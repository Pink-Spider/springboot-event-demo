# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A minimal demo of Spring application events: one service publishes `UserSignedUpEvent`, and three listeners consume it with different timing and threading. Spring Boot 4.1, Java 25 toolchain (auto-provisioned via foojay), Gradle 9 wrapper, H2 in-memory DB. README and comments are in Korean. The Boot 3.x version of the code lives on the `spring-boot-3x` branch.

## Commands

```bash
./gradlew build          # compile + test
./gradlew test           # only test is a @SpringBootTest contextLoads
./gradlew test --tests '*SpringEventDemoApplicationTests.contextLoads'
./gradlew bootRun        # port 8080, H2 console at /h2-console (jdbc:h2:mem:testdb)
curl -X POST "http://localhost:8080/api/users/signup?email=test@co.kr&name=YG"
```

No linter or formatter is configured. Behaviour is checked by reading the listener log lines (`AUDIT`, `SEND EMAIL`, `AFTER_COMMIT`), which include the thread name.

## Event flow

`SignUpController` → `SignUpService.signUp()` (`@Transactional`) saves a `User`, then publishes the event inside the transaction. The README documents this in detail. What matters when changing it:

- `AuditLogListener`: plain `@EventListener`, so it runs synchronously on the request thread inside the publisher's transaction. An exception here rolls back the sign-up.
- `WelcomeEmailListener`: `@Async` + `@EventListener(condition = "#event.email.endsWith('co.kr')")`. It runs on the default Boot task executor (`task-N` threads), enabled by `config/AsyncConfig` (`@EnableAsync`). No AOP/AspectJ starter is needed for this.
- `PostCommitListener`: `@TransactionalEventListener(AFTER_COMMIT)`. It runs only after the commit, and it is skipped entirely if the event is published outside a transaction (unless `fallbackExecution = true`).

## Notes

- The listener package is spelled `user/listner` (sic). Use the existing package name rather than creating a `listener` package next to it.
- Boot 4 module split: the H2 console needs the explicit `spring-boot-h2console` dependency, and the web starter is `spring-boot-starter-webmvc`.
- `logging.level.com.example` in `application.yml` doesn't match the actual base package `io.pinkspider.springeventdemo`.
