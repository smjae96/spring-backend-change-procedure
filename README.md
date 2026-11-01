# spring-backend-change-procedure

> Spring Boot 백엔드 코드를 바꿀 때 Claude가 따르는 표준 작업 절차 스킬

한 스타트업의 운영 중인 Spring Boot 서비스에서 실제로 쓰던 스킬을, 회사·서비스 정보를 모두 걷어내고 어느 프로젝트에나 쓸 수 있게 일반화했습니다.

## 왜 만들었나

- AI가 파일 이름만 보고 "그런 배치는 없다"고 단정했는데, 실제로는 이름과 무관한 서비스 클래스의 `@Scheduled` 메서드에 들어 있는 일이 반복됐다.
- 계획 공유 없이 여러 파일을 한꺼번에 고쳐, 어디까지 바뀌었는지 리뷰하기 어려웠다.
- DDL에 collation을 손으로 적어 `Illegal mix of collations` 에러가 나고, 같은 상태 목록이 여러 클래스와 SQL에 복사돼 정책 하나 바꾸는 데 커밋이 여러 개 필요했다.

## 언제 쓰나

- "이 기능 구현해줘"
- "주문 취소 로직 수정해줘"
- "이 서비스 리팩토링해줘"
- "이 API 이제 안 쓰니까 막아줘"
- "새 테이블 DDL 만들어줘"

## 무엇을 하나

- 변경 전 관련 파일을 전부 직접 읽는다. "없다"고 말하기 전에 `@Scheduled`·`@Async`·`@EventListener`·`@TransactionalEventListener`를 전체 grep하고, 읽지 않은 파일이 남아 있으면 단정을 보류한다.
- N+1, 트랜잭션 범위, 캐시 정합성 등 성능·설계 우려가 있으면 구현 전에 먼저 알린다.
- 수정할 파일과 파일별 변경 내용을 먼저 보여주고 승인받는다. 5개 이상이거나 흐름 전체에 영향이 있으면 단계를 나눈다.
- 호출부·DTO·Repository와, API 형태가 바뀌면 그 API를 쓰는 클라이언트까지 영향 범위에 넣는다.
- 끝나면 파일별로 "무엇이 바뀌었나 / 왜 바꿨나"를 요약한다.
- DDL은 collation을 명시하지 않고 스키마 기본값을 상속시킨다.
- 상태(enum) 목록은 enum의 공개 상수 한 곳에만 두고, SQL에는 파라미터로 넘긴다.
- 사용 중단 API는 컨트롤러를 고치지 않고 인터셉터 + URL 패턴 등록으로 410 GONE을 반환한다.

## 출력 예시

변경 계획 공유(Step 3):

```text
수정할 파일
1. OrderService.java — 취소 가능 여부 판단을 OrderStatus.CANCELLABLE 상수로 교체
2. OrderQueryRepository.java — 상태 IN 절을 :cancellableStatuses 파라미터로 변경
3. OrderStatus.java — CANCELLABLE 공개 상수 추가(Javadoc에 쓰는 기능 목록)

이 계획으로 진행해도 될까요?
```

## 설치

Claude Code

```bash
mkdir -p ~/.claude/skills/spring-backend-change-procedure
curl -L https://raw.githubusercontent.com/smjae96/spring-backend-change-procedure/main/SKILL.md -o ~/.claude/skills/spring-backend-change-procedure/SKILL.md
```

Claude 앱(claude.ai): 이 레포 폴더를 zip으로 묶어 설정 > 기능 > 스킬에서 업로드

DDL 규칙은 MySQL/MariaDB 기준입니다. 다른 DB를 쓰면 해당 절을 지우거나 바꿔서 쓰세요.

## 함께 쓰면 좋은 스킬

- [api-change-spec](https://github.com/smjae96/api-change-spec) — 변경을 마친 뒤 프론트엔드에 전달할 API 변경점 명세를 만드는 스킬
- [user-flow-qa-scenario](https://github.com/smjae96/user-flow-qa-scenario) — 변경을 마친 뒤 비개발 직원용 QA 시나리오와 개발자 체크리스트를 만드는 스킬
- 모음 레포: https://github.com/smjae96/claude-skills

## License

MIT
