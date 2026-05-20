# 시스템 아키텍처 설계서 (Architecture Design)
## CampusNav - ICT폴리텍대학 캠퍼스 자원 통합 네비게이션 시스템

**프로젝트명:** ICT CampusNav  
**버전:** v1.0  
**작성일:** 2026-05-20  
**방법:** 역공학 기반 설계 추출 (Reverse Engineering)

---

## 1. 시스템 개요 (System Overview)

### 1.1 목표 (Objectives - 기획서 기반)

```
기획서 목표
├─ 1) 교내 자원 통합 검색 플랫폼 구축
│  └─ 통합 DB 기반 단일 검색창 제공
├─ 2) 맞춤형 추천 서비스 제공
│  └─ 사용자 유형/목적 기반 자동 추천
├─ 3) 실내외 연계 네비게이션 기능
│  └─ GPS 기반 경로 안내 (현재 미구현)
└─ 4) 데이터 기반 운영 효율 개선
   └─ 공간 이용 현황 분석/시각화

현재 구현 상태 (v1.0)
├─ ✅ 통합 검색 플랫폼 (기본)
├─ ❌ 맞춤형 추천 (설계 필요)
├─ ⚠️ 네비게이션 (Placeholder만 존재)
└─ ❌ 운영 분석 (미구현)
```

### 1.2 아키텍처 스타일 (Architecture Style)

```
현재: JSP-직접 JDBC 아키텍처 (3-Tier JSP)
├─ Presentation Tier: JSP (서버 렌더링)
├─ Business Logic: JSP 내 JDBC (미분리)
└─ Data Tier: MySQL

문제점
├─ 비즈니스 로직이 JSP에 산재
├─ 코드 재사용성 낮음
├─ 테스트 곤란
└─ 유지보수 어려움

권장: Servlet-JSP-JDBC 분리 구조로 점진적 전환
```

---

## 2. 아키텍처 구성 (Architecture Components)

### 2.1 계층별 구성도 (Layered Architecture)

```
┌─────────────────────────────────────────────────────────────────┐
│                    Presentation Layer (UI)                       │
│  ┌───────────────┬────────────────┬────────────────┬──────────┐ │
│  │ campuslogin   │    search      │    detail      │ reserve  │ │
│  │    .jsp       │    .jsp        │    .jsp        │  .jsp    │ │
│  ├───────────────┼────────────────┼────────────────┼──────────┤ │
│  │ main_*        │   transfer     │  professor     │ etc.     │ │
│  │    .jsp       │    .jsp        │    .jsp        │          │ │
│  └───────────────┴────────────────┴────────────────┴──────────┘ │
│  + Bootstrap 5.3.3, Bootstrap Icons, DM Sans Font              │
└──────────────────────────────┬──────────────────────────────────┘
                               │ JDBC + 직접 쿼리
┌──────────────────────────────┴──────────────────────────────────┐
│              Application/Servlet Layer (Logic)                   │
│  ┌─────────────┬──────────────┬────────────┬────────────────┐  │
│  │ LoginServ.  │ LogoutServ.  │ GuestServ. │ VisitorServ.   │  │
│  └─────────────┴──────────────┴────────────┴────────────────┘  │
│                    (web.xml에 등록)                              │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ 미사용 Servlet들 (web.xml 미등록)                          │  │
│  │ AssetServ, SearchServ, DetailServ, ReserveServ 등       │  │
│  └─────────────────────────────────────────────────────────┘  │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │        공유 유틸리티                                       │  │
│  │  - DBUtil.java: 공유 연결 헬퍼                            │  │
│  │  - 기타 유틸리티 부재 (권장: QueryHelper, 예외처리 등)    │  │
│  └─────────────────────────────────────────────────────────┘  │
└──────────────────────────────┬──────────────────────────────────┘
                               │ MySQL JDBC Driver
┌──────────────────────────────┴──────────────────────────────────┐
│              Data Access Layer (Database)                         │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │         MySQL 8.x (localhost:3306/campusnav)            │  │
│  │  Tables:                                                │  │
│  │  ├─ users (로그인 사용자)                                 │  │
│  │  ├─ assets (자원 - 8,401개)                              │  │
│  │  ├─ reservations (예약)                                  │  │
│  │  ├─ asset_transfer (이관)                                │  │
│  │  ├─ asset_disposal (폐기)                                │  │
│  │  ├─ professors (교수)                                    │  │
│  │  ├─ prof_subjects (교수 과목)                            │  │
│  │  └─ prof_skills (교수 기술)                              │  │
│  └──────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘

배포 환경
├─ 웹 서버: Apache Tomcat 9.x (localhost:8080)
├─ 데이터베이스: MySQL 8.x (localhost:3306)
├─ 실행 환경: Java 11+ (Servlet 4.0, JSP 2.3)
└─ 개발 환경: Windows 11, Git
```

