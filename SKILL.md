---
name: "spring-backend-change-procedure"
description: "Spring Boot 백엔드 코드를 변경하거나 새 기능을 구현할 때 적용하는 표준 작업 절차. 실제 파일을 먼저 읽고, 스케줄러·비동기·이벤트 리스너까지 찾아본 뒤 부재를 판단하며, 변경 계획을 승인받고 영향 범위(호출부·DTO·Repository·API를 쓰는 클라이언트)를 함께 고친 다음 변경점을 요약한다. DDL 작성, 상태(enum) 목록 정의, API 사용 중단(deprecated) 처리 규칙을 포함한다. \"구현해줘\", \"변경해줘\", \"추가해줘\", \"수정해줘\", \"리팩토링\" 등 코드 변경 요청 시 사용하고, 특정 API를 비활성화하거나 사용 중단 처리할 때도 반드시 사용."
---

# Spring BE 코드 변경 표준 절차

## Step 1. 관련 코드 직접 확인
- 변경 전 반드시 관련 파일을 읽고 현재 상태를 확인한다.
- 추측하지 말고 실제 코드 기준으로만 판단한다.
- grep/find로 나온 파일 목록은 파일명과 무관하게 전부 읽는다.
- 파일명("Chat", "User", "Admin" 등)으로 내용을 추론해서 건너뛰지 않는다.
- Spring Boot에서는 도메인 경계와 무관한 파일에 핵심 로직이 있는 경우가 빈번하다. (예: `NotificationBatchService`에 주문 자동 구매확정 배치가 들어 있을 수 있음)
- "X가 없다", "X가 미구현이다" 등 부재를 주장하기 전에 반드시 아래를 실행한다
1. @Scheduled, @Async, @EventListener, @TransactionalEventListener 전체 grep
   → 예상치 못한 곳에서 같은 기능이 실행될 수 있다
2. 확인한 파일 목록을 명시하고,
   "이 목록 외에서 동일 기능이 발견될 가능성"을 스스로 검토한다
3. grep 결과 목록에서 아직 읽지 않은 파일이 있으면 읽기 전까지 부재 단정을 보류한다

## Step 2. 성능·설계 우려 선제 검토 (해당할 때만)
아래 항목에 해당하면 구현 전 사용자에게 먼저 알리고 방향을 확인한다:
- 기존 대비 DB 조회 수 증가 또는 N+1 발생 가능성
- 트랜잭션 범위·전파 방식 변경
- 캐시 정합성 문제
- 성능 저하가 예상되는 구조 변경

## Step 3. 변경 계획 사전 공유
구현에 앞서 아래 내용을 요약해서 사용자에게 먼저 보여주고 승인받는다:
- 수정할 파일 목록
- 파일별 핵심 변경 내용 (무엇을 왜 바꾸는지)

수정 파일이 5개 이상이거나 로직 흐름 전체에 영향을 주는 작업(리팩토링, 마이그레이션 등)은
단계(Phase)를 나눠 각 단계별 계획을 순차 승인받은 후 진행한다.

구현 중 계획에 없던 추가 변경이 필요하거나 바람직하다고 판단되면,
진행을 멈추고 해당 내용을 추가로 공유한 뒤 승인받는다.
승인 후 구현을 시작한다.

## Step 4. 영향 범위 파악 후 구현
- 변경 시 영향받는 파일(호출부, DTO, Repository 등)을 모두 함께 수정한다
- API의 요청·응답 형태가 바뀌면 그 API를 호출하는 클라이언트(앱·웹·어드민 등)도 영향 범위에 넣는다. 클라이언트 코드를 직접 고치지 않는 경우에도 어떤 화면·호출부가 영향을 받는지 Step 5 요약에 따로 적는다
- 상태 목록(enum 값을 골라 담은 List/Set)을 새로 만들거나 바꿀 때는 아래 「상태 목록(enum 집합) 정의 규칙」을 먼저 따른다
- 미사용 import는 함께 제거한다
- 변경 파일이 여러 개라면 순서를 명확히 하고 하나씩 처리한다

## Step 5. 변경점 비교 요약
모든 수정이 끝난 후 파일별로 핵심 변경 지점과 이유를 명확히 요약한다.
길이는 내용에 맞게 조절하되, 다음 두 가지는 반드시 포함한다:

- **무엇이 바뀌었는가**: 변경 전과 후의 핵심 차이를 구체적으로 서술한다
- **왜 바꿨는가**: 해당 변경이 필요했던 이유나 의도를 명확히 밝힌다

