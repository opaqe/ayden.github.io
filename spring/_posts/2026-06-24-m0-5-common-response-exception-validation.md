---
layout: post
title: "M0-5: 공통 응답·예외·검증 처리 (Spring Boot 3.5 / Java 17)"
date: 2026-06-24 10:00:00 +0900
tags: [spring-boot, exception-handling, validation, mockmvc, rest-api]
description: "컨트롤러마다 제각각이던 응답·에러 처리를 ApiResponse 표준 래퍼와 전역 예외 핸들러로 통일하고, @Valid 검증까지 더해 모든 API가 같은 모양의 성공/실패 응답을 반환하도록 계약(contract)을 고정한 과정."
---

> 모든 API가 **같은 모양의 성공/실패 응답**을 돌려주도록 표준을 만드는 단계.
> 컨트롤러마다 제각각이던 응답·에러 처리를 한 곳으로 모아, 이후 모든 모듈이 따르는 "계약(contract)"을 고정한다.

---

## 1. 목표와 완료 기준

| 항목 | 내용 |
|---|---|
| 목표 | 모든 API가 동일한 형태의 응답/에러를 반환 |
| 완료 기준(DoD) | 임의 컨트롤러에서 일관된 성공/실패 응답이 나옴 |
| 테스트 | 정상 / 검증 실패 / 도메인 예외 3케이스 (MockMvc) |
| 선행 의존 | M0-1 (Spring Boot 스캐폴드) |

---

## 2. 익혀야 할 핵심 개념

| 개념 | 한 줄 설명 |
|---|---|
| 표준 응답 래퍼 | 성공이든 실패든 JSON 모양을 하나로 통일하는 객체 (`ApiResponse<T>`) |
| `@RestControllerAdvice` | 모든 컨트롤러의 예외를 한 곳에서 가로채는 전역 핸들러 |
| `@ExceptionHandler` | "이 종류 예외가 오면 이 메서드가 처리"를 지정 |
| Bean Validation | `@NotBlank` 등 제약을 붙이고 `@Valid`로 자동 검증 |
| 에러 코드 체계 | 에러 종류를 enum 한 곳에 모아 (코드·HTTP상태·메시지) 관리 |
| Jackson 직렬화 | Java 객체 → JSON 자동 변환 (`@JsonInclude`로 null 필드 제외) |
| 도메인 예외 | 업무 규칙 위반을 표현하는 커스텀 예외 (`BusinessException`) |

---

## 3. 전체 구조 (파일 지도)

| 파일 | 위치 | 역할 |
|---|---|---|
| `ApiResponse.java` | `backend/src/main/java/com/ixikey/common/response/` | 표준 응답 래퍼 |
| `ErrorCode.java` | `backend/src/main/java/com/ixikey/common/error/` | 에러 코드 enum |
| `BusinessException.java` | `backend/src/main/java/com/ixikey/common/error/` | 도메인 예외 |
| `GlobalExceptionHandler.java` | `backend/src/main/java/com/ixikey/common/error/` | 전역 예외 처리 |
| `CommonResponseTest.java` | `backend/src/test/java/com/ixikey/common/` | 검증 테스트 |
| `build.gradle` | `backend/` | validation 의존성 추가 |

데이터 흐름:

```
요청 → 컨트롤러(@Valid 검증)
        ├─ 정상            → ApiResponse.ok(data)          → JSON
        ├─ 검증 실패        ┐
        ├─ 도메인 예외       ├→ GlobalExceptionHandler      → ApiResponse.error(...) → JSON
        └─ 예상 못한 예외    ┘
```

---

## 4. 구성요소 상세

### 4-1. 표준 응답 래퍼 `ApiResponse<T>`

`backend/src/main/java/com/ixikey/common/response/ApiResponse.java`

```java
package com.ixikey.common.response;

import com.fasterxml.jackson.annotation.JsonInclude;
import java.time.Instant;

@JsonInclude(JsonInclude.Include.NON_NULL)
public record ApiResponse<T>(
        boolean success,   // 성공 여부
        String code,        // 에러 코드 (실패일 때만)
        String message,     // 설명 메시지
        T data,             // 실제 데이터 (성공일 때만)
        Instant timestamp   // 응답 생성 시각
) {
    public static <T> ApiResponse<T> ok(T data) {
        return new ApiResponse<>(true, null, null, data, Instant.now());
    }

    public static <T> ApiResponse<T> ok() {
        return new ApiResponse<>(true, null, null, null, Instant.now());
    }

    public static <T> ApiResponse<T> error(String code, String message) {
        return new ApiResponse<>(false, code, message, null, Instant.now());
    }
}
```