### 2.2 컴포넌트별 책임 (Component Responsibilities)

#### Presentation Tier

| JSP/페이지 | 책임 | 현황 |
|-----------|------|------|
| **campuslogin.jsp** | 로그인 폼 + 역할 선택 | ✅ 구현 |
| **main_*.jsp** | 역할별 메인 페이지 | ✅ 구현 (6가지) |
| **search.jsp** | 자원 검색 + 필터링 + 페이지네이션 | ✅ 구현 |
| **detail.jsp** | 자산 상세조회 + 이관이력 + 예약현황 | ✅ 구현 |
| **reserve.jsp** | 예약 생성 + AJAX 중복 검사 | ✅ 구현 |
| **transfer.jsp** | 이관 이력 조회 | ✅ 구현 |
| **professor.jsp** | 교수 정보 + 과목/기술 태그 | ✅ 구현 |
| **asset_manage.jsp** | 관리자 자산 CRUD | ✅ 부분 |
| **floorNav.jsp** | 층별 네비게이션 (Placeholder) | ⚠️ 미완성 |
| **navigationTest1.jsp** | 네비게이션 테스트 | ⚠️ 미완성 |

#### Application/Servlet Tier

| Servlet | 책임 | 등록상태 | 현황 |
|---------|------|---------|------|
| **LoginServlet** | 인증 + 세션 생성 + 역할 설정 | ✅ web.xml | ⚠️ 하드코딩 자격증명 |
| **LogoutServlet** | 세션 무효화 | ✅ web.xml | ✅ 구현 |
| **GuestServlet** | 게스트 모드 진입 | ✅ web.xml | ✅ 구현 |
| **VisitorServlet** | 외부인 모드 진입 | ✅ web.xml | ✅ 구현 |
| **AssetServlet** | (미사용) | ❌ 미등록 | 미구현 |
| **SearchServlet** | (미사용) | ❌ 미등록 | 미구현 |
| **DetailServlet** | (미사용) | ❌ 미등록 | 미구현 |
| **ReserveServlet** | (미사용) | ❌ 미등록 | 미구현 |

#### Data Access Tier

| 컴포넌트 | 책임 |
|---------|------|
| **DBUtil.java** | JDBC 연결 헬퍼 (getConnection, close) |
| **MySQL Driver** | JDBC 드라이버 (mysql-connector-j) |
| **campusnav DB** | 8개 테이블, 8,401개 자산 데이터 |

### 2.3 데이터 흐름 (Data Flow)

#### 로그인 플로우
```
campuslogin.jsp (사용자 입력)
    ↓ POST /login
LoginServlet
    ├─ 자격증명 검증 (현재: 하드코딩 배열)
    ├─ 역할 조회 (필요: users 테이블에서)
    └─ Session 설정
        ├─ loginUser = 사용자ID
        ├─ loginName = 사용자명
        └─ loginRole = 역할 (student/assistant/professor/admin/guest/visitor)
    ↓ 리다이렉트
main_{role}.jsp 렌더링
```

