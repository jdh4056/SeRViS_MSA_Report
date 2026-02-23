# Spring Boot 모놀리식을 Gradle 멀티모듈 MSA로 전환한 과정

> 실제 프로젝트에서 모놀리식 Spring Boot 애플리케이션을 3개의 마이크로서비스로 분리하며 겪은 기술적 도전과 해결 과정을 공유합니다.

---

## 1. 프로젝트 소개

**SeRVe(Secure Repository for Vector Embeddings)**는 로봇(Physical AI)이 수집한 데이터를 End-to-End 암호화하여 저장하고, 팀 단위로 안전하게 공유하는 플랫폼입니다. Zero-Trust 보안 모델을 기반으로, 서버가 평문 데이터나 사용자 개인키를 절대 볼 수 없도록 설계되어 있습니다.

| 항목 | 스택 |
|------|------|
| 언어/프레임워크 | Java 17, Spring Boot 3.4.0, Spring Cloud 2024.0.0 |
| 데이터베이스 | MariaDB 11.4 |
| 암호화 | Google Tink (AES-256-GCM / ECIES) |
| 인증 | JWT HS256 |
| 서비스 간 통신 | Spring Cloud OpenFeign |

### 왜 MSA로 전환했는가

초기에는 빠른 개발을 위해 단일 Spring Boot 애플리케이션으로 구축했지만, 두 가지 문제가 명확해졌습니다.

1. **독립적 스케일링 필요**: 인증 서비스(Auth)는 로그인 폭주 시 개별 확장이 필요한데, 모놀리식에서는 전체를 통째로 스케일해야 했습니다.
2. **배포 독립성**: 팀 관리 기능을 수정했는데 인증 서비스까지 재배포해야 하는 상황이 반복되었습니다.

### 전환 전후 아키텍처

**Before — 모놀리식:**
```
클라이언트 → Spring Boot (:8080, 단일 JAR)
                 │
                 ▼
            MariaDB (단일 스키마)
```

**After — MSA (3개 서비스):**
```
클라이언트 → Nginx Gateway (:8080)
                 │ URL 기반 라우팅
          ┌──────┼──────────────┐
          ▼      ▼              ▼
        Auth   Team           Core
        :8081  :8082          :8083
          │      │              │
          ▼      ▼              ▼
        ┌──────────────────────────┐
        │  MariaDB (물리 1개)       │
        │  ┌──────┐┌──────┐┌────┐ │
        │  │auth  ││team  ││core│ │
        │  │_db   ││_db   ││_db │ │
        │  └──────┘└──────┘└────┘ │
        └──────────────────────────┘
```

---

## 2. 전환 전략: Gradle 멀티모듈 모노레포

서비스를 완전히 별도 레포지토리로 분리할 수도 있었지만, **Gradle 멀티모듈 모노레포** 방식을 선택했습니다.

```
SeRViS_server/
├── SeRVe-Common/    # 공통 모듈 (JWT, 암호화, 예외처리)
├── SeRVe-Auth/      # 인증 서비스 (:8081)
├── SeRVe-Team/      # 팀/멤버 관리 (:8082)
├── SeRVe-Core/      # 태스크/데모 데이터 (:8083)
├── build.gradle     # 루트 빌드 설정
└── settings.gradle
```

### 왜 모노레포인가

- **Common 모듈 공유**: JWT 검증, 암호화 유틸(Google Tink), 예외 처리 같은 공통 코드를 별도 모듈로 분리하여 세 서비스가 동일한 코드를 의존합니다. 별도 레포였다면 Maven/Gradle 패키지를 발행하고 버전 관리하는 오버헤드가 생깁니다.
- **Feign DTO 공유**: 서비스 간 통신에 사용하는 DTO(`UserInfoResponse`, `MemberRoleResponse` 등)를 Common 모듈에 두고 양쪽 서비스가 참조합니다. 타입 불일치 위험이 사라집니다.
- **단일 빌드**: `./gradlew build` 한 번으로 전체 서비스를 빌드하고 테스트할 수 있습니다.

### Common 모듈의 핵심: `java-library` 플러그인

Common 모듈은 실행 가능한 애플리케이션이 아니라 라이브러리이므로, `bootJar`를 생성하지 않아야 합니다.

