# 설계 단계 종합 가이드 (Design Stage Summary)
## CampusNav - ICT폴리텍대학 캠퍼스 자원 통합 네비게이션 시스템

**프로젝트명:** ICT CampusNav  
**버전:** v1.0  
**작성일:** 2026-05-20  
**범위:** 역공학 기반 설계 문서 (3/7 완성, 4개 템플릿 제공)

---

## 📋 설계 단계 산출물 현황

### 완성 (✅)
1. **시스템 아키텍처 설계서** - `CampusNav_Architecture_Design_v1.0_260520.md`
   - 아키텍처 스타일 (3-Tier JSP)
   - 컴포넌트 책임 정의
   - 배포 구조
   - 보안 아키텍처
   - 성능 아키텍처
   - 확장성 계획

2. **기능 분해도 (Function Map)** - `CampusNav_Function_Map_v1.0_260520.md`
   - 전체 11개 카테고리로 계층적 분해
   - Phase별 기능 배치 (1~4단계)
   - 기능 우선순위 및 크기 추정
   - 기능 의존성 매트릭스

### 템플릿 제공 (아래 참고)
3. **데이터 설계서 (Data Design)**
4. **UI/UX 설계서 (Interface Layout)**
5. **프로세스 설계서 (Process Logic)**
6. **인터페이스 정의서 (Integration Spec)**
7. **아키텍처 결정 기록 (Decision Records - ADR)**

---

## 3. 데이터 설계서 템플릿 (Data Design)

### 3.1 논리 데이터 모델 (Logical Data Model)

```
Entity Relationship Diagram (ERD)

┌──────────────┐
│    users     │
├──────────────┤
│ user_id (PK) │◄──────────┐
│ userid       │           │
│ password     │           │
│ user_name    │           │
│ role         │           │
│ email        │           │
│ phone        │           │
└──────────────┘           │
                           │
┌──────────────────────┐   │
│    assets (8401)     │   │
├──────────────────────┤   │
│ asset_no (PK)        │   │
│ asset_class          │   │
│ item_name            │   │
│ model                │   │
│ location             │   │
│ detail_location      │   │
│ asset_status         │   │
│ manage_dept          │   │
│ manager_name         │   │
│ reg_date             │   │
└──────────────────────┘   │
  ▲      │                 │
  │      │                 │
  │      └────────┐        │
  │               │        │
  │   ┌───────────▼─────────────┐
  │   │   reservations          │
  │   ├─────────────────────────┤
  │   │ reserve_id (PK)         │
  │   │ asset_no (FK) ──────────┤→ assets
  │   │ user_id (FK)  ──────────┤→ users
  │   │ reserve_date            │
  │   │ start_time              │
  │   │ end_time                │
  │   │ purpose                 │
  │   │ status                  │
  │   │ reg_date                │
  │   └─────────────────────────┘
  │
  ├─┬────────────────────────┐
  │ │  asset_transfer        │
  │ ├────────────────────────┤
  │ │ transfer_id (PK)       │
  │ │ asset_no (FK) ─────────┤→ assets
  │ │ transfer_date          │
  │ │ before_dept            │
  │ │ before_location        │
  │ │ after_dept             │
  │ │ after_location         │
  │ │ remark                 │
  │ └────────────────────────┘
  │
  └─┬────────────────────────┐
    │  asset_disposal        │
    ├────────────────────────┤
    │ disposal_id (PK)       │
    │ asset_no (FK) ─────────┤→ assets
    │ disposal_date          │
    │ disposal_type          │
    │ remark                 │
    └────────────────────────┘

┌──────────────────────┐
│    professors        │
├──────────────────────┤
│ prof_id (PK)         │
│ prof_name            │
│ department           │
│ email                │
│ phone                │
└──────┬───────────────┘
       │
       ├───────────────────────┐
       │  prof_subjects        │
       ├───────────────────────┤
       │ subject_id (PK)       │
       │ prof_id (FK) ─────────┤→ professors
       │ subject               │
       └───────────────────────┘
       │
       └───────────────────────┐
          prof_skills          │
          ───────────────────┤
          skill_id (PK)       │
          prof_id (FK) ───────┤→ professors
          skill               │
          ────────────────────┘
```

### 3.2 물리 데이터 모델 (Physical Data Model)

