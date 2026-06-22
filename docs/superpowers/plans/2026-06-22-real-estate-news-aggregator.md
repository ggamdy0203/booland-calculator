# 부동산 뉴스 통합 수집기 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 한국경제/매일경제/네이버/다음/부동산114/직방/호갱노노의 부동산 관련 글을 3~4시간마다 자동 수집해 `data/news.json`에 저장하고, GitHub Pages 정적 페이지에서 통합 타임라인 + 사이트별 필터로 보여준다.

**Architecture:** Python 스크립트가 각 소스의 RSS 피드를 `feedparser`로 읽어 `NewsItem` 딕셔너리 목록으로 변환한다. 직접 RSS가 없는 소스는 `site:` 조건을 건 Google 뉴스 RSS 검색을 사용한다 (즉 모든 소스를 "RSS 읽기"라는 동일한 메커니즘으로 통일 — 사이트별 HTML 구조에 의존하는 스크래퍼보다 훨씬 덜 깨진다). GitHub Actions가 3~4시간마다 이 스크립트를 실행하고 결과를 커밋한다. `index.html`은 같은 도메인의 `data/news.json`을 fetch해서 렌더링한다.

**Tech Stack:** Python 3.12, `feedparser`, `pytest`, GitHub Actions, 순수 HTML/CSS/Vanilla JS.

**설계 변경 사항 (스펙 대비):** 원래 스펙은 "RSS 가능하면 RSS, 아니면 사이트별 HTML 스크래퍼(`sources/naver.py` 등 7개 파일)"였다. 구현 전 조사 결과, 네이버/다음/부동산114/직방/호갱노노는 모두 `site:도메인` 조건의 Google 뉴스 RSS로 안정적으로 글을 가져올 수 있음을 확인했다(직접 검증 완료). 이에 따라 7개의 개별 스크래퍼 대신 **하나의 RSS 읽기 모듈 + 소스별 URL 설정 테이블**로 단순화한다. 기능적으로는 스펙의 요구사항(7개 소스 전수 수집, 필터 없음, 3~4시간 주기)을 동일하게 만족하면서 유지보수 부담은 크게 줄어든다.

---

## 파일 구조

```
real-estate-news/
├── .github/workflows/collect.yml   # 3~4시간마다 실행되는 GitHub Actions
├── scraper/
│   ├── requirements.txt
│   ├── sources.py                  # 소스 라벨 + RSS URL 설정 테이블
│   ├── rss_fetch.py                # RSS URL → NewsItem 목록 변환
│   ├── dedupe.py                   # 중복 제거 + 14일 보관 필터
│   └── main.py                     # 전체 수집 실행 (기존 데이터 병합 + 저장)
├── tests/
│   ├── test_rss_fetch.py
│   └── test_dedupe.py
├── data/
│   └── news.json                   # 수집 결과 (최근 14일치 누적)
└── index.html                      # GitHub Pages로 보여주는 화면
```

---

### Task 1: 레포지토리 스캐폴딩

**Files:**
- Create: `real-estate-news/scraper/requirements.txt`
- Create: `real-estate-news/.gitignore`

- [ ] **Step 1: 새 디렉터리와 기본 파일 생성**

```bash
mkdir -p real-estate-news/scraper real-estate-news/tests real-estate-news/data
cd real-estate-news
git init
```

- [ ] **Step 2: requirements.txt 작성**

`real-estate-news/scraper/requirements.txt`:
```
feedparser==6.0.11
```

- [ ] **Step 3: .gitignore 작성**

`real-estate-news/.gitignore`:
```
__pycache__/
*.pyc
.pytest_cache/
```

- [ ] **Step 4: pytest 설치 확인**

```bash
pip install -r scraper/requirements.txt pytest
```

Expected: 에러 없이 설치 완료

- [ ] **Step 5: Commit**

```bash
git add scraper/requirements.txt .gitignore
git commit -m "chore: 프로젝트 스캐폴딩"
```

