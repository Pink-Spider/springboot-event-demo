# Spring Event Demo

[![Java](https://img.shields.io/badge/Java-25-ED8B00?logo=openjdk&logoColor=white)](https://openjdk.org/projects/jdk/25/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-4.1.1-6DB33F?logo=springboot&logoColor=white)](https://spring.io/projects/spring-boot)
[![Gradle](https://img.shields.io/badge/Gradle-9.8.2-02303A?logo=gradle&logoColor=white)](https://gradle.org)
[![H2](https://img.shields.io/badge/H2-in--memory-1F6FEB)](https://www.h2database.com)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow)](LICENSE)

Spring Boot에서 `ApplicationEventPublisher`를 활용한 이벤트 기반 아키텍처 예제입니다.

## 기술 스택

| 항목 | 버전 |
|------|------|
| Java | 25 (Gradle toolchain) |
| Spring Boot | 4.1.1 |
| Gradle | 9.8.2 (wrapper) |
| DB | H2 (in-memory) |

> Spring Boot 3.x 기반 코드는 [`spring-boot-3x`](https://github.com/Pink-Spider/springboot-event-demo/tree/spring-boot-3x) 브랜치에 있습니다.

## 프로젝트 구조

```
src/main/java/io/pinkspider/springeventdemo/
├── config/
│   └── AsyncConfig.java           # 비동기 처리 설정
└── user/
    ├── api/
    │   └── SignUpController.java  # REST API 엔드포인트
    ├── event/
    │   └── UserSignedUpEvent.java # 이벤트 정의
    ├── listner/
    │   ├── AuditLogListener.java      # 동기 리스너
    │   ├── WelcomeEmailListener.java  # 비동기 리스너
    │   └── PostCommitListener.java    # 트랜잭션 커밋 후 리스너
    ├── model/
    │   └── User.java              # JPA 엔티티
    ├── repo/
    │   └── UserRepository.java    # JPA 레포지토리
    └── service/
        └── SignUpService.java     # 이벤트 발행 서비스
```

## 핵심 개념

### 1. 이벤트 정의 (Event)

```java
public record UserSignedUpEvent(Long userId, String email, Instant occurredAt) {
}
```

- Java 17의 `record`를 사용하여 불변(immutable) 이벤트 객체 정의
- Spring 4.2 이후부터는 `ApplicationEvent`를 상속할 필요 없이 POJO로 이벤트 정의 가능

### 2. 이벤트 발행 (Publisher)

```java
@Service
public class SignUpService {

    private final ApplicationEventPublisher publisher;

    @Transactional
    public Long signUp(String email, String name) {
        User user = userRepository.save(new User(email, name));
        publisher.publishEvent(new UserSignedUpEvent(user.getId(), user.getEmail(), Instant.now()));
        return user.getId();
    }
}
```

- `ApplicationEventPublisher`를 주입받아 `publishEvent()` 메서드로 이벤트 발행
- 트랜잭션 내에서 이벤트를 발행하면 리스너 타입에 따라 실행 시점이 달라짐

### 3. 이벤트 리스너 (Listener)

이 프로젝트에서는 3가지 타입의 리스너를 구현합니다.

#### 3-1. 동기 리스너 (`@EventListener`)

```java
@Component
public class AuditLogListener {

    @Order(0)
    @EventListener
    public void onUserSignedUp(UserSignedUpEvent event) {
        log.info("AUDIT: userId={}, email={}, at={}", event.userId(), event.email(), event.occurredAt());
    }
}
```

| 특징 | 설명 |
|------|------|
| 실행 시점 | 이벤트 발행 즉시 (동기) |
| 스레드 | Publisher와 동일한 스레드 |
| 트랜잭션 | Publisher의 트랜잭션에 참여 |
| 순서 제어 | `@Order` 어노테이션으로 실행 순서 지정 가능 |
| 사용 예 | 감사 로그, 유효성 검증 등 즉시 처리가 필요한 작업 |

#### 3-2. 비동기 리스너 (`@Async` + `@EventListener`)

```java
@Component
public class WelcomeEmailListener {

    @Async
    @EventListener(condition = "#event.email.endsWith('co.kr')")
    public void sendWelcomeEmail(UserSignedUpEvent event) {
        log.info("SEND EMAIL: welcome -> {}", event.email());
    }
}
```

| 특징 | 설명 |
|------|------|
| 실행 시점 | 이벤트 발행 즉시 (별도 스레드) |
| 스레드 | 별도의 스레드 풀에서 실행 |
| 트랜잭션 | 별도 트랜잭션 (Publisher 트랜잭션과 무관) |
| 조건부 실행 | `condition` 속성으로 SpEL 표현식 사용 가능 |
| 사용 예 | 이메일 발송, 알림 전송 등 시간이 오래 걸리는 작업 |

> `@Async`를 사용하려면 `@EnableAsync` 설정이 필요합니다.

```java
@Configuration
@EnableAsync
public class AsyncConfig {
}
```

#### 3-3. 트랜잭션 커밋 후 리스너 (`@TransactionalEventListener`)

```java
@Component
public class PostCommitListener {

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void afterCommit(UserSignedUpEvent event) {
        log.info("AFTER_COMMIT: create coupon for userId={}", event.userId());
    }
}
```

| 특징 | 설명 |
|------|------|
| 실행 시점 | 트랜잭션 커밋 이후 |
| 스레드 | Publisher와 동일한 스레드 (기본값) |
| 트랜잭션 | Publisher 트랜잭션 커밋 후 실행 (롤백 시 실행 안 됨) |
| Phase 옵션 | `BEFORE_COMMIT`, `AFTER_COMMIT`, `AFTER_ROLLBACK`, `AFTER_COMPLETION` |
| 사용 예 | 쿠폰 발급, 외부 API 호출 등 DB 커밋 보장이 필요한 작업 |

## 실행 흐름

```
[HTTP Request] POST /api/users/signup
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│  SignUpService.signUp() - @Transactional                    │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ 1. User 저장 (userRepository.save)                     │ │
│  │ 2. UserSignedUpEvent 발행 (publisher.publishEvent)     │ │
│  │    │                                                    │ │
│  │    ├──→ AuditLogListener (동기, 즉시 실행)              │ │
│  │    └──→ WelcomeEmailListener (비동기, 별도 스레드)      │ │
│  │ 3. return user.getId()                                 │ │
│  └────────────────────────────────────────────────────────┘ │
│                          │                                   │
│                    [트랜잭션 커밋]                            │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
            PostCommitListener (커밋 후 실행)
```

## 실행 방법

```bash
# 애플리케이션 실행
./gradlew bootRun

# 다른 터미널에서 API 호출
curl -X POST "http://localhost:8080/api/users/signup?email=test@co.kr&name=YG"
```

JDK 25가 설치되어 있지 않아도 Gradle toolchain(foojay resolver)이 자동으로 내려받습니다.
H2 콘솔은 http://localhost:8080/h2-console 에서 확인할 수 있습니다 (JDBC URL: `jdbc:h2:mem:testdb`).

## 예상 로그 출력

```
AUDIT: userId=1, email=test@co.kr, at=2024-01-01T00:00:00Z
SEND EMAIL: welcome -> test@co.kr
AFTER_COMMIT: create coupon for userId=1
```

> 비동기 리스너(`WelcomeEmailListener`)는 `condition = "#event.email.endsWith('co.kr')"`로 설정되어 있어,
> 이메일이 `co.kr`로 끝나는 경우에만 실행됩니다.

## 리스너 비교표

| 구분 | @EventListener | @Async + @EventListener | @TransactionalEventListener |
|------|----------------|-------------------------|----------------------------|
| 실행 시점 | 즉시 | 즉시 (비동기) | 트랜잭션 Phase에 따라 |
| 실행 스레드 | 동일 | 별도 스레드 풀 | 동일 (기본) |
| 트랜잭션 | 동일 트랜잭션 | 별도 트랜잭션 | 커밋/롤백 후 |
| 예외 영향 | 롤백 발생 | 영향 없음 | 영향 없음 |
| 순서 제어 | @Order | 불가 | @Order |

## 주의사항

1. **동기 리스너 예외**: 동기 리스너에서 예외가 발생하면 전체 트랜잭션이 롤백됩니다.
2. **비동기 리스너 트랜잭션**: `@Async` 리스너는 별도 트랜잭션이므로 Publisher의 롤백과 무관하게 실행됩니다.
3. **TransactionalEventListener 주의**: 트랜잭션이 없는 컨텍스트에서 발행된 이벤트는 기본적으로 무시됩니다. `fallbackExecution = true` 옵션으로 변경 가능합니다.

## Spring Boot 3 → 4 마이그레이션 요약

| 변경 | Boot 3.x | Boot 4.x |
|------|----------|----------|
| Web 스타터 | `spring-boot-starter-web` | `spring-boot-starter-webmvc` |
| H2 콘솔 | 자동 설정 내장 | `spring-boot-h2console` 모듈 추가 필요 |
| AOP 스타터 | `spring-boot-starter-aop` | 제거 (`@Async`는 spring-context의 프록시만으로 동작, AspectJ 불필요) |
| 테스트 | `spring-boot-starter-test` | 동일 + Gradle 9 대응 `junit-platform-launcher` 추가 |
| Java / Gradle | 23 / 8.14.3 | 25 / 9.8.2 |

## 참고 자료

- [Spring Framework - Application Events](https://docs.spring.io/spring-framework/reference/core/beans/context-introduction.html#context-functionality-events)
- [Spring Boot - @Async](https://docs.spring.io/spring-framework/reference/integration/scheduling.html#scheduling-annotation-support-async)

## License

[MIT](LICENSE)