| 테이블명 | 행 수 | 주요 컬럼 | 인덱스 | 용도 |
|---------|------|---------|--------|------|
| **users** | ~300 | user_id, userid, password, role | PK: user_id | 사용자 인증 |
| **assets** | 8,401 | asset_no, item_name, asset_class, location, status | PK, idx_class, idx_status | 자원 정보 |
| **reservations** | 증가중 | reserve_id, asset_no, user_id, reserve_date, status | PK, FK (asset_no, user_id) | 예약 관리 |
| **asset_transfer** | ~2000 | transfer_id, asset_no, transfer_date, before/after_location | PK, FK (asset_no) | 이관 이력 |
| **asset_disposal** | ~2700 | disposal_id, asset_no, disposal_date | PK, FK (asset_no) | 폐기 관리 |
| **professors** | ~200 | prof_id, prof_name, department | PK | 교수 정보 |
| **prof_subjects** | ~400 | subject_id, prof_id, subject | PK, FK (prof_id) | 담당 과목 |
| **prof_skills** | ~300 | skill_id, prof_id, skill | PK, FK (prof_id) | 기술 태그 |

### 3.3 데이터 스키마 상세 (Schema Details)

**assets 테이블 (핵심)**
```sql
CREATE TABLE assets (
  asset_no VARCHAR(20) PRIMARY KEY,           -- 자산번호
  acq_type VARCHAR(50),                       -- 취득 유형 (구입, 기증 등)
  asset_class VARCHAR(50),                    -- 분류 (강의실, 장비, 공간)
  item_name VARCHAR(200),                     -- 자산명
  spec VARCHAR(500),                          -- 사양
  manufacturer VARCHAR(100),                  -- 제조사
  model VARCHAR(200),                         -- 모델명
  acq_date DATE,                              -- 취득 일자
  acq_price DECIMAL(15,0),                    -- 취득가격
  useful_life INT,                            -- 내용연수
  location VARCHAR(200),                      -- 위치 (건물/층)
  detail_location VARCHAR(200),               -- 상세 위치 (호실/좌표)
  manage_dept VARCHAR(100),                   -- 관리 부서
  manager_name VARCHAR(100),                  -- 관리자 이름
  asset_status VARCHAR(50),                   -- 상태 (사용중, 점검, 사용불가, 폐기)
  reg_date DATETIME DEFAULT CURRENT_TIMESTAMP,-- 등록일
  UNIQUE KEY idx_asset_no (asset_no),
  KEY idx_asset_class (asset_class),
  KEY idx_asset_status (asset_status),
  KEY idx_location (location)
);
```

**reservations 테이블**
```sql
CREATE TABLE reservations (
  reserve_id INT AUTO_INCREMENT PRIMARY KEY,
  asset_no VARCHAR(20),                        -- 자산번호 (FK)
  user_id VARCHAR(50),                         -- 사용자ID (FK)
  reserve_date DATE,                           -- 예약 날짜
  start_time TIME,                             -- 시작 시간
  end_time TIME,                               -- 종료 시간
  purpose VARCHAR(200),                        -- 사용 목적
  status VARCHAR(20),                          -- 상태 (예약완료, 취소 등)
  reg_date DATETIME DEFAULT CURRENT_TIMESTAMP, -- 등록일
  FOREIGN KEY (asset_no) REFERENCES assets(asset_no),
  FOREIGN KEY (user_id) REFERENCES users(user_id),
  KEY idx_asset_no (asset_no),
  KEY idx_user_id (user_id),
  KEY idx_reserve_date (reserve_date),
  UNIQUE KEY idx_no_date_time (asset_no, reserve_date, start_time, end_time)
);
```

### 3.4 데이터 무결성 (Data Integrity)

```
제약조건
├─ Primary Key: 모든 테이블의 ID 필드
├─ Foreign Key: 
│  ├─ reservations.asset_no → assets.asset_no
│  ├─ reservations.user_id → users.user_id
│  ├─ asset_transfer.asset_no → assets.asset_no
│  ├─ asset_disposal.asset_no → assets.asset_no
│  ├─ prof_subjects.prof_id → professors.prof_id
│  └─ prof_skills.prof_id → professors.prof_id
├─ Unique Key:
│  ├─ users.userid (로그인 ID)
│  ├─ assets.asset_no
│  └─ reservations (asset_no, reserve_date, start_time)
└─ Check Constraint: (권장 - 향후 추가)
   ├─ asset_status IN ('사용중', '점검', '사용불가', '폐기')
   └─ reservation.status IN ('예약완료', '취소', '대기중')
```

