# Flood Escape Lab — 구현 가이드라인 & API 명세

> 작성일: 2026-06-26  
> 목적: Claude Code + MCP를 활용한 교육용 침수 시뮬레이션 게임 구현 로드맵

---

## 1. 기술 스택 선택

| 역할 | 선택 | 이유 |
|---|---|---|
| UI 프레임워크 | **React + Vite** | 컴포넌트 분리, 빠른 HMR |
| 단면도 애니메이션 | **SVG + CSS animation** | Canvas보다 DOM 접근 쉬움, 반응형 |
| 수위 그래프 | **Recharts** | React 친화적, 실시간 업데이트 쉬움 |
| 상태 관리 | **Zustand** | 가볍고 보일러플레이트 없음 |
| 영속 데이터(도감·배지) | **localStorage** | 서버 없이 구현 가능 |
| 스타일 | **Tailwind CSS** | 빠른 프로토타이핑 |

---

## 2. MCP 역할 분담

```
Claude Code (나)
├── Obsidian MCP     → 기획 문서 저장·검색, 개발 로그, 변수 명세 관리
├── Figma MCP        → 단면도 SVG 레이아웃, UI 컴포넌트 시안
└── Context7 MCP     → Recharts·Zustand 최신 API 레퍼런스 조회
```

**실전 활용 패턴:**
- 변수 조합 추가할 때 → Obsidian에서 명세 읽어서 코드 생성
- 새 화면 만들 때 → Figma에서 시안 가져와서 SVG 구현
- 라이브러리 API 헷갈릴 때 → Context7로 최신 문서 조회

---

## 3. 핵심 물리 모델 (Simulation Engine)

### 3.1 기본 공식

```
매 tick(0.5초)마다:
수위(t+1) = 수위(t) + (유입량 - 유출량) / 공간_단면적

유입량 = 기본유입(강수량, 패턴) + 범람유입(하천범람 여부, 시점)
유출량 = 펌프배수량(펌프상태) + 물막이효과(물막이판)
공간_단면적 = 공간유형 × 깊이
```

### 3.2 변수별 계산 계수

```typescript
// 강수량 → 기본 유입 속도 (cm/min)
const RAINFALL_INFLOW: Record<number, number> = {
  30:  0.8,
  50:  1.5,
  80:  3.2,
  100: 5.0,
  150: 9.0,
}

// 강우 패턴 배수
const PATTERN_MULTIPLIER = {
  steady:       1.0,   // 일정형
  concentrated: 2.4,   // 집중형 (피크 구간 기준)
}

// 펌프 상태 → 유출 속도 (cm/min)
const PUMP_OUTFLOW = {
  normal:  2.0,
  broken:  0.0,
}

// 물막이판 → 유입 지연 시간(초) + 유입량 감쇄
const BARRIER = {
  installed:     { delay: 180, reduction: 0.6 },
  not_installed: { delay: 0,   reduction: 1.0 },
}

// 공간 유형 → 단면적 계수 (클수록 수위 느리게 상승)
const SPACE_VOLUME = {
  underground_parking: 8.0,  // 지하주차장
  underground_road:    3.5,  // 지하차도
  semi_basement:       2.0,  // 반지하 (유입 지연 없음)
}

// 지하 깊이 → 위험수위까지 여유 (cm)
const DEPTH_CAPACITY = {
  B1: 280,
  B2: 560,
}

// 하천범람 → 추가 유입 (범람 시점 이후 급증)
const RIVER_OVERFLOW = {
  none:     { trigger_sec: Infinity, surge: 0   },
  overflow: { trigger_sec: 240,      surge: 8.0 }, // 4분 시점에 cm/min 급증
}
```

### 3.3 위험 구간 기준

```typescript
const DANGER_THRESHOLD = {
  warning:  30,  // cm — 차량 문 열기 어려움
  critical: 60,  // cm — 보행 불가
  fatal:    100, // cm — 탈출 불가
}
```

---

## 4. 데이터 구조 API 명세

### 4.1 SimulationConfig (입력)

```typescript
interface SimulationConfig {
  rainfall: 30 | 50 | 80 | 100 | 150          // mm/h
  rainPattern: 'steady' | 'concentrated'
  pumpStatus: 'normal' | 'broken'
  barrierInstalled: boolean
  spaceType: 'underground_parking' | 'underground_road' | 'semi_basement'
  depth: 'B1' | 'B2'
  riverOverflow: boolean
}
```

