# EDYN 프로젝트 초기화 프롬프트 (Claude Code용)

---

## 📋 프로젝트 컨텍스트

You are a senior full-stack engineer building **EDYN (Event-Driven Yield Navigator)** — a Korean bond market analysis system that matches current macro events to historical yield patterns using semantic similarity.

**Goal:** Build a working prototype in 3 days. Prioritize core functionality over perfection.

---

## 🎯 Day 1 Task: Project Scaffold + Data Pipeline + Vector DB

### Step 1. Create project structure

Create the following directory structure:

```
edyn/
├── backend/
│   ├── main.py               # FastAPI entry point
│   ├── config.py             # Settings
│   ├── db/
│   │   ├── vector_store.py   # ChromaDB wrapper
│   │   └── schemas.py        # Pydantic models
│   ├── modules/
│   │   ├── semantic_matcher.py   # Module 1: Semantic Event Matching
│   │   ├── yield_analyzer.py     # Module 2: Yield Deviation Analysis
│   │   └── duration_scanner.py  # Module 3: Event Duration Scanner
│   ├── data/
│   │   ├── events.csv            # Historical events dataset (50 events)
│   │   └── yield_data/           # KTB yield time series (CSV per event)
│   └── scripts/
│       └── seed_db.py            # DB seeding script
├── frontend/
│   ├── src/
│   │   ├── App.tsx
│   │   ├── components/
│   │   │   ├── EventInput.tsx        # News input box
│   │   │   ├── MatchedEvents.tsx     # Top-K similar events
│   │   │   ├── YieldOverlay.tsx      # TradingView chart overlay
│   │   │   └── SignalBadge.tsx       # BUY/SELL signal
│   │   └── api/
│   │       └── edyn.ts              # API client
│   └── package.json
├── requirements.txt
└── README.md
```

---

### Step 2. requirements.txt

Generate `requirements.txt` with:
- fastapi, uvicorn
- chromadb
- sentence-transformers (use `jhgan/ko-sbert-sts` model for Korean)
- pandas, numpy, scipy
- httpx
- python-dotenv

---

### Step 3. Historical Events Dataset

Create `backend/data/events.csv` with **50 historical financial events** covering:
- 컬럼: `event_id, date, label_ko, label_en, category, description_ko, duration_days, peak_deviation_3y_bp, peak_deviation_10y_bp`
- category 값: `중앙은행`, `신용위기`, `지정학`, `인플레이션`, `외환위기`
- 반드시 포함할 이벤트:
  - 1994 Fed 기습 인상
  - 1997 한국 외환위기
  - 2001 9/11 테러
  - 2008 리먼브라더스 파산
  - 2010 유럽 재정위기
  - 2013 Taper Tantrum
  - 2020 코로나 쇼크
  - 2022 Fed 75bp 연속 인상
  - 2022 레고랜드 사태
  - 2023 SVB 파산
  - 나머지 40건은 위 카테고리 내에서 적절히 생성

---

### Step 4. Synthetic Yield Path Data

For each event in events.csv, generate synthetic KTB yield path data:
- `backend/data/yield_data/{event_id}.csv`
- 컬럼: `t_day (T-5 ~ T+90), yield_3y, yield_10y`
- T+0 = 이벤트 발생일 기준 상대 일수
- 실제 역사적 흐름과 유사하게 생성 (급등 후 점진적 회귀 패턴)

---

### Step 5. Vector DB (ChromaDB) 구현

`backend/db/vector_store.py`:
- ChromaDB persistent client (로컬 저장)
- Collection name: `edyn_events`
- 임베딩: `jhgan/ko-sbert-sts` 로 description_ko 인코딩
- metadata에 event_id, date, label_ko, category, duration_days, peak_deviation_3y_bp 저장
- 함수:
  - `seed_collection(events_df)` → CSV 읽어서 DB 초기화
  - `search_similar(query_text, top_k=5)` → 유사 이벤트 반환

---

### Step 6. Semantic Matcher Module

`backend/modules/semantic_matcher.py`:
- Input: 뉴스 텍스트 (한국어)
- Output: Top-5 유사 이벤트 리스트 (similarity_score, event metadata 포함)
- `jhgan/ko-sbert-sts` 모델로 쿼리 임베딩 후 ChromaDB cosine search

---

### Step 7. Yield Analyzer Module

`backend/modules/yield_analyzer.py`:
- Input: matched event_ids 리스트, 현재 금리 값 (3y, 10y)
- Output:
  - 각 만기별 historical yield path bundle (mean μ, std σ, 각 이벤트 경로)
  - deviation_score = (current_yield - μ) / σ
  - signal: STRONG_BUY / BUY / NEUTRAL / SELL / STRONG_SELL

---

### Step 8. FastAPI Endpoints

`backend/main.py`:

```
POST /api/analyze
  body: { "news_text": "...", "current_yield_3y": 3.25, "current_yield_10y": 3.85 }
  response: {
    "matched_events": [...],
    "yield_analysis": {
      "mean_path_3y": [...],
      "sigma_path_3y": [...],
      "deviation_score_3y": float,
      "signal": "BUY"
    },
    "duration_probability": float
  }

GET /api/events
  response: 전체 이벤트 목록
```

---

### Step 9. Seed Script

`backend/scripts/seed_db.py`:
- events.csv 읽어서 ChromaDB에 전체 임베딩 후 저장
- 실행: `python -m scripts.seed_db`
- 완료 시 "Seeded {n} events into ChromaDB" 출력

---

## ⚠️ 제약 조건

- 외부 API 키 불필요 (망분리 환경 대응) — 모든 모델은 로컬 실행
- Mock 허용 범위: Event Duration Scanner는 규칙 기반 간소화 구현 허용
- 언어: 백엔드 Python, 프론트엔드 TypeScript(React)
- ChromaDB는 `./chroma_db` 디렉토리에 persistent 저장

---

## ✅ Day 1 완료 기준

- [ ] `python -m scripts.seed_db` 실행 시 50건 이벤트 ChromaDB 저장 완료
- [ ] `POST /api/analyze` 에 뉴스 텍스트 입력 시 Top-5 유사 이벤트 + Deviation Score 반환
- [ ] 서버 `uvicorn main:app --reload` 정상 실행
