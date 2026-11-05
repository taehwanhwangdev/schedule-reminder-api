# Spec

## 1. 시간 계산 (DST 포함)

### 규칙

- 입력: `startsAtLocal`(오프셋 없는 현지 시각, 예: "2026-03-08T02:30") + `timeZone`(IANA 이름, 예: "America/New_York")
  - 오프셋이 붙은 시각, 유효하지 않은 시간대 이름은 400
- 저장: 등록 시 UTC로 변환해 저장. 원래 입력(현지 시각·시간대)도 함께 저장
- 존재하지 않는 시각(DST 시작 구간): 전환 전 오프셋으로 해석한다 → 벽시계 기준 구간 길이만큼 뒤로 밀림 (02:30 → 03:30)
- 두 번 있는 시각(DST 종료 구간): 먼저 오는 쪽(전환 전 오프셋)을 쓴다. 알림은 늦는 것보다 이른 것이 안전하다
- 위 두 규칙은 Temporal의 기본 동작(`compatible`)과 같다
- 등록 응답에 해석된 현지 시각과 오프셋을 포함한다 (예: 입력 02:30 → 응답 "2026-03-08T03:30-04:00")
- 알림 시점: `remindBeforeMinutes`(예: [1440, 60]) — UTC 기준 절대 시간으로 뺀다
  - DST를 넘으면 현지 시각 기준으로 1시간 차이가 날 수 있다(의도된 동작, README에 명시)
- 과거 검증: 일정 시각이 이미 지났으면 400. 알림 시각만 이미 지났으면 그 알림은 만들지 않는다
- 발송 대상 판정: `notifyAt <= now` (경계 포함). 허용 지연·조회 주기는 2절에서 정한다

### 테스트 케이스 (단위)

| #   | 입력                                               | 기대값                                                    |
| --- | -------------------------------------------------- | --------------------------------------------------------- |
| T1  | Asia/Seoul 2026-11-20T10:00, 60분 전               | 일정 2026-11-20T01:00Z, 알림 00:00Z                       |
| T2  | America/New_York 2026-07-01T09:00 (EDT)            | 2026-07-01T13:00Z                                         |
| T3  | America/New_York 2026-03-08T02:30 (존재하지 않음)  | 2026-03-08T07:30Z, 응답 현지 시각 03:30-04:00             |
| T4  | America/New_York 2026-11-01T01:30 (두 번 있음)     | 먼저 오는 EDT = 2026-11-01T05:30Z (EST였다면 06:30Z)      |
| T5  | America/New_York 2026-03-08T10:00 (EDT), 1440분 전 | 2026-03-07T14:00Z = 현지 09:00 EST (현지 시각 1시간 차이) |
| T6  | 시간대 "Mars/Base"                                 | 400                                                       |
| T7  | 일정 시각이 과거                                   | 400                                                       |
| T8  | 일정 30분 뒤, 알림 [60, 10]                        | 10분 전 알림만 생성                                       |
| T9  | notifyAt == now                                    | 발송 대상                                                 |

## 2. 중복·누락

### 보장 수준

- 최소 1회 발송(at-least-once). 정확히 1회는 보장하지 않는다(외부 발송과 DB 갱신을 하나로 묶을 수 없음)
- 중복이 생길 수 있는 경우: 발송 성공 후 SENT 기록 전에 실행이 중단된 경우. lease 만료(최대 5분) 뒤 재발송될 수 있다
- 동시 실행·중복 호출(EventBridge 포함)로 인한 중복 발송은 선점 조건부 쓰기로 막는다

### 데이터

- 일정 1건 + 알림 시점별 알림 항목 N건(등록 시 생성, 알림 시점은 최대 5개)
- 알림 항목: `notifyAt`(UTC), `status`, `attempts`, `leaseUntil`, `leaseOwner`
- 상태: PENDING → PROCESSING → SENT / 실패 시 PENDING 복귀 또는 FAILED / EXPIRED / CANCELED
- 발송 대기 GSI(희소 인덱스): PENDING·PROCESSING 항목에만 `pendingKey`를 두고, 정렬 키는 `notifyAt`. SENT·FAILED·EXPIRED·CANCELED가 되면 `pendingKey`를 지운다

### 스케줄러 (1분 주기)