### 4.2 SimulationResult (출력)

```typescript
interface SimulationResult {
  id: string                    // uuid — 도감 저장용
  config: SimulationConfig
  curve: WaterLevelPoint[]      // 수위 시계열
  warningReachedAt: number | null   // 초 (30cm 도달 시각)
  criticalReachedAt: number | null  // 초 (60cm)
  fatalReachedAt: number | null     // 초 (100cm)
  maxWaterLevel: number         // 시뮬레이션 종료 시 최고 수위 cm
  grade: 'safe' | 'warning' | 'critical' | 'fatal'
  curveShape: 'flat' | 'gradual' | 'surge' | 'immediate' // 도감 카드 분류
  createdAt: string             // ISO timestamp
}

interface WaterLevelPoint {
  time: number       // 초
  waterLevel: number // cm
  phase: 'normal' | 'overflow_surge' | 'barrier_active'
}
```

### 4.3 CurveCard (도감)

```typescript
interface CurveCard {
  id: string
  result: SimulationResult
  label: string         // 자동생성: "집중형 + 범람 + 펌프고장"
  color: string         // curveShape에 따라 자동 배정
  grade: string         // Safe / Warning / Critical / Fatal
  unlockedAt: string
}
```

### 4.4 Badge

```typescript
interface Badge {
  id: string
  name: string
  description: string
  condition: (results: SimulationResult[]) => boolean
  unlockedAt: string | null
}

// 배지 목록
const BADGES: Badge[] = [
  { id: 'accurate_observer',    name: '정확한 관찰자',      condition: r => predictionError(r) <= 30  },
  { id: 'sharp_reasoner',       name: '예리한 추론가',      condition: r => predictionError(r) <= 60  },
  { id: 'brave_explorer',       name: '도전하는 탐험가',    condition: _ => true                       },
  { id: 'local_rain_observer',  name: '국지성 호우 관찰자', condition: r => completedMission(r, 2)    },
  { id: 'complex_disaster_analyst', name: '복합재해 분석가', condition: r => completedMission(r, 6)  },
  { id: 'flood_escape_graduate',name: 'Flood Escape Lab 수료', condition: r => allMissionsComplete(r) },
]
```

---

## 5. 컴포넌트 구조

```
App
├── MissionGate         — 단계적 변수 잠금해제 관리
├── ControlPanel        — 7개 변수 입력 슬라이더/토글
│   ├── RainfallSlider
│   ├── PatternToggle
│   ├── PumpToggle
│   ├── BarrierToggle
│   ├── SpaceTypeSelect
│   ├── DepthSelect
│   └── RiverOverflowToggle
├── PredictionInput     — 실행 전 위험수위 도달 시간 예측
├── SimulationView
│   ├── CrossSectionSVG — 단면도 애니메이션 (핵심)
│   └── WaterLevelChart — Recharts 실시간 곡선
├── ResultPanel         — 결과 + 배지 표시
└── CurveCollection     — 도감 (카드 그리드)
```

---

## 6. CrossSectionSVG 구현 핵심

단면도는 공간 유형에 따라 SVG viewBox를 동적으로 교체:

```typescript
// 반지하: 경사로 없음, 지면과 거의 같은 높이 → 즉시 유입
// 지하차도: 좁고 긴 형태, 양쪽 경사로
// 지하주차장: 넓은 공간, 단일 진입 경사로

// 수위 애니메이션: waterLevel(cm) → SVG y좌표 변환
const waterLevelToY = (cm: number, svgHeight: number, capacity: number) =>
  svgHeight - (cm / capacity) * svgHeight
```

물 색상은 위험도에 따라 변경:
- 0~30cm: `#60a5fa` (파랑)  
- 30~60cm: `#f59e0b` (주황)  
- 60cm+: `#ef4444` (빨강)

---

## 7. 구현 순서 (권장)

```
Week 1
  [x] 프로젝트 세팅 (Vite + React + Tailwind + Zustand)
  [ ] SimulationEngine 순수 함수로 구현 + 단위 테스트
  [ ] 시나리오 A·B·C 수치 검증

Week 2
  [ ] ControlPanel UI
  [ ] WaterLevelChart (Recharts)
  [ ] CrossSectionSVG — 지하주차장 버전 먼저

Week 3
  [ ] 나머지 공간 유형 SVG
  [ ] MissionGate (잠금해제 시스템)
  [ ] CurveCollection 도감

Week 4
  [ ] PredictionInput + 배지 시스템
  [ ] 시나리오 A·B·C 큐레이션 진입점
  [ ] 반응형 + 접근성 점검
```

