# Architecture

RoutineDiary는 **오프라인 우선(Offline-first)** 을 기본 전제로 설계된 루틴/투두/일기/챌린지 기록 앱입니다.  
UI는 로컬(Room) 데이터를 기반으로 즉시 반응하며, 로그인 이후에는 Firestore 연동/동기화를 확장할 수 있도록 구성했습니다.

---

## 핵심 목표

- **네트워크 유무와 관계없이 즉시 사용 가능**
- **Room을 중심으로 화면을 그린다(로컬이 기본 진실원천)**
- 로그인 이후에는 **원격(Firestore)과의 동기화/마이그레이션**을 붙일 수 있는 구조
- 카테고리 변경/삭제 이후에도 과거 기록 UI가 유지되는 **스냅샷 전략**

---

## 시스템 구성도

```text
App (Application)
 ├─ MobileAds.initialize()
 └─ Firestore persistence enabled

MainActivity (Compose Root)
 ├─ Theme / Typography (AppSettingsViewModel)
 ├─ InterstitialAdManager (전면광고 제어)
 └─ NavHost + BottomBar + BannerAdView

UI Screen (Compose)
 └─ ViewModel (Hilt)
     └─ Repository (Hilt)
         ├─ Local: Room DAO
         ├─ Session: DataStore(SessionManager)
         └─ Remote: Firestore/Auth (확장)
```

## 의존성 주입(Hilt) 구성

RoutineDiary는 Hilt를 사용해 **데이터 계층과 UI 계층을 명확히 분리**하고,  
앱 전반에서 필요한 의존성을 중앙에서 관리합니다.

### CoreModule
앱 전역에서 단일 인스턴스로 유지되어야 하는 **기본 인프라 레이어**를 제공합니다.

- `RoutineDiaryDatabase` (Room)
- 모든 DAO (`TodoDao`, `DiaryDao`, `CategoryDao`, `ChallengeDao`, `ChallengeTaskDao`, `ChallengeCheckDao`, `UserProfileDao`)
- `SessionManager` (DataStore 기반 세션 관리)
- `FirebaseAuth`
- `FirebaseFirestore`

> CoreModule은 “앱 전체에서 공유되는 핵심 도구 세트” 역할을 합니다.

---

### RepositoryModule
DAO, 세션, 인증 정보를 조합하여 **비즈니스 단위의 Repository**를 구성합니다.

- `AuthRepository`
- `TodoRepository`  
  - TodoDao + SessionManager + CategoryDao
- `DiaryRepository`  
  - DiaryDao + SessionManager
- `ChallengeRepository`  
  - ChallengeDao / TaskDao / CheckDao + SessionManager + CategoryDao
- `UserProfileRepository`
- `CategoryRepository`

> Repository는 로컬(Room), 원격(Firestore), 세션 상태를 **하나의 진입점**으로 통합합니다.

---

### 책임 분리 원칙
- **ViewModel**
  - 화면 상태(State) 관리
  - 사용자 입력(Intent)을 해석
  - Repository 호출을 통해 데이터 변경 요청
- **Repository**
  - 로컬/원격 데이터 접근
  - 로그인 상태(Session)에 따른 분기 처리
  - 데이터 정합성 및 가공 책임

---

## 데이터 흐름

### UI → 데이터 변경(쓰기) 흐름

```text
User Action (Compose)
 → ViewModel intent (e.g. add/update/toggle)
   → Repository
     → Room DAO write (insert/update/delete)
     → (Signed-in일 경우) Remote write (Firestore) 확장 가능
 → DAO의 Flow가 emit
 → ViewModel StateFlow 갱신
 → UI 리컴포지션
```

### 데이터 → UI(읽기/관찰) 흐름

```text
Room DAO observe (Flow)
 → Repository (필요 시 가공)
 → ViewModel state (StateFlow)
 → UI collectAsState()
```
핵심은  
**“UI는 상태만 보고”, “상태는 Flow / StateFlow에서 오며”,  
“실제 데이터 변경은 Repository를 통해 일어난다”** 입니다.

---

## 오프라인 우선 + 동기화 전략

### 로그인 전 (Guest)
- **Room만 사용**
- 모든 CRUD는 로컬에서 즉시 수행
- UI는 DAO의 `observe*()` Flow를 통해 실시간 반영

### 로그인 후 (Signed-in)
- 기존 로컬 데이터는 유지
- 세션 정보(uid 등)는 `SessionManager`가 관리
- 이후 전략은 다음 두 가지로 확장 가능