```gradle
// SeRVe-Common/build.gradle
plugins {
    id 'java-library'  // org.springframework.boot가 아님!
}

jar {
    enabled = true
}

dependencies {
    api 'org.springframework.boot:spring-boot-starter-web'
    api 'org.springframework.boot:spring-boot-starter-security'
    api 'io.jsonwebtoken:jjwt-api:0.11.5'
    api 'com.google.crypto.tink:tink:1.15.0'
}
```

각 서비스 모듈에서는 이 Common을 의존합니다:

```gradle
// SeRVe-Auth/build.gradle (Team, Core도 동일)
plugins {
    id 'org.springframework.boot' version '3.4.0'
}

dependencies {
    implementation project(':SeRVe-Common')
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.cloud:spring-cloud-starter-openfeign'
    runtimeOnly 'org.mariadb.jdbc:mariadb-java-client'
}
```

`api`로 선언된 Common의 의존성(Spring Web, Security, JWT, Tink)은 하위 서비스 모듈에 전이되므로, 각 서비스에서 중복 선언할 필요가 없습니다.

---

## 3. 가장 어려웠던 부분: JPA 엔티티 참조 제거

MSA 전환에서 가장 많은 코드 변경이 발생한 부분입니다. 모놀리식에서는 모든 엔티티가 하나의 DB에 있으므로 JPA `@ManyToOne`으로 자유롭게 연관관계를 맺을 수 있었습니다. MSA에서는 각 서비스가 **독립된 DB 스키마**를 사용하므로, 다른 서비스의 테이블을 JPA로 참조할 수 없습니다.

### 3.1 RepositoryMember: User 참조 제거

**Before (모놀리식) — User 엔티티를 직접 JPA 참조:**
```java
@Entity
@Table(name = "repository_members")
public class RepositoryMember {

    @EmbeddedId
    private RepositoryMemberId id;

    @MapsId("teamId")
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "team_id")
    private Team team;

    @MapsId("userId")
    @ManyToOne(fetch = FetchType.LAZY)  // ← Auth DB의 users 테이블과 JOIN
    @JoinColumn(name = "user_id")
    private User user;                   // ← User 엔티티 직접 참조

    private Role role;
    private String encryptedTeamKey;
}
```

이 구조에서는 `member.getUser().getEmail()` 같은 호출이 자유롭게 가능했습니다. JPA가 알아서 `users` 테이블을 JOIN 해주니까요.

**After (MSA) — String userId로 대체:**
```java
@Entity
@Table(name = "repository_members")
public class RepositoryMember {

    @EmbeddedId
    private RepositoryMemberId id;

    @MapsId("teamId")
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "team_id")
    private Team team;                   // ← 같은 DB(serve_team_db)이므로 유지

    @Column(insertable = false, updatable = false)
    private String userId;               // ← 단순 문자열. Feign으로 정보 조회

    private Role role;
    private String encryptedTeamKey;
}
```

`Team`은 같은 `serve_team_db`에 있으므로 JPA 참조를 유지하지만, `User`는 `serve_auth_db`에 있으므로 문자열 ID로 대체했습니다. 사용자 정보가 필요할 때는 Auth 서비스에 Feign 호출을 합니다.

### 3.2 Document(Task): Team과 User 참조 모두 제거

Core 서비스의 Document(현재 Task) 엔티티는 더 극적인 변화를 겪었습니다.

**Before (모놀리식):**
```java
@Entity
@Table(name = "documents")
public class Document {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "team_id")
    private Team team;           // ← team_db의 teams 테이블 참조

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "uploader_id")
    private User uploader;       // ← auth_db의 users 테이블 참조

    private String originalFileName;
    private String fileType;
}
```

**After (MSA):**
```java
@Entity
@Table(name = "tasks")
public class Task {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "team_id")
    private String teamId;       // ← 단순 문자열

    @Column(name = "uploader_id")
    private String uploaderId;   // ← 단순 문자열

    private String originalFileName;
    private String fileType;
}
```

`@ManyToOne` 2개가 전부 `String`으로 바뀌었습니다. JPA의 편리한 연관관계 탐색(`document.getTeam().getName()`, `document.getUploader().getEmail()`)이 모두 사라지는 대가를 치렀습니다.

### 3.3 Repository 쿼리 메서드의 연쇄 변경

