# Discord 안정 해시 학습 페이지 디자인 시스템

## 0. Research Log

- Embedded refs: Intercom, Notion, Figma를 검토하고 `minimalist-skill` + Intercom을 선택했다. 긴 한국어 설명의 인지 부담은 낮추고, 따뜻한 종이색과 또렷한 상태 색으로 친근함을 유지하기 위해서다.
- Lazyweb: `children education interactive explainer lesson`, `technical documentation educational storytelling diagrams` 2개 질의로 Kahoot, Tappity, moonrepo 화면 3개를 확인했다. 강한 장 구분 띠, 순서가 보이는 이야기 흐름, 개념 카드 문법만 가져오고 화면은 복제하지 않는다.
- Imagen drafts: `exec-8e49e6ba-e87e-4ec5-b201-e698650ca589.png`, `exec-2a1b9401-1c0b-4f10-9caf-316f8d789557.png`을 비교해 `.omo/ulw-loop/019f4b86-33a3-7f51-943f-cce7857bd18e/evidence/reference-concept-school-map.png`을 기준안으로 골랐다.
- UI/UX DB: content-first 구조, 한국어 가독성, progressive disclosure, 375/768/1280px 검증을 채택했다. 외부 폰트 권고는 self-contained 제약 때문에 사용하지 않는다.
- 방향: 따뜻한 교실 학습지다. 종이와 잉크처럼 평평하고 또렷한 표면 위에서, 병목은 coral/orange, 공유 읽기는 indigo/sky로 구분한다. 기억에 남는 장면은 `한 안내 데스크에 줄 서기`와 `각자 같은 학교 지도를 읽기`의 전후 비교다.

## 1. Atmosphere & Identity

처음 보는 분산 시스템도 겁나지 않는 교실용 현장 학습지다. 장식보다 이야기 순서가 먼저 보이고, 모든 기술 용어는 `비유`, `실제 사실`, `계산값`, `주의` 중 하나로 출처 범위를 드러낸다. 대표 장면은 학교 지도 전후 비교이며, 안정 해시 링은 별도의 순환 노선 그림으로 설명한다.

## 2. Color

| 역할 | 토큰 | 값 | 용도 |
|---|---|---|---|
| 바탕 | `--surface-canvas` | `#F8F4EC` | 페이지 전체의 따뜻한 종이색 |
| 기본 표면 | `--surface-card` | `#FFFDF8` | 카드와 도식 배경 |
| 보조 표면 | `--surface-soft` | `#F1EBDD` | 보조 설명과 표 머리 |
| 기본 글자 | `--text-primary` | `#172033` | 제목과 본문 |
| 보조 글자 | `--text-secondary` | `#4E5969` | 캡션과 부연 |
| 경계선 | `--border-oat` | `#D8CDBA` | 카드와 구분선 |
| 병목 | `--accent-hot` | `#C84E2F` | 위험, 느린 경로, 오답 |
| 병목 배경 | `--accent-hot-soft` | `#FBE7DE` | 병목 카드 |
| 공유 읽기 | `--accent-shared` | `#244C86` | 해결 경로, 링크, 초점 |
| 공유 읽기 배경 | `--accent-shared-soft` | `#E6F0FC` | 해결 카드 |
| 실제 사실 | `--fact` | `#356B4A` | 사실 라벨과 정답 |
| 사실 배경 | `--fact-soft` | `#E7F1E9` | 사실 라벨 배경 |
| 주의 | `--caution` | `#805B13` | 역사 범위와 과장 방지 |
| 주의 배경 | `--caution-soft` | `#FFF2CE` | 주의 박스 |
| 초점선 | `--focus-ring` | `#0B5FFF` | 키보드 초점 |

색만으로 의미를 전달하지 않는다. 모든 상태는 텍스트 라벨과 모양을 함께 쓴다. 본문 대비는 WCAG 2.2 AA 4.5:1 이상을 유지한다.

## 3. Typography

- 기본 글꼴: `system-ui, -apple-system, BlinkMacSystemFont, "Apple SD Gothic Neo", "Noto Sans KR", sans-serif`.
- 고정폭: `ui-monospace, "SFMono-Regular", Consolas, monospace`.
- 외부 폰트나 CDN은 사용하지 않는다.

| 단계 | 크기 | 굵기 | 줄 높이 | 용도 |
|---|---:|---:|---:|---|
| Display | `clamp(2.25rem, 6vw, 4.5rem)` | 800 | 1.12 | 페이지 제목 |
| H2 | `clamp(1.75rem, 4vw, 2.75rem)` | 800 | 1.2 | 장 제목 |
| H3 | `clamp(1.25rem, 2.6vw, 1.75rem)` | 750 | 1.3 | 카드 제목 |
| Lead | `clamp(1.0625rem, 2vw, 1.25rem)` | 500 | 1.7 | 도입 문장 |
| Body | `1rem` | 400 | 1.75 | 기본 설명 |
| Small | `0.875rem` | 500 | 1.6 | 출처와 보조 라벨 |

