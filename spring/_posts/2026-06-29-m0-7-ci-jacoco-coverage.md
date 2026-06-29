---
layout: post
title: "M0-7: GitHub Actions CI + JaCoCo 커버리지 (자체 호스팅 배지)"
date: 2026-06-29 15:00:00 +0900
tags: [spring-boot, github-actions, ci, jacoco, code-coverage]
description: "PR·push마다 빌드·테스트·커버리지를 자동 검증하는 GitHub Actions CI를 구축하고, JaCoCo로 커버리지를 측정해 csv→SVG 자체 호스팅 배지를 main에 자동 커밋하도록 만든 과정."
---

> PR마다 빌드·테스트·커버리지를 자동으로 검증하는 CI 파이프라인을 구축한다.
> 핵심 학습: **CI의 역할**, **JaCoCo 커버리지 측정**, **GitHub Actions 워크플로우 구조**, **Testcontainers와 CI의 관계**, **자체 호스팅 커버리지 배지**.

---

## 1. 목표와 범위

| 구분 | 내용 |
|---|---|
| 목표 | 코드 변경마다 "빌드되는가 / 테스트 통과하는가 / 커버리지는 얼마인가"를 자동 검증 |
| 완료 기준(DoD) | PR에 CI 그린(초록 체크) + 커버리지 리포트 생성 |
| 포함 | 빌드·테스트 자동화, JaCoCo 리포트(xml/csv/html), 커버리지 배지, PR 트리거 |
| 제외 | 커버리지 80% 게이트(M11-6으로 이연), 자동 배포(CD), 코드 포맷 게이트(Spotless 별도 유닛) |

---

## 2. 핵심 개념 정리

### 2-1. CI vs CD

| 항목 | CI (이번 작업) | CD (이번 작업 아님) |
|---|---|---|
| 정의 | Continuous Integration — 변경마다 빌드·테스트로 코드 건강성 검증 | Continuous Deployment — 검증된 산출물을 서버에 배포 |
| 결과물 | 통과 여부(초록/빨강), 커버리지 리포트 | 실제 동작하는 배포본 |
| 빌드 산출물 | 검증 후 폐기(러너 VM 소멸) | 서버로 전달 |
| 본 프로젝트 위치 | M0-7 | M9(무중단 배포, Blue-Green) |

> CI가 초록이라는 것은 "코드가 건강하다"는 증명일 뿐, "서버에 반영됐다"는 뜻이 아니다.

### 2-2. JaCoCo — 테스트 커버리지

**Ja**va **Co**de **Co**verage. 테스트가 실제 코드의 몇 %를 실행했는지 측정한다.

| 커버리지 종류 | 측정 대상 |
|---|---|
| Line coverage | 실행된 코드 줄의 비율 |
| Branch coverage | if/switch 등 분기 중 실행된 비율 |

리포트는 세 형식으로 나온다.

| 형식 | 용도 |
|---|---|
| html | 사람이 브라우저로 보는 상세 리포트 |
| xml | CI 도구 연동용 |
| csv | 배지 생성기가 커버리지 % 숫자를 읽는 입력 |

### 2-3. 커버리지 배지와 SVG

배지(badge)는 README 상단에 보이는 작은 라벨 이미지(`coverage 87%` 같은 것). SVG는 그 이미지 형식으로, 픽셀이 아니라 도형·텍스트를 좌표로 기술하는 벡터 포맷이라 확대해도 안 깨지고 파일이 작다.

| 배지 종류 | 생성 방식 | 저장 위치 |
|---|---|---|
| CI 상태 배지 | GitHub이 요청 시 즉석 생성 | 저장 안 함(URL 엔드포인트) |
| Coverage·Branches 배지 | CI가 csv를 읽어 SVG 파일로 생성 후 커밋 | `.github/badges/*.svg` |

### 2-4. Testcontainers vs docker compose

이 프로젝트의 통합 테스트는 `compose.yml`을 쓰지 않고 **Testcontainers**를 쓴다. 둘은 역할이 다르다.

