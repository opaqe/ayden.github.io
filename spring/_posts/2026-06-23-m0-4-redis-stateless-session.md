---
layout: post
title: "M0-4: Redis + Stateless 세션 (Spring Session / Testcontainers)"
date: 2026-06-23 10:00:00 +0900
tags: [spring-boot, redis, spring-session, testcontainers, stateless]
description: "톰캣 인메모리 세션을 Redis로 외부화해 앱을 무상태로 만들고, Testcontainers의 실제 Redis로 세션이 재시작·증설에도 유지됨을 통합 테스트로 증명한 과정."
---

> **목표**: 세션을 톰캣 인메모리가 아니라 Redis에 저장해 애플리케이션을 **무상태(stateless)** 로 만든다.
> **완료 기준(DoD)**: 인스턴스가 재시작되거나 여러 대로 늘어나도 세션이 유지됨을 **테스트로 증명**한다.

## 1. 왜 무상태(stateless)인가

기본 Spring Boot(내장 톰캣)는 세션을 **JVM 힙 메모리**에 저장한다. 이 구조의 한계는 다음과 같다.

| 상황 | 인메모리 세션의 문제 |
|---|---|
| 인스턴스 재시작 | 힙이 비워져 **모든 세션 소실** → 사용자 재로그인 |
| 인스턴스 2대 이상(로드밸런싱) | 1번에서 로그인한 세션을 2번이 모름 → 요청이 다른 인스턴스로 가면 인증 끊김 |
| Blue-Green 무중단 배포 | Green으로 트래픽 전환 시 Blue의 세션이 사라짐 |

세션을 **외부 공유 저장소(Redis)** 로 빼면 위 문제가 모두 사라진다. 어떤 인스턴스든 같은 Redis를 보므로, 인스턴스는 세션 상태를 들고 있지 않아도 된다 = **무상태**. 이것이 로드밸런싱·무중단 배포의 전제 조건이다.

## 2. 학습 개념

| 개념 | 한 줄 정의 | 본 단계에서의 역할 |
|---|---|---|
| **Spring Session** | `HttpSession`을 외부 저장소로 추상화하는 모듈 | 코드 변경 없이 세션 저장 위치를 Redis로 전환 |
| **무상태(Stateless)** | 인스턴스가 세션 상태를 보관하지 않는 설계 | 인스턴스 교체·증설에도 세션 유지 |
| **Redis** | 인메모리 Key-Value 저장소 | 세션 공유 저장소(이후 캐시·Pub/Sub도 담당) |
| **Testcontainers** | 테스트 시 실제 도커 컨테이너를 띄워 검증하는 라이브러리 | Mock/H2 대신 **실제 Redis**로 통합 테스트 |
| **`@ServiceConnection`** | 테스트 컨테이너 접속정보를 스프링에 자동 주입하는 어노테이션 | host/port를 `spring.data.redis.*`에 자동 연결 |

## 3. 구현 단계 요약

| 순서 | 작업 | 파일 |
|---|---|---|
| 1 | Redis + Spring Session 의존성 추가 | `build.gradle` |
| 2 | Redis 접속정보·세션 설정 외부화 | `application.yaml`, `.env.example` |
| 3 | 테스트용 Redis 컨테이너 추가 | `TestcontainersConfiguration.java` |
| 4 | 세션 저장/조회 통합 테스트 | `SessionPersistenceIT.java` |

## 4. 소스

### 4-1. 의존성 — `backend/build.gradle`

```groovy
dependencies {
    // --- 기존 ---
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'dev.langchain4j:langchain4j-spring-boot-starter:1.16.3-beta26'

    // --- M0-3: 영속성 ---
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.flywaydb:flyway-core'
    implementation 'org.flywaydb:flyway-database-postgresql'
    runtimeOnly   'org.postgresql:postgresql'

    // --- M0-4: 세션/캐시(Redis) ---
    implementation 'org.springframework.boot:spring-boot-starter-data-redis'
    implementation 'org.springframework.session:spring-session-data-redis'

    // --- 테스트 ---
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.boot:spring-boot-testcontainers'
    testImplementation 'org.testcontainers:postgresql'
    testImplementation 'org.testcontainers:junit-jupiter'
    testRuntimeOnly    'org.junit.platform:junit-platform-launcher'
}
```

| 의존성 | 역할 |
|---|---|
| `spring-boot-starter-data-redis` | Redis 클라이언트(기본 Lettuce) + `RedisConnectionFactory` 자동설정 |
| `spring-session-data-redis` | `HttpSession`을 Redis에 저장하는 Spring Session 구현체 |

> 버전 핀 없음 — Spring Boot 3.5.x BOM이 두 의존성의 버전을 관리한다.
> Redis 전용 Testcontainers 모듈은 추가하지 않았다(이유는 6장 참고).

### 4-2. 설정 외부화 — `backend/src/main/resources/application.yaml`