---

## 8. Claude Code + MCP 활용 팁

1. **SimulationEngine 먼저 완성** — 나머지는 UI니까 나중에 갈아도 됨
2. **시나리오 A·B·C를 테스트 케이스로** — 엔진 검증 기준으로 고정
3. **Figma MCP로 SVG 초안 추출** — 직접 SVG 코딩보다 훨씬 빠름
4. **Obsidian MCP로 변수 명세 관리** — 숫자 바꿀 때 여기 먼저 수정, 코드에 반영
5. **Context7 MCP로 Recharts API 조회** — 실시간 업데이트 관련 props 자주 바뀜


---

## 9. UI 방향 전면 변경 (v2 — 2026-06-26)

> **결정:** 변수 조작 패널 방식 폐기 → 4화면 게임 플로우

### 9-1. 화면 구성

```
화면 1: Lobby
  FLOOD ESCAPE LAB
  오늘의 재난 미션 텍스트 (집중호우 50mm/h, 하천 범람)
  [PLAY] [시나리오 선택] [랭킹 보기]

화면 2: EquipmentShop (예산: 500)
  [물막이판 100] — 초반 유입 차단
  [배수펌프 300] — 수위 상승 속도 감소
  [경보시스템 150] — 대피 시작 시간 단축
  [탈출사다리 200] — 대피 성공률 증가

화면 3: PlayScreen
  좌/중앙: 침수되는 지하차도 2D/아이소메트릭 맵
  우측: 남은시간 · 현재수위 · 대피인원 · 위험도 게이지
  하단: [펌프] [물막이] [경보] [탈출유도] 스킬 버튼

화면 4: ResultScreen
  생존 등급: S/A/B/C/F
  대피 성공률 / 최고 수위 / 위험 감지 시간
  획득 배지: 빠른 경보, 펌프 마스터, 하천범람 생존자
  학습 포인트 (교육 메시지)
```

### 9-2. 컴포넌트 구조 (v2)

```
App (useGameStore)
├── LobbyScreen
│   └── MissionCard
├── EquipmentScreen
│   ├── BudgetBar
│   └── EquipmentCard × 4
├── PlayScreen
│   ├── FloodMap (SVG 애니메이션)
│   ├── StatusPanel (타이머/수위/인원/위험도)
│   └── SkillBar
└── ResultScreen
    ├── GradeDisplay (S/A/B/C/F)
    ├── StatRow × 3
    ├── BadgeList
    └── EducationCard
```

### 9-3. Zustand 상태 구조 (v2)

```typescript
interface GameStore {
  screen: 'lobby' | 'equipment' | 'play' | 'result'
  budget: number                  // 초기 500
  purchased: EquipmentId[]
  playState: PlayState | null
  result: ResultState | null
  buyEquipment(id: EquipmentId): void
  startPlay(): void
  activateSkill(id: SkillId): void
  finishGame(result: ResultState): void
  goLobby(): void
}

interface PlayState {
  timeLeft: number           // 초 (240 = 4분)
  currentWaterLevel: number  // cm
  evacuatedCount: number
  totalPeople: number
  dangerLevel: 'low' | 'medium' | 'high' | 'critical'
}

interface ResultState {
  grade: 'S' | 'A' | 'B' | 'C' | 'F'
  evacuationRate: number
  maxWaterLevel: number
  warningDetectedAt: number
  badges: string[]
  educationMessage: string
}
```

### 9-4. 등급 산정

```typescript
function calcGrade(evacuationRate: number, detectedAt: number): Grade {
  if (evacuationRate >= 90 && detectedAt <= 60)  return 'S'
  if (evacuationRate >= 75 && detectedAt <= 120) return 'A'
  if (evacuationRate >= 55)                       return 'B'
  if (evacuationRate >= 30)                       return 'C'
  return 'F'
}
```

### 9-5. 기존 엔진 재활용

- `engine.ts`의 `runSimulation()` 그대로 사용
- 장비 선택 → `SimulationConfig` 변환 후 엔진 실행
- `warningReachedAt`, `maxWaterLevel`, `grade` → ResultState에 직접 매핑


---

## 10. 리워드 시스템 (v2 — 2026-06-26)

### 10-1. 생존 등급