사소한 정리(import 제거 등)는 생략하고, 로직·구조·동작 방식이 달라진 지점에 집중한다.

---

## DB 스키마 변경(DDL) 규칙 — collation 미지정 원칙

새 테이블이나 컬럼을 만드는 DDL을 사용자에게 제공할 때는 **`CHARACTER SET` / `COLLATE`를 명시하지 않는다.** (MySQL/MariaDB 기준)

먼저 사용하는 스키마(운영·개발 모두)의 기본 collation을 확인하고, 그 값이 기존 테이블들과 같은지 본다.

```sql
SELECT SCHEMA_NAME, DEFAULT_CHARACTER_SET_NAME, DEFAULT_COLLATION_NAME
FROM information_schema.SCHEMATA
WHERE SCHEMA_NAME = DATABASE();
```

이유: DDL에 collation을 지정하지 않으면 새 오브젝트가 스키마 기본값을 자동 상속해 **항상 일치**한다. 반대로 collation을 손으로 적으면(예: 기본이 `utf8mb4_0900_ai_ci`인데 `utf8mb4_unicode_ci`를 적는 경우) 기존 테이블과 어긋나고, 문자열(UUID 등) 컬럼으로 JOIN·서브쿼리 비교 시 **`Illegal mix of collations`** 에러가 발생한다. 이 에러는 DB 클라이언트 툴뿐 아니라 **JPA/QueryDSL/native query 실행 시에도 동일하게** 발생한다. (실제로 일부 테이블이 다른 collation으로 만들어져, 나중에 전체 테이블 collation을 통일하는 작업이 필요했던 일이 있었다.)

규칙:
1. `CREATE TABLE` 끝에 `COLLATE` / `CHARACTER SET` 절을 붙이지 않는다.
2. 개별 컬럼에도 `CHARACTER SET` / `COLLATE`를 붙이지 않는다. 타입만 적는다 (`VARCHAR(36)`, `CHAR(36)`, `TEXT` 등).
3. 기존 DDL 파일에 다른 collation(`collate = utf8mb4_unicode_ci` 등)이 남아 있어도 그대로 복사해오지 않는다.
4. 새 스키마(`CREATE DATABASE`)를 만드는 경우도 동일하다. 굳이 명시할 거면 기존 스키마와 같은 값(예: `CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci`)으로 적는다.
5. `CREATE TABLE ... LIKE` / `AS SELECT`는 원본 collation을 복제하므로, 원본이 기본과 다르면 만든 직후 `ALTER TABLE ... CONVERT TO CHARACTER SET utf8mb4 COLLATE <스키마 기본 collation>`으로 맞춘다.

```sql
-- 나쁨: 기본과 다른 collation을 명시 → 문자열 조인 시 Illegal mix of collations
CREATE TABLE order_message (
  id CHAR(36) NOT NULL,
  order_id CHAR(36) NOT NULL,
  ...
  PRIMARY KEY (id)
) COMMENT '주문 문의 메시지' COLLATE = utf8mb4_unicode_ci;

-- 좋음: charset/collation을 아예 명시하지 않음 → 스키마 기본 상속
CREATE TABLE order_message (
  id CHAR(36) NOT NULL,
  order_id CHAR(36) NOT NULL,
  ...
  PRIMARY KEY (id)
) COMMENT '주문 문의 메시지';
```

점검(선택): 새 테이블 추가 후 아래가 0행이면 스키마 기본 collation과 일치하는 것이다.
```sql
SET @base := (SELECT DEFAULT_COLLATION_NAME FROM information_schema.SCHEMATA
              WHERE SCHEMA_NAME = DATABASE());

SELECT t.table_name, c.column_name, c.collation_name
FROM information_schema.tables t
LEFT JOIN information_schema.columns c
  ON c.table_schema = t.table_schema AND c.table_name = t.table_name
 AND c.collation_name IS NOT NULL AND c.collation_name <> @base
WHERE t.table_schema = DATABASE() AND t.table_type = 'BASE TABLE'
  AND (t.table_collation <> @base OR c.column_name IS NOT NULL);
```

---

## 상태 목록(enum 집합) 정의 규칙 — 같은 규칙은 한 곳에만

「이 상태들 중 하나면 ~한다」에 쓰는 상태 목록(예: `List.of(PENDING, PAID, …)`)을 만들거나 고칠 때 적용한다.

