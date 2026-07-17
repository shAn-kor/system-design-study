# 7장 참고문헌: 분산 시스템을 위한 유일 ID 생성기 설계

책에서 다루는 UUID, Flickr Ticket Server, Twitter Snowflake, NTP를 원문과 공식 표준으로 다시 확인한 자료다.

## 1. UUID

- Link: [RFC 9562 - Universally Unique IDentifiers](https://www.rfc-editor.org/rfc/rfc9562.html)
- Source grade: standard
- Key evidence:
  - UUID는 128비트 식별자다.
  - 중앙 등록소 없이 각 노드가 독립적으로 생성할 수 있다.
  - UUIDv4는 122비트의 난수 영역을 사용한다.
  - UUIDv7은 Unix Epoch 기준 밀리초 시간을 앞부분에 배치해 시간순 정렬에 유리하다.
- Re-read location: RFC 9562의 [1. Introduction](https://www.rfc-editor.org/rfc/rfc9562.html#section-1), [5.4. UUID Version 4](https://www.rfc-editor.org/rfc/rfc9562.html#section-5.4), [5.7. UUID Version 7](https://www.rfc-editor.org/rfc/rfc9562.html#section-5.7)
- Why this source matters:
  - 중앙 ID 발급 서버가 없어도 생성할 수 있으므로 가용성과 확장성이 높다.
  - 128비트라서 64비트 정수보다 저장 공간과 인덱스 비용이 크다.
  - UUIDv4는 생성 순서가 키 정렬 순서에 반영되지 않지만 UUIDv7은 시간순 정렬이 필요한 설계의 대안이 된다.

## 2. Flickr Ticket Servers

- Link: [Ticket Servers: Distributed Unique Primary Keys on the Cheap](https://code.flickr.net/2010/02/08/ticket-servers-distributed-unique-primary-keys-on-the-cheap/)
- Source grade: official-blog
- Key evidence:
  - 전용 MySQL 서버의 단일 행에 `REPLACE INTO`를 실행하고 `LAST_INSERT_ID()`로 새 64비트 ID를 얻는다.
  - 두 Ticket Server가 홀수와 짝수 ID 공간을 나눠 사용한다.
  - 두 서버 사이에는 ID 발급을 위한 복제나 잠금이 없고, 요청은 round-robin으로 분산한다.
  - Flickr 전체에서 유일하고 순차적인 정수 키를 제공하지만 두 서버의 발급 진도는 서로 다를 수 있다.
- Re-read location: `REPLACE INTO`, `Putting It All Together`, `SPOFs` 절
- Why this source matters:
  - 구현은 단순하고 MySQL의 auto-increment를 활용할 수 있다.
  - ID 발급이 Ticket Server에 의존하므로 서버 장애와 처리량이 전체 시스템의 제약이 된다.
  - 홀수/짝수 분할은 두 서버 사이의 요청별 합의 없이 유일성을 확보하는 실제 사례다.

## 3. Twitter Snowflake

- Link: [Announcing Snowflake](https://blog.x.com/engineering/en_us/a/2010/announcing-snowflake)
- Source grade: official-blog
- Additional primary source: [Twitter Snowflake `IdWorker.scala`](https://github.com/twitter-archive/snowflake/blob/b3f6a3c6ca8e1b6847baa6ff42bf72201e2c2231/src/main/scala/com/twitter/service/snowflake/IdWorker.scala)
- Source grade: repository
- Key evidence:
  - 목표는 초당 수만 개의 ID, 높은 가용성, 대략적인 시간순 정렬, 64비트 크기였다.
  - ID는 `timestamp + datacenter ID + worker ID + sequence`로 구성한다.
  - 공개된 구현은 5비트 datacenter ID, 5비트 worker ID, 12비트 sequence를 사용한다. 나머지 41비트가 밀리초 timestamp 영역이며 최상위 1비트는 부호 비트로 남는다.
  - 같은 밀리초에는 sequence를 증가시키고 12비트를 모두 사용하면 다음 밀리초까지 기다린다.
  - 시스템 시간이 직전 발급 시각보다 뒤로 이동하면 ID 발급을 거부한다.
- Re-read location: X Engineering 글의 `The Problem`, `Solution` 절과 `IdWorker.scala`의 비트 상수, `nextId()` 구현
- Why this source matters:
  - ID 발급 요청마다 중앙 서버와 합의하지 않으므로 수평 확장과 높은 처리량에 유리하다.
  - 유일성을 보장하려면 datacenter ID와 worker ID 조합이 중복되지 않아야 한다.
  - 시간 기반 정렬은 시스템 시계에 의존하므로 clock rollback 감지와 시간 동기화 정책이 필수다.

```text
0 | timestamp 41bit | datacenter 5bit | worker 5bit | sequence 12bit
```

## 4. NTP

- Link: [RFC 5905 - Network Time Protocol Version 4](https://www.rfc-editor.org/rfc/rfc5905.html)
- Source grade: standard
- Key evidence:
  - NTPv4는 분산된 서버와 클라이언트의 시스템 시계를 동기화하는 프로토콜이다.
  - 클라이언트와 서버가 교환한 네 시각 `T1`, `T2`, `T3`, `T4`로 clock offset과 round-trip delay를 계산한다.
  - 여러 시간 소스 중 신뢰할 수 있는 후보를 선택하고 로컬 시계를 보정한다.
- Re-read location: RFC 5905의 [2. Modes of Operation](https://www.rfc-editor.org/rfc/rfc5905.html#section-2), [8. On-Wire Protocol](https://www.rfc-editor.org/rfc/rfc5905.html#section-8), [11. Clock Discipline Algorithm](https://www.rfc-editor.org/rfc/rfc5905.html#section-11)
- Why this source matters:
  - Snowflake처럼 timestamp를 ID에 포함하는 방식은 노드 간 시계 차이에 영향을 받는다.
  - NTP는 시계 오차를 줄이는 기반이지만 단조 증가 시각을 보장하지는 않는다.
  - 따라서 ID 생성기는 NTP 사용 여부와 별개로 clock rollback을 감지하고 발급 중단, 대기 또는 논리 시각 사용 정책을 가져야 한다.

## 비교 요약

| 방식 | 크기 | 중앙 조정 | 정렬성 | 주요 위험 |
|---|---:|---|---|---|
| UUIDv4 | 128비트 | 없음 | 없음 | 큰 인덱스, 난수 품질 |
| UUIDv7 | 128비트 | 없음 | 시간순 | 큰 인덱스, 시계 의존 |
| Flickr Ticket Server | 64비트 | 발급 서버 의존 | 순차적 | 발급 서버 장애와 병목 |
| Snowflake | 64비트 | 요청 시 없음 | 대략적인 시간순 | worker ID 충돌, clock rollback |