```typescript
function calcGrade(evacuationRate: number, detectedAt: number): 'S'|'A'|'B'|'C'|'F' {
  if (evacuationRate >= 90 && detectedAt <= 60)  return 'S'
  if (evacuationRate >= 75 && detectedAt <= 120) return 'A'
  if (evacuationRate >= 55)                       return 'B'
  if (evacuationRate >= 30)                       return 'C'
  return 'F'
}
```

### 10-2. 플레이 중 획득 배지

| 조건 | 배지 |
|------|------|
| 경보를 60초 내 발동 | 빠른 경보 |
| 펌프를 첫 30초 내 가동 | 펌프 마스터 |
| 하천범람 조건에서 생존 | 하천범람 생존자 |
| 예산 100 이하 남기고 S등급 | 알뜰 구조대 |
| 장비 없이 B등급 이상 | 맨몸 탈출 |
| 3회 연속 A등급 이상 | 재난 전문가 |

### 10-3. 결과 화면 학습 메시지

```typescript
const EDUCATION_MESSAGES: Record<Grade, string> = {
  S: '하천 범람 + 펌프 고장 조건에서 경보가 2분 빠르면 생존율이 35% 올라갑니다.',
  A: '배수펌프 하나로 수위 상승 속도를 절반으로 줄일 수 있었습니다.',
  B: '물막이판이 초반 유입을 막아 대피 시간을 확보했습니다.',
  C: '집중호우 50mm/h에서 무방비 지하차도는 4분 안에 위험 수위에 도달합니다.',
  F: '같은 강수량이라도 하천 범람 + 펌프 고장이 겹치면 수위는 일정하게 오르지 않고 급격히 상승합니다.',
}
```

### 10-4. 대피 성공률 계산

```typescript
// 장비 조합에 따른 대피 성공률 산정
function calcEvacuationRate(
  result: SimulationResult,
  purchased: EquipmentId[],
  warningDetectedAt: number  // 사용자가 경보 스킬 발동한 시각(초)
): number {
  let baseRate = 50
  if (result.grade === 'safe')     baseRate = 95
  if (result.grade === 'warning')  baseRate = 70
  if (result.grade === 'critical') baseRate = 40
  if (result.grade === 'fatal')    baseRate = 15

  if (purchased.includes('alarm'))  baseRate += 15
  if (purchased.includes('ladder')) baseRate += 10
  if (warningDetectedAt <= 60)      baseRate += 10
  if (warningDetectedAt <= 120)     baseRate += 5

  return Math.min(100, Math.max(0, baseRate))
}
```


---

## 11. AX 자동화 진행 현황

### Phase 1 — 설계 자동화 ✅ 완료 (2026-06-26)

- [x] Obsidian MCP: 기획 명세 읽기
- [x] 컴포넌트 구조 자동 설계 (v2 4화면 게임 플로우)
- [x] Notion MCP: 작업계획서 생성 및 v2 전면 반영
- [x] Figma MCP: 파일 생성 + UI 목업 확정 (파일키: 3yQFoZFSi9FwJZVmMuqZTJ)
- [x] 리워드 시스템 v2 설계 (등급/배지/학습메시지/대피율 함수)
- [x] Obsidian & Notion 양방향 동기화 완료

### Phase 2 — 코드 생성 (Claude Code) ← 현재 단계

- [ ] `gameStore.ts` — Zustand (screen/budget/purchased/playState/result)
- [ ] `LobbyScreen.tsx` — 미션 텍스트 + 3개 버튼
- [ ] `EquipmentScreen.tsx` — BudgetBar + EquipmentCard × 4
- [ ] `PlayScreen.tsx` — FloodMap(SVG) + StatusPanel + SkillBar
- [ ] `ResultScreen.tsx` — GradeDisplay + BadgeList + EducationCard
- [ ] `App.tsx` — screen 상태에 따른 4화면 라우팅

### Phase 3 — Playwright 자동 검증

- [ ] Lobby 렌더링 + PLAY 버튼
- [ ] 예산 초과 시 구매 버튼 비활성화
- [ ] 풀 플레이스루 (240초 시뮬레이션)
- [ ] 등급 + 배지 노출

### Phase 4 — 게임 루프 검증

- [ ] 최적 조합 → S등급 확인
- [ ] 무장비 → F등급 + 학습 메시지 확인
- [ ] localStorage 결과 누적

### Phase 5 — 문서화

- [ ] Obsidian: 개발 로그 업데이트
- [ ] Notion: 최종 명세 반영