배경: 「주문을 수정할 수 있는 상태」가 서비스 두 곳과 배치 한 곳에 `private` 상수로 복사돼 있었다. 「재주문 차단」은 자바 메서드와 SQL 목록 여러 개가 서로 다른 값이었고, 「아직 결제 전」은 자바 상수와 SQL 글자로 따로 있었다. 기능이 다른 시기에 만들어지면서 각자 목록을 들고 시작했고, 기획이 「같은 범위」라고 정한 뒤에도 합치지 않았다. 정책 하나를 바꾸는 데 커밋 여러 개가 필요했고, 한 곳만 고치면 기능끼리 어긋날 수 있었다.

규칙:
1. **만들기 전에 enum부터 확인한다.** 해당 enum(예: `OrderStatus`)의 공개 상수와 Javadoc을 읽고, 같은 뜻의 상수(예: `EDITABLE`, `CANCELLABLE`, `BLOCKS_REORDER`)가 있으면 그걸 쓴다. 이어서 `grep`으로 같은 값 조합(`List.of(…`, `Set.of(…`, `IN ('…`)이 다른 파일에 있는지도 찾는다.
2. **두 군데 이상에서 쓰거나, 기획이 같은 규칙이라고 정했으면 enum의 `public static final` 상수로 만든다.** 서비스 · 배치 클래스 안에 `private static final List<…>`로 복사하지 않는다. 배치 등 다른 모듈도 공통 모듈의 enum을 참조할 수 있다.
3. **SQL에 상태 값을 글자로 쓰지 않는다.** `IN (:xxxStatuses)` 파라미터로 받고, 넘기는 쪽에서 enum 상수의 `name()` 목록을 만든다(`STATUSES.stream().map(Enum::name).toList()`).
4. **자바 판단 메서드와 SQL이 같은 판단이면 같은 상수를 본다.** 예: `Order.isReorderBlocked()`은 `OrderStatus.BLOCKS_REORDER.contains(status)`, 목록 SQL은 같은 상수를 `:blockingStatuses`로. 엔티티 메서드 안에 상태 비교를 따로 늘어놓지 않는다.
5. **값이 같아도 뜻이 다르면 합치지 않는다.** 예: 주문 목록의 쿠폰 표시 조건이 「아직 결제 전」과 값이 같더라도 다른 규칙이면 별도 상수(`COUPON_DISPLAY_STATUSES`)로 둔다. 상수 Javadoc에 **무엇에 쓰는지 · 쓰는 기능 목록 · 무엇과는 다른지**를 적는다.
6. **이름은 값이 아니라 뜻으로 짓는다.** `BEFORE_SHIPPING`(값 설명)이 아니라 `EDITABLE`(규칙의 뜻)처럼, 정책이 바뀌어 값이 달라져도 이름이 틀린 말이 되지 않게 한다.
7. **나눌 이유가 사라지면 그때 합친다.** 시간 조건(예: 유효기간) 때문에 자바와 SQL을 따로 둘 수밖에 없던 경우, 그 조건이 없어지는 작업에서 함께 합친다. 기획이 여러 기능을 「같은 범위」로 묶는 결정을 하면 그 작업 안에서 합친다.
8. **상수의 값을 바꾸는 변경은 쓰는 기능 전체를 확인한다.** Javadoc의 기능 목록(또는 `grep` 참조 결과)을 따라 한 번씩 확인하고, 변경 요약에 「같이 바뀌는 기능」을 적는다. SQL 쪽 집합에 값을 더할 때는 결과가 달라지는 데이터가 있는지 읽기 전용 SQL로 먼저 확인한다.

```java
// 나쁨: 같은 규칙을 클래스마다 복사, SQL은 글자로
private static final List<OrderStatus> EDITABLE_STATUSES =
    List.of(OrderStatus.PENDING, OrderStatus.PAID);   // 서비스 A
private static final List<OrderStatus> MODIFIABLE =
    List.of(OrderStatus.PENDING, OrderStatus.PAID);   // 배치 B
sql.append("AND o.status IN ('PENDING', 'PAID', 'PREPARING') ");

// 좋음: enum에 뜻으로 이름 붙인 공개 상수 하나 + 모두 참조, SQL은 파라미터
public enum OrderStatus {
  ...;
  /** 구매자가 주문 내용을 수정할 수 있는 상태. 쓰는 곳: 주문 수정 · 배송지 변경 · 수정 알림 배치. */
  public static final List<OrderStatus> EDITABLE =
      List.of(PENDING, PAID, PREPARING);
}
sql.append("AND o.status IN (:editableStatuses) ");
params.put("editableStatuses", OrderStatus.EDITABLE.stream().map(Enum::name).toList());
```