| 항목 | docker compose (`compose.yml`) | Testcontainers |
|---|---|---|
| 목적 | 실제 앱 스택을 로컬에서 기동 | 테스트 중 일회용 컨테이너 자동 생성·정리 |
| 실행 주체 | 사람이 직접 `up` | 테스트 코드가 Docker API 직접 호출 |
| 생명주기 | 수동 종료 | 테스트 종료 시 자동 정리(Ryuk 청소부 컨테이너) |
| 필요 조건 | — | **Docker 엔진이 떠 있어야 함** |

> CI 러너 `ubuntu-latest`에는 Docker가 내장돼 있어, Testcontainers가 별도 설정 없이 PostgreSQL·Redis 컨테이너를 띄운다.

### 2-5. Gradle UP-TO-DATE 캐싱 (자주 혼동하는 부분)

Gradle은 태스크 입력이 직전과 같으면 **재실행하지 않고 성공으로 건너뛴다(UP-TO-DATE)**. 그래서 로컬에서 Docker를 끈 채 `test`를 돌려도 "통과"로 보일 수 있는데, 이는 실제 실행이 아니라 캐시된 결과다.

| 상황 | 동작 |
|---|---|
| 로컬, 변경 없음 | `test` UP-TO-DATE → 컨테이너 안 뜸, 실제 검증 아님 |
| 로컬 강제 재실행(`--rerun-tasks`) | 실제 실행 → Docker 필요 |
| CI | 매번 빈 체크아웃 → 캐시 없음 → 항상 실제 실행 |

---

## 3. 의사결정 기록

| # | 결정 | 선택 | 근거 |
|---|---|---|---|
| 1 | 커버리지 도구 | JaCoCo | Java 표준 커버리지 도구, Gradle 플러그인 기본 제공 |
| 2 | 커버리지 배지 방식 | 자체 호스팅(SVG 생성·커밋) | 외부 서비스(Codecov 등) 가입·토큰 없이 운영. 의존성 최소화 |
| 3 | 배지 생성 도구 | `cicirello/jacoco-badge-generator` | csv 리포트를 읽어 SVG 생성, 외부 서비스 불필요 |
| 4 | 배지 갱신 시점 | main 한정 | PR 브랜치 되커밋 혼란 방지, main 기준 커버리지 표시 |
| 5 | 80% 커버리지 게이트 | M0-7 제외 | M0은 골격 단계라 실 로직 적음. 게이트는 M11-6에서 강제 |
| 6 | 코드 포맷 게이트(Spotless) | M0-7 제외, 별도 유닛 | 빌드 JDK 영향(아래 4-4) 때문에 범위 분리 |
| 7 | 통합 테스트 실행 방식 | 한 job에서 전체 실행 | 규모가 작아 분리 불필요. 느려지면 그때 분리 |
| 8 | 러너/JDK | `ubuntu-latest` + Temurin 17 | Docker 내장(Testcontainers), 프로젝트 타깃 Java 17 |

---

## 4. 구현 — 소스 전문

### 4-1. `backend/build.gradle`

JaCoCo 플러그인 추가 + 리포트 설정. 변경 핵심은 ① `plugins`에 `jacoco`, ② `test` 태스크에 `finalizedBy`, ③ 파일 하단의 `jacoco`/`jacocoTestReport` 설정 블록.

