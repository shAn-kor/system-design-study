# Snowflake와 NTP만으로 분산 사건 순서를 보장할 수 있을까

## 먼저 답부터

Snowflake와 NTP를 함께 사용해도 여러 서버에서 일어난 사건의 엄격한 전역 순서는 보장되지 않는다.

- NTP는 각 서버의 시계 오차를 줄인다.
- Snowflake는 분산 환경에서 ID의 유일성과 대략적인 시간 정렬을 제공한다.
- 주문별 순서, 사건의 인과관계, 시스템 전체의 전역 총순서는 별도의 문제다.

이 글에서 말하는 Snowflake는 데이터 웨어하우스 제품이 아니라 분산 ID 생성 방식이다.

주문 서버 A가 사건을 먼저 만들고 결제 서버 B가 나중에 만들었다고 하자. 두 서버의 시계가 조금만 달라도 B의 timestamp가 더 작을 수 있다. Snowflake ID를 숫자순으로 정렬하면 실제 발생 순서와 어긋난다. NTP는 이 오차를 줄일 뿐 없애지는 못한다.

순서에는 두 가지 뜻이 있다. **실제 발생 순서**는 현실의 선후다. **합의된 처리 순서**는 시스템이 모두가 따를 순서를 정한 결과다. 합의 복제 로그는 두 번째를 보장한다.

## 1. Snowflake와 NTP가 보장하는 범위

### NTP

NTP(Network Time Protocol)는 서버가 시간 기준과 자기 시계를 비교해 오차를 줄이도록 돕는 프로토콜이다. 날짜와 시각을 표시하는 시스템 시계를 `wall clock`이라고 한다. NTP는 클라이언트와 시간 소스가 기록한 시각값(timestamp)으로 시계 차이(clock offset)와 네트워크 왕복 지연을 추정한다.

하지만 다음은 보장하지 않는다.

- 모든 서버의 시계가 완전히 같아지는 것
- 시계가 절대 감소하지 않는 것
- 여러 서버의 사건이 실제 발생 순서대로 정렬되는 것
- Snowflake ID의 유일성

### Snowflake

Snowflake는 중앙 데이터베이스에서 번호를 하나씩 받지 않고 각 서버가 직접 ID를 만드는 방식이다. 일반적인 Snowflake ID는 다음 값을 조합한다.

```text
timestamp + workerId + sequence
```

- `timestamp`: ID를 만든 시각에 가까운 값
- `workerId`: ID를 발급한 서버나 프로세스의 번호
- `sequence`: 같은 시간 단위에 한 worker가 만든 여러 ID의 일련번호

worker ID가 겹치지 않고 sequence를 올바르게 관리하면 ID 중복을 막을 수 있다. 시스템 시계가 이전 값보다 작아지는 현상을 clock rollback이라고 한다. rollback이 발생하면 시계가 따라잡을 때까지 기다리거나 ID 발급을 실패시키는 방어도 ID 생성기가 맡는다.

Snowflake가 제공하는 것은 엄격한 전역 순서가 아니라 **대략적인 시간순 정렬**이다.

## 2. 엄격한 시간·사건 순서를 보장하지 못하는 이유

### 서버별 시계 오차가 남는다

NTP로 보정한 뒤에도 서버 A와 B의 wall clock 값에는 작은 차이가 남을 수 있다. 실제로 A의 사건이 먼저 발생했더라도 B의 timestamp가 더 작게 기록될 수 있다.

### 네트워크 지연은 일정하지 않다

메시지 전달 시간은 요청마다 다르다. 요청과 응답 경로의 지연도 서로 다를 수 있다. 관측한 wall clock 값만으로는 실제 사건의 선후나 원인 관계를 확정할 수 없다.

### ID 숫자 순서와 실제 사건 순서는 다르다

Snowflake ID에는 worker와 sequence가 포함된다. 두 서버가 독립적으로 ID를 만들기 때문에 숫자가 작은 ID가 반드시 먼저 일어난 사건이라는 뜻은 아니다.

### 시간상 먼저라고 원인인 것은 아니다

사건 X의 timestamp가 Y보다 작아도 X가 Y를 발생시켰다고 결론 내릴 수 없다. 두 사건이 서로 메시지를 주고받지 않았다면 인과적으로 무관한 **동시 발생 관계(concurrent)** 일 수 있다. 여기서 concurrent는 두 사건이 같은 순간에 일어났다는 뜻이 아니다. 한 사건이 다른 사건의 원인이라고 판단할 근거가 없다는 뜻이다.

## 3. 필요한 순서 보장을 먼저 구분한다

