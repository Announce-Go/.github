# 시스템 설계서

## 1. 기술 스택

### 1.1 Backend

| 구분 | 기술 | 버전 | 선정 사유 |
|------|------|------|-----------|
| Language | Python | 3.11+ | 크롤링 생태계, 빠른 개발 |
| Framework | FastAPI | 0.100+ | 비동기 지원, 자동 API 문서화, 타입 힌트 |
| ORM | SQLAlchemy | 2.0+ | 비동기 지원, 성숙한 생태계 |
| Migration | Alembic | 1.12+ | SQLAlchemy 공식 마이그레이션 도구 |

### 1.2 크롤링

| 구분 | 기술 | 용도 |
|------|------|------|
| HTTP Client | httpx | 비동기 HTTP 요청 |
| HTML Parser | BeautifulSoup4 | 정적 HTML 파싱 |
| Browser Automation | Playwright | 동적 렌더링 대응 (필요 시) |

### 1.3 스케줄링 & 작업 큐

| 구분 | 기술 | 용도 |
|------|------|------|
| Task Queue | Celery | 비동기 작업 처리, 분산 처리 |
| Scheduler | Celery Beat | 주기적 작업 스케줄링 |
| Broker | Redis | 메시지 브로커, 작업 큐 |

### 1.4 데이터베이스

| 구분 | 기술 | 용도 |
|------|------|------|
| Primary DB | PostgreSQL 15+ | 메인 데이터 저장 |
| Cache | Redis | 세션 저장, 캐싱 |

### 1.5 인증

| 구분 | 기술 | 설명 |
|------|------|------|
| Session | Cookie 기반 | HTTP-Only, Secure Cookie |
| Remember Me | 장기 세션 토큰 | 별도 테이블 관리, 만료 기간 연장 |
| Password | bcrypt | 비밀번호 해싱 |

### 1.6 인프라

| 구분 | 기술 | 용도 |
|------|------|------|
| Container | Docker | 애플리케이션 컨테이너화 |
| Orchestration | Docker Compose | 로컬/개발 환경 구성 |
| Cloud | TBD | 클라우드 배포 (AWS/GCP/NCP 등) |

---

## 2. 시스템 아키텍처

```
┌─────────────────────────────────────────────────────────────┐
│                          Client                              │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      FastAPI Server                          │
│  ┌───────────┐  ┌─────────────┐  ┌───────────────────────┐  │
│  │ Auth API  │  │  Rank API   │  │  Keyword Mgmt API     │  │
│  └─────┬─────┘  └──────┬──────┘  └───────────┬───────────┘  │
│        │               │                     │              │
│        │               ▼                     │              │
│        │     ┌─────────────────┐             │              │
│        │     │  RankService    │             │              │
│        │     └────────┬────────┘             │              │
│        │              │                      │              │
│        │              ▼                      │              │
│        │     ┌─────────────────┐             │              │
│        │     │    Crawler      │ ◄───────────┼──────┐       │
│        │     │ (place, post)   │             │      │       │
│        │     └─────────────────┘             │      │       │
└────────┼──────────────┬──────────────────────┼──────┼───────┘
         │              │                      │      │
         ▼              │                      ▼      │
┌───────────────┐       │              ┌─────────────────────────┐
│    Redis      │       │              │      Celery Worker      │
│(Session/Cache)│       │              │  ┌───────────────────┐  │
└───────────────┘       │              │  │   RankService     │  │
                        │              │  │   (check & save)  │  │
                        ▼              │  └───────────────────┘  │
                ┌─────────────┐        └────────────┬────────────┘
                │ PostgreSQL  │                     │
                │    (DB)     │ ◄───────────────────┘
                └─────────────┘          (순위 저장)
                        ▲
                        │
              ┌─────────┴─────────┐
              │    Celery Beat    │
              │ (Daily Scheduler) │
              └───────────────────┘
```

### 2.1 주요 흐름

| 케이스 | 흐름 | 저장 |
|--------|------|------|
| 실시간 순위 조회 | Client → Rank API → RankService → Crawler → 네이버 | X |
| 히스토리 조회 | Client → Rank API → PostgreSQL | - |
| 일일 배치 | Celery Beat → Celery Worker → RankService → Crawler → PostgreSQL | O |

### 2.2 공통 모듈

- **Crawler**: 네이버 크롤링 로직 (place, popular_post)
- **RankService**: 크롤러 호출 + 비즈니스 로직
  - `check_rank()`: 순위 조회만 (실시간용)
  - `check_and_save_rank()`: 순위 조회 + DB 저장 (배치용)

---

## 3. 프로젝트 구조

```
naver-rank-tracker/
├── docker-compose.yml
├── Dockerfile
├── requirements.txt
├── alembic/
│   └── versions/
├── app/
│   ├── main.py
│   ├── config.py
│   ├── database.py
│   ├── api/
│   │   ├── v1/
│   │   │   ├── auth.py
│   │   │   ├── keywords.py
│   │   │   └── ranks.py
│   │   └── deps.py
│   ├── models/
│   │   ├── user.py
│   │   ├── keyword.py
│   │   └── rank_history.py
│   ├── schemas/
│   │   ├── user.py
│   │   ├── keyword.py
│   │   └── rank.py
│   ├── services/
│   │   ├── auth.py
│   │   ├── keyword.py
│   │   └── rank.py
│   ├── crawler/
│   │   ├── base.py
│   │   ├── place.py
│   │   └── popular_post.py
│   └── tasks/
│       ├── celery.py
│       └── rank_check.py
├── tests/
└── docs/
```