엔티티가 바뀌면 Repository의 Spring Data JPA 쿼리 메서드도 변경되어야 합니다.

```java
// Before: User 엔티티를 파라미터로 받음
public interface MemberRepository extends JpaRepository<RepositoryMember, ...> {
    List<RepositoryMember> findAllByUser(User user);
    Optional<RepositoryMember> findByTeamAndUser(Team team, User user);
    boolean existsByTeamAndUser(Team team, User user);
}

// After: String userId로 변경
public interface MemberRepository extends JpaRepository<RepositoryMember, ...> {
    List<RepositoryMember> findAllByUserId(String userId);
    Optional<RepositoryMember> findByTeamAndUserId(Team team, String userId);
    boolean existsByTeamAndUserId(Team team, String userId);
}
```

메서드 시그니처만 바뀌는 것 같지만, 이 Repository를 사용하는 모든 Service 코드도 함께 수정해야 하므로 변경 범위가 넓습니다.

---

## 4. JWT 인증 체계 분리

모놀리식에서 MSA로 전환할 때 인증 처리 방식이 근본적으로 달라져야 했습니다. 이 부분이 단순히 코드를 나누는 것 이상의 **설계 결정**이 필요한 지점이었습니다.

### 4.1 모놀리식의 인증: 모든 곳에서 User 객체 접근

모놀리식에서는 JWT 필터가 `UserDetailsService`를 통해 DB에서 User를 로드하고, 모든 컨트롤러에서 `@AuthenticationPrincipal User user`로 사용자 엔티티에 직접 접근할 수 있었습니다.

```java
// 모놀리식 JwtTokenProvider.getAuthentication()
public Authentication getAuthentication(String token) {
    String email = getUserEmail(token);
    UserDetails userDetails = userDetailsService.loadUserByUsername(email);
    return new UsernamePasswordAuthenticationToken(
        userDetails, "", userDetails.getAuthorities());
    // Principal = User 객체 (DB에서 로드)
}
```

```java
// 모놀리식 컨트롤러 — 어디서든 User 엔티티 접근 가능
@GetMapping("/api/teams/{teamId}/members")
public ResponseEntity<?> getMembers(
        @PathVariable Long teamId,
        @AuthenticationPrincipal User user) {    // User 객체 바로 사용
    memberService.getMembers(teamId, user);
}
```

### 4.2 MSA에서의 문제

Team 서비스와 Core 서비스에는 `users` 테이블이 없습니다. `UserDetailsService.loadUserByUsername()`을 호출하면 테이블이 없으니 오류가 납니다. 그렇다고 JWT 검증을 위해 매번 Auth 서비스에 Feign 호출을 하면 모든 API 요청마다 네트워크 호출이 추가됩니다.

### 4.3 해결: 서비스별 JWT 필터 분리

**Common 모듈의 JwtTokenProvider (Team/Core용) — DB 조회 없이 JWT claims만 사용:**
```java
public Authentication getAuthentication(String token) {
    String email = getUserEmail(token);
    String userId = getUserId(token);
    List<GrantedAuthority> authorities =
        List.of(new SimpleGrantedAuthority("ROLE_USER"));

    UsernamePasswordAuthenticationToken auth =
        new UsernamePasswordAuthenticationToken(userId, null, authorities);
    auth.setDetails(Map.of("email", email, "userId", userId));
    return auth;
    // Principal = String userId (DB 조회 없음!)
}
```

JWT 토큰 자체에 `userId`와 `email`이 들어있으므로, DB를 조회하지 않아도 인증 정보를 구성할 수 있습니다. 대신 Principal이 `User` 객체가 아닌 `String userId`가 됩니다.

**Auth 전용 AuthJwtAuthenticationFilter (신규 클래스) — full User 로드 유지:**
```java
// Auth 서비스에서만 사용하는 JWT 필터
String email = jwtTokenProvider.getUserEmail(token);
UserDetails userDetails = userDetailsService.loadUserByUsername(email);
UsernamePasswordAuthenticationToken auth =
    new UsernamePasswordAuthenticationToken(
        userDetails, "", userDetails.getAuthorities());
// Principal = User 객체 (Auth DB에서 로드)
```

