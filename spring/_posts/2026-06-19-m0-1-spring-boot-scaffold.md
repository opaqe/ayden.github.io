---
layout: post
title: "M0-1: Spring Boot 프로젝트 스캐폴드"
date: 2026-06-19 15:00:00 +0900
tags: [spring-boot, gradle, java, actuator]
description: "Spring Boot 3.5 + Gradle로 빈 앱을 스캐폴드하고 /actuator/health 200까지 — 버전 선택 근거와 트러블슈팅 정리."
---

빈 Spring Boot 앱을 세우고 `/actuator/health`가 200을 반환하는 것까지를 목표로, 버전 선택의 근거와 실제로 부딪힌 빌드 에러를 정리한다.

## 1. 목표

| 항목 | 내용 |
|---|---|
| 결과물 | 빈 Spring Boot 앱이 `/actuator/health`에서 200 반환 |
| 완료 기준(DoD) | `./gradlew test` 그린 + 로컬 health 200 |
| 범위 | 기반 골격까지. DB·Redis·Docker·공통 처리·CI는 이후 단계 |

## 2. 스택 결정

| 항목 | 선택 | 근거 |
|---|---|---|
| Java | 17 LTS | Spring Boot 3.5·4.0·LangChain4j 모두의 최소 요구. (virtual threads는 21+, 의식적 보류) |
| Spring Boot | 3.5.15 | LangChain4j가 Boot 3 starter 트랙(`-spring-boot-starter`)을 사용. 4.x는 별도 트랙(`-spring-boot4-starter`). 안정·호환 우선 |
| 빌드 도구 | Gradle 8.14.5 (Groovy DSL) | 자료 풍부. (Kotlin DSL은 빌드 전용 문법이라 Kotlin 앱 개발 경험과 무관) |
| 의존성 관리 | io.spring.dependency-management 1.1.7 | Spring Boot BOM 기반 버전 정렬 |
| LLM 의존성 | `langchain4j-spring-boot-starter:1.16.3-beta26` | Spring 자동설정 제공. 코어(`langchain4j`)가 아님. M0-1에선 미사용(부팅 확인용) |
| 레포 구조 | monorepo, `backend/` 하위 | 백엔드·프론트·nginx가 한 레포. 컴포넌트별 폴더 분리 |
| 앱 포트 | 8090 | 예약 포트 회피 (아래 표) |

### 포트 예약

| 포트 | 용도 |
|---|---|
| 8090 / 8091 | 앱 인스턴스 (현재 8090) |
| 3000 | Grafana |
| 9090 | Prometheus |
| 5432 | PostgreSQL |
| 6379 | Redis |
| 80 / 443 | nginx |

## 3. 디렉토리 구조

```
ixikey/
├─ AGENTS.md  CLAUDE.md  PROJECT.md  DESIGN.md
├─ .gitignore
├─ docs/
└─ backend/
   ├─ build.gradle
   ├─ settings.gradle
   ├─ gradlew  gradlew.bat  gradle/
   ├─ .gitignore
   └─ src/
      ├─ main/java/com/ixikey/IxikeyApplication.java
      ├─ main/resources/application.yml
      └─ test/java/com/ixikey/
         ├─ IxikeyApplicationTests.java
         └─ HealthEndpointTest.java
```

## 4. Spring Initializr 설정

| 필드 | 값 |
|---|---|
| Project | Gradle - Groovy |
| Language | Java |
| Spring Boot | 3.5.15 |
| Group | `com.ixikey` |
| Artifact | `ixikey` |
| Package name | `com.ixikey` |
| Packaging | Jar |
| Java | 17 |
| Dependencies | Spring Web, Spring Boot Actuator |

생성물의 `build.gradle`·`src/`·`gradlew` 등을 `backend/` 폴더에 배치한다.

## 5. 소스 코드

### `backend/build.gradle`

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
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'dev.langchain4j:langchain4j-spring-boot-starter:1.16.3-beta26'  // 미사용(부팅 확인용)

    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}

