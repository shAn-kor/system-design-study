# chapter05-week1-inital-design.md

## 1. 일반적인 해시 문제점

일반적인 해시 방식은 `hash(key) % 서버 수`처럼 서버 수를 계산식에 직접 넣는다. 이 방식은 단순하지만 서버가 추가되거나 삭제되면 서버 수가 바뀌기 때문에, 기존 키 대부분의 저장 위치가 다시 계산된다.

즉 서버 한 대를 추가했을 뿐인데도 전체 데이터가 다시 섞일 수 있다.

## 2. 안정 해시와 해시 링

안정 해시는 서버 수를 직접 나머지 연산에 넣지 않는다. 대신 키와 서버를 같은 해시 공간에 올리고, 그 해시 공간을 원형 링으로 본다.

해시 링은 특정 해시 함수로 만들어진 일렬의 해시 공간을 끝과 시작이 이어진 원형 구조로 생각한 것이다. 서버도 해시해서 링 위에 배치하고, 데이터 키도 해시해서 링 위에 배치한다. 이후 데이터는 자기 위치에서 시계 방향으로 이동했을 때 가장 먼저 만나는 서버에 저장된다.

![안정 해시 기본 구조](images/01-hash-ring-basic.svg)

### 기본 서버 5대 구조에서 데이터 배치와 저장

서버 5대가 링 위에 있을 때 각 데이터는 자기 해시 위치에서 시계 방향으로 가장 가까운 서버에 저장된다. 아래 예시는 데이터 A~E가 어느 서버에 저장되는지 보여준다.

![서버 5대 구조에서 데이터 배치와 저장](images/02-five-server-placement.svg)

## 3. 서버 추가/삭제 시 재배치

안정 해시에서는 서버가 추가되거나 삭제되어도 전체 데이터를 다시 배치하지 않는다.

서버가 추가되면 새 서버가 들어간 위치와 그 직전 서버 사이의 키만 새 서버로 이동한다. 서버가 삭제되면 삭제된 서버가 맡던 키만 시계 방향 다음 서버로 이동한다. 나머지 서버가 맡던 데이터는 그대로 유지된다.

![서버 추가 삭제 시 데이터 변경 범위](images/03-server-add-delete.svg)

## 4. 안정 해시의 문제점

안정 해시는 데이터 재배치를 줄여주지만, 기본 구조만으로는 키가 항상 균등하게 분산된다고 보장하기 어렵다.

물리 서버를 링 위에 한 지점씩만 배치하면 각 서버가 맡는 구간 크기가 우연에 크게 좌우된다. 서버 추가/삭제가 반복되면 어떤 서버는 큰 구간을 맡고, 어떤 서버는 작은 구간만 맡아 부하가 치우칠 수 있다.

![가상 노드가 없을 때 불균등 분배](images/04-imbalance-without-vnodes.svg)

## 5. 가상 노드

가상 노드는 물리 서버 하나를 링 위의 여러 지점에 나눠 배치한 것이다. 링에는 `S1-1`, `S1-2`처럼 가상 노드가 올라가지만, 실제 저장과 운영 책임은 물리 서버 S1이 가진다.

가상 노드를 사용하면 한 물리 서버가 하나의 큰 구간이 아니라 여러 작은 구간을 나누어 맡는다. 그래서 특정 서버에만 데이터가 몰릴 가능성을 줄이고, 서버 추가/삭제 시에도 여러 서버에서 작은 조각 단위로 데이터가 이동한다.

### 가상 노드의 실제 서버 배치 구조

![가상 노드와 실제 서버 배치 구조](images/05-virtual-node-mapping.svg)

### 가상 노드 사용 시 데이터 저장 모습

데이터는 먼저 시계 방향으로 가장 가까운 가상 노드에 배정된다. 이후 그 가상 노드가 가리키는 물리 서버에 실제로 저장된다.