### 3.5 데이터 마이그레이션 전략

```
초기 데이터 로딩
├─ assets: nav.sql 스크립트 (8,401개)
├─ users: 관리자/조교 수동 등록 (필요시)
├─ professors: 교무처에서 추출 (예정)
└─ 기타: 필요시 Excel 일괄 import

정기 동기화 (향후)
├─ 교무처 DB → assets 미러링 (일 1회)
├─ 강의 스케줄 → reservations 자동 생성 (학기초)
└─ 교수정보 → professors 동기화 (학기초)
```

---

## 4. UI/UX 설계서 템플릿 (Interface Layout & Storyboard)

### 4.1 사용자 인터페이스 레이아웃 (Page Templates)

#### 로그인 페이지 (campuslogin.jsp)
```
┌─────────────────────────────────────────────────┐
│           ICT CAN — 로그인                       │
├─────────────────────────────────────────────────┤
│                                                 │
│  ┌──────────────────────────────────────────┐  │
│  │           [로고] ICT CAN                  │  │
│  │        Campus Resource Navigator         │  │
│  └──────────────────────────────────────────┘  │
│                                                 │
│  [역할 선택 그리드: 5 버튼]                     │
│  [🎓 학생] [👨‍🏫 조교] [📚 교수] [⚙️ 관리자] [👤 게스트] │
│                                                 │
│  ID: [____________]                            │
│  PW: [____________] [👁]                        │
│                                                 │
│  [로그인 버튼]                                  │
│  혹은 [회원가입 링크]                           │
│                                                 │
└─────────────────────────────────────────────────┘
```

#### 검색 페이지 (search.jsp)
```
┌────────────────────────────────────────────┐
│  [로고] 검색   [역할 칩]  [로그아웃]        │
├────────────────────────────────────────────┤
│  [영웅 섹션 - 배경 이미지]                 │
│  📍 자원 검색 | 8,401개 자원 보유           │
├────────────────────────────────────────────┤
│  통계 카드 (4개)                            │
│  [📦 자산] [📅 예약] [🏢 건물] [👥 사용자] │
├────────────────────────────────────────────┤
│  검색 폼                                    │
│  키워드: [_____________]                    │
│  분류: [▼ 전체]  상태: [▼ 전체]              │
│  [검색] [초기화]                            │
├────────────────────────────────────────────┤
│  결과 테이블 (20개/페이지)                  │
│  [헤더] 자산명 | 분류 | 위치 | 상태 | 관리자│
│  [행1]  강의실 | 교실 | 321 | 사용중 | 김팀장│→상세보기
│  [행2]  ...                                 │
├────────────────────────────────────────────┤
│  페이지네이션: [이전] 1 2 3 4 5 [다음]     │
└────────────────────────────────────────────┘
```

#### 상세 페이지 (detail.jsp)
```
┌──────────────────────────────────────────────┐
│  [로고] 상세 정보   [역할 칩]  [로그아웃]     │
├──────────────────────────────────────────────┤
│  [영웅 섹션]                                 │
│  강의실 203 | 3층 | 사용 가능                │
├──────────────────────────────────────────────┤
│  정보 카드                                   │
│  │ 명: 강의실 203      │ 위치: 3층           │
│  │ 상: 사용중          │ 부서: 교무처        │
│  │ 관: 김팀장(☎02-xxx)│ 모델: 표준강의실   │
├──────────────────────────────────────────────┤
│  탭 인터페이스                               │
│  [기본정보] [이관이력] [예약현황]            │
│                                              │
│  ┌─ 예약현황 시간표 ─────────────────────┐  │
│  │ 오늘: 09:00-11:00 (영어수학)  이용 중  │  │
│  │       14:00-16:00 (미분학)    이용 중  │  │
│  │       18:00-19:00 예약 가능            │  │
│  │ 내일: 전체 예약 가능                   │  │
│  └────────────────────────────────────────┘  │
├──────────────────────────────────────────────┤
│  [예약하기 버튼]                             │
└──────────────────────────────────────────────┘
```