---

### Task 2: 소스 설정 테이블 (`sources.py`)

**Files:**
- Create: `real-estate-news/scraper/sources.py`

- [ ] **Step 1: sources.py 작성**

`real-estate-news/scraper/sources.py`:
```python
"""수집 대상 소스 목록. 각 항목은 (소스 라벨, RSS URL) 튜플."""

SOURCES = [
    ("한국경제", "https://www.hankyung.com/feed/realestate"),
    (
        "매일경제",
        "https://news.google.com/rss/search?q=%EB%B6%80%EB%8F%99%EC%82%B0+site:mk.co.kr&hl=ko&gl=KR&ceid=KR:ko",
    ),
    (
        "네이버",
        "https://news.google.com/rss/search?q=%EB%B6%80%EB%8F%99%EC%82%B0+site:news.naver.com&hl=ko&gl=KR&ceid=KR:ko",
    ),
    (
        "다음",
        "https://news.google.com/rss/search?q=%EB%B6%80%EB%8F%99%EC%82%B0+site:v.daum.net&hl=ko&gl=KR&ceid=KR:ko",
    ),
    (
        "부동산114",
        "https://news.google.com/rss/search?q=%EB%B6%80%EB%8F%99%EC%82%B0+site:r114.com&hl=ko&gl=KR&ceid=KR:ko",
    ),
    (
        "직방",
        "https://news.google.com/rss/search?q=%EB%B6%80%EB%8F%99%EC%82%B0+site:zigbang.com&hl=ko&gl=KR&ceid=KR:ko",
    ),
    (
        "호갱노노",
        "https://news.google.com/rss/search?q=%EB%B6%80%EB%8F%99%EC%82%B0+site:hogangnono.com&hl=ko&gl=KR&ceid=KR:ko",
    ),
]
```

- [ ] **Step 2: Commit**

```bash
git add scraper/sources.py
git commit -m "feat: 수집 소스 설정 테이블 추가"
```

---

### Task 3: RSS 읽기 모듈 (`rss_fetch.py`)

**Files:**
- Create: `real-estate-news/scraper/rss_fetch.py`
- Test: `real-estate-news/tests/test_rss_fetch.py`

`feedparser.parse()`는 URL뿐 아니라 XML 문자열을 직접 받아도 동일하게 동작하므로, 테스트에서는 실제 네트워크 호출 없이 샘플 XML 문자열을 그대로 "URL" 인자로 넘겨서 검증한다.

- [ ] **Step 1: 실패하는 테스트 작성**

`real-estate-news/tests/test_rss_fetch.py`:
```python
import sys
import os

sys.path.insert(0, os.path.join(os.path.dirname(__file__), "..", "scraper"))

from rss_fetch import fetch_rss

SAMPLE_FEED = """<?xml version="1.0" encoding="UTF-8"?>
<rss version="2.0">
<channel>
  <title>샘플 피드</title>
  <item>
    <title>부산 아파트 매매가 상승</title>
    <link>https://example.com/article/1</link>
    <pubDate>Mon, 22 Jun 2026 09:00:00 +0900</pubDate>
    <description>부산 지역 아파트 매매가가 상승세를 보이고 있다.</description>
  </item>
  <item>
    <title>전세 시장 안정세</title>
    <link>https://example.com/article/2</link>
    <pubDate>Mon, 22 Jun 2026 08:00:00 +0900</pubDate>
    <description>전세 시장이 안정세를 보이고 있다.</description>
  </item>
</channel>
</rss>
"""


def test_fetch_rss_returns_parsed_items():
    items = fetch_rss("테스트소스", SAMPLE_FEED)

    assert len(items) == 2
    first = items[0]
    assert first["source"] == "테스트소스"
    assert first["title"] == "부산 아파트 매매가 상승"
    assert first["url"] == "https://example.com/article/1"
    assert first["summary"] == "부산 지역 아파트 매매가가 상승세를 보이고 있다."
    assert first["publishedAt"].startswith("2026-06-22")
    assert first["id"]


def test_fetch_rss_skips_items_without_title_or_link():
    feed_with_broken_item = SAMPLE_FEED.replace(
        "<title>전세 시장 안정세</title>", "<title></title>"
    )
    items = fetch_rss("테스트소스", feed_with_broken_item)

    assert len(items) == 1
```

