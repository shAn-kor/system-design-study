# chapter05-week2-design

# 디스코드 문서 정리
### 기존 구조
session process
→ "guild:42는 어느 node야?"라고 ring 담당 process에 메시지
→ ring 담당 process가 hash ring 조회
→ "node B야"라고 답장
→ session process가 node B로 감

S1 ─┐
S2 ─┼→ ring 담당 process → hash ring → node B
S3 ─┤
S4 ─┘

### 변경된 구조
session process
→ 공유된 read-only hash ring을 직접 읽음
→ guild:42를 해시해서 링 위 위치 계산
→ 시계 방향으로 첫 node 찾기
→ node B로 감

S1 ─┐
S2 ─┼→ 공유 hash ring → node B
S3 ─┤
S4 ─┘


| 구분 | 기존 | 변경 후 |
|---|---|---|
| 누가 링을 읽나 | ring 담당 process 한 명 | 각 session process |
| 요청 방식 | 메시지 보내고 답장 대기 | 자기 자리에서 바로 조회 |
| 대기 줄 | 생김 | 없음 |
| 안정 해시 규칙 | `key hash → 시계 방향 첫 node` | 똑같음 |


## 정말 나아졌을까

### ETS 구조
ring을 갱신하는 쪽
→ 최신 hash ring을 ETS 표에 저장
ETS: BEAM VM 안의 여러 process가 직접 찾을 수 있는 공유 인메모리 표

#### session process
→ ETS 표에서 최신 hash ring을 직접 꺼냄
→ 그런데 큰 ring 값이 session process의 heap으로 복사됨
→ guild:42를 해시해서 링 위 위치 계산
→ 시계 방향으로 첫 node 찾기
→ node B로 감

#### ring 갱신 process
→ ETS 표
S1 ─┐
S2 ─┼→ ETS 표 → 큰 ring 복사 → 각 session heap → node B
S3 ─┤
S4 ─┘

### FastGlobal 구조
#### ring을 갱신하는 쪽
→ 최신 hash ring을 read-only shared heap에 올림

#### session process
→ 공유된 ring 값을 복사 없이 읽음
→ guild:42를 해시해서 링 위 위치 계산
→ 시계 방향으로 첫 node 찾기
→ node B로 감

#### ring 갱신 process
→ 공유 read-only ring
S1 ─┐
S2 ─┼→ 같은 ring을 복사 없이 읽음 → node B
S3 ─┤
S4 ─┘


ETS        : 모두 같은 표에서 읽지만, 큰 ring은 각자 heap으로 복사함
FastGlobal : 모두 같은 ring 자체를 복사 없이 읽음

요청 처리 process
→ FastGlobal에서 현재 ring 지도 읽기
→ guild:42를 해시해서 링 위 위치 계산
→ 시계 방향으로 첫 node 찾기
→ node B 선택

node 구성 변경
→ 새 node 목록으로 새 ring 계산
→ 새 ring을 담은 module 생성
→ compile
→ BEAM VM에 load
→ 이후 요청은 새 ring 읽기

## 해시 알고리즘
비 암호화 해시 알고리즘이 여러가지 이며 이중 분산을 위해 많이 쓰이는건 크게 2가지가 있다

#### murmurhash3
- MU = Multiply
- R = Rotate
- Hash
- 3 = 세 번째 설계 세대

#### xxh3 (Extremely fast Hash algorithm)
- xxHash = Extremely fast Hash algorithm

둘 다 결국은 **문자열/바이트를 규칙적으로 섞어서 고정 길이 숫자로 바꾸는 과정**이야.  
예를 들어 `"guild:42"`를 넣으면, 원래 글자를 복원할 수 없는 32비트·64비트 숫자로 바뀌고 그 숫자를 해시 링 위치로 써.

아래는 이해하기 쉽게 대표 형태로 풀어쓴 거야.

## MurmurHash3: 한 덩어리씩 순서대로 강하게 휘젓기

대표적으로 `MurmurHash3_x86_32`는 32비트 결과를 만드는 버전이야.

```text
입력: "guild:42"
→ 바이트: g u i l d : 4 2
→ 4바이트씩 나눔
→ 각 덩어리를 섞어서 하나의 누적 숫자 h1에 합침
→ 남은 바이트와 전체 길이도 반영
→ 마지막에 전체 숫자를 한 번 더 세게 섞음
→ 32비트 해시값
```

조금 더 실제 과정에 가깝게 쓰면:

1. **seed로 시작 숫자를 만든다**

   ```text
   h1 = seed
   ```

   seed가 다르면 같은 `"guild:42"`도 다른 결과가 나와.  
   같은 시스템 안에서는 모두 같은 seed를 써야 같은 node를 고르게 돼.

2. **입력을 4바이트씩 읽는다**

   예를 들어 앞 네 글자 `"guil"`을 하나의 32비트 숫자 `k1`로 본다.