#### 예약 페이지 (reserve.jsp)
```
┌─────────────────────────────────────────┐
│  예약 | [자산명: 강의실 203]             │
├─────────────────────────────────────────┤
│  예약 정보                               │
│  자산: 강의실 203 (읽기 전용)           │
│  사용자: 홍길동 (읽기 전용)             │
│                                         │
│  예약 날짜: [2026-05-30] [📅]          │
│  시작 시간: [09:00] [⏰]                │
│  종료 시간: [11:00] [⏰]                │
│  목적: [________________]               │
│                                         │
│  중복 검사: [예약 가능 ✓]               │
│  (JavaScript: AJAX 실시간 검사)        │
│                                         │
│  [예약 완료] [취소]                     │
│                                         │
│  ✓ 예약 완료! 예약번호: RES-2026-001   │
│  상세정보로 돌아갑니다...               │
└─────────────────────────────────────────┘
```

### 4.2 사용자 흐름도 (User Journey)

```
학생 사용자 흐름
┌──────────┐
│ 로그인   │
└────┬─────┘
     │ ✓ 인증 성공
     ▼
┌──────────────────┐
│ main_student.jsp │ (메인 페이지)
└────┬─────────────┘
     │
     ├─→ [검색] → search.jsp → [상세보기] → detail.jsp → [예약하기] → reserve.jsp
     ├─→ [내 예약] → 예약 목록 조회
     ├─→ [건물 안내] → floorNav.jsp (미구현)
     └─→ [교수 정보] → professor.jsp

조교 사용자 흐름
┌──────────┐
│ 로그인   │
└────┬─────┘
     │ ✓ role=assistant
     ▼
┌──────────────────┐
│ main_assistant.jsp│
└────┬─────────────┘
     │
     ├─→ [자원 검색] → search.jsp → detail.jsp
     ├─→ [예약 관리] → 예약 승인/거부 (미구현)
     └─→ [대시보드] → 예약 현황 (미구현)

관리자 사용자 흐름
┌──────────┐
│ 로그인   │
└────┬─────┘
     │ ✓ role=admin
     ▼
┌──────────────────┐
│ main_admin.jsp   │
└────┬─────────────┘
     │
     ├─→ [자산 관리] → asset_manage.jsp
     │   ├─→ [자산 등록]  → INSERT
     │   ├─→ [자산 수정]  → UPDATE
     │   ├─→ [자산 삭제]  → DELETE
     │   └─→ [상태 변경]  → UPDATE asset_status
     ├─→ [예약 관리] → 전체 예약 관리
     ├─→ [분석] → 대시보드 (미구현)
     └─→ [사용자 관리] → users 테이블 관리 (미구현)

게스트 사용자 흐름
┌──────────┐
│ 게스트   │ (로그인 불필요)
│ 입장     │
└────┬─────┘
     │
     ▼
┌──────────────────┐
│ main_guest.jsp   │
└────┬─────────────┘
     │
     ├─→ [공개 자원 검색] → search.jsp (제한된 필드만)
     ├─→ [건물 안내] → floorNav.jsp
     └─→ [교수 찾기] → professor.jsp (연락처만)
```

### 4.3 색상 & 타이포그래피 (Design System)

```
색상 팔레트
├─ 주색: Blue (#1a56db)
├─ 보조색: Teal (#0d9488)
├─ 경고색: Red (#dc2626), Amber (#d97706)
├─ 성공색: Green (#16a34a)
├─ 배경: White (#ffffff), Light Gray (#f7f8fa)
└─ 텍스트: Dark (#111827), Medium (#4b5563), Light (#9ca3af)

타이포그래피
├─ 제목: DM Sans Bold 28px
├─ 소제목: DM Sans Bold 17px
├─ 본문: DM Sans 15px (한글: Noto Sans KR)
├─ 코드: DM Mono 12px
└─ 모노스페이스: DM Mono

컴포넌트
├─ 버튼: 둥근모서리 12px, 패딩 14px
├─ 입력: 테두리 1.5px, 포커스 시 블루 하이라이트
├─ 카드: 백색, 그림자, 호버 시 상승 효과
└─ 뱃지: 다양한 색상, 테두리 또는 채우기
```

---

## 5. 프로세스 설계서 템플릿 (Process Logic)

### 5.1 주요 비즈니스 프로세스