Auth 서비스는 `users` 테이블을 직접 갖고 있으므로, 기존 방식대로 full User를 로드합니다. 이를 통해 Auth 컨트롤러에서만 `@AuthenticationPrincipal User user`를 유지합니다.

### 4.4 컨트롤러 인증 패턴 변경

이 JWT 분리의 결과로, Team/Core 서비스의 모든 컨트롤러가 변경되었습니다.

```java
// Before (모놀리식) — User 객체 직접 사용
@PostMapping("/api/teams/{teamId}/members")
public ResponseEntity<Void> inviteMember(
        @PathVariable Long teamId,
        @AuthenticationPrincipal User user,     // User 객체
        @RequestBody InviteMemberRequest req) {
    memberService.inviteMember(teamId, req);
}

// After (MSA, Team/Core) — Authentication에서 userId String 추출
@PostMapping("/api/teams/{teamId}/members")
public ResponseEntity<Void> inviteMember(
        @PathVariable String teamId,
        Authentication authentication,          // Authentication 객체
        @RequestBody InviteMemberRequest req) {
    String userId = (String) authentication.getPrincipal();  // String 추출
    memberService.inviteMember(teamId, userId, req);
}
```

미묘하지만 중요한 차이입니다:
- `@AuthenticationPrincipal User user` → `Authentication authentication` + `(String) auth.getPrincipal()`
- 서비스 메서드에 `User` 객체 대신 `String userId`를 전달
- Auth 서비스만 기존 패턴(`@AuthenticationPrincipal User`) 유지

---

## 5. 서비스 간 통신: Feign Client 도입

JPA 참조를 제거하면서 잃어버린 데이터 접근 능력을 **Spring Cloud OpenFeign**으로 대체했습니다.

### 5.1 내부 API 패턴: `/internal/**`

각 서비스는 다른 서비스가 호출할 수 있는 내부 전용 API를 `/internal/**` 경로로 노출합니다. 이 경로는 SecurityConfig에서 인증 없이 허용(`permitAll`)됩니다.

```java
// Auth 서비스 — InternalUserController (신규)
@RestController
@RequestMapping("/internal/users")
public class InternalUserController {

    private final UserRepository userRepository;

    @GetMapping("/{userId}")
    public ResponseEntity<UserInfoResponse> getUserInfo(@PathVariable String userId) {
        User user = userRepository.findById(userId)
                .orElseThrow(() -> new IllegalArgumentException("사용자를 찾을 수 없습니다."));
        return ResponseEntity.ok(UserInfoResponse.builder()
                .userId(user.getUserId())
                .email(user.getEmail())
                .publicKey(user.getPublicKey())
                .build());
    }

    @GetMapping("/{userId}/exists")
    public ResponseEntity<Boolean> userExists(@PathVariable String userId) {
        return ResponseEntity.ok(userRepository.existsById(userId));
    }
}
```

### 5.2 Feign Client 인터페이스

호출하는 쪽에서는 Feign Client 인터페이스를 정의합니다.

```java
// Team 서비스에서 Auth 서비스를 호출하는 Feign Client
@FeignClient(name = "auth-service", url = "${service.auth.url}")
public interface AuthServiceClient {

    @GetMapping("/internal/users/{userId}")
    UserInfoResponse getUserInfo(@PathVariable String userId);

    @GetMapping("/internal/users/by-email/{email}")
    UserInfoResponse getUserByEmail(@PathVariable String email);

    @GetMapping("/internal/users/{userId}/exists")
    boolean userExists(@PathVariable String userId);
}
```

`application.yml`에서 URL을 환경변수로 관리합니다:

```yaml
service:
  auth:
    url: ${AUTH_SERVICE_URL:http://localhost:8081}
```

### 5.3 실제 전환 예시: MemberService.inviteMember()

멤버 초대 로직의 Before/After를 통해 Feign 전환의 실체를 보여드리겠습니다.