| 요구 | 의미 | 예시 |
|---|---|---|
| 전역 총순서(total order) | 모든 참여자가 모든 사건을 동일한 하나의 순서로 처리 | 하나의 전역 원장, 합의된 명령 로그 |
| 인과 순서(causal order) | 원인 사건이 결과 사건보다 먼저임을 보존 | 주문 생성 → 결제 승인 |
| 같은 엔터티(aggregate) 내부 순서 | 주문 한 건처럼 함께 일관성을 지킬 상태의 변경만 순서대로 처리 | 같은 주문의 상태 version 7 → 8 |
| 대략적 시간 정렬 | 화면·검색·관측에서 실제 시각에 가깝게 정렬 | 로그 타임라인, 최신순 목록 |
| 유일성 | 서로 다른 사건이나 엔터티가 같은 ID를 갖지 않음 | `eventId`, `orderId` |

유일한 ID가 있다고 순서가 생기지는 않는다. 순서가 있다고 인과관계가 증명되는 것도 아니다.

### 논리 시계는 무엇을 더해 주나

물리 시각만으로 인과관계를 알 수 없을 때는 논리 시계를 쓴다.

`Lamport clock`은 사건이 발생하거나 메시지를 주고받을 때 숫자를 증가시킨다. X가 Y의 원인이면 X의 값은 Y보다 작다. 반대로 X의 값이 작다는 사실만으로 X가 원인이라고 단정할 수는 없다.

`Vector clock`은 노드별 카운터를 묶어서 기록한다. 두 값을 성분별로 비교하면 인과관계와 concurrent 관계를 구분할 수 있다. 대신 참여 노드가 늘수록 저장 공간과 전송량이 커진다.

`HLC(Hybrid Logical Clock)`는 wall clock에 가까운 물리 시각과 논리 카운터(logical counter)를 결합한다. timestamp 크기는 고정되어 있지만 HLC 값만으로 concurrent 여부를 판정하거나 여러 노드의 순서를 합의할 수는 없다.

## 4. 요구별 대안과 비용

| 방식 | 제공하는 보장 | 추가 지연 | 가용성 영향 | 저장·운영 비용 | 주요 한계 |
|---|---|---:|---|---|---|
| Snowflake | 전역 유일 ID, 대략적 시간 정렬 | 로컬 생성 | 별도 합의 없음 | worker ID와 rollback 관리 | 엄격한 전역 순서·인과관계 없음 |
| 주문별 sequence/version | 한 aggregate 내부 순서 | 저장소 갱신 | 저장소에 의존 | 충돌 재시도와 version 관리 | 서로 다른 aggregate 사이 순서 없음 |
| `causationId` | 직접 원인 링크 | 추가 왕복 없음 | 별도 합의 없음 | 원인 이벤트 보관·조회 필요 | concurrent 사건 전체 판정은 못 함 |
| Lamport clock | 인과관계가 있으면 논리 시각 증가 | 추가 왕복 없음 | 별도 합의 없음 | 고정 크기 | 값의 대소만으로 인과관계 역추론·concurrent 판정 불가 |
| Vector clock | 인과관계와 concurrent 관계 판정 | 메시지 크기 증가 | 별도 합의 없음 | 노드 수에 비례한 메타데이터와 참여 노드 목록 관리 | 동적 대규모 노드에서 비쌈 |
| HLC | 실제 시각에 가까운 정렬과 인과 방향 | 추가 왕복 없음 | 별도 합의 없음 | 고정 크기, 시계 상태 관리 | concurrent 판정·합의된 전역 총순서 보장 불가 |
| 단일 순서 발급기(sequencer) | 한 지점이 부여한 전역 순서 | sequencer 왕복 | sequencer 장애에 취약 | 장애 조치 설계 필요 | 병목과 단일 장애점 |
| DB sequence | 번호 할당 순서 | DB 왕복 | DB에 의존 | DB 운영과 병목 관리 | 번호 할당 순서가 트랜잭션 확정 순서·실제 발생 순서와 같지는 않음 |
| 합의 복제 로그 | 참여 노드가 합의한 전역 처리 순서 | 리더·과반수 왕복과 디스크 기록 | 과반수 노드 필요 | 리더 선출·복제·복구 비용 높음 | 실제 발생 시각을 알아내는 것이 아니라 처리 순서를 새로 확정함 |

### 같은 주문의 엄격한 상태 순서

주문별 `sequence` 또는 `aggregateVersion`을 사용한다. 숫자 필드만 추가해서는 동시 쓰기 충돌을 막지 못한다. 이전 version이 그대로일 때만 상태와 다음 version을 함께 저장해야 한다. 이를 조건부 원자 갱신 또는 compare-and-set(CAS)이라고 부른다.

```sql
UPDATE orders
SET state = :next_state,
    version = 8
WHERE order_id = 'order-10'
  AND version = 7;
```