```yaml
spring:
  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: ${REDIS_PORT:26379}
      password: ${REDIS_PASSWORD:}
      timeout: 2s                 # Redis 명령 타임아웃 (연결 응답 한계)

  session:
    timeout: 30m                  # 세션 만료 시간
    redis:
      namespace: ixikey:session   # Redis 키 접두어 (충돌 방지)
      flush-mode: on_save
```

| 속성 | 의미 |
|---|---|
| `spring.data.redis.*` | Redis **접속정보**. `${ENV:기본값}` 으로 외부화(12-factor) |
| `spring.data.redis.timeout` | Redis **명령** 타임아웃(2초 내 미응답 시 실패) |
| `spring.session.timeout` | **세션 만료** 시간(30분 무활동 시 폐기) — 위 명령 타임아웃과 별개 개념 |
| `spring.session.redis.namespace` | Redis 키 접두어 → 키는 `ixikey:session:{sessionId}` 형태 |
| `spring.session.redis.flush-mode: on_save` | `save()` 호출 시점에 Redis로 기록 |

> 주의: 접속(`spring.data.redis`) 경로는 Spring Boot 3.x 표준이며, 구버전 `spring.redis.*` 가 아니다.

### 4-3. 환경 변수 템플릿 — `backend/.env.example`

```properties
SPRING_PROFILES_ACTIVE=local
SERVER_PORT=8090

# --- M0-3 PostgreSQL ---
DB_URL=jdbc:postgresql://localhost:25432/ixikey
DB_USERNAME=ixikey
DB_PASSWORD=changeme

# --- M0-4 Redis ---
REDIS_HOST=localhost
REDIS_PORT=26379
REDIS_PASSWORD=
```

> 로컬 포트 `26379` 는 M0-3의 Postgres(`25432`)와 맞춘 "2xxxx 비표준 포트" 관례. 로컬 Redis는 인증이 없어 password를 비운다.

### 4-4. 테스트용 Redis 컨테이너 — `backend/src/test/java/com/ixikey/support/TestcontainersConfiguration.java`

```java
package com.ixikey.support;

import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.springframework.context.annotation.Bean;
import org.testcontainers.containers.GenericContainer;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.utility.DockerImageName;

@TestConfiguration(proxyBeanMethods = false)
public class TestcontainersConfiguration {

    @Bean
    @ServiceConnection                       // URL·user·password를 컨텍스트에 자동 주입
    PostgreSQLContainer<?> postgresContainer() {
        return new PostgreSQLContainer<>("postgres:16-alpine");
    }

    @Bean
    @ServiceConnection(name = "redis")       // GenericContainer는 정체를 모르므로 name으로 명시
    @SuppressWarnings("resource")            // 빈 생명주기는 Spring이 소유(누수 아님)
    GenericContainer<?> redisContainer() {
        return new GenericContainer<>(DockerImageName.parse("redis:7-alpine"))
                .withExposedPorts(6379);
    }
}
```

| 포인트 | 설명 |
|---|---|
| `@ServiceConnection(name = "redis")` | `GenericContainer`는 무슨 서비스인지 추론 불가 → `name="redis"`로 알려줘야 host/port가 `spring.data.redis.*`에 자동 주입됨 (Spring Boot 3.1+) |
| `GenericContainer` 사용 | Redis 전용 모듈 없이 범용 컨테이너로 처리(6장 결정) |
| `@SuppressWarnings("resource")` | `GenericContainer`가 `Closeable`이라 린트가 누수 경고 → 실제로는 스프링이 정리하므로 억제 |
| 빈은 싱글톤 | 컨테이너는 스위트 전체에서 1회만 기동·재사용 |

### 4-5. 세션 영속 통합 테스트 — `backend/src/test/java/com/ixikey/SessionPersistenceIT.java`

```java
package com.ixikey;

import com.ixikey.support.TestcontainersConfiguration;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.annotation.Import;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.session.Session;
import org.springframework.session.SessionRepository;

import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@Import(TestcontainersConfiguration.class)
class SessionPersistenceIT {

    @Autowired
    SessionRepository<? extends Session> sessionRepository;

    @Autowired
    StringRedisTemplate redisTemplate;       // Redis에 실제로 저장됐는지 직접 검증용

    @Test
    void sessionSurvivesInRedisAndIsRetrievable() {
        saveAndVerify(sessionRepository);
    }

    // 타입 변수 S가 createSession()·save()·findById()를 하나로 묶어
    // 와일드카드 캡처 불일치를 해소하고, 비공개 RedisSession 타입을 이름으로 부르지 않는다.
    private <S extends Session> void saveAndVerify(SessionRepository<S> repo) {
        S session = repo.createSession();
        session.setAttribute("user", "ayden");
        String sessionId = session.getId();
        repo.save(session);

        // 1) 실제 Redis에 키 존재 (인메모리 아님)
        Set<String> keys = redisTemplate.keys("ixikey:session:*");
        assertThat(keys).isNotEmpty();

        // 2) 재조회 = Redis에서 읽어옴 → 재시작/타 인스턴스도 동일 조회
        S loaded = repo.findById(sessionId);
        assertThat(loaded).isNotNull();
        assertThat(loaded.<String>getAttribute("user")).isEqualTo("ayden");
    }
}
```

