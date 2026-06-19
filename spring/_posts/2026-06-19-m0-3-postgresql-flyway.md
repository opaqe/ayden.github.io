---
layout: post
title: "M0-3: PostgreSQL + Flyway 연동 (Spring Boot 3.5 / Java 17)"
date: 2026-06-19 17:00:00 +0900
tags: [spring-boot, postgresql, flyway, testcontainers, jpa]
description: "앱을 PostgreSQL에 연결하고 Flyway로 스키마 마이그레이션을 자동 적용, Testcontainers로 H2 없이 운영과 동일한 DB로 통합 테스트를 그린으로 만든 과정."
---

> 앱을 실제 DB에 붙이고, 스키마 마이그레이션을 자동 적용하고, **H2가 아닌 운영과 동일한 PostgreSQL**로 통합 테스트를 그린으로 만드는 단계.

## 1. 한눈에 보기

| 항목 | 내용 |
|---|---|
| 목표 | 앱이 PostgreSQL에 연결되고 Flyway 마이그레이션이 부팅 시 자동 적용 |
| 익힐 개념 | JPA/Hibernate, Flyway 버전 마이그레이션, Testcontainers, `@ServiceConnection` |
| DoD | 통합 테스트 그린 (**H2 미사용**) |
| 선행 | M0-1(스캐폴드), M0-2(설정 외부화) |
| 스택 | Java 17 · Spring Boot 3.5.15 · Gradle(Groovy) · PostgreSQL 16 |

## 2. 핵심 개념 정리

| 개념 | 한 줄 정의 | 다루는 대상 | 동작 시점 |
|---|---|---|---|
| **Hibernate(JPA)** | 객체 ↔ 테이블 자동 매핑 ORM | 데이터(행) INSERT/SELECT | 앱 실행 중(요청 처리) |
| **Flyway** | 스키마 변경 이력을 버전 SQL로 관리(=DB의 git) | 구조(테이블) CREATE/ALTER | 앱 부팅 시 1회 |
| **Testcontainers** | 테스트 때 Docker로 실 DB를 띄웠다 버리는 라이브러리 | 테스트용 일회성 컨테이너 | 테스트 실행 중 |
| **@ServiceConnection** | 띄운 컨테이너의 접속정보(URL·user·pw)를 컨텍스트에 자동 주입 | 테스트 DataSource 배선 | 테스트 컨텍스트 로드 |

**Flyway ↔ Hibernate 역할 분담 (이 단계의 핵심 원칙)**

| | Flyway | Hibernate(JPA) |
|---|---|---|
| 책임 | 스키마 **생성·변경** (단일 소유자) | 엔티티 ↔ 테이블 **검증(validate)** |
| 부팅 순서 | 1순위(테이블 생성) | 2순위(생성된 테이블 검증) |

## 3. 의사결정 정리

| # | 결정 | 선택 | 근거 |
|---|---|---|---|
| 1 | Flyway PostgreSQL 지원 | `flyway-database-postgresql` **추가** | Flyway 10+부터 DB 벤더 지원이 별도 모듈로 분리. 없으면 PostgreSQL 마이그레이션 동작 안 함 |
| 2 | 스키마 자동생성 정책 | `ddl-auto: validate` | 스키마 주인은 Flyway 하나로 고정. `update`/`create`는 Hibernate가 스키마를 바꿔 Flyway와 충돌 |
| 3 | OSIV | `open-in-view: false` | 뷰 렌더링까지 커넥션 점유 방지. 기본값 `true`는 부팅 경고 발생 |
| 4 | JDBC 드라이버 스코프 | `runtimeOnly` | 컴파일 시점에 직접 참조 없음 → 의존 범위 최소화 |
| 5 | 테스트 DB | **Testcontainers PostgreSQL** (H2 금지) | H2는 SQL 방언·타입이 달라 "테스트 통과, 운영 실패" 위험. 향후 pgvector(M8)는 H2 불가 |
| 6 | Postgres 이미지 | `postgres:16-alpine` 핀 고정 | 재현성. 테스트와 로컬 실행 환경 일치 |
| 7 | 컨테이너 수명 | `@TestConfiguration` 싱글톤 빈 | 스위트 전체에서 1회 기동·재사용 → 테스트 속도 확보 |
| 8 | 접속정보 관리 | `${ENV:기본값}` 외부화(M0-2 패턴 유지) | 12-factor. 코드에 접속정보 하드코딩 금지 |