본문 한 줄은 데스크톱 65~75자, 모바일 35~60자를 목표로 한다. 한국어 조사가 한 줄에 홀로 남지 않도록 제목 폭과 `text-wrap: balance`를 조정한다.

## 4. Spacing & Layout

- 기본 단위: 4px.
- 간격 토큰: `--space-1: 4px`, `--space-2: 8px`, `--space-3: 12px`, `--space-4: 16px`, `--space-5: 20px`, `--space-6: 24px`, `--space-8: 32px`, `--space-10: 40px`, `--space-12: 48px`, `--space-16: 64px`, `--space-20: 80px`.
- 콘텐츠 최대 폭: 1120px. 긴 문단 최대 폭: 760px.
- breakpoints: 375px 소형 화면, 768px 태블릿, 1280px 데스크톱 검증 기준.
- 375px: 한 열, 16px gutter, 도식은 세로 재배치, 가로 스크롤 금지.
- 768px: 24px gutter, 비교 카드 2열 가능하되 200% 확대에서는 한 열로 복귀.
- 1280px: 32px gutter, 최대 폭 안에서 비대칭 12열 구성을 허용한다.

## 5. Components

### Chapter band

- 구조: 장 번호, 짧은 제목, 한 문장 목표.
- 변형: 기본, 현재 위치.
- 상태: 링크일 때 hover, active, `:focus-visible`.
- 접근성: 순차적인 heading 구조와 명확한 링크 문구.

### Evidence label

- 변형: `비유`, `실제 사실`, `계산값`, `주의`.
- 색과 함께 텍스트를 항상 표시한다.
- 장식 아이콘 대신 짧은 단어와 테두리를 사용한다.

### Diagram card

- 구조: 제목, inline SVG, 설명 목록, 대체 텍스트 역할의 `<figcaption>`.
- 변형: 링, 전후 흐름, 성능 단계.
- 상태: 정적. 의미 없는 hover나 motion은 없다.

### Comparison card

- 변형: `hot` 안내 데스크, `shared` 직접 지도 읽기.
- 구조: 경로, 속도, 대기, 확장성 설명.
- 모바일에서는 읽기 순서가 이전 뒤 이후가 되도록 한 열로 바꾼다.

### Real-system disclosure

- 토글: `#real-system-toggle`.
- 패널: `#real-system-panel`.
- 상태: 접힘, 펼침, hover, active, focus. 토글은 `aria-expanded`와 `aria-controls`를 갱신한다.
- 패널을 숨길 때 `hidden`을 사용하고 펼침 상태는 키보드 Enter/Space로도 동작한다.

### Quiz

- 정답 선택자: `[data-quiz-answer="correct"]`.
- 피드백: `#quiz-feedback`, `aria-live="polite"`.
- 상태: 기본, hover, focus, 선택, 정답, 오답, 완료. 정답/오답은 색과 함께 문장으로 알린다.
- 모든 선택지는 최소 44px 높이이며 키보드로 접근 가능하다.

### Source card

- 공식 글 제목, 작성자, 2017-07-06 날짜, 원문 링크와 역사적 범위 주의를 함께 표시한다.
- 링크는 새 창 여부를 문구로 알리고 명확한 focus ring을 가진다.

## 6. Motion & Interaction

- micro: 120ms ease-out, 버튼 누름과 focus 상태에만 사용.
- standard: 220ms ease-in-out, disclosure의 opacity/transform에만 사용.
- layout 속성은 애니메이션하지 않는다. `transform`과 `opacity`만 허용한다.
- `prefers-reduced-motion: reduce`에서는 모든 비필수 전환을 제거한다.
- 장식용 입장 애니메이션, 자동 재생, 스크롤 효과는 사용하지 않는다.

## 7. Depth & Surface

전략은 borders + tonal shift다. 카드에는 `1px solid var(--border-oat)`를 쓰고, 표면색 차이로 층을 만든다. 큰 그림자, gradient, glassmorphism, dark mode는 사용하지 않는다. 모서리는 버튼 6px, 카드 12px, 큰 도식 16px로 제한한다.

## 8. Accessibility Constraints & Accepted Debt

- 목표: WCAG 2.2 AA.
- 375px, 768px, 1280px과 200% 확대에서 가로 스크롤과 잘림이 없어야 한다.
- 모든 조작은 keyboard-only로 가능하고 focus ring을 제거하지 않는다.
- screen reader 순서는 시각적 순서와 같아야 하며 inline SVG는 제목과 설명을 제공한다.
- CJK 줄바꿈은 자연스러워야 하며 한 글자 고아 줄, 잘린 받침, 글리프 누락이 없어야 한다.
- `prefers-reduced-motion`을 존중하고 색만으로 정답, 위험, 해결을 구분하지 않는다.
- accepted debt: 없음.

