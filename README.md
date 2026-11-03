# schedule-reminder-api

- 목적: 일정을 등록하면 지정한 시점(예: 24시간 전, 1시간 전)에 알림을 보내는 서버리스 API. 알림 누락·중복 발송·시간대 문제를 테스트로 검증하는 데 집중한다.
- 범위: 일정 등록·조회·삭제 API, 주기 실행 스케줄러, 발송 채널 1개 (TypeScript · AWS Lambda · EventBridge · DynamoDB)
- 범위 밖: 화면(웹·앱), SMS, 다중 채널

> 진행 중 (2026-11 착수)