- [ ] **Step 2: 테스트 실행 후 실패 확인**

```bash
cd real-estate-news
python -m pytest tests/test_rss_fetch.py -v
```

Expected: FAIL — `ModuleNotFoundError: No module named 'rss_fetch'` (아직 파일이 없음)

- [ ] **Step 3: rss_fetch.py 구현**

`real-estate-news/scraper/rss_fetch.py`:
```python
"""RSS URL을 NewsItem(딱셔너리) 목록으로 변환하는 모듈."""

import hashlib
from datetime import datetime, timezone

import feedparser


def fetch_rss(source_label, rss_url):
    """rss_url을 읽어 source_label로 태그된 NewsItem 목록을 반환한다.

    title 또는 link가 비어있는 항목은 건너뛴다.
    """
    feed = feedparser.parse(rss_url)
    items = []
    for entry in feed.entries:
        title = entry.get("title", "").strip()
        link = entry.get("link", "").strip()
        if not title or not link:
            continue
        items.append(
            {
                "id": _make_id(source_label, link),
                "source": source_label,
                "title": title,
                "url": link,
                "publishedAt": _parse_published(entry),
                "summary": entry.get("summary", "").strip(),
            }
        )
    return items


def _parse_published(entry):
    parsed = entry.get("published_parsed")
    if parsed:
        return datetime(*parsed[:6], tzinfo=timezone.utc).isoformat()
    return datetime.now(timezone.utc).isoformat()


def _make_id(source_label, link):
    digest = hashlib.sha1(f"{source_label}:{link}".encode("utf-8")).hexdigest()[:12]
    return f"{source_label}-{digest}"
```

- [ ] **Step 4: 테스트 실행 후 통과 확인**

```bash
python -m pytest tests/test_rss_fetch.py -v
```

Expected: PASS (2 passed)

- [ ] **Step 5: Commit**

```bash
git add scraper/rss_fetch.py tests/test_rss_fetch.py
git commit -m "feat: RSS 읽기 모듈 추가"
```

---

### Task 4: 중복 제거 + 보관기간 필터 (`dedupe.py`)

**Files:**
- Create: `real-estate-news/scraper/dedupe.py`
- Test: `real-estate-news/tests/test_dedupe.py`

- [ ] **Step 1: 실패하는 테스트 작성**

`real-estate-news/tests/test_dedupe.py`:
```python
import sys
import os
from datetime import datetime, timezone, timedelta

sys.path.insert(0, os.path.join(os.path.dirname(__file__), "..", "scraper"))

from dedupe import dedupe, filter_recent


def _item(title, published_at, source="테스트"):
    return {
        "id": f"{source}-{title}",
        "source": source,
        "title": title,
        "url": "https://example.com",
        "publishedAt": published_at,
        "summary": "",
    }


def test_dedupe_removes_items_with_identical_normalized_title():
    items = [
        _item("부산 아파트 매매가 상승!", "2026-06-22T09:00:00+00:00"),
        _item("부산  아파트   매매가 상승", "2026-06-22T08:00:00+00:00"),
        _item("전세 시장 안정세", "2026-06-22T07:00:00+00:00"),
    ]

    result = dedupe(items)

    assert len(result) == 2
    assert result[0]["title"] == "부산 아파트 매매가 상승!"
    assert result[1]["title"] == "전세 시장 안정세"


def test_filter_recent_keeps_only_items_within_days():
    now = datetime.now(timezone.utc)
    recent = _item("최근 기사", (now - timedelta(days=1)).isoformat())
    old = _item("오래된 기사", (now - timedelta(days=20)).isoformat())

    result = filter_recent([recent, old], days=14)

    assert result == [recent]
```