version 7을 읽은 두 요청이 동시에 실행돼도 먼저 도착한 요청만 한 행을 바꾼다. 나중 요청은 변경된 행이 0개이므로 충돌을 감지하고 다시 읽거나 실패 처리한다.

### 도메인을 넘은 인과관계

각 사건은 자기 ID와 직접 원인의 ID를 구분해 기록한다.

```yaml
eventId: e-payment-92
aggregateId: order-10
aggregateSequence: 8
causationId: e-order-37
correlationId: checkout-202
occurredAt: 2026-07-25T10:30:00.123+09:00
hlc: [1753407000123, 1]
```

- `eventId`: 현재 사건의 전역 고유 ID
- `aggregateSequence`: 같은 주문 내부의 상태 변경 순서
- `causationId`: 현재 사건을 직접 발생시킨 입력 사건
- `correlationId`: 하나의 업무 흐름을 묶는 식별자
- `occurredAt` 또는 HLC: 대략적 시간 정렬

`eventId`와 `causationId`를 뒤바꾸면 안 된다. 현재 결제 이벤트는 `e-payment-92`, 원인이 된 주문 이벤트는 `e-order-37`이다. HLC의 두 번째 값 `1`은 같은 물리 시각 안의 순서를 구분하는 논리 카운터다.

### 모든 서버의 전역 총순서

장애를 견디면서 모든 참여자가 같은 순서를 따라야 한다면 Raft 같은 합의 프로토콜로 복제 로그를 운영한다. Raft에서는 리더가 로그 위치를 정하고 과반수(quorum) 노드가 저장한 항목을 확정한다.

```text
요청 → 리더가 로그 위치 부여 → 복제 노드 전송
     → 과반수 저장 확인 → 항목 확정 → 동일한 순서로 적용
```

이 방식은 실제 세계에서 어느 사건이 먼저 일어났는지를 시계로 알아내는 기술이 아니다. 시스템이 모든 사건에 하나의 **합의된 처리 순서**를 부여하는 기술이다.

그 대가로 다음 비용이 생긴다.

- 과반수까지의 네트워크 왕복과 디스크 기록으로 지연 증가
- 망이 분리되어 과반수와 통신하지 못하는 쪽의 쓰기 중단
- 리더 선출, 로그 복제, 스냅샷, 복구 운영
- 하나의 순서로 모으는 데 따른 처리량 병목

## 5. 선택 기준

요구사항을 아래 순서로 좁히면 불필요한 전역 합의를 피할 수 있다.

1. ID가 겹치지만 않으면 되는가?
   - Snowflake 같은 분산 ID 생성기를 사용한다.
2. 같은 주문의 상태 변경만 순서대로 처리하면 되는가?
   - 주문별 version과 조건부 원자 갱신을 사용한다.
3. 주문·결제·배송 사이의 원인 흐름을 추적하면 되는가?
   - `eventId`, `causationId`, 필요하면 `correlationId`를 사용한다.
4. 인과관계와 concurrent 여부를 판정해야 하는가?
   - 노드 규모와 메타데이터 비용을 감수할 수 있으면 Vector clock을 검토한다.
5. 실제 시각에 가까운 정렬과 인과 방향이 함께 필요한가?
   - HLC를 검토한다.
6. 모든 사건을 동일한 하나의 순서로 확정해야 하는가?
   - 단일 순서 발급기, DB sequence, 합의 복제 로그의 장애·지연·운영 비용을 비교한다.

## 근거

### 주 학습 자료가 다루는 범위

- NTP의 clock offset과 round-trip delay
- step과 slew
- NTP 보정 후에도 남는 서버별 시각 차이
- Snowflake의 rollback 검사, worker ID, sequence
- Snowflake ID의 대략적 정렬과 엄격한 전체 순서의 한계

### 외부 1차 자료로 보강한 범위

- [RFC 5905: Network Time Protocol Version 4](https://www.rfc-editor.org/rfc/rfc5905)
- [Twitter Snowflake 원본 구현](https://github.com/twitter-archive/snowflake/blob/b3f6a3c6ca8e1b6847baa6ff42bf72201e2c2231/src/main/scala/com/twitter/service/snowflake/IdWorker.scala)
- [Raft 합의 복제 로그](https://raft.github.io/raft.pdf)
- [Lamport: Time, Clocks, and the Ordering of Events](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/12/Time-Clocks-and-the-Ordering-of-Events-in-a-Distributed-System.pdf)
- [Mattern: Virtual Time and Global States of Distributed Systems](https://vs.inf.ethz.ch/publ/papers/VirtTimeGlobStates.pdf)
- [Hybrid Logical Clock](https://cse.buffalo.edu/~demirbas/publications/hlc.pdf)