**Before (모놀리식) — UserRepository 직접 호출:**
```java
public void inviteMember(Long teamId, InviteMemberRequest req) {
    Team team = teamRepository.findById(teamId)
            .orElseThrow(() -> ...);

    // 같은 DB의 UserRepository에서 직접 조회
    User invitee = userRepository.findByEmail(req.getEmail())
            .orElseThrow(() -> ...);

    if (memberRepository.existsByTeamAndUser(team, invitee)) {
        throw new IllegalArgumentException("이미 멤버입니다.");
    }

    RepositoryMember newMember = RepositoryMember.builder()
            .id(new RepositoryMemberId(team.getId(), invitee.getUserId()))
            .team(team)
            .user(invitee)              // User 엔티티 직접 설정
            .role(Role.MEMBER)
            .encryptedTeamKey(req.getEncryptedTeamKey())
            .build();

    memberRepository.save(newMember);
}
```

**After (MSA) — AuthServiceClient Feign 호출:**
```java
public void inviteMember(String teamId, String inviterUserId, InviteMemberRequest req) {
    Team team = teamRepository.findByTeamId(teamId)
            .orElseThrow(() -> ...);

    // 초대자 권한 검증 (모놀리식에서는 없던 로직)
    RepositoryMember inviter = memberRepository.findByTeamAndUserId(team, inviterUserId)
            .orElseThrow(() -> new SecurityException("초대자가 멤버가 아닙니다."));
    if (inviter.getRole() != Role.ADMIN) {
        throw new SecurityException("ADMIN 권한이 필요합니다.");
    }

    // Auth 서비스에 Feign 호출로 사용자 조회
    UserInfoResponse inviteeInfo = authServiceClient.getUserByEmail(req.getEmail());

    if (memberRepository.existsByTeamAndUserId(team, inviteeInfo.getUserId())) {
        throw new IllegalArgumentException("이미 멤버입니다.");
    }

    RepositoryMember newMember = RepositoryMember.builder()
            .id(new RepositoryMemberId(team.getTeamId(), inviteeInfo.getUserId()))
            .team(team)
            // .user(invitee) ← 삭제됨. User 엔티티 참조 없음
            .role(Role.MEMBER)
            .encryptedTeamKey(req.getEncryptedTeamKey())
            .build();

    memberRepository.save(newMember);
}
```

핵심 변경점:
1. `userRepository.findByEmail()` → `authServiceClient.getUserByEmail()` (로컬 DB → 네트워크 호출)
2. `User invitee` → `UserInfoResponse inviteeInfo` (JPA 엔티티 → Feign DTO)
3. `.user(invitee)` 삭제 — JPA 참조 대신 `RepositoryMemberId`에 userId String을 직접 설정

### 5.4 DTO 팩토리 메서드 변경

JPA 참조 제거의 파급 효과가 DTO까지 미칩니다. 연관 엔티티에서 가져오던 정보를 외부 파라미터로 받아야 합니다.

```java
// Before — JPA로 로드된 User에서 이메일 획득
public static MemberResponse from(RepositoryMember member) {
    return MemberResponse.builder()
        .userId(member.getUser().getUserId())
        .email(member.getUser().getEmail())    // member → user → email
        .role(member.getRole().name())
        .build();
}

// After — Feign으로 조회한 이메일을 파라미터로 주입
public static MemberResponse from(RepositoryMember member, String email) {
    return MemberResponse.builder()
        .userId(member.getUserId())
        .email(email)                          // 외부에서 주입
        .role(member.getRole().name())
        .build();
}
```

### 5.5 N+1 Feign 호출 문제

멤버 목록을 조회할 때, 각 멤버의 이메일을 알기 위해 멤버 수만큼 Feign 호출이 발생합니다.

```java
// 멤버 목록 조회 — 멤버 수(N)만큼 Feign 호출
return memberRepository.findAllByTeam(team).stream()
    .map(member -> {
        String email;
        try {
            email = authServiceClient.getUserInfo(member.getUserId()).getEmail();
        } catch (Exception e) {
            email = "Unknown";  // Feign 실패 시 폴백
        }
        return MemberResponse.from(member, email);
    })
    .collect(Collectors.toList());
```

모놀리식에서는 JPA가 한 번의 JOIN으로 해결하던 것이, MSA에서는 N번의 HTTP 호출로 바뀌었습니다. 현재는 `try-catch`로 개별 실패를 처리하고 있으며, 향후 배치 API(`/internal/users/batch?ids=...`)나 Resilience4j 서킷브레이커로 개선할 수 있습니다.

---

## 6. DB 스키마 분리

### 물리 1개, 논리 3개 전략