#### 자원 검색 플로우
```
search.jsp 로드 (GET /search.jsp)
    ├─ 검색 조건 입력 (keyword, type, status)
    └─ 페이지네이션 (page=N)
    ↓
검색 쿼리 구성
    ├─ SELECT COUNT(*) → 총 건수 계산
    ├─ SELECT ... LIMIT 20 OFFSET ((page-1)*20) → 페이지 데이터
    └─ 동적 WHERE 절 구성
        ├─ keyword LIKE %?% (5개 필드)
        ├─ asset_class = ?
        └─ asset_status LIKE %?%
    ↓
ResultSet → Map<String,String> 변환
    ↓
search.jsp 렌더링
    ├─ 결과 테이블 출력
    ├─ 페이지네이션 버튼
    └─ 상세 페이지 링크
```

#### 자산 상세 조회 플로우
```
detail.jsp?id={asset_no}
    ↓
3개 쿼리 실행 (순차)
    ├─ SELECT * FROM assets WHERE asset_no=?
    │  └─ 자산 기본정보 조회
    ├─ SELECT ... FROM asset_transfer WHERE asset_no=?
    │  └─ 이관 이력 조회 (최신순)
    └─ SELECT ... FROM reservations WHERE asset_no=?
       └─ 향후 예약 현황 조회 (조인: users)
    ↓
detail.jsp 렌더링
    ├─ 자산 정보 카드
    ├─ 이관 이력 타임라인
    └─ 예약 현황 시간표
    ↓
예약 버튼 클릭
    └─ reserve.jsp?id={asset_no} 이동
```

#### 예약 생성 플로우
```
reserve.jsp?id={asset_no} (초기로드)
    ├─ 자산 정보 로드
    └─ 예약 폼 렌더링

사용자 입력 (예약날짜, 시간)
    ↓
AJAX: getReservations.jsp 호출
    ├─ 해당 날짜의 예약 현황 쿼리
    └─ JSON 응답: 예약된 시간대
    ↓
JavaScript: 중복 검사 및 경고

예약 버튼 클릭
    ↓ POST /reserve.jsp
INSERT INTO reservations
    ├─ asset_no, user_id, reserve_date
    ├─ start_time, end_time, purpose
    ├─ status = '예약완료'
    └─ reg_date = NOW()
    ↓
성공 메시지 + 상세 페이지로 리다이렉트
```

---

## 3. 기술 스택 (Technology Stack)

### 3.1 스택 상세

| 계층 | 기술 | 버전 | 역할 | 상태 |
|-----|------|------|------|------|
| **Web Server** | Apache Tomcat | 9.x | 서블릿/JSP 컨테이너 | ✅ |
| **Front-end Framework** | Bootstrap | 5.3.3 | UI 컴포넌트 + 반응형 | ✅ |
| **Icons** | Bootstrap Icons | 1.11.3 | 아이콘 라이브러리 | ✅ |
| **Font** | DM Sans + Noto Sans KR | - | 타이포그래피 | ✅ |
| **Language** | Java | 11+ | 서블릿/비즈니스 로직 | ✅ |
| **Servlet API** | Servlet API | 4.0 | 웹 요청 처리 | ✅ |
| **JSP** | JSP | 2.3 | 서버 렌더링 템플릿 | ✅ |
| **Database** | MySQL | 8.x | 데이터 저장소 | ✅ |
| **JDBC Driver** | mysql-connector-j | 8.0.x | DB 통신 | ✅ |
| **Version Control** | Git | - | 소스 관리 | ✅ |
| **Encoding** | UTF-8 | - | 한글 인코딩 | ✅ |

### 3.2 라이브러리 의존성

```
Tomcat 제공 (내장)
├─ servlet-api.jar
├─ jsp-api.jar
└─ CATALINA_HOME/lib/ 기타

외부 라이브러리 (필수)
├─ mysql-connector-j-*.jar
│  └─ CATALINA_HOME/lib/ 에 배치
└─ bootstrap, icons (CDN 사용)
```