- [ ] **Step 2: 테스트 실행 후 실패 확인**

```bash
python -m pytest tests/test_dedupe.py -v
```

Expected: FAIL — `ModuleNotFoundError: No module named 'dedupe'`

- [ ] **Step 3: dedupe.py 구현**

`real-estate-news/scraper/dedupe.py`:
```python
"""중복 기사 제거 및 보관기간 필터링."""

import re
from datetime import datetime, timezone, timedelta

_NORMALIZE_RE = re.compile(r"[\s\W]+")


def _normalize_title(title):
    return _NORMALIZE_RE.sub("", title).lower()


def dedupe(items):
    """공백/특수문자를 제거한 제목이 완전히 같은 항목만 중복으로 간주해 제거한다.

    먼저 나온 항목을 우선 유지한다.
    """
    seen = set()
    result = []
    for item in items:
        key = _normalize_title(item["title"])
        if key in seen:
            continue
        seen.add(key)
        result.append(item)
    return result


def filter_recent(items, days=14):
    """publishedAt이 days일 이내인 항목만 반환한다."""
    cutoff = datetime.now(timezone.utc) - timedelta(days=days)
    result = []
    for item in items:
        published = datetime.fromisoformat(item["publishedAt"])
        if published.tzinfo is None:
            published = published.replace(tzinfo=timezone.utc)
        if published >= cutoff:
            result.append(item)
    return result
```

- [ ] **Step 4: 테스트 실행 후 통과 확인**

```bash
python -m pytest tests/test_dedupe.py -v
```

Expected: PASS (2 passed)

- [ ] **Step 5: Commit**

```bash
git add scraper/dedupe.py tests/test_dedupe.py
git commit -m "feat: 중복 제거 및 보관기간 필터 추가"
```

---

### Task 5: 전체 수집 실행 스크립트 (`main.py`)

**Files:**
- Create: `real-estate-news/scraper/main.py`

수동 통합 테스트(이번 태스크는 실제 네트워크를 호출하므로 단위 테스트 대신 수동 실행으로 검증한다 — Task 7에서 전체 파이프라인을 다시 한번 수동으로 확인한다).

- [ ] **Step 1: main.py 구현**

`real-estate-news/scraper/main.py`:
```python
"""모든 소스를 수집해 data/news.json을 갱신한다."""

import json
import os
from datetime import datetime, timezone

from sources import SOURCES
from rss_fetch import fetch_rss
from dedupe import dedupe, filter_recent

DATA_PATH = os.path.join(os.path.dirname(__file__), "..", "data", "news.json")
RETENTION_DAYS = 14


def load_existing():
    if not os.path.exists(DATA_PATH):
        return []
    with open(DATA_PATH, "r", encoding="utf-8") as f:
        data = json.load(f)
    return data.get("items", [])


def collect_all():
    items = []
    for label, url in SOURCES:
        try:
            fetched = fetch_rss(label, url)
            print(f"[OK] {label}: {len(fetched)}건")
            items.extend(fetched)
        except Exception as exc:  # noqa: BLE001 - 한 소스 실패가 전체를 막으면 안 됨
            print(f"[ERROR] {label} 수집 실패: {exc}")
    return items


def main():
    existing = load_existing()
    new_items = collect_all()

    combined = dedupe(existing + new_items)
    combined = filter_recent(combined, days=RETENTION_DAYS)
    combined.sort(key=lambda item: item["publishedAt"], reverse=True)

    output = {
        "updatedAt": datetime.now(timezone.utc).isoformat(),
        "items": combined,
    }

    os.makedirs(os.path.dirname(DATA_PATH), exist_ok=True)
    with open(DATA_PATH, "w", encoding="utf-8") as f:
        json.dump(output, f, ensure_ascii=False, indent=2)

    print(f"총 {len(combined)}건 저장 완료 → {DATA_PATH}")


if __name__ == "__main__":
    main()
```