```groovy
plugins {
	id 'java'
	id 'org.springframework.boot' version '3.5.15'
	id 'io.spring.dependency-management' version '1.1.7'
	id 'jacoco'
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

	// --- M0-4: 세션/캐시(Redis) ---
	implementation 'org.springframework.boot:spring-boot-starter-data-redis'
	implementation 'org.springframework.session:spring-session-data-redis'

	// --- M0-5: Validation ---
	implementation 'org.springframework.boot:spring-boot-starter-validation'

	// --- 테스트 ---
	testImplementation 'org.springframework.boot:spring-boot-starter-test'
	testImplementation 'org.springframework.boot:spring-boot-testcontainers'
	testImplementation 'org.testcontainers:postgresql'
	testImplementation 'org.testcontainers:junit-jupiter'
	testRuntimeOnly    'org.junit.platform:junit-platform-launcher'
}

tasks.named('test') {
	useJUnitPlatform()
	finalizedBy tasks.named('jacocoTestReport')   // 테스트 끝나면 자동으로 커버리지 리포트 생성
	testLogging {
		events "passed", "skipped", "failed"
		showStandardStreams = false
		exceptionFormat = "full"

		afterTest { desc, result ->
			def ms = result.endTime - result.startTime
			print "\n${desc.className.tokenize('.').last()} > ${desc.name}: ${result.resultType} (${ms}ms)"
		}
	}
}

// bootJar 외에 실행 불가능한 plain jar가 같이 생기는 걸 막는다
tasks.named('jar') {
	enabled = false
}

// JaCoCo: 테스트 커버리지 측정 도구
jacoco {
	toolVersion = '0.8.12'  // JDK 17 호환 최신. 명시로 재현성 확보
}

// 커버리지 리포트 생성 규칙
tasks.named('jacocoTestReport') {
	dependsOn tasks.named('test')   // 리포트는 테스트 실행 결과(.exec)에 의존
	reports {
		xml.required = true   // CI 도구 연동용
		csv.required = true   // 배지 생성기가 csv를 읽음 → 반드시 켜야 함
		html.required = true  // 사람이 브라우저로 보는 리포트
	}
}
```

| 설정 | 의미 |
|---|---|
| `id 'jacoco'` | JaCoCo 플러그인 활성화 → `jacocoTestReport` 태스크 생성됨 |
| `finalizedBy` | `test`가 끝나면 항상 리포트 태스크를 이어서 실행 |
| `dependsOn test` | 리포트는 테스트 실행 데이터(`.exec`)에 의존 |
| `csv.required = true` | 배지 생성의 입력. 끄면 배지가 안 만들어짐 |
| `toolVersion = '0.8.12'` | JaCoCo 엔진 버전 고정(재현성) |

### 4-2. `.github/workflows/ci.yml`

```yaml
name: CI

permissions:
  contents: write   # 배지 SVG를 main에 커밋하기 위해

# 언제 돈다: main으로의 PR, 그리고 main에 push될 때
on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest        # Docker 내장 → Testcontainers 그대로 동작
    defaults:
      run:
        working-directory: backend   # gradlew가 backend/에 있으므로 run 기준 경로 고정

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'

      - name: Set up Gradle (의존성/빌드 캐시 자동)
        uses: gradle/actions/setup-gradle@v4

      - name: Make gradlew executable
        run: chmod +x gradlew        # Windows에서 커밋 시 실행권한이 빠질 수 있어 보정

      - name: Test with coverage
        run: ./gradlew test jacocoTestReport

      # 테스트 실패해도 리포트는 올려서 원인 확인 가능하게
      - name: Upload coverage report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: jacoco-report
          path: backend/build/reports/jacoco/test/html

      # 커버리지 csv를 읽어 배지 SVG 생성 (main에서만)
      - name: Generate coverage badge
        if: github.ref == 'refs/heads/main'
        uses: cicirello/jacoco-badge-generator@v2
        with:
          jacoco-csv-file: backend/build/reports/jacoco/test/jacocoTestReport.csv
          badges-directory: .github/badges
          generate-coverage-badge: true
          generate-branches-badge: true

      # 배지가 바뀌었으면 main에 되커밋
      - name: Commit coverage badge
        if: github.ref == 'refs/heads/main'
        working-directory: ${{ github.workspace }}   # 레포 루트에서 작업
        run: |
          if [ -n "$(git status --porcelain .github/badges)" ]; then
            git config user.name "github-actions[bot]"
            git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
            git add .github/badges
            git commit -m "docs: update coverage badge [skip ci]"
            git push
          else
            echo "Badge unchanged"
          fi
```

### 4-3. `README.md`

```markdown
# ixikey

PL·개발자용 문서 업무 보조 서비스.

![CI](https://github.com/opaqe/ixikey/actions/workflows/ci.yml/badge.svg)
![Coverage](./.github/badges/jacoco.svg)
![Branches](./.github/badges/branches.svg)

> 상세 문서는 [`docs/`](./docs) 참고. (README 본문은 M11-5에서 확장)
```

### 4-4. 빌드 JDK와 포맷터 — Spotless를 분리한 이유