3. **`k1`을 세게 섞는다**

   ```text
   k1 = k1 × 상수1
   k1 = 비트 회전
   k1 = k1 × 상수2
   ```

   - 곱하기: 비트들이 넓게 퍼지게 함
   - 비트 회전: 끝쪽 비트도 다른 자리로 이동
   - 다시 곱하기: 규칙적인 모양을 더 깨뜨림

4. **섞인 `k1`을 지금까지 결과 `h1`에 합친다**

   ```text
   h1 = h1 XOR k1
   h1 = 비트 회전
   h1 = h1 × 5 + 상수
   ```

   여기서 `5`와 뒤의 상수는 입력이나 사용자가 정하는 값이 아니라,
   **MurmurHash3 구현에 미리 정해져 있는 고정 상수**야.

   - `h1`: 입력을 읽을 때마다 달라지는 누적 숫자
   - `5`: 고정 상수. 홀수라서 32비트 안에서 비트 정보를 쉽게 뭉개지 않고, `× 4 + 자기 자신`처럼 빠르게 계산할 수 있음
   - 뒤의 상수: 고정 상수. 반복 처리 중 단순한 패턴이나 고정된 결과에 빠지는 것을 막음
   - `seed`: 사용자가 정할 수 있는 시작값

   따라서 `5`나 뒤의 상수를 바꾸면 같은 입력의 결과가 달라지고,
   더 이상 표준 MurmurHash3 결과가 아니야.

   이 과정을 다음 4바이트 덩어리에도 반복해.  
   그래서 앞부분이 달라져도 뒤의 누적 결과가 달라져.

5. **4바이트로 딱 나뉘지 않은 끝부분도 넣는다**

   `"guild:42"`처럼 끝에 1~3바이트가 남으면 버리지 않고 따로 `k1`에 넣어 섞는다.

