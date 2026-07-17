# Flickr Ticket Server

## 참고문헌

- [Ticket Servers: Distributed Unique Primary Keys on the Cheap](https://code.flickr.net/2010/02/08/ticket-servers-distributed-unique-primary-keys-on-the-cheap/)

## 해결하려던 문제

Flickr는 데이터를 여러 MySQL shard에 나눠 저장했다. 데이터가 shard 사이에서 이동할 수 있으므로 기본 키는 특정 데이터베이스 안에서만 유일해서는 안 되고 Flickr 전체에서 유일해야 했다.

각 shard의 `auto_increment`는 물리적으로 분리된 데이터베이스 전체의 유일성을 보장하지 못한다. Flickr는 ID 발급 책임을 전용 MySQL 서버로 모았다.

## 기본 구조

Ticket Server에는 `Tickets32`, `Tickets64`처럼 ID 크기별 테이블이 있다. `Tickets64`는 실제 데이터 대신 한 행의 `stub`만 유지한다.

```sql
CREATE TABLE Tickets64 (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  stub CHAR(1) NOT NULL DEFAULT '',
  PRIMARY KEY (id),
  UNIQUE KEY stub (stub)
) ENGINE=InnoDB;
```

새 ID가 필요할 때 같은 `stub` 값을 `REPLACE`하고 MySQL이 새로 발급한 auto-increment 값을 읽는다.

```sql
REPLACE INTO Tickets64 (stub) VALUES ('a');
SELECT LAST_INSERT_ID();
```

테이블은 한 행을 유지하지만 `REPLACE INTO`가 기존 행을 교체하면서 새로운 ID를 발급한다.

## 두 서버로 나누는 방법

한 Ticket Server만 사용하면 단일 장애 지점이 된다. Flickr는 두 서버가 서로 다른 ID 공간을 사용하도록 홀수와 짝수를 나눴다.

```text
TicketServer1
auto-increment-increment = 2
auto-increment-offset = 1

TicketServer2
auto-increment-increment = 2
auto-increment-offset = 2
```

- 서버 1은 `1, 3, 5, ...`를 발급한다.
- 서버 2는 `2, 4, 6, ...`을 발급한다.
- 두 서버 사이에 ID 발급 상태를 복제하거나 잠글 필요가 없다.
- 요청은 두 서버에 round-robin으로 분산할 수 있다.

## 장점

- 구현이 단순하다.
- 숫자형 64비트 ID를 만들 수 있다.
- 중앙에서 발급하므로 전체 시스템에서 유일성을 설명하기 쉽다.
- 대체로 증가하는 ID라 운영, 보고, 디버깅이 편하다.

## 한계

- 모든 ID 발급 요청이 Ticket Server 계층에 의존한다.
- 서버 수를 늘릴 때 offset과 increment 전략을 다시 설계해야 한다.
- 두 서버의 발급 속도가 다르면 전체 ID가 생성 시간 순서대로 엄격하게 정렬되지 않는다.
- ID 발급 계층 장애가 서비스 쓰기 전체로 전파될 수 있다.
- 여러 리전에 배치하면 네트워크 지연과 리전 간 ID 공간 분할 문제가 생긴다.

## 7장에서의 판단

Ticket Server는 작은 규모나 중간 규모에서는 실용적이다. 하지만 7장은 높은 가용성과 확장성을 함께 요구한다. 중앙 조정 지점을 없애고도 64비트와 대략적인 시간 순서를 만족하는 Snowflake 방식이 최종안에 더 가깝다.

## 설계 시 질문

- ID 발급 서버 장애 시 쓰기를 중단할 것인가?
- ID 공간을 홀수와 짝수보다 더 많은 서버로 어떻게 분할할 것인가?
- 리전별로 ID 범위를 나눌 것인가?
- ID가 중간에 건너뛰어도 되는가?
- 엄격한 생성 순서가 필요한가?

## 핵심 정리

Ticket Server의 핵심은 실제 데이터를 중앙화하는 것이 아니라 숫자 발급만 중앙화하는 것이다. 단순하고 이해하기 쉽지만 ID 생성 경로에 중앙 의존성이 생긴다.