마이크로서비스별로 별도의 DB 인스턴스를 사용하는 것이 이상적이지만, 비용과 관리 부담을 고려하여 **하나의 MariaDB에 3개 논리 스키마**를 생성했습니다.

```sql
-- docker/init-db.sql
CREATE DATABASE IF NOT EXISTS serve_auth_db;
CREATE DATABASE IF NOT EXISTS serve_team_db;
CREATE DATABASE IF NOT EXISTS serve_core_db;

GRANT ALL PRIVILEGES ON serve_auth_db.* TO 'serve_user'@'%';
GRANT ALL PRIVILEGES ON serve_team_db.* TO 'serve_user'@'%';
GRANT ALL PRIVILEGES ON serve_core_db.* TO 'serve_user'@'%';
```

각 서비스는 자신의 스키마에만 연결합니다:

```yaml
# SeRVe-Auth: serve_auth_db만 사용
spring:
  datasource:
    url: jdbc:mariadb://localhost:3306/serve_auth_db

# SeRVe-Team: serve_team_db만 사용
spring:
  datasource:
    url: jdbc:mariadb://localhost:3306/serve_team_db

# SeRVe-Core: serve_core_db만 사용
spring:
  datasource:
    url: jdbc:mariadb://localhost:3306/serve_core_db
```

물리적으로는 같은 인스턴스지만, 각 서비스는 다른 서비스의 테이블에 접근할 수 없습니다. 이것이 JPA 참조를 제거해야 하는 근본적인 이유입니다. 나중에 트래픽이 증가하면 스키마를 별도 인스턴스로 분리하기만 하면 됩니다. 애플리케이션 코드 변경 없이 `application.yml`의 URL만 바꾸면 됩니다.

---

## 7. API Gateway와 클라이언트 호환성

### Nginx URL 기반 라우팅

모놀리식에서는 모든 API가 단일 포트(:8080)로 서비스되었습니다. MSA 전환 후에는 클라이언트가 여전히 하나의 엔드포인트로 접근할 수 있도록 Nginx API Gateway를 두었습니다.

```nginx
# gateway/nginx.conf — 핵심 라우팅 규칙
upstream auth-service { server host.docker.internal:8081; }
upstream team-service { server host.docker.internal:8082; }
upstream core-service { server host.docker.internal:8083; }

server {
    listen 8080;

    # Auth 서비스
    location /auth/ { proxy_pass http://auth-service; }

    # Team 서비스
    location /api/repositories { proxy_pass http://team-service; }
    location ~ ^/api/teams/([^/]+)/members { proxy_pass http://team-service; }

    # Core 서비스
    location ~ ^/api/teams/([^/]+)/tasks { proxy_pass http://core-service; }
    location = /api/tasks { proxy_pass http://core-service; }
    location ~ ^/api/teams/([^/]+)/demos { proxy_pass http://core-service; }
    location /api/security/ { proxy_pass http://core-service; }
    location /api/sync/ { proxy_pass http://core-service; }
}
```

### 클라이언트 하위 호환: Rewrite 규칙

MSA 전환과 동시에 도메인 리네이밍(Document→Task)을 진행했는데, 기존 Python 클라이언트가 `/api/documents` 경로를 사용하고 있었습니다. 클라이언트를 즉시 업데이트할 수 없는 상황에서, Nginx의 rewrite 규칙으로 호환성을 유지했습니다.

```nginx
# 기존 클라이언트: /api/documents → 새 경로: /api/tasks
location = /api/documents {
    rewrite ^/api/documents$ /api/tasks break;
    proxy_pass http://core-service;
}

location ~ ^/api/documents/([^/]+)$ {
    rewrite ^/api/documents/([^/]+)$ /api/tasks/$1/data break;
    proxy_pass http://core-service;
}
```

이 방식으로 서버 코드는 새로운 URL(`/api/tasks`)만 처리하면 되고, 기존 클라이언트는 아무 수정 없이 계속 동작합니다.

---

## 8. 전환 후 달라진 점 — 한눈에 보기