#### 1) 실시간 미러링
- Firestore 스냅샷을 수신해 Room에 반영
- UI는 **Room 데이터만 관찰**하여 화면을 렌더링

#### 2) 마이그레이션 업로드
- 기존 로컬 데이터를 Firestore로 업로드 (WorkManager 사용)
- 업로드 완료 후 실시간 연동 시작

### Firestore 오프라인 캐시
- 앱 시작 시 Firestore persistence를 활성화하여
  네트워크가 끊겨도 Firestore 측에서 캐시 및 재전송을 지원하도록 설정

---

## 데이터 설계 핵심

### 1) Category 스냅샷 전략
카테고리는 시간이 지나면서 이름이나 색상이 변경되거나 삭제될 수 있습니다.  
이를 대비해 투두(및 필요 시 다른 기록)에 다음 정보를 **스냅샷으로 함께 저장**합니다.

- `categoryId` (nullable)
- `categoryNameSnapshot`
- `categoryColorSnapshot`

**효과**
- 카테고리 수정·삭제 이후에도 과거 카드 UI가 당시 모습 그대로 유지
- FK가 null이 되어도 스냅샷 정보만으로 화면 재현 가능

---

### 2) Flow 기반 실시간 반응
DAO 레벨에서 `observeAll()` 과 같은 Flow 쿼리를 제공하여,  
데이터 변경 시 UI가 즉시 갱신되도록 합니다.

**적용 예시**
- Category: `observeAll()`
- Diary: `observeAll()`, `observeByDate()`, `searchByKeyword()`, `observeByEmotion()`
- Challenge / Task / Check: `observeAll()` 및 집계 쿼리 지원

---

### 3) 챌린지 집계 쿼리 (성능 고려)
챌린지는 체크 로그가 누적되기 때문에  
화면에 필요한 통계를 **DAO 레벨에서 직접 계산**합니다.

- 총 완료 수
- 특정 날짜 완료 수
- 기간 완료 수

ViewModel / Compose에서 반복 계산하지 않고  
Room SQL로 계산하여 **UI 렌더링 비용을 최소화**하는 방식입니다.

## Firestore 구조(권장 스키마)

```text
users/{uid}
├─ profile
│  └─ nickname, photoUrl, optional fields
├─ categories
│  └─ {categoryId}
├─ todos
│  └─ {todoId}
├─ diaries
│  └─ {diaryId}
├─ challenges
│  └─ {challengeId}
│     └─ tasks
│        └─ {taskId}
└─ challengeChecks
   └─ {checkId}
```

## 원칙

- 모든 컬렉션은 **사용자(uid) 기준으로 분리**한다
- 로컬(Room) 데이터 구조와 **유사한 형태를 유지**하여
  동기화 시 구조적 복잡도를 최소화한다
- 다중 디바이스 동기화를 고려해
  **문서 키 및 필드 규약을 일관되게 유지**한다

---

## 네비게이션 구조

- Bottom Navigation 기반의 주요 탭 구성
  - Home
  - Calendar
  - My Page
- Editor / Detail 화면은 **Stack 방식**으로 전환
- 외부 공유를 위한 **딥링크(Deep Link) 지원**
  - `routinediary://share?...`

---

## 광고(AdMob) 연동 구조

### 배너
- 메인 레이아웃 하단에 고정 노출  
  (`BannerAdView`)

### 전면 광고 (Interstitial)

#### 노출 트리거
- 챌린지 완료 시
- 하루 투두 전체 완료 시

#### 사용자 경험 보호 정책 (`InterstitialAdManager`)
- 세션당 최대 **2회** 노출
- 광고 간 **최소 3분 간격** 유지

> 광고는 화면 진입 시점이 아니라  
> **“의미 있는 이벤트 발생 시점”에만 연결**하여,  
> 앱의 핵심 플로우(기록 / 회고)를 방해하지 않도록 설계합니다.

---

## 에러 / 엣지 케이스 가이드

- **네트워크 없음**
  - Room 기반 기능은 정상 동작
  - UI 즉시 반응 유지

- **전면 광고 로드 실패**
  - 즉시 `onFinished` 처리 후 기존 로직 진행

- **카테고리 삭제**
  - FK가 끊겨도 스냅샷 정보로 과거 UI 유지

- **동기화 충돌 (확장 영역)**
  - 기본 정책은 **Last Write Wins**
  - 추후 `updatedAt` 기반 정책으로 고도화 여지
