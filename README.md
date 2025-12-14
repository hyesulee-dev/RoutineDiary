# RoutineDiary (루틴다이어리)

루틴, 할 일, 감정일기, 챌린지를 하나의 흐름으로 기록하고  
하루를 돌아볼 수 있도록 설계된 **오프라인 우선 개인 다이어리 앱**입니다.

이 프로젝트는  
“루틴을 체크하는 앱”과 “감정을 기록하는 일기 앱”이  
분리되어 있는 기존 방식의 불편함에서 출발했습니다.

---

## 📖 프로젝트 시작 배경

RoutineDiary는 단순한 할 일 관리 앱이 아닙니다.

- 오늘 무엇을 했는지  
- 얼마나 꾸준했는지  
- 그날의 감정은 어땠는지  

이 모든 기록을 **하루 단위의 하나의 흐름**으로 남기고,  
나중에 돌아보며 스스로를 평가하거나 자책하지 않도록  
**항상 긍정적인 회고 경험**을 제공하는 것을 목표로 합니다.

---

## 🎯 초기 앱 기획 방향

- 루틴 / 투두 / 일기 / 챌린지를 분리하지 않는다
- 로그인 없이도 완전 사용 가능해야 한다
- 나중에 로그인해도 데이터가 사라지지 않아야 한다
- 카테고리를 바꿔도 과거 기록의 모습은 유지되어야 한다
- 숫자와 통계는 보여주되, 상처 주는 피드백은 하지 않는다

---

## 🧩 핵심 설계 원칙

### 1. 오프라인 우선 (Offline-first)
- 로그인 전: 모든 데이터는 **Room 로컬 DB**에만 저장
- 로그인 후: Firestore로 마이그레이션 + 실시간 동기화
- 네트워크 상태와 상관없이 앱은 항상 즉시 반응

### 2. 카테고리 스냅샷 전략
- 투두 생성 시 카테고리의 **이름/색상을 스냅샷으로 함께 저장**
- 이후 카테고리가 수정·삭제되더라도  
  과거 기록 화면은 당시 모습 그대로 유지

### 3. 회고 중심 UX
- 투두: 성과를 숫자와 기록으로 보여줌
- 챌린지: 패턴과 습관 흐름을 분석
- 일기: 감정 분포를 시각화하고, 항상 긍정적인 코멘트 제공

---

## 🧠 핵심 기능

### 루틴 / 투두 (Todo)
- 날짜 기반 투두 관리 (LocalDate)
- 완료/미완료 토글 즉시 반영 (Flow 기반)
- 카테고리 선택 및 색상 표시
- 카테고리 변경·삭제 이후에도 과거 투두 화면 유지 (스냅샷 전략)
- 전체 완료 시 이벤트 트리거 (광고 등)

### 감정 일기 (Diary)
- 날짜별 감정 이모지 + 텍스트 기록
- 사진 첨부 지원
- 달력 / 리스트 기반 조회
- 감정별 / 키워드 검색
- 기간별 일기 조회 (통계 / 보관함 용)

### 챌린지 (Challenge)
- 기간(start/end)을 가지는 챌린지
- 챌린지 내 체크리스트(Task) 구성
- 일자별 체크 로그 기록
- 총 완료 수 / 일별 완료 수 / 기간 완료 수 집계 쿼리 제공
- 챌린지 완료 시 이벤트 트리거 (광고 등)

### 계정 / 세션
- 로그인 없이도 모든 기능 사용 가능
- 로그인 시 기존 로컬 데이터 유지
- 세션 정보(uid, 이름, 프로필 사진)를 DataStore로 관리

### 광고 (AdMob)
- 하단 배너 광고 상시 노출
- 전면 광고(Interstitial)
  - 챌린지 완료 시
  - 오늘의 투두 전체 완료 시
- 세션 제한 정책
  - 세션당 최대 2회
  - 광고 간 최소 3분 간격

---

## 🛠 기술 스택

### Android
- Kotlin
- Jetpack Compose
- Material 3
- Navigation Compose
- ViewModel + StateFlow

### 로컬 데이터
- Room
- DAO + Flow 기반 실시간 관찰
- LocalDate TypeConverter
- Repository 패턴

### 비동기 / 백그라운드
- Kotlin Coroutines
- Flow
- WorkManager (마이그레이션)

### 의존성 주입
- Hilt  
  - Application / Activity / ViewModel / Repository / Worker 전반 적용

### 서버 / 클라우드
- Firebase Auth
- Firebase Firestore  
  - 오프라인 캐시 활성화 (persistenceEnabled)

### 설정 / 세션
- DataStore (Preferences)

### 광고
- Google AdMob
  - Banner
  - Interstitial

---

## 🏗 아키텍처 개요

```text
UI (Jetpack Compose)
└─ ViewModel
   └─ Repository
      ├─ Local (Room DAO)
      ├─ Session (DataStore)
      └─ Remote (Firestore)

```

### 설계 원칙

- UI는 **ViewModel이 제공하는 상태(StateFlow)** 만 관찰한다
- ViewModel은 **Repository 인터페이스**에만 의존한다
- Repository는 **로컬 데이터, 원격 데이터, 세션 상태를 통합 관리**한다
- 네트워크 상태와 관계없이 UI는 **항상 즉시 반응**하도록 설계한다

---

## 🔁 오프라인 우선 + 동기화 전략

### 기본 동작 흐름

#### 로그인 전
- 모든 데이터는 **Room 로컬 DB**에만 저장
- 네트워크 연결 여부와 무관하게 앱의 모든 기능 사용 가능

#### 로그인 후
- 기존 로컬 데이터는 **유지**
- Firestore와 연동 가능한 상태로 전환
- 이후 데이터는 로컬과 원격이 함께 관리됨

### Firestore 동작 설정
- 오프라인 퍼시스턴스 활성화
- 네트워크 복구 시 Firestore가 자동으로 동기화 수행

---

## 🗂 데이터 설계 핵심

### Category 스냅샷 전략
- 투두 생성 시 다음 정보를 함께 저장
  - `categoryId` (nullable)
  - `categoryNameSnapshot`
  - `categoryColorSnapshot`
- 카테고리 수정·삭제 이후에도
  - 과거 투두 카드의 이름과 색상을 그대로 유지
  - 과거 기록의 UI 일관성 보장

### 실시간 반응 구조
- 모든 주요 엔티티(Category, Todo, Diary, Challenge)는
  - Flow 기반 `observe` 쿼리 제공
  - 데이터 변경 시 UI가 즉시 갱신됨


## 🔥 Firebase / Firestore 구조


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

- 모든 컬렉션은 **사용자(uid) 기준으로 분리**되어 관리된다
- 로컬(Room) 데이터 구조와 **최대한 유사한 형태로 설계**하여
  동기화 시 구조적 복잡도를 최소화한다
- 추후 **다중 디바이스 간 실시간 동기화 확장**을 고려한 스키마 구성

---

## 📐 네비게이션 구조

- Bottom Navigation 기반의 주요 탭 구성
  - Home
  - Calendar
  - My Page
- Editor / Detail 화면은 **Stack 방식**으로 전환
- 외부 공유를 위한 **딥링크(Deep Link) 지원**
  - `routinediary://share?...`

---

## 📢 광고 노출 정책

### 배너 광고
- 메인 화면 하단에 고정 노출

### 전면 광고 (Interstitial)
- 노출 트리거
  - 챌린지 완료 시
  - 하루 투두 전체 완료 시

### 사용자 경험 보호 정책
- 세션당 최대 **2회** 노출
- 광고 간 **최소 3분 간격** 유지