| 포인트 | 설명 |
|---|---|
| `record` | 필드만 담는 불변 객체를 짧게 정의하는 Java 16+ 문법 |
| `<T>` 제네릭 | `data`에 어떤 타입이든 담을 수 있게 함 (문자열·DTO 등) |
| `@JsonInclude(NON_NULL)` | null인 필드는 JSON에서 제외 → 성공 응답엔 `code/message` 안 보이고, 실패 응답엔 `data` 안 보임 |
| 정적 팩토리 | `new` 대신 `ok()` / `error()`로 의미가 분명한 생성 |

### 4-2. 에러 코드 enum `ErrorCode`

`backend/src/main/java/com/ixikey/common/error/ErrorCode.java`

```java
package com.ixikey.common.error;

import org.springframework.http.HttpStatus;

public enum ErrorCode {
    INVALID_INPUT(HttpStatus.BAD_REQUEST, "입력값이 올바르지 않습니다."),          // 400
    NOT_FOUND(HttpStatus.NOT_FOUND, "요청한 리소스를 찾을 수 없습니다."),          // 404
    INTERNAL_ERROR(HttpStatus.INTERNAL_SERVER_ERROR, "서버 오류가 발생했습니다."); // 500

    private final HttpStatus httpStatus;
    private final String defaultMessage;

    ErrorCode(HttpStatus httpStatus, String defaultMessage) {
        this.httpStatus = httpStatus;
        this.defaultMessage = defaultMessage;
    }

    public HttpStatus getHttpStatus() {
        return httpStatus;
    }

    public String getDefaultMessage() {
        return defaultMessage;
    }

    public String getCode() {
        return name();   // enum 이름 문자열 반환. 예: "INVALID_INPUT"
    }
}
```

| 코드 | HTTP 상태 | 기본 메시지 |
|---|---|---|
| `INVALID_INPUT` | 400 | 입력값이 올바르지 않습니다. |
| `NOT_FOUND` | 404 | 요청한 리소스를 찾을 수 없습니다. |
| `INTERNAL_ERROR` | 500 | 서버 오류가 발생했습니다. |

> 새 에러가 필요하면 이 enum에 한 줄만 추가하면 된다. (코드·상태·메시지를 한 묶음으로 관리)

### 4-3. 도메인 예외 `BusinessException`

`backend/src/main/java/com/ixikey/common/error/BusinessException.java`

```java
package com.ixikey.common.error;

public class BusinessException extends RuntimeException {

    private final ErrorCode errorCode;

    public BusinessException(ErrorCode errorCode) {
        super(errorCode.getDefaultMessage());
        this.errorCode = errorCode;
    }

    public BusinessException(ErrorCode errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }

    public ErrorCode getErrorCode() {
        return errorCode;
    }
}
```

| 포인트 | 설명 |
|---|---|
| `extends RuntimeException` | unchecked 예외 → try-catch 강제 안 함 |
| `ErrorCode` 보유 | 예외가 "어떤 에러인지(코드·상태·메시지)"를 들고 다님 |
| 사용법 | 서비스에서 `throw new BusinessException(ErrorCode.NOT_FOUND)` |
| 생성자 2개 | 기본 메시지 사용 / 상황별 메시지 직접 지정 |

### 4-4. 전역 예외 처리 `GlobalExceptionHandler`

`backend/src/main/java/com/ixikey/common/error/GlobalExceptionHandler.java`