### 3.3 시스템 요구사항

```
개발 환경
├─ OS: Windows 11 (또는 Linux)
├─ Java: JDK 11+
├─ MySQL: 8.0+
├─ Tomcat: 9.x
├─ IDE: Eclipse, IntelliJ (권장)
└─ Browser: Chrome, Firefox, Safari, Edge

프로덕션 환경 (예정)
├─ OS: Linux (권장 CentOS/Ubuntu)
├─ Java: OpenJDK 11 LTS
├─ MySQL: 8.0+
├─ Tomcat: 9.x (또는 10.x)
├─ 메모리: 최소 4GB
└─ 디스크: 최소 50GB (자산 이미지 저장)
```

---

## 4. 배포 구조 (Deployment Topology)

### 4.1 현재 로컬 배포 구조

```
개발자 로컬 머신
├─ C:\Program Files\Apache Software Foundation\Tomcat 9.0
│  ├─ bin\
│  │  ├─ startup.bat
│  │  ├─ shutdown.bat
│  │  └─ catalina.sh
│  ├─ lib\
│  │  ├─ servlet-api.jar
│  │  ├─ mysql-connector-j-8.0.x.jar ← 필수 배치
│  │  └─ ...
│  └─ webapps\
│     └─ CampusNav\ ← 프로젝트 폴더
│        ├─ *.jsp (JSP 파일들)
│        ├─ css\
│        │  └─ style.css
│        ├─ WEB-INF\
│        │  ├─ web.xml (배포 기술서)
│        │  ├─ src\ (소스)
│        │  │  └─ com\campus\nav\*.java
│        │  └─ classes\ (컴파일된 클래스)
│        │     └─ com\campus\nav\*.class
│        └─ compile.bat (빌드 스크립트)
│
├─ MySQL Server (localhost:3306)
│  └─ campusnav DB
│     ├─ users
│     ├─ assets
│     ├─ reservations
│     └─ ...
│
└─ 브라우저
   └─ http://localhost:8080/CampusNav/campuslogin.jsp
```

### 4.2 향후 프로덕션 배포 (단계별)

#### Phase 1: 온프레미스 (현재)
```
로컬 개발 환경
├─ 수동 배포 (compile.bat + Tomcat 재시작)
├─ 로컬 MySQL
└─ http://localhost:8080/CampusNav
```

#### Phase 2: 서버 배포 (권장)
```
ICT폴리텍 내부 서버 (향후)
├─ Linux 서버 (CentOS/Ubuntu)
├─ Tomcat 9.x
├─ MySQL 8.x (또는 MariaDB)
├─ CI/CD 파이프라인 (Jenkins/GitLab CI)
└─ http://campusnav.ict.ac.kr 또는 내부 IP

구성
┌─────────────────────────────────────────┐
│     Nginx (리버스 프록시, SSL)          │
│     :443 (HTTPS)                        │
└───────────────┬─────────────────────────┘
                │
┌───────────────┴─────────────────────────┐
│   Tomcat Cluster (3-5 인스턴스)         │
│   :8080, :8081, :8082, ...             │
└───────────────┬─────────────────────────┘
                │
        ┌───────┴──────────┐
        │                  │
   ┌────▼───┐      ┌──────▼───┐
   │ MySQL  │      │ 파일저장소│
   │Primary │      │(이미지)   │
   └─────┬──┘      └───────────┘
         │
    ┌────▼───────┐
    │ Backup DB  │
    │(MySQL Repl)│
    └────────────┘
```

#### Phase 3: 클라우드 배포 (장기)
```
AWS / Azure / Google Cloud (예정)
├─ RDS (관리형 MySQL)
├─ ECS/Fargate (컨테이너)
├─ CloudFront (CDN)
├─ S3/Blob Storage (파일저장소)
├─ Lambda (자동화 함수)
└─ CloudWatch (모니터링)
```

---