## 4. 구현 (전체 소스 · 파일 경로 포함)

### 4-1. 의존성

**`backend/build.gradle`**

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.5.15'
    id 'io.spring.dependency-management' version '1.1.7'
}

group = 'com.ixikey'
version = '0.0.1-SNAPSHOT'

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(17)
    }
}

repositories {
    mavenCentral()
}

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

    // --- 테스트 ---
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.boot:spring-boot-testcontainers'
    testImplementation 'org.testcontainers:postgresql'
    testImplementation 'org.testcontainers:junit-jupiter'
    testRuntimeOnly    'org.junit.platform:junit-platform-launcher'
}

tasks.named('test') {
    useJUnitPlatform()
}
```

> 버전을 명시하지 않는 이유: `io.spring.dependency-management`(Spring Boot BOM)가 Flyway·PostgreSQL·Testcontainers 버전을 일괄 관리한다. 직접 명시하면 BOM과 어긋날 위험이 있다.

| 의존성 | 스코프 | 역할 |
|---|---|---|
| `spring-boot-starter-data-jpa` | implementation | Hibernate + Spring Data JPA + HikariCP |
| `flyway-core` | implementation | 마이그레이션 엔진 |
| `flyway-database-postgresql` | implementation | Flyway의 PostgreSQL 벤더 지원(10+ 필수) |
| `postgresql` | runtimeOnly | JDBC 드라이버 |
| `spring-boot-testcontainers` | testImplementation | `@ServiceConnection` 지원 |
| `org.testcontainers:postgresql` | testImplementation | PostgreSQL 컨테이너 |
| `org.testcontainers:junit-jupiter` | testImplementation | JUnit5 컨테이너 라이프사이클 |

### 4-2. 마이그레이션

**`backend/src/main/resources/db/migration/V1__init.sql`**

```sql
-- M0-3: 마이그레이션 파이프라인 검증용 health 테이블
CREATE TABLE health_check (
    id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    status     VARCHAR(20)  NOT NULL,
    checked_at TIMESTAMPTZ  NOT NULL DEFAULT now()
);