1. GSI 조회: `notifyAt <= now`, 페이지 단위(50건)로 반복. Lambda 제한 시간 여유가 부족하면 중단하고 다음 실행에 넘긴다
2. 건별 처리(한 건의 실패가 다른 건에 영향을 주지 않는다)
   - `notifyAt < now - 30분` → EXPIRED (조건: PENDING), 로그 남김
   - 선점: 조건 `status = PENDING` 또는 `(status = PROCESSING 이고 leaseUntil < now)` → PROCESSING, `leaseUntil = now + 5분`, `leaseOwner = 실행 ID`, `attempts + 1`
     - 조건 실패 = 다른 실행이 처리 중 → 건너뛴다
   - 발송
   - 성공 → SENT (조건: `leaseOwner = 내 실행 ID`)
   - 실패 → `attempts < 3`이면 PENDING 복귀, 아니면 FAILED (같은 조건)
3. 발송 여부는 GSI가 아니라 기본 테이블 조건부 쓰기(강한 일관성)로 판단한다. GSI 반영 지연으로 늦게 보인 건은 다음 실행에서 처리된다

### 삭제

- 일정 삭제 + 알림 항목 취소를 한 트랜잭션으로 처리한다: PENDING → CANCELED
- PROCESSING인 알림이 있으면 트랜잭션 실패 → 409 (발송 진행 중, 잠시 후 재시도)
- SENT·FAILED·EXPIRED 알림은 그대로 두고 삭제한다

### 테스트 케이스 (통합, DynamoDB Local)

| #   | 상황                                          | 기대값                                      |
| --- | --------------------------------------------- | ------------------------------------------- |
| I1  | 같은 알림을 두 실행이 동시에 처리             | 발송 1회, 한쪽은 선점 실패로 건너뜀         |
| I2  | 선점 후 발송 전에 중단(lease 남음)            | 다른 실행이 건너뜀                          |
| I3  | 선점 후 중단, lease 만료                      | 다음 실행이 재선점해 1회 발송               |
| I4  | lease 만료 후 이전 실행이 늦게 SENT 기록 시도 | `leaseOwner` 불일치로 거부                  |
| I5  | 3건 중 1건 발송 실패                          | 나머지 2건 SENT, 실패 건 PENDING·attempts 1 |
| I6  | 3회 연속 실패                                 | FAILED, GSI에서 빠짐                        |
| I7  | `notifyAt`이 31분 지난 PENDING                | EXPIRED, 발송 안 함                         |
| I8  | 도래한 알림 120건(페이지 50)                  | 120건 모두 처리                             |
| I9  | 삭제 시 PENDING 알림                          | CANCELED, 이후 발송 안 함                   |
| I10 | 삭제 시 PROCESSING 알림                       | 409, 아무것도 바뀌지 않음                   |

- GSI 반영 지연은 DynamoDB Local에서 재현하기 어렵다 → 판단을 기본 테이블 조건부 쓰기로 하는 설계로 대응하고, 이 한계를 README에 명시한다

## 3. 데이터 모델·API (최소안)

### 테이블 (단일 테이블)

| 항목 | PK                      | SK                |
| ---- | ----------------------- | ----------------- |
| 일정 | `SCHEDULE#<scheduleId>` | `META`            |
| 알림 | `SCHEDULE#<scheduleId>` | `NOTIF#<minutes>` |

- 일정과 그 알림을 같은 PK에 둬서, 한 번의 Query로 일정 + 알림 상태를 함께 읽는다(GET /schedules/{id})
- 모든 UTC 시각은 `toISOString()` 형식(`YYYY-MM-DDTHH:mm:ss.sssZ`)으로 고정한다. 형식이 같아야 문자열 비교 결과가 시간 순서와 일치한다(GSI 정렬·조건식에서 사용)

#### 일정 항목 (SK = `META`)

| 속성                | 타입   | 설명                                                                  | 예시                         |
| ------------------- | ------ | --------------------------------------------------------------------- | ---------------------------- |
| scheduleId          | string | 일정 ID(UUID), PK에서도 쓰지만 응답용으로 저장                        | `"3f2a..."`                  |
| title               | string | 일정 제목, 1~100자                                                    | `"치과 예약"`                |
| startsAtLocal       | string | 사용자가 입력한 현지 시각(오프셋 없음), 원본 보존용                   | `"2026-03-08T02:30"`         |
| timeZone            | string | IANA 시간대 이름                                                      | `"America/New_York"`         |
| startsAtUtc         | string | DST 규칙을 적용해 변환한 일정 시각(UTC)                               | `"2026-03-08T07:30:00.000Z"` |
| resolvedLocal       | string | 실제로 해석된 현지 시각 + 오프셋(존재하지 않는 시각 보정 결과 확인용) | `"2026-03-08T03:30-04:00"`   |
| remindBeforeMinutes | number | 일정 몇 분 전에 알릴지, 정수 1~10080                                  | `60`                         |
| createdAt           | string | 생성 시각(UTC)                                                        | `"2026-11-05T01:00:00.000Z"` |