## 5. 보안 아키텍처 (Security Architecture)

### 5.1 인증 (Authentication)

```
현재 구현
├─ LoginServlet: 하드코딩 자격증명 배열
├─ Session: loginUser, loginName, loginRole 저장
├─ 타임아웃: 30분 유휴 시 자동 로그아웃
└─ ❌ 암호화 없음, DB 미연동

개선 방안
├─ ✅ DB 기반 사용자 조회 (users 테이블)
├─ ✅ bcrypt 암호화 저장
├─ ✅ 실패 기록 및 잠금 (5회 실패 후 잠금)
├─ ✅ HTTPS 강제 적용
└─ ✅ 로그인 로그 기록
```

### 5.2 접근 제어 (Authorization)

```
현재 구현
├─ 역할 기반 접근 제어 (RBAC): 6가지 역할
│  ├─ student (학부생) → main_student.jsp
│  ├─ assistant (조교) → main_assistant.jsp
│  ├─ professor (교수) → main_professor.jsp
│  ├─ admin (관리자) → main_admin.jsp
│  ├─ guest (게스트) → main_guest.jsp
│  └─ visitor (외부인) → main_visitor.jsp
└─ JSP 페이지에서 세션 확인: loginRole 속성 검사

개선 방안
├─ ✅ 서블릿 필터로 일괄 인증 검사
├─ ✅ 역할별 API 엔드포인트 보호
├─ ✅ 세분화된 권한 관리 (예: 자신의 예약만 수정)
└─ ✅ 감사 로그 (audit trail)
```

### 5.3 데이터 보안 (Data Security)

```
현재 문제점
├─ ❌ SQL Injection 위험: 동적 SQL 구성
├─ ❌ XSS 위험: 사용자 입력 미검증
├─ ❌ CSRF 보호: CSRF 토큰 없음
├─ ❌ 비밀번호: 평문 저장 (하드코딩)
└─ ❌ 데이터 암호화: 민감정보 평문 저장

개선 방안 (즉시)
├─ ✅ Prepared Statement 사용 (SQL Injection 방지)
├─ ✅ HTML 인코딩 (XSS 방지)
├─ ✅ CSRF 토큰 검증
├─ ✅ bcrypt 암호화
└─ ✅ 민감정보 암호화 저장 (AES-256)
```

### 5.4 통신 보안 (Transport Security)

```
현재
├─ ❌ HTTP (비암호화)
└─ localhost만 사용

개선 방안
├─ ✅ HTTPS (SSL/TLS) 강제
├─ ✅ 인증서 설정 (self-signed 또는 공식)
└─ ✅ HSTS 헤더 설정
```

---

## 6. 성능 아키텍처 (Performance Architecture)

### 6.1 데이터베이스 성능

```
현재 상태
├─ 인덱싱: 기본 (PK만)
├─ 쿼리 최적화: 미실행
├─ 캐싱: 없음
└─ 커넥션 풀: 없음

개선 방안
├─ ✅ 인덱싱 추가
│  ├─ assets: (asset_class, asset_status, detail_location)
│  ├─ reservations: (asset_no, reserve_date)
│  └─ users: (user_id)
├─ ✅ 쿼리 최적화 (EXPLAIN 분석)
├─ ✅ 커넥션 풀 도입 (HikariCP 또는 C3P0)
└─ ✅ 캐싱 레이어 추가 (Redis - 향후)
```

### 6.2 애플리케이션 성능

```
현재 상태
├─ 응답 시간: 측정 필요
├─ 동시접속: 100명 정도 가능
├─ 메모리: Tomcat 힙 1GB
└─ 리소스 누수: 가능성 높음 (try-finally 부족)

개선 방안
├─ ✅ try-with-resources 사용 (자동 리소스 해제)
├─ ✅ 연결 풀 (커넥션 재사용)
├─ ✅ 페이지네이션 강제 (대량 데이터 방지)
├─ ✅ 이미지 압축 (LCP 개선)
└─ ✅ CDN (정적 파일 배포)
```