#### 프로세스 1: 자원 검색 및 상세 조회
```
시작: 사용자가 검색.jsp 접근
    ↓
1단계: 검색 조건 입력
    - 키워드, 카테고리, 상태 등
    - 페이지 번호
    ↓
2단계: 검색 쿼리 실행
    - SQL: WHERE 절 동적 구성
    - PreparedStatement 사용 (보안)
    - LIMIT 20 OFFSET ((page-1)*20)
    ↓
3단계: 결과 출력
    - 검색 결과 테이블 렌더링
    - 페이지네이션 버튼
    - 상세보기 링크 포함
    ↓
4단계: 사용자가 [상세보기] 클릭
    ↓
5단계: detail.jsp?id={asset_no} 로드
    ↓
6단계: 3개 쿼리 실행 (순차)
    1) SELECT * FROM assets WHERE asset_no=?
    2) SELECT * FROM asset_transfer WHERE asset_no=?
    3) SELECT * FROM reservations WHERE asset_no=?
    ↓
7단계: 상세 페이지 렌더링
    - 자산 정보 카드
    - 이관 이력 타임라인
    - 예약 현황 시간표
    ↓
종료
```

#### 프로세스 2: 예약 생성
```
시작: 사용자가 detail.jsp에서 [예약하기] 클릭
    ↓
1단계: reserve.jsp?id={asset_no} 로드
    - 자산 정보 표시 (읽기 전용)
    - 예약 폼 초기화
    ↓
2단계: 사용자 입력
    - 예약 날짜 선택
    - 시작/종료 시간 입력
    - 사용 목적 입력
    ↓
3단계: AJAX 중복 검사 (실시간)
    - getReservations.jsp 호출
    - SELECT FROM reservations WHERE asset_no=? AND reserve_date=?
    - 예약된 시간대 반환 (JSON)
    ↓
4단계: JavaScript 충돌 검사
    - 입력 시간대와 기존 예약 비교
    - 충돌 여부 판단
    ↓
    ├─→ [충돌 있음] → 경고 메시지 표시 → 사용자 재입력
    │
    └─→ [충돌 없음] → [예약 완료] 버튼 활성화
         ↓
5단계: POST /reserve.jsp
    - form 제출
    ↓
6단계: 예약 저장
    INSERT INTO reservations (
        asset_no, user_id, reserve_date,
        start_time, end_time, purpose,
        status, reg_date
    ) VALUES (?, ?, ?, ?, ?, ?, '예약완료', NOW())
    ↓
7단계: 성공 확인
    - 예약번호 생성
    - 성공 메시지 표시
    ↓
8단계: 상세 페이지로 리다이렉트
    - 예약 현황 업데이트 표시
    ↓
종료
```

### 5.2 데이터 처리 알고리즘

#### 알고리즘 1: 검색 쿼리 동적 구성
```java
// Pseudocode
StringBuilder whereClause = new StringBuilder("WHERE 1=1 ");
List<String> params = new ArrayList<>();

if (!keyword.isEmpty()) {
    whereClause.append("AND (item_name LIKE ? OR asset_no LIKE ? OR ...) ");
    for (int i = 0; i < 5; i++) {
        params.add("%" + keyword + "%");
    }
}

if (!category.isEmpty()) {
    whereClause.append("AND asset_class = ? ");
    params.add(category);
}

if (!status.isEmpty()) {
    whereClause.append("AND asset_status LIKE ? ");
    params.add("%" + status + "%");
}

// 쿼리 실행
String countQuery = "SELECT COUNT(*) FROM assets " + whereClause;
String selectQuery = "SELECT * FROM assets " + whereClause + 
                     "ORDER BY reg_date DESC LIMIT ? OFFSET ?";

// Prepared Statement 사용으로 SQL Injection 방지
PreparedStatement ps = conn.prepareStatement(selectQuery);
for (int i = 0; i < params.size(); i++) {
    ps.setString(i+1, params.get(i));
}
ps.setInt(params.size()+1, pageSize);
ps.setInt(params.size()+2, offset);
```