```java
package com.ixikey.common.error;

import com.ixikey.common.response.ApiResponse;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.util.stream.Collectors;

@RestControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    // [1] @Valid 검증 실패
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiResponse<Void>> handleValidation(MethodArgumentNotValidException ex) {
        String message = ex.getBindingResult().getFieldErrors().stream()
                .map(this::formatFieldError)
                .collect(Collectors.joining(", "));
        ErrorCode code = ErrorCode.INVALID_INPUT;
        return ResponseEntity.status(code.getHttpStatus().value())
                .body(ApiResponse.error(code.getCode(), message));
    }

    // [2] 도메인 예외
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ApiResponse<Void>> handleBusiness(BusinessException ex) {
        ErrorCode code = ex.getErrorCode();
        return ResponseEntity.status(code.getHttpStatus().value())
                .body(ApiResponse.error(code.getCode(), ex.getMessage()));
    }

    // [3] 예상 못한 모든 예외 (최후 방어선)
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiResponse<Void>> handleUnexpected(Exception ex) {
        ErrorCode code = ErrorCode.INTERNAL_ERROR;
        log.error("처리되지 않은 예외", ex);   // 내부 정보는 로그에만
        return ResponseEntity.status(code.getHttpStatus().value())
                .body(ApiResponse.error(code.getCode(), code.getDefaultMessage()));
    }

    private String formatFieldError(FieldError error) {
        return error.getField() + ": " + error.getDefaultMessage();
    }
}
```

| 분기 | 잡는 예외 | 결과 | 비고 |
|---|---|---|---|
| [1] 검증 실패 | `MethodArgumentNotValidException` | 400 `INVALID_INPUT` | 틀린 필드명+메시지를 모아 반환 |
| [2] 도메인 예외 | `BusinessException` | 예외가 든 `ErrorCode` 대로 | 코드·상태를 예외에서 그대로 사용 |
| [3] 그 외 전부 | `Exception` | 500 `INTERNAL_ERROR` | 스택트레이스는 로그에만, 응답엔 일반 메시지 |

### 4-5. Validation 의존성

`backend/build.gradle` (dependencies 블록)

```groovy
// --- M0-5: Validation ---
implementation 'org.springframework.boot:spring-boot-starter-validation'
```

> 이 의존성이 있어야 `jakarta.validation.*`(`@NotBlank`, `@Valid` 등)이 동작한다.
> 없으면 import 자체가 풀리지 않고, `@Valid`가 무시된다.

---

## 5. 입력 검증(@Valid) 패턴

| 단계 | 코드 | 의미 |
|---|---|---|
| 1. DTO에 제약 | `record EchoRequest(@NotBlank String message)` | null/빈문자열/공백이면 위반 |
| 2. 파라미터에 `@Valid` | `echo(@Valid @RequestBody EchoRequest req)` | 들어온 본문을 검사하라 |
| 3. 위반 시 | Spring이 `MethodArgumentNotValidException` 발생 | 핸들러 [1]이 받아 400 응답 |

자주 쓰는 제약 애너테이션:

| 애너테이션 | 검사 내용 |
|---|---|
| `@NotNull` | null 아님 |
| `@NotBlank` | null·빈문자열·공백 아님 (문자열용) |
| `@NotEmpty` | null·빈 컬렉션 아님 |
| `@Size(min, max)` | 길이/크기 범위 |
| `@Email` | 이메일 형식 |

---

## 6. 테스트

`backend/src/test/java/com/ixikey/common/CommonResponseTest.java`

```java
package com.ixikey.common;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.ixikey.common.error.BusinessException;
import com.ixikey.common.error.ErrorCode;
import com.ixikey.common.error.GlobalExceptionHandler;
import com.ixikey.common.response.ApiResponse;
import jakarta.validation.Valid;
import jakarta.validation.constraints.NotBlank;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.setup.MockMvcBuilders;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

class CommonResponseTest {

    private final ObjectMapper objectMapper = new ObjectMapper();
    private MockMvc mockMvc;

    @BeforeEach
    void setUp() {
        mockMvc = MockMvcBuilders.standaloneSetup(new TestController())
                .setControllerAdvice(new GlobalExceptionHandler())
                .build();
    }

    @Test
    void 정상_요청은_success_true() throws Exception {
        String body = objectMapper.writeValueAsString(new EchoRequest("hello"));
        mockMvc.perform(post("/test/echo")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(body))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.success").value(true))
                .andExpect(jsonPath("$.data").value("hello"));
    }

    @Test
    void 검증_실패는_INVALID_INPUT_400() throws Exception {
        String body = objectMapper.writeValueAsString(new EchoRequest(""));
        mockMvc.perform(post("/test/echo")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(body))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.success").value(false))
                .andExpect(jsonPath("$.code").value("INVALID_INPUT"));
    }

    @Test
    void 도메인_예외는_해당_코드와_status() throws Exception {
        mockMvc.perform(post("/test/boom")
                        .contentType(MediaType.APPLICATION_JSON))
                .andExpect(status().isNotFound())
                .andExpect(jsonPath("$.success").value(false))
                .andExpect(jsonPath("$.code").value("NOT_FOUND"));
    }

    // --- 테스트 전용 더미 컨트롤러/요청 객체 (실코드에는 두지 않음) ---
    @RestController
    static class TestController {
        @PostMapping("/test/echo")
        ApiResponse<String> echo(@Valid @RequestBody EchoRequest request) {
            return ApiResponse.ok(request.message());
        }

        @PostMapping("/test/boom")
        ApiResponse<Void> boom() {
            throw new BusinessException(ErrorCode.NOT_FOUND);
        }
    }

    record EchoRequest(@NotBlank String message) {
    }
}
```