![가상 노드 사용 시 데이터 저장 모습](images/06-virtual-node-storage.svg)

### 가상 노드 사용 시 노드/서버 추가·삭제 변경 범위

가상 노드를 쓰면 물리 서버 하나가 담당하던 범위가 여러 작은 조각으로 나뉜다. 서버를 추가하거나 삭제할 때도 큰 구간 하나가 통째로 이동하는 대신, 여러 서버에서 작은 조각들이 조금씩 이동한다.

![가상 노드 사용 시 서버 추가 삭제 변경 범위](images/07-vnode-add-delete.svg)

## 공식/원문 그림 기준으로 다시 읽기

아래 참고자료들은 바로 논문식 설명부터 읽으면 어렵다. 그래서 먼저 원문에서 제공하는 그림을 보고, 그 그림이 이 문서의 어느 구조와 연결되는지 맞춰서 읽는 편이 이해하기 쉽다. 원문 그림은 그대로 가져오지 않고 링크로 연결하고, 이 문서의 로컬 그림과 함께 읽도록 정리했다.

### 1. Cassandra 공식 문서: 해시값 범위가 노드의 책임 범위가 된다

- 원문 그림: [DataStax Cassandra - Hash values in a four node cluster](https://docs.datastax.com/en/cassandra-oss/3.0/cassandra/images/arc_hashValueRange.png)
- 같이 볼 그림: [서버 5대 구조에서 데이터 배치와 저장](images/02-five-server-placement.svg)
- 핵심 문서: [DataStax Cassandra - Consistent hashing](https://docs.datastax.com/en/cassandra-oss/3.0/cassandra/architecture/archDataDistributeHashing.html)

이 그림에서 링은 서버를 예쁘게 그린 장식이 아니라, 전체 해시값 범위를 원형으로 이어 붙인 것이다. Cassandra 문서에서는 partition key를 해시해서 나온 값이 어느 token range에 들어가는지를 보고 담당 노드를 정한다고 설명한다.

우리 문서의 그림으로 바꾸면, 데이터 A~E도 각각 해시된 위치를 가진다. 그리고 자기 위치에서 시계 방향으로 처음 만나는 서버가 그 데이터를 맡는다. 그래서 안정 해시의 기본 규칙은 다음 한 문장으로 정리할 수 있다.

> 데이터 key를 해시해서 링 위에 올리고, 시계 방향으로 처음 만나는 서버에 저장한다.

### 2. Cassandra 공식 문서: 가상 노드는 큰 구간을 여러 작은 구간으로 쪼갠다

- 원문 그림: [DataStax Cassandra - Virtual vs single-token architecture](https://docs.datastax.com/en/cassandra-oss/3.0/cassandra/images/arc_vnodes_compare.png)
- 같이 볼 그림: [가상 노드와 실제 서버 배치 구조](images/05-virtual-node-mapping.svg), [가상 노드 사용 시 데이터 저장 모습](images/06-virtual-node-storage.svg)
- 핵심 문서: [DataStax Cassandra - How data is distributed across a cluster using virtual nodes](https://docs.datastax.com/en/cassandra-oss/3.0/cassandra/architecture/archDataDistributeDistribute.html)

이 원문 그림은 위아래를 비교해서 봐야 한다. 위쪽은 물리 서버 하나가 링 위에서 하나의 큰 연속 구간을 맡는 구조다. 이 경우 어떤 서버는 큰 구간을 맡고, 어떤 서버는 작은 구간만 맡을 수 있다.

아래쪽은 가상 노드를 사용한 구조다. 물리 서버 하나가 링 위의 여러 위치에 나뉘어 등장한다. 예를 들어 실제 서버 S1이 `S1-1`, `S1-2`, `S1-3` 같은 여러 가상 노드를 가진다. 데이터는 먼저 가장 가까운 가상 노드에 배정되고, 실제 저장은 그 가상 노드가 가리키는 물리 서버에 된다.

이 구조의 효과는 단순하다.

- 물리 서버 하나가 하나의 큰 구간을 통째로 맡지 않는다.
- 여러 작은 구간을 나눠 맡기 때문에 부하가 평균에 가까워진다.
- 서버 추가/삭제 시에도 큰 덩어리 하나가 아니라 작은 구간들이 조금씩 이동한다.

### 3. Dynamo 원문: 물리 노드 하나를 링 위 여러 지점에 배치한다

- 원문 자료: [Dynamo: Amazon's Highly Available Key-value Store](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)
- 같이 볼 그림: [가상 노드와 실제 서버 배치 구조](images/05-virtual-node-mapping.svg), [가상 노드 사용 시 서버 추가 삭제 변경 범위](images/07-vnode-add-delete.svg)

Dynamo의 안정 해시 설명도 Cassandra의 가상 노드 그림과 같은 방향으로 이해하면 된다. 핵심은 물리 서버 하나를 링 위의 한 점에만 두지 않는 것이다. 물리 서버는 여러 개의 virtual node를 가질 수 있고, 링 위에서는 이 virtual node들이 각각 독립된 위치를 가진다.

이렇게 하면 서버 한 대가 추가되거나 빠질 때 한 서버의 큰 연속 구간만 흔들리지 않는다. 여러 virtual node 단위로 책임 구간이 나뉘기 때문에 데이터 이동도 여러 작은 조각으로 분산된다. 이 문서의 5번 그림부터 7번 그림까지가 바로 그 구조를 단순화한 것이다.

### 4. Cassandra 논문: 링은 저장 위치뿐 아니라 복제 기준도 된다

- 원문 자료: [Cassandra - A Decentralized Structured Storage System](https://www.cs.cornell.edu/projects/ladis2009/papers/lakshman-ladis2009.pdf)
- 같이 볼 그림: [서버 추가 삭제 시 데이터 변경 범위](images/03-server-add-delete.svg), [가상 노드 사용 시 서버 추가 삭제 변경 범위](images/07-vnode-add-delete.svg)

Cassandra 논문에서 안정 해시는 단순히 "어느 서버에 저장할지"만 정하는 장치가 아니다. 링에서 key의 위치를 찾고, 그 위치를 담당하는 노드와 그 뒤쪽 노드들을 복제 후보로 볼 수 있다.

즉 저장소 시스템에서 링은 두 가지 기준이 된다.

- 첫 번째 기준: 이 key의 주 담당 노드는 누구인가?
- 두 번째 기준: 복제본을 둘 다음 노드들은 누구인가?

여기서 가상 노드가 들어가면 주의할 점이 생긴다. 링 위에서는 서로 다른 가상 노드처럼 보여도, 실제로는 같은 물리 서버를 가리킬 수 있다. 그래서 복제 대상을 고를 때는 같은 물리 서버가 중복되지 않도록 걸러야 한다.

### 5. Discord 글: 링 구조가 맞아도 조회 경로가 병목이 될 수 있다

- 원문 자료: [How Discord Scaled Elixir to 5,000,000 Concurrent Users](https://discord.com/blog/how-discord-scaled-elixir-to-5-000-000-concurrent-users)
- 같이 볼 그림: [안정 해시 기본 구조](images/01-hash-ring-basic.svg)

Discord 글은 "안정 해시 구조를 어떻게 그리느냐"보다 "그 링을 실제 요청 경로에서 얼마나 빠르게 조회하느냐"를 보여주는 사례로 읽으면 된다.

안정 해시를 쓰면 요청마다 보통 다음 일을 한다.

1. key를 해시한다.
2. 링에서 key 위치를 찾는다.
3. 시계 방향으로 담당 노드나 shard를 찾는다.

이 과정이 세션 재연결 같은 hot path에 들어가면, 링 알고리즘 자체가 맞더라도 조회 비용이 병목이 될 수 있다. 그래서 운영 시스템에서는 링을 어떻게 배치할지뿐 아니라, 링 정보를 어떤 자료구조로 들고 있고 여러 프로세스가 어떻게 빠르게 읽을지도 설계 대상이 된다.

### 6. Maglev 논문: 로드밸런서는 링보다 빠른 lookup table을 중시한다

- 원문 자료: [Maglev: A Fast and Reliable Software Network Load Balancer](https://research.google/pubs/maglev-a-fast-and-reliable-software-network-load-balancer/)
- 원문 PDF: [Maglev paper](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/44824.pdf)

Maglev는 분산 저장소가 아니라 로드밸런서 사례다. 그래서 관심사가 조금 다르다. 저장소에서는 "이 key의 데이터를 어느 서버가 책임지는가"가 중요하지만, 로드밸런서에서는 "이 연결을 어느 backend로 빠르게 보낼 것인가"가 중요하다.

Maglev는 backend 선택을 빠르게 하기 위해 큰 lookup table을 만든다. backend가 추가되거나 삭제되어도 기존 연결이 가능한 한 같은 backend로 가게 하면서, 동시에 backend들 사이에 트래픽이 고르게 나뉘도록 설계한다.

따라서 Maglev는 안정 해시를 그대로 저장소처럼 쓰는 예시라기보다, 같은 문제의식을 로드밸런서에 맞게 바꾼 사례로 보면 된다.

- 변경이 생겨도 기존 매핑이 너무 많이 흔들리면 안 된다.
- 특정 backend에 연결이 몰리면 안 된다.
- 매 요청마다 backend를 아주 빠르게 찾아야 한다.

### 7. 이 참고자료들을 읽는 순서

1. 먼저 Cassandra의 hash range 그림을 본다. 여기서 "해시값 범위 = 노드 책임 범위"라는 감을 잡는다.
2. 이 문서의 [서버 5대 구조에서 데이터 배치와 저장](images/02-five-server-placement.svg)을 본다. key가 시계 방향 첫 서버로 가는 규칙을 확인한다.
3. Cassandra의 vnode 비교 그림을 본다. 위쪽은 물리 서버 1개가 큰 구간 1개를 맡는 구조이고, 아래쪽은 물리 서버 1개가 작은 구간 여러 개를 맡는 구조다.
4. 이 문서의 [가상 노드 사용 시 서버 추가 삭제 변경 범위](images/07-vnode-add-delete.svg)를 본다. 가상 노드를 쓰면 서버 추가/삭제 때 데이터 이동이 작은 조각으로 나뉜다는 점을 확인한다.
5. 그 다음 Dynamo와 Cassandra 논문을 읽는다. 여기서는 안정 해시가 실제 분산 저장소에서 파티션, 복제, 노드 추가/삭제의 기준으로 쓰인다는 점을 보면 된다.
6. 마지막으로 Discord와 Maglev를 읽는다. 여기서는 안정 해시 구조 자체보다 운영 중 lookup 비용, load balancing, minimal disruption이 중요해진다는 점을 보면 된다.

정리하면, 안정 해시 설계에서 꼭 답해야 하는 질문은 다음과 같다.

1. key와 서버를 어떤 해시 함수로 같은 공간에 올릴 것인가?
2. 데이터는 시계 방향 첫 노드에 둘 것인가, 복제까지 고려해 여러 노드에 둘 것인가?
3. 물리 서버 하나당 가상 노드 또는 token을 몇 개 둘 것인가?
4. 가상 노드가 같은 물리 서버를 가리킬 때 복제 중복을 어떻게 막을 것인가?
5. 서버 추가/삭제 시 어느 구간의 데이터만 이동해야 하는가?
6. 링 조회가 요청 hot path에 있다면 lookup 자료구조는 충분히 빠른가?
7. 저장소처럼 데이터 위치가 중요한 시스템인가, 로드밸런서처럼 빠른 backend 선택이 중요한 시스템인가?