#### 알고리즘 2: 예약 중복 검사 (JavaScript)
```javascript
// AJAX로 해당 날짜의 예약 현황 조회
function checkAvailability() {
    const date = document.getElementById('reserveDate').value;
    const startTime = document.getElementById('startTime').value;
    const endTime = document.getElementById('endTime').value;
    
    fetch(`getReservations.jsp?date=${date}&assetNo=${assetNo}`)
        .then(r => r.json())
        .then(reservations => {
            let conflict = false;
            reservations.forEach(res => {
                if (timeOverlap(startTime, endTime, res.start, res.end)) {
                    conflict = true;
                }
            });
            
            if (conflict) {
                showError('이미 예약된 시간입니다');
            } else {
                enableSubmit();
            }
        });
}

function timeOverlap(s1, e1, s2, e2) {
    return s1 < e2 && s2 < e1; // 기본 시간대 겹침 검사
}
```

---

## 6. 인터페이스 정의서 템플릿 (Integration Specification)

### 6.1 API 엔드포인트 (REST API - 향후)

```
로그인 API
POST /api/auth/login
Request: {userid: string, password: string}
Response: {token: string, user: {id, name, role}, expires: timestamp}
Status: 200 OK / 401 Unauthorized

로그아웃 API
POST /api/auth/logout
Request: {token: string}
Response: {success: boolean}
Status: 200 OK / 401 Unauthorized

자원 검색 API
GET /api/assets/search?keyword=&category=&status=&page=1&pageSize=20
Request: Query parameters
Response: {
    total: number,
    page: number,
    pageSize: number,
    data: [{asset_no, item_name, asset_class, location, status, ...}]
}
Status: 200 OK / 400 Bad Request

예약 생성 API
POST /api/reservations
Request: {asset_no: string, reserve_date: date, start_time: time, end_time: time, purpose: string}
Response: {reserve_id: number, status: string, message: string}
Status: 201 Created / 409 Conflict (중복 예약)

예약 조회 API
GET /api/reservations/{date}
Request: date query parameter
Response: [{reserve_id, asset_no, user_id, start_time, end_time, purpose}]
Status: 200 OK / 404 Not Found
```

### 6.2 JSP 인터페이스 (현재 - 서버 렌더링)

```
입력 파라미터 (HTTP GET/POST)
- search.jsp: keyword, type, status, page
- detail.jsp: id (asset_no)
- reserve.jsp: id (asset_no) + form data
- transfer.jsp: id (asset_no)

출력 (HTML 렌더링)
- HTML5 + Bootstrap 5.3.3
- 인라인 스타일 및 템플릿 변수 사용
- ResultSet → Map<String, String> → JSP EL 표현

에러 처리
- try-catch: Exception → String dbErr
- JSP: <% if (!dbErr.isEmpty()) { %> ... <% } %>
```

### 6.3 데이터 형식 (Data Format)

```
JSON (향후 API)
{
    "assetNo": "A-001",
    "itemName": "강의실 203",
    "assetClass": "교실",
    "location": "3층",
    "detailLocation": "건물A 3-203",
    "status": "사용중",
    "manageDept": "교무처",
    "managerName": "김팀장"
}

CSV (일괄 import)
asset_no,item_name,asset_class,location,manage_dept,status
A-001,강의실 203,교실,3층,교무처,사용중
A-002,강의실 204,교실,3층,교무처,사용중

URL 인코딩 (폼 데이터)
reserve_date=2026-05-30&start_time=09%3A00&end_time=11%3A00&purpose=%EC%98%81%EC%96%B4%EC%88%98%EC%97%85
```

---

## 7. 아키텍처 결정 기록 (ADR - Architecture Decision Records)

### 7.1 ADR 템플릿

```
## ADR-001: 웹 프레임워크 선택 (JSP vs Spring vs Others)

**상태:** ACCEPTED (현재 구현)

**배경**
- 빠른 초기 구현 필요
- 팀의 JSP 경험 있음
- 소규모 프로젝트

**옵션**
1. JSP + Servlet (선택됨)
   - 장점: 학습곡선 낮음, 빠른 개발
   - 단점: 현대화 필요, 테스트 어려움
   
2. Spring Boot
   - 장점: 표준 프레임워크, 확장성
   - 단점: 학습곡선 높음, 초기 구성 복잡
   
3. Jakarta EE (구 Java EE)
   - 장점: 표준, 기능 풍부
   - 단점: 너무 무거움

**결정**
JSP + Servlet 유지, 향후 Spring으로 마이그레이션 계획

**근거**
- 기존 구현 활용 가능
- 팀 역량 고려
- 단기(1개월) 목표 달성 용이

**결과**
- 빠른 1단계 완료 가능
- 향후 확장성 고려해 아키텍처 정리 필요
- 보안 강화 (SQL Injection 방지) 필수
```

