---
layout: post
title: "M0-2: Spring Boot 설정 외부화"
date: 2026-06-19 10:01:00 +0900
tags: [spring-boot, gradle, java, actuator]
description: "모든 환경값을 환경변수/외부 파일로 분리"
---
# Spring Boot 설정 외부화 (12-factor) — M0-2

## 1. 무엇을 / 왜

| 항목 | 내용 |
|---|---|
| 목표 | 모든 환경값을 환경변수/외부 파일로 분리. 코드·깃에 하드코딩 0. |
| 근거 | [12-factor app](https://12factor.net/config)의 **Config** 원칙 — "설정은 코드와 분리하고 환경에서 주입한다". |
| 효과 | 같은 빌드 산출물(JAR)을 환경(로컬/운영)만 바꿔 재사용. 비밀값이 산출물·깃에 남지 않음. |

## 2. 학습 개념 정리

| 개념 | 설명 |
|---|---|
| `application.yaml` | 공통 설정. 모든 프로파일에 적용. |
| `application-{profile}.yaml` | 프로파일별 오버라이드. 활성 프로파일 파일만 추가 로드. |
| 프로파일 | 실행 환경 구분 단위. `spring.profiles.active`로 선택. |
| `${ENV:default}` | 플레이스홀더. 환경값 `ENV`가 있으면 그 값, 없으면 `default`. |
| `spring.config.import` | 외부 설정 파일을 property source로 끌어옴. `.env` 로드에 사용. |
| `@ConfigurationProperties` | 외부 설정을 타입드 객체(POJO/record)로 바인딩. (M0-2에선 미사용, 후속 도입) |
| 비밀값 분리 | 키·비밀번호는 깃에 올리지 않고 `.env`(깃 제외)로만 주입. |

## 3. 의사결정

### 3-1. 프로파일 구성

| 결정 | 값 | 근거 |
|---|---|---|
| 프로파일 | `local` / `pilot` | 개발 머신(local)과 운영 PC(pilot) 분리. 개발 계획서 표기와 일치. |
| 기본 프로파일 | `local` | 미지정 시 개발 환경으로 안전하게 부팅. |
| 활성화 방식 | `SPRING_PROFILES_ACTIVE` 환경변수 | 환경마다 변수만 바꿔 전환. |

### 3-2. `.env` 처리

| 쟁점 | 선택 | 탈락안 / 근거 |
|---|---|---|
| 파일 개수 | **단일 `.env` 공유** | 개발 머신 ≠ 운영 PC라 실파일이 물리적으로 분리됨 → 같은 키 이름, 머신별 다른 값으로 단순화. |
| 로딩 방식 | **네이티브 `spring.config.import`** | `spring-dotenv` 라이브러리 탈락 — 순수 `KEY=value`면 의존성 0으로 충분. |
| 파일 위치 | **`backend/` 루트** | `resources/` 금지 — 빌드 시 JAR(classpath)에 비밀값이 패키징됨. `file:` 접두사는 실행 디렉터리 기준 파일시스템을 봄. |
| 깃 처리 | `.env` 제외 / `.env.example` 커밋 | 실값 유출 방지 + 필요한 키 목록은 공유. |

### 3-3. 스코프 (지금 안 하는 것)

| 항목 | 시점 | 이유 |
|---|---|---|
| DB 설정(`spring.datasource.*`) | M0-3 | JPA 스타터가 그때 추가됨. 지금 넣어도 소비처 없음. |
| Redis 설정(`spring.data.redis.*`) | M0-4 | Redis 스타터가 그때 추가됨. |
| LLM 타입드 프로퍼티(`ixikey.llm.*`) | M4 | LLM 사용 시점. 동일한 `${ENV:default}` + `.env` 패턴으로 끼워넣음. |

> 모두 같은 패턴을 재사용하므로, 필요 시점에 블록만 추가하면 됨.

## 4. 소스 전체

### 4-1. `backend/src/main/resources/application.yaml`

```yaml
spring:
  application:
    name: ixikey
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:local}   # 미지정 시 local
  config:
    import: optional:file:.env[.properties]    # backend/ 루트의 단일 .env 로드 (없어도 OK)

server:
  port: ${SERVER_PORT:8090}                    # 환경값 우선, 없으면 8090
```

### 4-2. `backend/src/main/resources/application-local.yaml`

```yaml
logging:
  level:
    "[com.ixikey]": DEBUG     # 키에 점(.) 포함 → 대괄호 escape (Spring map key 표기)
```

### 4-3. `backend/src/main/resources/application-pilot.yaml`

```yaml
logging:
  level:
    "[com.ixikey]": INFO
    root: WARN
```

### 4-4. `backend/.env.example` (커밋 / 실 `.env`는 깃 제외)

```dotenv
SPRING_PROFILES_ACTIVE=local
SERVER_PORT=8090
```

### 4-5. `backend/.gitignore` (한 줄 추가)

```gitignore
.env
```

## 5. 테스트

> 타입드 프로퍼티가 없는 단계이므로 **프로파일 활성화 + 환경값 외부화 동작**을 검증한다.
> 전부 `WebEnvironment.NONE`(웹서버 미기동)으로 가볍게 실행. `.env`는 `optional:`이라 파일 없는 CI에서도 통과.

| 테스트 클래스 | 검증 내용 |
|---|---|
| `ConfigBindingTest` | 기본 프로파일 `local` / 포트 기본값 8090 해석 |
| `PortInjectionTest` | `SERVER_PORT` 주입 시 기본값을 덮어쓰는지 |
| `PilotProfileTest` | `pilot` 프로파일에서 로깅 레벨 오버라이드 적용 |

### 5-1. `backend/src/test/java/com/ixikey/config/ConfigBindingTest.java`

```java
package com.ixikey.config;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.core.env.Environment;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * M0-2 설정 외부화 검증 (기본 동작).
 *
 * 목적:
 *  - 'spring.profiles.active: ${SPRING_PROFILES_ACTIVE:local}' 기본값이 적용되어
 *    별도 지정 없이 local 프로파일로 뜨는지 확인.
 *  - 'server.port: ${SERVER_PORT:8090}' 플레이스홀더가 환경값 미주입 시
 *    기본값 8090 으로 해석되는지 확인.
 *
 * WebEnvironment.NONE: 실제 웹서버를 띄우지 않고 컨텍스트만 로드(가볍고 빠름).
 * .env 는 'optional:' 이므로 파일이 없는 CI 에서도 통과한다.
 */
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
class ConfigBindingTest {

    @Autowired
    Environment env;

    /** 환경변수 미지정 시 기본 프로파일이 local 인지 검증. */
    @Test
    void defaultProfileIsLocal() {
        assertThat(env.getActiveProfiles()).contains("local");
    }

    /** SERVER_PORT 미주입 시 ${SERVER_PORT:8090} 기본값이 적용되는지 검증. */
    @Test
    void serverPortFallsBackToDefault() {
        assertThat(env.getProperty("server.port")).isEqualTo("8090");
    }
}
```

### 5-2. `backend/src/test/java/com/ixikey/config/PortInjectionTest.java`

```java
package com.ixikey.config;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.core.env.Environment;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * M0-2 설정 외부화 검증 (주입값 우선).
 *
 * 목적:
 *  - 외부에서 SERVER_PORT 가 주입되면 ${SERVER_PORT:8090} 의 기본값이 아니라
 *    주입값이 사용되는지 = "설정 외부화가 실제로 동작"함을 증명.
 *
 * 실제 OS 환경변수/.env 대신 properties 로 SERVER_PORT 를 Environment 에 넣어
 * 동일한 플레이스홀더 해석 경로를 검증한다(테스트 격리·재현성 확보).
 */
@SpringBootTest(
        webEnvironment = SpringBootTest.WebEnvironment.NONE,
        properties = "SERVER_PORT=19090")
class PortInjectionTest {

    @Autowired
    Environment env;

    /** 주입된 SERVER_PORT(19090)가 기본값(8090)을 덮어쓰는지 검증. */
    @Test
    void injectedServerPortOverridesDefault() {
        assertThat(env.getProperty("server.port")).isEqualTo("19090");
    }
}
```

### 5-3. `backend/src/test/java/com/ixikey/config/PilotProfileTest.java`

```java
package com.ixikey.config;

import ch.qos.logback.classic.Level;
import ch.qos.logback.classic.Logger;
import org.junit.jupiter.api.Test;
import org.slf4j.LoggerFactory;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * M0-2 설정 외부화 검증 (프로파일별 오버라이드).
 *
 * 목적:
 *  - @ActiveProfiles("pilot") 로 띄웠을 때 application-pilot.yaml 의
 *    로깅 설정이 실제로 바인딩·적용되는지 확인.
 *  - raw 프로퍼티 키('[com.ixikey]')를 읽는 대신, 적용된 로거의 실제 레벨을
 *    조회해 "설정이 효력을 가졌다"를 확실히 검증한다.
 *    (local=DEBUG, pilot=INFO 로 프로파일마다 다름)
 *
 * 캐스팅: spring-boot-starter 의 기본 로깅 구현이 logback 이므로
 *         org.slf4j.Logger -> ch.qos.logback.classic.Logger 캐스팅이 가능.
 */
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
@ActiveProfiles("pilot")
class PilotProfileTest {

    /** pilot 프로파일에서 com.ixikey 로거 레벨이 INFO 로 적용되는지 검증. */
    @Test
    void pilotProfileSetsInfoLogLevel() {
        Logger logger = (Logger) LoggerFactory.getLogger("com.ixikey");
        assertThat(logger.getLevel()).isEqualTo(Level.INFO);
    }
}
```

## 6. 실행 / 검증

| 명령 | 용도 |
|---|---|
| `./gradlew test` | 전체 테스트 실행 (`BUILD SUCCESSFUL` = 그린) |
| `./gradlew test --tests "com.ixikey.config.*"` | 설정 테스트만 실행 |
| `./gradlew clean test` | 캐시 무시하고 재실행 |
| `build/reports/tests/test/index.html` | 결과 리포트(HTML) |
| `git grep -nE 'api[_-]?key\|secret\|password' -- '*.yaml' '*.java'` | 평문 비밀값 0 확인 |
| `git check-ignore .env` | `.env` 깃 제외 확인 (출력되면 정상) |

> Windows: `./gradlew` 대신 `.\gradlew`(PowerShell) 또는 `gradlew.bat`(cmd).

## 7. DoD (완료 기준)

| 체크 | 기준 |
|---|---|
| ☑ 테스트 그린 | `./gradlew test` 통과 (메서드 4개) |
| ☑ 비밀값 분리 | 코드·yaml에 평문 비밀값 없음 |
| ☑ 깃 제외 | `.env` 미추적, `.env.example`만 커밋 |
| ☑ 프로파일 전환 | `SPRING_PROFILES_ACTIVE=pilot` 시 pilot 설정 적용 |

## 8. 핵심 요약

| 키워드 | 한 줄 |
|---|---|
| 12-factor Config | 설정을 코드에서 분리해 환경에서 주입 |
| `${ENV:default}` | 환경값 우선 + 기본값 fallback |
| `spring.config.import` | `.env`를 네이티브로 로드(의존성 0) |
| `.env` 위치 | `resources/` 금지, `backend/` 루트 + 깃 제외 |
| 프로파일 | `local`(기본·DEBUG) / `pilot`(운영·INFO) |