6. **마지막 avalanche 단계**

   ```text
   h1 = h1 XOR (h1을 오른쪽으로 민 값)
   h1 = h1 × 상수
   h1 = h1 XOR (h1을 오른쪽으로 민 값)
   h1 = h1 × 상수
   h1 = h1 XOR (h1을 오른쪽으로 민 값)
   ```

   이 마지막 단계 때문에 입력 한 글자만 바뀌어도 결과 비트 대부분이 바뀌는 성질이 생겨. 이를 avalanche 효과라고 해. MurmurHash3의 실제 32비트 구현도 `4바이트 block → multiply/rotate/multiply → 누적 → fmix` 순서다. [MurmurHash3 원본 구현](https://chromium.googlesource.com/external/smhasher/%2B/daec2995d3e14a7c2eeca690ba0b86e34dcf853c/MurmurHash3.cpp)

---

## XXH3: 입력 길이에 따라 가장 빠른 섞는 길을 고르고, 여러 덩어리를 병렬로 섞기

XXH3도 결과적으로는 “입력 바이트 → 섞인 숫자”지만, **입력이 짧을 때와 길 때 방법을 아예 다르게 쓴다**는 점이 핵심이야.

```text
입력: "guild:42"

짧은 입력인가?
→ 예: 앞·뒤 바이트를 읽음
→ secret과 XOR
→ 64비트 곱셈 결과를 접어서 섞음
→ avalanche
→ 64비트 해시값
```

### 1. seed 말고 `secret`도 사용한다

XXH3 안에는 기본으로 준비된 긴 숫자 바이트표(`secret`)가 있어. 기본 secret은 약 192바이트의 **고정된 내부 재료**야.
이름은 secret이지만 사용자 비밀번호나 서버 비밀값은 아니야. 같은 XXH3 구현을 쓰는 곳은 기본적으로 같은 표를 사용해, 같은 입력에서 같은 해시값을 만들 수 있어.

이번에는 `XXH3_64bits("guild:42", seed = 0)`의 **4~8바이트 입력 경로**를 실제 기본 secret 값으로 보자.

```text
입력 앞 4바이트 = "guil"  → front
입력 뒤 4바이트 = "d:42"  → back

기본 secret의 offset 8~15 바이트 (0부터 세면 9~16번째)
= 7c 01 81 2c f7 21 ad 1c

컴퓨터가 little-endian 64비트 숫자로 읽으면
= 0x1cad21f72c81017c
```

`"guil"`과 `"d:42"`도 같은 방식으로 4바이트씩 읽는다.

```text
"guil" → 0x6c697567
"d:42" → 0x32343a64

둘을 앞·뒤로 붙임
input64 = 0x6c69756732343a64

seed = 0이므로
keyed = input64 XOR 0x1cad21f72c81017c
      = 0x70c454901eb53b18
```

여기까지가 “입력과 고정 내부 재료를 XOR로 만나는” 실제 모습이야. 그 다음 `keyed`에 회전, 곱셈, 오른쪽 이동 XOR을 여러 번 적용하는 `rrmxmx` 마무리 함수를 거쳐 최종 64비트 해시값이 나온다.

중요한 점은 이 8바이트 입력에서는 `S0`, `S1` 두 개를 임의로 고르는 것이 아니라, 구현이 정해 둔 정확한 위치인 **기본 secret의 offset 8~15 바이트**를 읽는다는 거야. 입력 길이가 9~16바이트가 되면 구현은 secret의 다른 위치들을 사용하고, 더 긴 입력에서는 16바이트·64바이트 단위마다 또 다른 secret 조각을 사용한다.

#### 왜 하필 secret의 offset 8~15인가?

입력의 “8번째 바이트”를 고른 것이 아니야. `"guild:42"`가 8바이트라서
XXH3의 **4~8바이트 전용 코드 경로**로 들어갔고, 그 코드 경로가 처음부터
`readLE64(secret + 8)`을 쓰도록 정해져 있기 때문이야.

```text
입력 길이 1~3바이트
→ 기본 secret의 앞부분을 사용

입력 길이 4~8바이트
→ readLE64(secret + 8)
→ offset 8~15, 즉 0부터 세면 9~16번째 바이트 사용

입력 길이 9~16바이트
→ 또 다른 secret offset들을 사용
```

왜 숫자 `8`이냐는 질문에는, 입력에서 계산해 낸 값이 아니라 알고리즘 작성자가
정한 **고정 offset**이라고 답하는 게 맞아. 각 길이 경로가 겹치지 않는 다른 secret
조각을 쓰게 하려는 설계이고, 이 특정 숫자에 특별한 뜻이 있거나 사용자가 바꾸는 값은 아니야.

`seed`를 `0` 대신 다른 값으로 주면 위 secret 값에 seed를 섞은 값이 XOR 재료가 된다. 그래서 같은 `"guild:42"`도 다른 결과가 나와. 다만 안정 해시에 쓸 때는 모든 process가 같은 seed를 사용해야 모두 같은 node를 고른다.

### 2. 짧은 입력은 “앞과 뒤를 빠르게 잡아 섞는다”

예를 들어 정확히 8바이트인 `"guild:42"`는 긴 반복문을 돌지 않고:

```text
앞 4바이트와 뒤 4바이트를 붙여 input64 만들기
기본 secret의 offset 8~15 바이트를 64비트로 읽기
input64 XOR secret 값
rrmxmx(회전·곱셈·오른쪽 이동 XOR)로 마무리
64비트 해시값
```

를 한다. 9~16바이트 입력부터는 앞·뒤 입력 조각에 서로 다른 secret 위치를 써서 64비트 곱셈과 fold를 적용한다.

여기서 **접기(fold)**는 이런 뜻이야.

```text
64비트 × 64비트
→ 128비트짜리 큰 결과

상위 64비트 XOR 하위 64비트
→ 다시 64비트
```

곱셈으로 비트가 넓게 섞이고, 위·아래 절반을 합치면서 정보가 한쪽에 몰리지 않게 해.

### 3. 중간 길이는 16바이트씩 여러 번 섞는다

입력이 조금 길어지면:

```text
입력 16바이트
→ secret의 해당 위치 16바이트와 XOR
→ 곱셈·fold
→ 누적값에 더함

다음 16바이트
→ 다른 secret 조각과 XOR
→ 곱셈·fold
→ 누적값에 더함
```

처럼 처리해.

MurmurHash3가 대체로 “하나의 누적값을 앞에서부터 계속 갱신”하는 느낌이라면, XXH3는 **각 덩어리에 서로 다른 secret 조각을 대고 섞은 뒤 결과를 모으는 방식**에 가까워.

### 4. 긴 입력은 여러 줄을 동시에 처리한다

아주 긴 입력에서는 CPU가 한 번에 여러 숫자를 계산할 수 있는 SIMD 명령을 활용한다.

```text
긴 입력
→ 여러 64바이트 줄(stripe)로 나눔
→ 한 줄 안의 여러 숫자를 여러 누적칸에 나눠 섞음
→ 다음 줄도 다른 secret 조각으로 섞음
→ 중간중간 누적칸을 다시 scramble
→ 마지막에 누적칸들을 하나로 합치고 avalanche
→ 최종 해시값
```

그래서 XXH3는 한 줄의 계산이 끝날 때까지 다음 계산이 마냥 기다리지 않도록 설계되어 있어. 현대 CPU의 병렬 계산 능력을 더 잘 쓴다는 뜻이야. XXH3는 64/128비트 계열이고, 작은 입력 성능과 SIMD 활용을 특히 목표로 설계됐다고 공식 문서가 설명해. [XXH3 공식 구현 문서](https://raw.githubusercontent.com/Cyan4973/xxHash/dev/xxhash.h)

---

## 차이를 한 문장으로

- **MurmurHash3**: 4바이트씩 가져와 하나의 누적 숫자를 차례대로 강하게 섞고, 마지막에 전체를 avalanche한다.
- **XXH3**: 짧은 입력은 앞·뒤를 빠르게 섞고, 긴 입력은 secret과 여러 누적값·SIMD를 이용해 여러 덩어리를 더 병렬적으로 섞는다.

둘 다 `"guild:42"`를 해시 링 위의 숫자 위치로 바꿀 수 있지만, XXH3는 특히 현대 CPU에서 그 숫자를 더 빨리 만들도록 설계된 쪽이야.