INSERT INTO health_check (status) VALUES ('OK');
```

| 규칙 | 설명 |
|---|---|
| 파일명 | `V` + 버전 + `__`(언더스코어 2개) + 설명 + `.sql`. 어기면 Flyway가 인식 못 함 |
| 적용 위치 | 기본 `classpath:db/migration` 자동 스캔 |
| 이력 관리 | Flyway가 `flyway_schema_history` 테이블에 적용 버전 기록 → 미적용분만 순서대로 실행 |

### 4-3. 설정 (외부화 + 프로파일)

**`backend/src/main/resources/application.yaml`**

```yaml
spring:
  application:
    name: ixikey
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:local}
  config:
    import: optional:file:.env[.properties]   # 루트(backend/)의 단일 .env 로드

  datasource:
    url: ${DB_URL:jdbc:postgresql://localhost:25432/ixikey}
    username: ${DB_USERNAME:ixikey}
    password: ${DB_PASSWORD:ixikey}
    driver-class-name: org.postgresql.Driver

  jpa:
    hibernate:
      ddl-auto: validate        # 스키마는 Flyway가 소유, Hibernate는 검증만
    open-in-view: false         # OSIV 비활성 (지연로딩 누수·커넥션 점유 방지)
    properties:
      hibernate:
        "[format_sql]": true

  flyway:
    enabled: true
    # locations 기본값 classpath:db/migration — 명시 불필요

server:
  port: ${SERVER_PORT:8090}
```

**`backend/src/main/resources/application-local.yaml`** (개발: SQL 로그 노출)

```yaml
spring:
  jpa:
    show-sql: true
logging:
  level:
    "[com.ixikey]": DEBUG
    "[org.hibernate.SQL]": DEBUG          # 실행 SQL
    "[org.flywaydb]": INFO                # 마이그레이션 적용 로그
```

**`backend/src/main/resources/application-pilot.yaml`** (운영: 로그 최소)

```yaml
spring:
  jpa:
    show-sql: false
logging:
  level:
    "[com.ixikey]": INFO
    root: WARN
```

**`backend/.env.example`** (실값 없는 템플릿 — 실제 `.env`는 `.gitignore` 처리)

```properties
SPRING_PROFILES_ACTIVE=local
SERVER_PORT=8090

# --- M0-3 PostgreSQL ---
DB_URL=jdbc:postgresql://localhost:25432/ixikey
DB_USERNAME=ixikey_admin
DB_PASSWORD=changeme
```

### 4-4. 테스트 — 공유 컨테이너 설정

**`backend/src/test/java/com/ixikey/support/TestcontainersConfiguration.java`**

```java
package com.ixikey.support;

import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.springframework.context.annotation.Bean;
import org.testcontainers.containers.PostgreSQLContainer;

/**
 * 통합 테스트용 공유 Testcontainers 설정.
 *
 * [임시 Docker 사용]
 *  - 이 테스트는 Dockerfile / compose.yml(M0-6) 같은 프로젝트 설정 파일이 아니라,
 *    머신에 떠 있는 Docker 엔진(데몬)을 사용한다.
 *  - 테스트 실행 시 Testcontainers가 Docker 엔진에 요청해 'postgres:16-alpine'
 *    이미지로 임시 컨테이너를 띄우고, 테스트가 끝나면 자동으로 폐기한다.
 *    (= 운영용이 아닌, 테스트 한정 일회성 컨테이너)
 *  - 따라서 테스트를 돌리려면 Docker 엔진(예: Docker Desktop)이 실행 중이어야 한다.
 *  - 컨테이너 빈은 싱글톤이라 스위트 전체에서 한 번만 기동·재사용된다(매 테스트 재기동 아님).
 *  - H2 미사용: 운영과 동일한 PostgreSQL로 검증하기 위함(DoD).
 */
@TestConfiguration(proxyBeanMethods = false)
public class TestcontainersConfiguration {

    @Bean
    @ServiceConnection                       // URL·user·password를 컨텍스트에 자동 주입
    PostgreSQLContainer<?> postgresContainer() {
        return new PostgreSQLContainer<>("postgres:16-alpine");
    }
}
```

### 4-5. 테스트 — 통합 테스트 (마이그레이션 + 연결 검증)

**`backend/src/test/java/com/ixikey/DatabaseMigrationIT.java`**

```java
package com.ixikey;

import com.ixikey.support.TestcontainersConfiguration;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.annotation.Import;
import org.springframework.jdbc.core.JdbcTemplate;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@Import(TestcontainersConfiguration.class)
class DatabaseMigrationIT {

    @Autowired JdbcTemplate jdbcTemplate;

    @Test
    void flyway가_V1을_적용한다() {
        Integer applied = jdbcTemplate.queryForObject(
            "SELECT count(*) FROM flyway_schema_history WHERE version = '1'",
            Integer.class);
        assertThat(applied).isEqualTo(1);
    }

    @Test
    void health_테이블에_연결되고_시드가_있다() {
        String status = jdbcTemplate.queryForObject(
            "SELECT status FROM health_check LIMIT 1", String.class);
        assertThat(status).isEqualTo("OK");
    }
}
```

| 테스트 | 증명하는 것 |
|---|---|
| `flyway가_V1을_적용한다` | `flyway_schema_history`에 V1 기록 존재 → **마이그레이션 적용** |
| `health_테이블에_연결되고_시드가_있다` | 실제 테이블 쿼리 성공 → **연결 + 테이블 존재** |

> 엔티티 없이 `JdbcTemplate`로 검증하는 이유: M0-3 범위에는 매핑 엔티티가 없다(엔티티는 M1-1 User부터).

### 4-6. 기존 테스트 수정 (JPA 도입 부작용 대응)

JPA starter가 들어오면 `@SpringBootTest`로 전체 컨텍스트를 띄우는 **모든 테스트가 DataSource를 요구**한다. 기존 5개 테스트에 동일 패턴으로 공유 컨테이너를 주입한다.

**적용 대상 (5개)**

| 파일 경로 |
|---|
| `backend/src/test/java/com/ixikey/HealthEndpointTest.java` |
| `backend/src/test/java/com/ixikey/IxikeyApplicationTests.java` |
| `backend/src/test/java/com/ixikey/config/ConfigBindingTest.java` |
| `backend/src/test/java/com/ixikey/config/PilotProfileTest.java` |
| `backend/src/test/java/com/ixikey/config/PortInjectionTest.java` |

**공통 수정 패턴 (import 2줄 + 어노테이션 1줄)**

```java
// import 추가
import org.springframework.context.annotation.Import;
import com.ixikey.support.TestcontainersConfiguration;

// 클래스 선언부에 어노테이션 추가
@Import(TestcontainersConfiguration.class)
```

**예시 — `backend/src/test/java/com/ixikey/HealthEndpointTest.java` (전체)**

```java
package com.ixikey;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.annotation.Import;
import org.springframework.test.web.servlet.MockMvc;

import com.ixikey.support.TestcontainersConfiguration;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@SpringBootTest
@AutoConfigureMockMvc
@Import(TestcontainersConfiguration.class)
class HealthEndpointTest {

    @Autowired
    MockMvc mockMvc;

    @Test
    void healthReturnsUp() throws Exception {
        mockMvc.perform(get("/actuator/health"))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$.status").value("UP"));
    }
}
```

## 5. 겪은 문제와 해결

| 증상 | 원인 | 해결 |
|---|---|---|
| 기존 5개 `@SpringBootTest` 테스트가 한꺼번에 실패 (`ConnectException` → `localhost:5432`) | JPA 도입으로 전체 컨텍스트가 DataSource 요구. 컨테이너 미배선 시 기본값 주소로 연결 시도 | 각 테스트에 `@Import(TestcontainersConfiguration.class)` 추가 |
| `gradlew test`는 통과하는데 `bootRun` 실행은 연결 실패 | 테스트는 Testcontainers가 임시 DB 제공. **앱 실행은 실 DB가 없음** | 로컬에 `.env`와 동일한 PostgreSQL 기동 (아래 명령) |
| "Dockerfile도 안 만들었는데 왜 Docker가 필요?" | Testcontainers는 **Docker 엔진(데몬)**을 사용. Dockerfile/compose(M0-6)와 별개 | Docker Desktop 등 엔진만 실행돼 있으면 됨 |

**로컬 실행용 PostgreSQL 기동 (M0-6 compose 전까지 임시)**

```bash
docker run -d --name ixikey-postgres \
  -p 25432:5432 \
  -e POSTGRES_DB=ixikey \
  -e POSTGRES_USER=ixikey_admin \
  -e POSTGRES_PASSWORD='<password>' \
  postgres:16-alpine
```

| 포인트 | 설명 |
|---|---|
| 포트 `25432:5432` | 호스트 25432 → 컨테이너 5432 (`.env`의 `DB_URL` 포트와 일치) |
| DB·계정·비번 | `.env` 값과 정확히 일치시켜야 연결됨 |
| 이미지 | 테스트와 동일 `postgres:16-alpine` |

**Docker의 두 의미 구분**

| 구분 | 정체 | 이번에 필요? |
|---|---|---|
| Docker 엔진(데몬) | 컨테이너를 띄우는 런타임(설치 프로그램) | ✅ 필요 (Testcontainers·로컬 DB) |
| Dockerfile / compose.yml | 직접 작성하는 설정 파일 | ❌ M0-6 작업 |

## 6. 검증 결과

```bash
cd backend && ./gradlew clean test
```

| 항목 | 결과 |
|---|---|
| 총 테스트 | 8건 (DatabaseMigrationIT 2 + health 1 + context 1 + config 3) |
| 결과 | 전부 그린 |
| DB | PostgreSQL(Testcontainers), **H2 미사용** |
| bootRun | 로컬 Postgres 기동 후 정상, Flyway V1 적용 확인 |

## 7. 핵심 요약

| 키워드 | 기억할 점 |
|---|---|
| 역할 분담 | 스키마=Flyway, 데이터=Hibernate, 검증=`ddl-auto: validate` |
| Flyway 10+ | PostgreSQL은 `flyway-database-postgresql` 별도 모듈 필수 |
| 테스트 DB | H2 금지, Testcontainers로 운영과 동일 DB 사용 |
| @ServiceConnection | 컨테이너 접속정보 자동 주입 → 배선 보일러플레이트 제거 |
| 부작용 | JPA 도입 후 모든 `@SpringBootTest`가 DataSource 필요 → 공유 컨테이너 Import |
| 테스트 vs 실행 | 테스트는 컨테이너 자동, 앱 실행은 실 DB를 별도로 기동해야 함 |