- [ ] **Step 2: 수동 실행으로 검증**

```bash
cd real-estate-news
python scraper/main.py
```

Expected: 각 소스별 `[OK] 소스명: N건` 로그가 출력되고, 마지막에 `총 N건 저장 완료` 메시지가 출력됨. 일부 소스가 `[ERROR]`로 실패해도 스크립트는 끝까지 실행되어야 한다.

- [ ] **Step 3: data/news.json 내용 확인**

```bash
python -c "import json; d=json.load(open('data/news.json', encoding='utf-8')); print(d['updatedAt']); print(len(d['items'])); print(d['items'][0])"
```

Expected: `updatedAt`, 항목 개수, 첫 항목이 `id`/`source`/`title`/`url`/`publishedAt`/`summary` 키를 모두 가지고 있음을 확인

- [ ] **Step 4: Commit**

```bash
git add scraper/main.py data/news.json
git commit -m "feat: 전체 수집 실행 스크립트 추가"
```

---

### Task 6: GitHub Actions 워크플로우

**Files:**
- Create: `real-estate-news/.github/workflows/collect.yml`

- [ ] **Step 1: 워크플로우 파일 작성**

`real-estate-news/.github/workflows/collect.yml`:
```yaml
name: Collect News

on:
  schedule:
    - cron: '0 0,4,8,12,16,20 * * *'
  workflow_dispatch:

permissions:
  contents: write

jobs:
  collect:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: 의존성 설치
        run: pip install -r scraper/requirements.txt

      - name: 뉴스 수집 실행
        run: python scraper/main.py

      - name: 변경사항 커밋
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add data/news.json
          git diff --staged --quiet || git commit -m "chore: 뉴스 데이터 갱신"
          git push
```

- [ ] **Step 2: Commit**

```bash
git add .github/workflows/collect.yml
git commit -m "ci: 3~4시간마다 뉴스 수집하는 GitHub Actions 추가"
```

---

### Task 7: 프런트엔드 (`index.html`)

**Files:**
- Create: `real-estate-news/index.html`

- [ ] **Step 1: index.html 작성**

`real-estate-news/index.html`:
```html
<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>부동산 뉴스 모음</title>
<style>
  body { font-family: -apple-system, "Malgun Gothic", sans-serif; max-width: 720px; margin: 0 auto; padding: 16px; background: #f5f6f8; }
  h1 { font-size: 20px; }
  #updatedAt { color: #888; font-size: 13px; margin-bottom: 16px; }
  .chips { display: flex; flex-wrap: wrap; gap: 6px; margin-bottom: 16px; }
  .chip { padding: 6px 12px; border-radius: 16px; border: 1px solid #ccc; background: #fff; cursor: pointer; font-size: 13px; }
  .chip.active { background: #2a4d8f; color: #fff; border-color: #2a4d8f; }
  .item { background: #fff; border-radius: 8px; padding: 12px 14px; margin-bottom: 8px; }
  .item a { color: #222; text-decoration: none; font-weight: 600; }
  .item a:hover { text-decoration: underline; }
  .meta { color: #999; font-size: 12px; margin-top: 4px; }
  .source-tag { color: #2a4d8f; font-weight: 600; }
</style>
</head>
<body>
  <h1>부동산 뉴스 모음</h1>
  <div id="updatedAt"></div>
  <div class="chips" id="chips"></div>
  <div id="list"></div>

  <script>
    let allItems = [];
    let activeSource = "전체";

    async function load() {
      const res = await fetch("data/news.json");
      const data = await res.json();
      allItems = data.items;
      document.getElementById("updatedAt").textContent =
        "마지막 업데이트: " + new Date(data.updatedAt).toLocaleString("ko-KR");
      renderChips();
      renderList();
    }

    function renderChips() {
      const sources = ["전체", ...new Set(allItems.map(i => i.source))];
      const chipsEl = document.getElementById("chips");
      chipsEl.innerHTML = "";
      for (const source of sources) {
        const chip = document.createElement("div");
        chip.className = "chip" + (source === activeSource ? " active" : "");
        chip.textContent = source;
        chip.onclick = () => {
          activeSource = source;
          renderChips();
          renderList();
        };
        chipsEl.appendChild(chip);
      }
    }

    function renderList() {
      const listEl = document.getElementById("list");
      const filtered = activeSource === "전체"
        ? allItems
        : allItems.filter(i => i.source === activeSource);

      listEl.innerHTML = filtered.map(item => `
        <div class="item">
          <a href="${item.url}" target="_blank" rel="noopener">${item.title}</a>
          <div class="meta">
            <span class="source-tag">${item.source}</span>
            · ${new Date(item.publishedAt).toLocaleString("ko-KR")}
          </div>
        </div>
      `).join("");
    }

    load();
  </script>
</body>
</html>
```