코드 포맷 자동화(Spotless + google-java-format)는 M0-7에서 의도적으로 뺐다. google-java-format은 **1.29.0부터 실행에 JDK 21을 요구**하기 때문이다.

| 구분 | JDK |
|---|---|
| Gradle 데몬(포맷터 실행) | 최신 포맷터는 21 필요 |
| 앱 컴파일·테스트(toolchain) | 17 유지 |
| 최종 바이트코드/런타임 | 17 보장 |

해결책으로 **google-java-format 1.28.0(JDK 17 호환 마지막 버전)으로 핀**하면 빌드 전체를 17로 유지할 수 있다. 이 결정·적용은 별도 유닛으로 분리했다. M0-7(JaCoCo)은 이 JDK 제약과 무관하므로 17 그대로 진행했다.

---

## 5. CI 워크플로우 단계별 해설

| 순서 | step | 하는 일 |
|---|---|---|
| 1 | Checkout | 해당 커밋 코드를 러너 VM에 내려받음 |
| 2 | Set up JDK 17 | Temurin 17 설치 |
| 3 | Set up Gradle | `~/.gradle` 캐시 자동 관리(두 번째 실행부터 빨라짐) |
| 4 | Make gradlew executable | 리눅스 러너 실행권한 보정 |
| 5 | Test with coverage | `./gradlew test jacocoTestReport` — Testcontainers로 DB/Redis 띄우고 전체 테스트 + 리포트 생성 |
| 6 | Upload coverage report | HTML 리포트를 아티팩트로 보관(PR에서 다운로드 가능) |
| 7 | Generate coverage badge | csv → SVG 배지 생성(main 한정) |
| 8 | Commit coverage badge | 배지 변경 시 main에 되커밋(main 한정) |

### 5-1. 트리거 동작

| 동작 | 결과 |
|---|---|
| feature 브랜치에만 push | CI 안 돎(트리거가 main PR/push 한정) |
| main 대상 PR 생성 | `pull_request` 트리거 → CI 첫 실행 |
| PR 열린 상태에서 추가 push | synchronize 이벤트 → CI 재실행 |
| main에 반영(push) | `push` 트리거 → CI 실행 + 배지 생성 |

### 5-2. `[skip ci]`의 역할

배지 커밋이 다시 `push` 트리거를 발동시키면 **CI가 자기 자신을 무한 호출**한다. 커밋 메시지에 `[skip ci]`를 넣으면 GitHub Actions가 그 push의 워크플로우 실행을 건너뛰어 루프를 차단한다.

---

## 6. 자주 막히는 포인트

| 증상 | 원인 | 해결 |
|---|---|---|
| Docker 껐는데 테스트 "통과" | `test`가 UP-TO-DATE로 건너뜀(캐시) | `--rerun-tasks`로 강제 실행해 실제 검증 |
| 커버리지 배지가 깨져 보임 | 배지 SVG가 아직 main에 없음(첫 머지 전) | main 첫 실행 후 생성됨. 또는 이미지 캐시 → 새로고침 |
| 배지 커밋 단계 실패(403) | 레포 Actions 권한이 read-only | Settings → Actions → General → Workflow permissions → **Read and write** |
| 배지 csv 못 읽음 | `csv.required`가 false | build.gradle 리포트 설정에 `csv.required = true` |
| YAML 인식 실패 | 들여쓰기에 탭 사용 | 스페이스만 사용 |

---

## 7. 파일 변경 요약

| 파일 | 변경 |
|---|---|
| `backend/build.gradle` | jacoco 플러그인 + 리포트(xml/csv/html) + `finalizedBy` |
| `.github/workflows/ci.yml` | 신규: 빌드·테스트·커버리지·아티팩트·배지 |
| `README.md` | 신규: 배지 3종(CI/Coverage/Branches) |
| `.github/badges/jacoco.svg`, `branches.svg` | CI가 자동 생성·커밋 |

---

## 8. 다음 단계

| 항목 | 위치 |
|---|---|
| 코드 포맷 게이트(Spotless, google-java-format 1.28.0 핀) | 별도 유닛 |
| 커버리지 80% 게이트(미달 시 빌드 실패) | M11-6 |
| 자동 배포(CD), 무중단 배포(Blue-Green) | M9 |