| 항목 | Before (모놀리식) | After (MSA) |
|------|-------------------|-------------|
| 엔티티 참조 | `@ManyToOne User user` | `String userId` |
| JWT Principal | `User` 객체 (DB 조회) | `String userId` (JWT claims만) |
| JWT 필터 | 전역 단일 필터 | Auth 전용 + Common 공통 필터 분리 |
| 컨트롤러 인증 | `@AuthenticationPrincipal User` | `Authentication` + `getPrincipal()` |
| 데이터 조회 | `userRepository.findByEmail()` | `authServiceClient.getUserByEmail()` |
| DTO 생성 | `member.getUser().getEmail()` | Feign 조회 후 파라미터 주입 |
| 내부 API | 불필요 | `/internal/**` 엔드포인트 신규 |
| 권한 검증 | 직접 DB 쿼리 | Feign으로 Team 서비스 호출 |
| DB | 단일 스키마 | 3개 논리 스키마 |
| 배포 | 단일 JAR | 서비스별 독립 배포 가능 |

### 신규 생성된 클래스

MSA 전환을 위해 완전히 새로 작성된 클래스들입니다:

| 분류 | 클래스 | 위치 |
|------|--------|------|
| Internal API | `InternalUserController` | Auth |
| Internal API | `InternalTeamController` | Team |
| Feign Client | `TeamServiceClient` | Auth |
| Feign Client | `AuthServiceClient` | Team |
| Feign Client | `TeamServiceClient` | Core |
| Feign Client | `AuthServiceClient` | Core |
| JWT 필터 | `AuthJwtAuthenticationFilter` | Auth |
| 공유 DTO | `UserInfoResponse` | Common |
| 공유 DTO | `MemberRoleResponse` | Common |
| 공유 DTO | `EdgeNodeAuthResponse` | Common |

기존 코드의 "수정"보다 이런 "신규 클래스 생성"이 전환 작업의 상당 부분을 차지했습니다.

---

## 9. 회고

### 잘한 점

- **Gradle 멀티모듈 모노레포**: Common 모듈로 JWT, 암호화, Feign DTO를 공유하면서도 서비스별 독립 빌드/배포가 가능합니다. 별도 레포였다면 패키지 발행/버전 관리 오버헤드가 컸을 것입니다.
- **단계적 전환**: Phase 0(모듈 구조)부터 Phase 5(인프라 정리)까지 나누어 진행하면서 각 단계마다 빌드가 성공하는 상태를 유지했습니다. 한 번에 전체를 바꾸려 했다면 디버깅이 훨씬 어려웠을 것입니다.
- **클라이언트 호환**: Nginx rewrite로 기존 클라이언트를 깨뜨리지 않으면서 도메인 리네이밍까지 동시에 진행한 것이 효과적이었습니다.

### 아쉬운 점

- **N+1 Feign 호출**: 멤버 목록 조회 등에서 멤버 수만큼 Auth 서비스에 HTTP 호출을 하고 있습니다. 배치 API나 캐싱을 도입하지 못했습니다.
- **모니터링 부재**: 서비스가 3개로 분리되면서 어디서 문제가 발생했는지 추적하기 어려워졌습니다. 분산 트레이싱(Zipkin/Jaeger)이나 중앙 로깅이 필요합니다.
- **Internal API 보안**: `/internal/**`이 `permitAll`이므로, 누군가 직접 내부 API에 접근하면 인증 없이 데이터를 조회할 수 있습니다. 네트워크 레벨 격리(Kubernetes NetworkPolicy 등)나 내부 토큰 검증이 필요합니다.

### 배운 점

모놀리식에서 MSA로의 전환은 "코드를 나누는 것"이 아니라 **"연관관계를 끊는 것"**이었습니다. JPA가 제공하던 편리한 엔티티 탐색(`document.getUploader().getEmail()`)을 포기하고, 네트워크 호출로 대체하는 과정에서 모든 서비스 레이어의 코드가 영향을 받았습니다.

가장 중요한 교훈은, MSA에서의 데이터 접근은 **"쿼리 한 번"이 아니라 "API 호출 한 번"**이라는 점입니다. 이 비용 차이를 인식하고 설계해야, 단순히 코드를 나누는 것을 넘어 실제로 동작하는 마이크로서비스를 만들 수 있습니다.

---

*이 글에서 다룬 프로젝트의 전체 코드는 Gradle 멀티모듈 구조로 구성되어 있으며, Spring Boot 3.4.0 + Spring Cloud 2024.0.0 기반입니다.*