## 5. "재시작에도 유지"를 어떻게 증명했나

한 JVM 테스트에서 물리적 재시작을 재현할 필요는 없다. 두 단언의 **조합**이 논리적 증명이 된다.

| 단언 | 코드 | 증명하는 것 |
|---|---|---|
| ① Redis에 키 존재 | `redisTemplate.keys("ixikey:session:*")` 가 비어있지 않음 | 세션이 앱 힙이 아니라 **외부 Redis에 물리적으로 저장**됨 |
| ② 재조회 성공 | `repo.findById(id)` 가 속성 반환 | **Redis에서 읽어옴** (기본 `RedisSessionRepository`는 로컬 캐시가 없어 매번 Redis 왕복) |

①(외부 저장) + ②(캐시 없는 Redis 읽기)가 성립하면 → 같은 Redis를 보는 **새 인스턴스/재시작된 인스턴스도 동일 세션을 읽는다**가 보장된다.

> 더 강한 증명(물리적 2-컨텍스트 재시작)은 무겁고 본 단계 범위를 넘어 "확장 가능" 메모로 보류했다.

## 6. 의사결정 근거

| 결정 | 선택 | 근거 |
|---|---|---|
| 세션 저장소 지정 방식 | `spring.session.store-type` **미사용** | Spring Boot 3.0에서 제거된 속성. 클래스패스의 Spring Session 구현체(`spring-session-data-redis`)가 **유일**하면 자동 선택되므로 불필요 |
| 테스트 Redis 기동 | 전용 모듈 대신 **`GenericContainer` + `@ServiceConnection(name="redis")`** | 별도 의존성 없이 처리 가능 → 최소 의존성(Simplicity First) |
| 테스트 더블 | **실제 Redis(Testcontainers)** | Mock/H2 미사용, 운영과 동일 환경 검증(프로젝트 철학) |
| 세션 저장소 주입 타입 | `SessionRepository<? extends Session>` + **제네릭 헬퍼 메서드** | 구현체(`RedisSessionRepository`)의 `RedisSession`이 **비공개 타입**이라 직접 못 부름 → 타입 변수 `S`로 우회 |
| 검증 신뢰성 | `StringRedisTemplate`로 **키 직접 확인** | "그냥 메모리에서 읽은 것 아니냐"는 반박 차단 |

## 7. 트러블슈팅 기록

| 증상 | 원인 | 해결 |
|---|---|---|
| `Unknown property 'spring.session.store-type'` | 3.0에서 제거된 속성 | 해당 줄 삭제 (단일 구현체 자동 선택) |
| `HealthEndpointTest` 503(DOWN) 실패 | Redis 의존성 추가로 **Actuator Redis 헬스 인디케이터** 활성화 → 테스트에 Redis 없음 | `TestcontainersConfiguration`에 Redis 컨테이너 추가 → 헬스 200 복구 |
| `save(...) is not applicable for (Session)` | `<? extends Session>` 와일드카드 **캡처 불일치** | 제네릭 헬퍼 메서드 `<S extends Session>`로 타입 통일 |
| `RedisSession is not visible` | 구현체의 `RedisSession`이 **package-private** | 비공개 타입을 이름으로 부르지 않고 타입 변수 `S`로만 다룸 |
| `Resource leak: ... never closed` | `GenericContainer`가 `Closeable` | `@SuppressWarnings("resource")` (생명주기는 Spring 소유) |

## 8. 검증

| 항목 | 결과 |
|---|---|
| 통합 테스트 | `SessionPersistenceIT` 1건 추가 → 전체 9건 그린 |
| 헬스 체크 | Redis 헬스 인디케이터 포함 200(UP) 복구 |
| 전제 조건 | 테스트 실행 시 Docker 엔진(예: Docker Desktop) 기동 필요 |

> 테스트 실행 중 브레이크포인트로 오래 멈추면 Testcontainers 정리기(Ryuk)가 컨테이너를 회수할 수 있다. 장시간 디버깅 시 `TESTCONTAINERS_RYUK_DISABLED=true`.

## 9. 핵심 요약

- **무상태 설계**는 로드밸런싱·무중단 배포의 전제이며, 세션을 Redis로 외부화해 달성한다.
- **Spring Session**은 코드 변경 없이 세션 저장 위치만 Redis로 바꾼다(의존성 + 설정만으로).
- **Testcontainers + `@ServiceConnection`** 으로 실제 Redis에 통합 테스트해 DoD를 닫는다.
- 의존성 추가가 **Actuator 헬스 체크 범위를 넓힌다**는 점(503 이슈)이 실무적 학습 포인트.