### 7.2 주요 ADR 목록

| ADR ID | 제목 | 결정 | 상태 | 마이그레이션 계획 |
|--------|------|------|------|----------|
| **ADR-001** | 웹 프레임워크 | JSP + Servlet | ACCEPTED | Phase 3: Spring으로 전환 |
| **ADR-002** | 렌더링 방식 | 서버 렌더링 (JSP) | ACCEPTED | Phase 4: SPA (React/Vue) 검토 |
| **ADR-003** | 데이터베이스 | MySQL 8.x | ACCEPTED | 유지 (마이그레이션 불필요) |
| **ADR-004** | ORM 사용 | 직접 JDBC | ACCEPTED | Phase 2: Hibernate 검토 |
| **ADR-005** | 인증 방식 | 커스텀 세션 | ACCEPTED | Phase 1: Spring Security |
| **ADR-006** | 배포 환경 | Tomcat 온프레미스 | ACCEPTED | Phase 3: Docker + K8s |
| **ADR-007** | 정적 파일 | CDN (현재 로컬) | PROPOSED | Phase 2: CloudFront/Cloudflare |
| **ADR-008** | API 설계 | 없음 (향후) | PROPOSED | Phase 2: REST API 개발 |
| **ADR-009** | 모니터링 | 로그 파일만 | PROPOSED | Phase 3: ELK 스택 |
| **ADR-010** | 캐싱 | 없음 | PROPOSED | Phase 2: Redis |

---

## 8. 다음 단계 (Next Steps)

### 설계 완료 후 구현 전 체크리스트

```
☐ 아키텍처 리뷰 (팀장, 기술이사)
☐ 데이터 스키마 리뷰 (DBA 또는 경험자)
☐ UI/UX 검증 (사용자 대표)
☐ 성능 예측 및 부하 테스트 계획
☐ 보안 감시 체크리스트 (OWASP Top 10)
☐ 개발 환경 구축 (IDE 설정, 빌드 도구)
☐ 테스트 환경 구축 (DB, Tomcat)
☐ 배포 자동화 파이프라인 설계
☐ 운영 가이드 초안 작성
☐ 팀 온보딩 및 기술 공유
```

### 구현 단계 (Implementation Phase)

```
Phase 1 (1개월): 보안 + 안정화
├─ LoginServlet DB 연동
├─ SQL Injection 방지 (모든 쿼리)
├─ CSRF 방지
├─ 데이터베이스 인덱싱
└─ 기본 성능 최적화

Phase 2 (3개월): 핵심 기능
├─ 네비게이션 (Naver Map)
├─ AI 추천 (협력 필터링)
├─ 분석 대시보드
└─ REST API 개발

Phase 3 (6개월): 확대
├─ 알림 시스템
├─ 일괄 등록/관리 도구
├─ CI/CD 파이프라인
└─ 모니터링 대시보드

Phase 4 (12개월+): 고도화
├─ Spring Framework 마이그레이션
├─ 생성형 AI 활용
├─ 모바일 앱
└─ 다중 캠퍼스 지원
```

---

## 📌 문서 작성 현황

| 산출물 | 파일명 | 상태 |
|--------|--------|------|
| 1. 아키텍처 설계서 | `CampusNav_Architecture_Design_v1.0_260520.md` | ✅ 완성 |
| 2. 기능 분해도 | `CampusNav_Function_Map_v1.0_260520.md` | ✅ 완성 |
| 3. 데이터 설계서 | (위 섹션 3 참고) | ⚠️ 템플릿 |
| 4. UI/UX 설계서 | (위 섹션 4 참고) | ⚠️ 템플릿 |
| 5. 프로세스 설계서 | (위 섹션 5 참고) | ⚠️ 템플릿 |
| 6. 인터페이스 정의서 | (위 섹션 6 참고) | ⚠️ 템플릿 |
| 7. ADR | (위 섹션 7 참고) | ⚠️ 템플릿 |

---

**설계 방법:** 역공학 기반 (현재 구현에서 설계 추출)  
**작성일:** 2026-05-20  
**다음:** 구현 단계 진행