tasks.named('test') {
    useJUnitPlatform()
}
```

### `backend/src/main/java/com/ixikey/IxikeyApplication.java`

```java
package com.ixikey;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class IxikeyApplication {

    public static void main(String[] args) {
        SpringApplication.run(IxikeyApplication.class, args);
    }
}
```

### `backend/src/main/resources/application.yml`

```yaml
server:
  port: 8090
```

### `backend/src/test/java/com/ixikey/IxikeyApplicationTests.java`

```java
package com.ixikey;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;

@SpringBootTest
class IxikeyApplicationTests {

    @Test
    void contextLoads() {
    }
}
```

### `backend/src/test/java/com/ixikey/HealthEndpointTest.java`

```java
package com.ixikey;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@SpringBootTest
@AutoConfigureMockMvc
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

## 6. 실행 / 검증

```bash
cd backend

# 빌드
./gradlew build

# 실행
./gradlew bootRun

# 다른 터미널에서 health 확인
curl http://localhost:8090/actuator/health
# → {"status":"UP"}

# 테스트 (DoD)
./gradlew test
```

## 7. 트러블슈팅

| 증상 | 원인 | 해결 |
|---|---|---|
| `Could not find dev.langchain4j:langchain4j-spring-boot-starter:1.16.3` | ① 코어(`langchain4j`)와 starter 혼동 ② 버전에 `-beta` 누락 | starter 사용 + `maven-metadata.xml`에서 실재 버전 확인 → `1.16.3-beta26` |
| `Web server failed to start. Port 8080 was already in use` | 8080을 다른 서비스가 점유 | `application.yml`에 `server.port: 8090` |
| `package org.junit... does not exist` (태스크 `:compileJava`) | 테스트 파일이 `src/main`에 위치. `testImplementation` 의존성은 test 소스셋에서만 보임 | 파일을 `src/test/java/com/ixikey/`로 이동 |

### LangChain4j 버전 확인

```
https://repo1.maven.org/maven2/dev/langchain4j/langchain4j-spring-boot-starter/maven-metadata.xml
```

```xml
<release>1.16.3-beta26</release>
```

### 포트 점유 확인 (Windows)

```powershell
# PowerShell
Get-NetTCPConnection -LocalPort 8090 -ErrorAction SilentlyContinue   # 출력 없으면 사용 가능
```

```cmd
:: cmd
netstat -ano | findstr :8090
```

## 8. 핵심 정리

| 포인트 | 내용 |
|---|---|
| 버전 선택 | 의존성 제약에서 역산 (LangChain4j → Spring Boot 3.5.x → Java 17) |
| 빌드 DSL | Kotlin DSL(`.kts`) ≠ Kotlin 앱 개발 경험 |
| 의존성 버전 | 추측 금지. `maven-metadata.xml`에서 실재 버전 확인 |
| 소스셋 | `testImplementation`은 `src/test`에서만 클래스패스에 올라옴 |
| 빌드 에러 | 실패 태스크명(`:compileJava`)과 파일 경로가 원인을 알려줌 |
| MockMvc | 실제 포트 미사용 → `server.port` 설정과 무관하게 테스트 통과 |

## 9. Git 작업 흐름

| 단계 | 명령 / 내용 |
|---|---|
| 브랜치 | `git checkout -b feature/M0-1-spring-boot-scaffold` (1 유닛 = 1 브랜치 = 1 PR) |
| 커밋 | Conventional Commits — `feat(M0-1): ...`, `chore: ...`, `docs: ...` |
| PR | 셀프 리뷰(`Comment`), 설명에 목표/변경/테스트/DoD/스코프/다음 명시 |
| 머지 후 | `git checkout main` → `git pull` → `git branch -d feature/...` |

## 10. 다음 단계

**M0-2 설정 외부화(12-factor)** — `local`/`pilot` 프로파일 분리, DB·Redis·LLM 키를 환경변수로 분리, `.env.example` 작성(비밀값 git 제외).
