# Twitter Snowflake

## 참고문헌

- [Announcing Snowflake](https://blog.x.com/engineering/en_us/a/2010/announcing-snowflake)
- [twitter-archive/snowflake](https://github.com/twitter-archive/snowflake)
- [2010 IdWorker.scala](https://github.com/twitter-archive/snowflake/blob/snowflake-2010/src/main/scala/com/twitter/service/snowflake/IdWorker.scala)

## 해결하려던 문제

Twitter는 Cassandra와 수평 분할된 MySQL에서 사용할 ID가 필요했다.

- 초당 수만 개의 ID를 생성해야 했다.
- 높은 가용성을 위해 ID 생성 때마다 중앙 조정을 피해야 했다.
- 트윗을 시간순으로 다루기 위해 ID가 대략 정렬돼야 했다.
- 기존 시스템과의 호환을 위해 64비트 안에 들어가야 했다.

Snowflake는 타임스탬프, 데이터센터 ID, worker ID, sequence를 한 숫자에 조합한다.

## 64비트 구조

```text
0 | timestamp 41비트 | datacenter 5비트 | worker 5비트 | sequence 12비트
```

| 영역 | 크기 | 역할 |
|---|---:|---|
| sign | 1비트 | 양수 범위를 사용하기 위해 0으로 유지 |
| timestamp | 41비트 | 기준 epoch 이후 지난 밀리초 |
| datacenter | 5비트 | 최대 32개 데이터센터 구분 |
| worker | 5비트 | 데이터센터마다 최대 32개 worker 구분 |
| sequence | 12비트 | 같은 worker가 같은 밀리초에 만든 ID 구분 |

41비트 밀리초는 약 69.7년을 표현한다. 12비트 sequence는 worker 하나가 같은 밀리초에 4,096개 ID를 만들 수 있다는 뜻이다.

## ID 생성식

공개된 2010년 구현은 다음 값을 bit shift와 OR 연산으로 결합한다.

```text
((timestamp - epoch) << 22)
| (datacenterId << 17)
| (workerId << 12)
| sequence
```

예를 들어 서로 다른 worker가 같은 밀리초에 `sequence = 0`을 사용하더라도 worker 비트가 다르므로 최종 ID는 겹치지 않는다.

## 같은 밀리초에 요청이 몰리면

1. 이전 ID와 현재 시각이 같은지 확인한다.
2. 같으면 sequence를 1 증가시킨다.
3. 12비트를 모두 사용해 sequence가 다시 0이 되면 다음 밀리초까지 기다린다.
4. 밀리초가 바뀌면 sequence를 0으로 초기화한다.

## 왜 대략적인 시간 순서가 만들어지는가

timestamp가 ID의 상위 비트에 있다. 시간이 증가하면 하위의 datacenter, worker, sequence 값보다 timestamp 변화가 더 큰 숫자 차이를 만든다.

다만 여러 서버의 시계가 완전히 같지 않으므로 전 세계 모든 ID가 실제 생성 순서와 정확히 일치한다고 보장할 수는 없다.

## 시계가 뒤로 가면

공개된 `IdWorker.scala`는 현재 timestamp가 마지막 timestamp보다 작으면 ID 생성을 거부하고 오류를 발생시킨다.

```text
Clock moved backwards. Refusing to generate id.
```

그대로 생성하면 과거에 사용했던 timestamp, worker ID, sequence 조합을 다시 사용할 수 있기 때문이다.

가능한 운영 정책은 다음과 같다.

- 짧은 역행이면 마지막 timestamp까지 기다린다.
- 긴 역행이면 해당 worker를 비정상 상태로 전환한다.
- 프로세스 재시작 후에도 마지막 timestamp를 안전하게 관리한다.
- NTP 상태와 clock offset을 모니터링한다.
- 서로 다른 프로세스에 같은 worker ID를 중복 할당하지 않는다.

## 장점

- 중앙 DB에서 순번을 발급받지 않는다.
- 숫자형 64비트 ID를 만든다.
- 높은 처리량을 낼 수 있다.
- ID만 보고 대략적인 생성 시간을 추정할 수 있다.

## 한계

- worker ID 할당 체계가 필요하다.
- 물리 시계가 역행하면 ID 생성이 중단될 수 있다.
- 비트 수를 한 번 정하면 데이터센터 수, worker 수, 초당 처리량, 수명 사이의 비율이 고정된다.
- 2010년 공개 저장소는 현재 archive 상태이며 최초 구현은 퇴역했다.

## 핵심 정리

Snowflake의 핵심은 분산 노드가 서로 통신하지 않고도 ID를 만들 수 있도록 충돌 가능성을 비트 영역으로 미리 분리하는 것이다. 시간은 정렬을, worker ID는 노드 구분을, sequence는 같은 밀리초 안의 중복 방지를 담당한다.
