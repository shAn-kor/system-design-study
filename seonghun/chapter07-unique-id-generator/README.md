# Chapter 07 - 분산 시스템을 위한 유일 ID 생성기 설계

책 7장의 참고문헌 4개를 설계 관점에서 다시 읽기 위한 문서 모음이다.

## 7장의 핵심 요구사항

- ID는 유일해야 한다.
- 숫자로만 구성한다.
- 64비트 안에 들어가야 한다.
- 생성 시간 순서로 대략 정렬할 수 있어야 한다.
- 초당 10,000개 이상 생성할 수 있어야 한다.

## 참고문헌별 문서

1. [UUID](01-uuid.md)
2. [Flickr Ticket Server](02-flickr-ticket-server.md)
3. [Twitter Snowflake](03-twitter-snowflake.md)
4. [Network Time Protocol](04-network-time-protocol.md)
5. [1차 출처 조사 기록](SOURCES.md)

## 빠른 비교

| 방식 | 중앙 조정 | 크기 | 시간 순서 | 핵심 문제 |
|---|---:|---:|---:|---|
| UUID | 불필요 | 128비트 | 버전에 따라 다름 | 7장의 64비트 숫자 조건을 만족하지 못함 |
| Ticket Server | 필요 | 64비트 가능 | 발급 서버 안에서는 순차적 | 중앙 발급 계층의 병목과 장애 처리 필요 |
| Snowflake | ID 발급 시 불필요 | 64비트 | 대략 보장 | worker ID 할당과 시계 역행 처리 필요 |
| NTP | 시간 동기화 계층 | 해당 없음 | 시계 오차를 줄임 | 단조 증가 시계를 보장하는 장치는 아님 |

## 읽는 순서

1. UUID로 조정 없는 ID 생성의 장점을 확인한다.
2. Ticket Server로 중앙 순번 발급의 장단점을 확인한다.
3. Snowflake가 시간, 노드, 순번을 한 ID에 조합한 이유를 확인한다.
4. NTP를 읽고 시간 기반 ID가 물리 시계에 의존할 때 생기는 문제를 확인한다.

## 결론

책의 요구사항에는 Snowflake 계열이 가장 잘 맞는다. 다만 ID 비트 배치만 구현하면 끝나는 것이 아니다. worker ID 중복, 같은 밀리초의 sequence 소진, 시계 역행, generator 장애를 함께 다뤄야 한다.