### 6.3 프런트엔드 성능

```
현재 상태
├─ 번들 크기: Bootstrap 5.3.3 (작음)
├─ CSS: 인라인 스타일 (병렬화 부족)
├─ JS: 기본 vanilla JS + jQuery
└─ 폰트: Google Fonts CDN

개선 방안
├─ ✅ CSS 외부화 및 최소화
├─ ✅ 지연 로딩 (Lazy loading)
├─ ✅ 이미지 최적화 (WebP)
├─ ✅ Gzip 압축
└─ ✅ 브라우저 캐싱 (1시간)
```

---

## 7. 확장성 아키텍처 (Scalability Architecture)

### 7.1 수평 확장 (Horizontal Scaling)

```
현재: 단일 Tomcat 인스턴스

향후 구조:
┌──────────────┐
│ Load Balancer│ (Nginx/HAProxy)
└────────┬─────┘
    ┌────┼────┬────────────┐
    │    │    │            │
 ┌──▼──┐┌─▼──┐┌─▼──┐   ┌──▼──┐
 │Tom1 ││Tom2││Tom3│...│Tom-N│
 └──┬──┘└─┬──┘└─┬──┘   └──┬──┘
    │     │    │          │
    └─────┼────┼──────────┘
          │    │
      ┌───▼────▼──┐
      │   MySQL   │
      │  (Primary)│
      └───┬───────┘
          │
      ┌───▼────────┐
      │   Backup   │
      │  Replica   │
      └────────────┘

세션 공유 전략
├─ Sticky Session (로드밸런서 레벨)
├─ 또는 Redis 기반 세션 저장소 (향후)
└─ 또는 데이터베이스 기반 세션 (현재)
```

### 7.2 수직 확장 (Vertical Scaling)

```
현재: 로컬 개발 환경
├─ Tomcat 힙: 1GB
├─ MySQL 메모리: 512MB
└─ 디스크: 필요한 만큼

향후: 프로덕션 서버
├─ Tomcat 힙: 4-8GB
├─ MySQL 메모리: 8-16GB
├─ CPU: 4-8 코어
└─ 디스크: 500GB+ (자산 이미지)
```

### 7.3 다중 캠퍼스 확산 (Multi-Campus)

```
현재: ICT 폴리텍대학 전용

향후 고려사항:
├─ 테넌트 격리
│  ├─ 데이터베이스 분리 또는
│  └─ schema 분리
├─ 설정 외부화
│  ├─ campus_id 파라미터
│  └─ 브랜딩 커스터마이징
└─ API 개발 (다중 클라이언트 지원)
```

---

## 8. 아키텍처 결정 (Architecture Decisions)

### 8.1 주요 결정사항

| 결정 | 선택 | 근거 | 트레이드오프 |
|-----|------|------|-----------|
| **프레임워크** | JSP + Servlet | 학습곡선 낮음, 기존 코드 활용 | 현대적 프레임워크 대비 복잡도 높음 |
| **렌더링** | 서버 렌더링 (JSP) | 빠른 초기 구현 | SPA 대비 UX 제한 |
| **DB** | MySQL | 오픈소스, 한국 생태계 풍부 | NoSQL 대비 확장성 제한 |
| **ORM** | 없음 (직접 JDBC) | 빠른 개발, 유연성 | 보안 위험 (SQL Injection) |
| **인증** | 커스텀 (하드코딩) | 간단함 | 보안 취약점, 유지보수 곤란 |
| **프론트엔드** | Bootstrap | 빠른 구현, 반응형 | 커스터마이징 제한 |
| **배포** | 온프레미스 Tomcat | 비용 낮음, 제어력 높음 | 운영 부담, 장애 대응 어려움 |

### 8.2 향후 아키텍처 개선 (ADR)