- [ ] **Step 2: 로컬에서 수동 확인**

```bash
cd real-estate-news
python -m http.server 8000
```

브라우저에서 `http://localhost:8000` 접속.

Expected:
- "마지막 업데이트: ..." 텍스트가 표시됨
- 칩(전체/한국경제/매일경제/...)이 표시되고, 클릭하면 해당 소스만 필터링됨
- 기사 제목 클릭 시 새 탭에서 원문이 열림

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: 부동산 뉴스 타임라인 프런트엔드 추가"
```

---

### Task 8: 새 GitHub 레포지토리에 배포

**Files:** 없음 (배포 작업)

- [ ] **Step 1: GitHub에 새 레포지토리 생성**

GitHub 웹사이트에서 `ggamdy0203` 계정으로 새 레포지토리 `real-estate-news`를 생성한다 (Public, README 없이 생성).

- [ ] **Step 2: 원격 저장소 연결 및 푸시**

```bash
cd real-estate-news
git remote add origin https://github.com/ggamdy0203/real-estate-news.git
git branch -M main
git push -u origin main
```

- [ ] **Step 3: GitHub Pages 활성화**

레포지토리 Settings → Pages → Source를 `main` 브랜치 / `/ (root)`로 설정.

- [ ] **Step 4: GitHub Actions 워크플로우 수동 실행으로 확인**

레포지토리 Actions 탭 → "Collect News" 워크플로우 → "Run workflow" 클릭.

Expected: 워크플로우가 성공(녹색 체크)하고, `data/news.json`이 갱신된 새 커밋이 생성됨.

- [ ] **Step 5: 배포된 페이지 확인**

`https://ggamdy0203.github.io/real-estate-news/` 접속.

Expected: 최신 `data/news.json` 내용이 화면에 표시됨.

---

## Self-Review 결과

- **스펙 커버리지:** 7개 소스 수집(Task 2,3,5) / 필터 없음(Task 7 — 칩은 전환용이며 기본은 "전체")  / 3~4시간 주기(Task 6) / 알림 제외(범위 외, 미포함) / 통합 타임라인+사이트 칩(Task 7) / 중복 제거(Task 4) / 14일 보관(Task 4,5) / 에러 격리(Task 5) / 새 레포(Task 8) — 모두 대응하는 태스크 있음.
- **플레이스홀더 스캔:** 없음 — 모든 코드가 완전한 형태로 작성됨.
- **타입/이름 일관성:** `NewsItem` 키(`id`, `source`, `title`, `url`, `publishedAt`, `summary`)가 `rss_fetch.py`, `dedupe.py`, `main.py`, `index.html` 전체에서 동일하게 사용됨. `fetch_rss(source_label, rss_url)` 시그니처가 `sources.py`의 `(label, url)` 튜플 순서와 일치함.