---

## Deprecated API 처리 패턴

특정 API를 더 이상 사용하지 않기로 결정했을 때 적용하는 표준 패턴이다.
각 메서드에 직접 예외를 삽입하는 방식 대신, **인터셉터 + URL 패턴 등록** 방식을 사용한다.
이렇게 하면 컨트롤러를 건드리지 않고도 비활성화할 수 있고, 향후 추가/해제도 한 파일에서 관리된다.

### 1. 공통 인프라 확인 또는 생성

프로젝트에 아래 항목이 이미 있는지 먼저 확인한다. 없으면 생성한다.
패키지 경로·에러 응답 클래스는 프로젝트에 이미 있는 것에 맞춘다. (아래 `com.example`, `ErrorResponse`는 예시)
멀티 모듈이면 예외·핸들러는 공통 모듈, 인터셉터·설정은 API 모듈에 둔다. 단일 모듈이면 같은 프로젝트에 둔다.

**`DeprecatedApiException`** (공통 모듈)
```java
package com.example.shared.exception;

public class DeprecatedApiException extends RuntimeException {
  public DeprecatedApiException() {
    super("지원이 종료된 기능입니다.");
  }
}
```

**`GlobalExceptionHandler`에 핸들러 추가** (공통 모듈)
```java
@ExceptionHandler(DeprecatedApiException.class)
public ResponseEntity<Object> handleDeprecatedApi(DeprecatedApiException ex) {
  ErrorResponse errorResponse = ErrorResponse.builder()
      .error("DEPRECATED_API")
      .message(ex.getMessage())
      .httpStatus(HttpStatus.GONE)
      .timestamp(ZonedDateTime.now())
      .build();
  return new ResponseEntity<>(errorResponse, HttpStatus.GONE);
}
```

**`DeprecatedApiInterceptor`** (API 모듈)
```java
@Component
public class DeprecatedApiInterceptor implements HandlerInterceptor {
  @Override
  public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
    throw new DeprecatedApiException();
  }
}
```

**`DeprecatedApiWebConfig`** (API 모듈)
```java
@Configuration
@RequiredArgsConstructor
public class DeprecatedApiWebConfig implements WebMvcConfigurer {
  private final DeprecatedApiInterceptor deprecatedApiInterceptor;

  @Override
  public void addInterceptors(InterceptorRegistry registry) {
    registry.addInterceptor(deprecatedApiInterceptor)
        .addPathPatterns(
            "/api/v1/some-feature/**",   // 비활성화할 URL 패턴 목록
            "/api/v2/another-feature/**"
        );
  }
}
```

### 2. 새 deprecated API 추가 시

`DeprecatedApiWebConfig`의 `addPathPatterns`에 URL 패턴 한 줄만 추가한다. 컨트롤러는 건드리지 않는다.

### 3. 컨트롤러 어노테이션 처리

인터셉터로 요청은 막히지만, 코드를 보는 다른 개발자를 위해 아래 어노테이션을 컨트롤러 클래스 레벨에 추가한다.

```java
@Deprecated           // IDE에서 취소선으로 표시
@Hidden               // Swagger(springdoc)에서 숨김
@Tag(name = "...", description = "지원이 종료된 API입니다.")
public class SomeController {
```

### 4. STOMP(WebSocket) 엔드포인트 처리

`@MessageMapping` 메서드는 HTTP 인터셉터가 적용되지 않는다.
해당 메서드에만 `@Deprecated` 어노테이션과 함께 직접 throw를 넣는다.

```java
@MessageMapping("/some-topic/{id}")
@Deprecated
public SomeMessage sendMessage(...) {
  throw new DeprecatedApiException();
}
```

### 5. 기존 방식(메서드 직접 수정)으로 처리된 deprecated API가 있다면

`DeprecatedApiWebConfig` 패턴으로 통합하는 것을 고려한다.
단, 엔드포인트 전체가 아니라 **특정 파라미터 조건에서만** 막는 경우(예: `legacyMode=true`일 때만 차단)는 인터셉터로 통합이 불가능하므로 메서드 내부 분기에 `DeprecatedApiException`을 throw하는 방식을 유지한다.