```
1. Servlet 기반 아키텍처 정리
   ├─ 상태: PROPOSED
   ├─ 영향도: HIGH
   └─ 기간: 3개월

2. Spring Framework 마이그레이션
   ├─ 상태: ACCEPTED (향후)
   ├─ 영향도: VERY HIGH
   └─ 기간: 6개월

3. REST API 개발 (모바일 앱 지원)
   ├─ 상태: ACCEPTED (단기)
   ├─ 영향도: MEDIUM
   └─ 기간: 2개월

4. 데이터베이스 보안 강화
   ├─ 상태: ACCEPTED (즉시)
   ├─ 영향도: CRITICAL
   └─ 기간: 1개월

5. CI/CD 파이프라인 구축
   ├─ 상태: ACCEPTED (중기)
   ├─ 영향도: HIGH
   └─ 기간: 1개월
```

---

## 9. 리스크 및 대응 (Risk Management)

| 리스크 | 발생 확률 | 영향도 | 대응 방안 |
|--------|---------|--------|---------|
| **SQL Injection 공격** | HIGH | CRITICAL | Prepared Statement 즉시 도입 |
| **스케일 부족** (동시 1000명) | MEDIUM | HIGH | 데이터베이스 최적화, 캐싱 |
| **세션 공유 문제** | LOW | MEDIUM | Redis 기반 세션 저장소 |
| **Tomcat 메모리 누수** | MEDIUM | MEDIUM | try-with-resources 강제 |
| **네트워크 단절** | LOW | MEDIUM | 오프라인 모드 (향후) |
| **데이터 손실** | LOW | CRITICAL | 일일 백업 + 복제 DB |

---

## 10. 모니터링 및 관찰성 (Observability)

### 10.1 로깅 전략

```
현재
├─ Tomcat 기본 로그만 사용
└─ 애플리케이션 로그 미흡

권장 (구현 예정)
├─ 구조화된 로그 (JSON 형식)
│  ├─ timestamp, level, logger, message
│  ├─ user_id, session_id (추적성)
│  └─ performance metrics
├─ 로그 레벨
│  ├─ FATAL: 시스템 중단 수준
│  ├─ ERROR: 오류 (시스템 계속 가능)
│  ├─ WARN: 경고 (잠재적 문제)
│  ├─ INFO: 일반 정보 (로그인, 주요 작업)
│  └─ DEBUG: 상세 정보 (개발용)
└─ 저장소
   ├─ 파일: /var/log/campusnav/*.log
   ├─ DB: logs 테이블 (향후)
   └─ 중앙 집계: ELK/Splunk (장기)
```

### 10.2 메트릭 수집

```
권장 메트릭
├─ 성능
│  ├─ 요청 응답 시간 (ms)
│  ├─ 처리된 요청 수 (requests/min)
│  └─ 에러율 (%)
├─ 리소스
│  ├─ CPU 사용률
│  ├─ 메모리 사용률
│  ├─ 디스크 I/O
│  └─ 네트워크 대역폭
└─ 비즈니스
   ├─ 활성 사용자 수
   ├─ 예약 건수
   ├─ 검색 조회수
   └─ 오류 로그
```

### 10.3 알림 (Alerting)

```
임계값 설정
├─ CPU > 80% → 경고
├─ 메모리 > 85% → 경고
├─ 응답시간 > 5초 → 경고
├─ 에러율 > 1% → 경고
└─ 디스크 > 90% → 경고

알림 채널 (향후)
├─ Slack
├─ 이메일
├─ SMS
└─ 온콜 시스템
```

---

## 11. 참고문서 (References)

- 기획서: `docs/기획서.md`
- 데이터베이스: `nav.sql`
- 배포 설정: `WEB-INF/web.xml`
- 빌드 스크립트: `compile.bat`
- 코드: `WEB-INF/src/com/campus/nav/`

---

**작성 방법:** 역공학 기반 (현재 구현 분석)  
**작성일:** 2026-05-20  
**검토자:** AI Agent (Claude Code)  
**승인 대기:** 기술 이사