#### 알림 항목 (SK = `NOTIF#<minutes>`)

| 속성        | 타입   | 설명                                                                                                                    | 예시                         |
| ----------- | ------ | ----------------------------------------------------------------------------------------------------------------------- | ---------------------------- |
| notifyAt    | string | 발송 예정 시각(UTC) = startsAtUtc − remindBeforeMinutes. GSI 정렬 키                                                    | `"2026-03-08T06:30:00.000Z"` |
| status      | string | PENDING / PROCESSING / SENT / FAILED / EXPIRED / CANCELED                                                               | `"PENDING"`                  |
| pendingKey  | string | 발송 대기 GSI 파티션 키. 값은 상수 `"PENDING"`, status가 PENDING·PROCESSING일 때만 존재하고 그 외에는 제거(희소 인덱스) | `"PENDING"`                  |
| attempts    | number | 선점 횟수. 선점할 때마다 +1, 3회를 넘기면 FAILED                                                                        | `1`                          |
| leaseUntil  | string | 선점 만료 시각(UTC) = 선점 시각 + 5분. 이 시각이 지나면 다른 실행이 재선점 가능                                         | `"2026-03-08T06:35:00.000Z"` |
| leaseOwner  | string | 선점한 실행의 ID(Lambda 요청 ID). SENT·실패 기록 시 조건으로 써서, lease가 끝난 이전 실행의 기록을 막는다               | `"c1d9..."`                  |
| title       | string | 발송 메시지용 복사본(스케줄러가 일정 항목을 다시 읽지 않도록)                                                           | `"치과 예약"`                |
| startsAtUtc | string | 발송 메시지용 복사본                                                                                                    | `"2026-03-08T07:30:00.000Z"` |
| timeZone    | string | 발송 메시지에 현지 시각을 표시하기 위한 복사본                                                                          | `"America/New_York"`         |
| updatedAt   | string | 마지막 상태 변경 시각(UTC)                                                                                              | `"2026-03-08T06:30:05.000Z"` |

- 생성 시: status = PENDING, pendingKey = "PENDING", attempts = 0, leaseUntil·leaseOwner 없음
- 알림 시각이 이미 지난 경우 알림 항목을 만들지 않는다

### API

| 메서드 | 경로            | 성공                                                               | 실패 |
| ------ | --------------- | ------------------------------------------------------------------ | ---- |
| POST   | /schedules      | 201 `{ scheduleId, startsAtUtc, resolvedLocal, notifyAt \| null }` | 400  |
| GET    | /schedules/{id} | 200 `{ 일정 + 알림 상태 }`                                         | 404  |
| DELETE | /schedules/{id} | 204                                                                | 404  |

- POST 본문: `{ title, startsAtLocal, timeZone, remindBeforeMinutes }`
- 검증(400): 필수값 누락, title 1~100자, 오프셋 포함 시각, 유효하지 않은 시간대, 과거 일정, remindBeforeMinutes 정수 1~10080
- 오류 형식: `{ "error": { "code": "...", "message": "..." } }`
- 목록 조회는 범위 밖(사용자 구분·인증이 필요해짐)

### 삭제 (단순 취소)

- 알림을 조건부로 취소(PENDING → CANCELED, pendingKey 제거)한 뒤 일정 항목 삭제. 순서를 이렇게 해서 중간에 실패하면 다시 시도할 수 있게 둔다
- 이미 PROCESSING인 알림은 발송될 수 있다(최대 lease 5분). README에 명시

### 발송·인프라

- 발송: Slack Incoming Webhook 1개. 발송부는 인터페이스(`Notifier`)로 분리. URL은 로컬 `.env`(커밋 금지), 배포 시 SSM Parameter Store
- 인프라: AWS CDK(TypeScript). 로컬 개발·테스트는 핸들러 직접 호출 + DynamoDB Local
- 배포 시 공개 API 남용 방지: API Gateway 사용 계획(API 키 + 낮은 스로틀)은 2주차 배포 때 결정