테스트 케이스 요약:

| 테스트 | 입력 | 기대 상태 | 기대 응답 |
|---|---|---|---|
| 정상 | `{"message":"hello"}` | 200 | `success=true`, `data="hello"` |
| 검증 실패 | `{"message":""}` | 400 | `success=false`, `code="INVALID_INPUT"` |
| 도메인 예외 | `/test/boom` 호출 | 404 | `success=false`, `code="NOT_FOUND"` |

핵심 도구:

| 도구 | 역할 |
|---|---|
| `MockMvcBuilders.standaloneSetup` | DB/Redis 없이 웹 계층만 띄우는 미니 환경 |
| `.setControllerAdvice(...)` | 전역 예외 핸들러를 함께 등록해야 예외가 잡힘 |
| `ObjectMapper` | 요청 본문을 객체 → JSON 문자열로 변환 |
| `jsonPath("$.필드")` | 응답 JSON의 특정 필드 값 검증 |

---

## 7. 의사결정 근거

| 결정 | 선택 | 이유 | 대안(미채택) |
|---|---|---|---|
| 응답 모양 | `ApiResponse<T>` 단일 래퍼 | 성공/실패 형태 통일, 이후 전 모듈의 계약 | 컨트롤러별 자유 응답 → 일관성 깨짐 |
| 응답 스키마 시점 | M0-5에서 고정 | 후속 모듈이 모두 의존하므로 일찍 굳혀야 함 | 나중에 변경 → 광범위 수정 발생 |
| 예외 처리 위치 | `@RestControllerAdvice` 전역 | try-catch 중복 제거, 한 곳에서 통일 | 컨트롤러마다 try-catch |
| 에러 코드 | enum 한 곳 집중 | 코드·상태·메시지 묶음 관리, 추가 용이 | 문자열 상수 흩뿌리기 |
| 500 응답 | 내부 메시지/스택 비노출 | 보안 — 내부 구조 유출 방지 (로그로만) | 예외 메시지 그대로 노출 |
| 테스트 방식 | `standaloneSetup` + 더미 컨트롤러 | 컨텍스트·Testcontainers 불필요, 빠름 / 실코드 오염 방지 | `@SpringBootTest` 전체 기동 |
| 모킹 | Mockito 미사용 (Fake/더미) | 프로젝트 규칙 — 실객체·수작업 Fake 선호 | Mockito mock |
| `ResponseEntity` 상태 | `getHttpStatus().value()` (int) | IDE annotation null 분석 false positive 회피 | `HttpStatus` 직접 전달 |

---

## 8. 응답 예시

성공:

```json
{
  "success": true,
  "data": "hello",
  "timestamp": "2026-06-24T02:00:00Z"
}
```

검증 실패:

```json
{
  "success": false,
  "code": "INVALID_INPUT",
  "message": "message: 공백일 수 없습니다",
  "timestamp": "2026-06-24T02:00:00Z"
}
```

> `@JsonInclude(NON_NULL)` 덕분에 성공엔 `code/message`가, 실패엔 `data`가 빠진다.

---

## 9. 마무리 체크리스트

| 항목 | 상태 |
|---|---|
| `ApiResponse<T>` 표준 래퍼 | ✅ |
| `ErrorCode` enum (400/404/500) | ✅ |
| `BusinessException` 도메인 예외 | ✅ |
| `GlobalExceptionHandler` 3분기 | ✅ |
| `@Valid` 검증 패턴 + validation 의존성 | ✅ |
| 테스트 3종 (정상/검증실패/도메인예외) | ✅ |

**다음 단계**: M0-6 Docker Compose 로컬 스택
