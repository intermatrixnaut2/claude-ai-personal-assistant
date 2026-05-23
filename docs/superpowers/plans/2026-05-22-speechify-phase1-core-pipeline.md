# Speechify Clone — Phase 1: Core Pipeline Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build an end-to-end working system: photograph book title → auto-find and download free PDF → split into chapters → listen with Edge TTS → chapter navigation and resume where you left off.

**Architecture:** Python FastAPI server on Mac mini handles book search, PDF processing, OCR, and TTS. React Native/Expo iOS app handles camera, player, and UI. Server is built and tested first with curl, then the app connects over Tailscale. Phase 1 uses Edge TTS as the voice engine — XTTS-v2 voice cloning is Phase 2.

**Tech Stack:** Python 3.11 / FastAPI / edge-tts / pypdf / pytesseract / httpx · React Native / Expo SDK 52 / TypeScript / expo-av / expo-camera / expo-file-system / @react-navigation/native-stack / @react-native-async-storage/async-storage

---

## File Structure

### Server (`speechify-server/`)
```
speechify-server/
├── main.py                    # FastAPI app, mounts all routers
├── requirements.txt
├── models/
│   └── schemas.py             # All Pydantic request/response models
├── routers/
│   ├── health.py              # GET /health
│   ├── books.py               # POST /search-book, /download-book, /upload-book, GET /chapter-text
│   ├── ocr.py                 # POST /ocr-title
│   └── synthesis.py           # POST /synthesize, GET /voices
├── services/
│   ├── book_search.py         # Open Library + Gutenberg API
│   ├── pdf_processor.py       # Download, extract text, split chapters, cache
│   ├── tts_edge.py            # Edge TTS + ffmpeg speed + audio file cache
│   └── ocr.py                 # pytesseract wrapper
├── cache/
│   ├── audio/                 # Synthesized MP3s: book_id/chapter_voice_speed.mp3
│   └── books/                 # Downloaded PDFs + extracted chapter .txt files
└── tests/
    ├── conftest.py
    ├── test_health.py
    ├── test_books.py
    ├── test_ocr.py
    └── test_synthesis.py
```

### App (`speechify-app/`)
```
speechify-app/
├── App.tsx                    # NavigationContainer + Stack
├── app.json                   # Expo config, bundle ID, camera permission
├── eas.json                   # EAS build: preview profile → unsigned IPA
├── package.json
├── tsconfig.json
└── src/
    ├── types/
    │   └── index.ts           # All shared TypeScript interfaces (single source of truth)
    ├── services/
    │   ├── api.ts             # All HTTP calls to server
    │   ├── storage.ts         # AsyncStorage: books, progress, server URL
    │   └── player.ts          # expo-av + expo-file-system audio wrapper
    ├── screens/
    │   ├── LibraryScreen.tsx
    │   ├── AddBookScreen.tsx
    │   ├── PlayerScreen.tsx
    │   └── SettingsScreen.tsx
    └── components/
        ├── BookCard.tsx
        ├── SpeedSelector.tsx
        └── ChapterList.tsx
```

---

## Task 1: Mac Server — Scaffold + Health Endpoint

**Files:**
- Create: `speechify-server/requirements.txt`
- Create: `speechify-server/main.py`
- Create: `speechify-server/routers/health.py`
- Create: `speechify-server/tests/conftest.py`
- Create: `speechify-server/tests/test_health.py`

- [ ] **Step 1: Create directory structure**

```bash
mkdir -p speechify-server/{routers,services,models,cache/audio,cache/books,tests}
touch speechify-server/routers/__init__.py
touch speechify-server/services/__init__.py
touch speechify-server/models/__init__.py
touch speechify-server/tests/__init__.py
```

- [ ] **Step 2: Write requirements.txt**

```
fastapi==0.115.0
uvicorn[standard]==0.32.0
python-multipart==0.0.12
edge-tts==6.1.12
pypdf==5.1.0
pytesseract==0.3.13
Pillow==11.0.0
httpx==0.27.2
pytest==8.3.3
pytest-asyncio==0.24.0
```

- [ ] **Step 3: Install dependencies + Tesseract**

```bash
cd speechify-server && pip install -r requirements.txt -q
brew install tesseract
```

- [ ] **Step 4: Write the failing test**

```python
# speechify-server/tests/conftest.py
import pytest
from fastapi.testclient import TestClient
from main import app

@pytest.fixture
def client():
    return TestClient(app)
```

```python
# speechify-server/tests/test_health.py
def test_health_returns_ok(client):
    response = client.get("/health")
    assert response.status_code == 200
    assert response.json() == {"status": "ok"}
```

- [ ] **Step 5: Run — confirm failure**

```bash
pytest tests/test_health.py -v
```
Expected: `ModuleNotFoundError: No module named 'main'`

- [ ] **Step 6: Write health router**

```python
# speechify-server/routers/health.py
from fastapi import APIRouter

router = APIRouter()

@router.get("/health")
def health_check():
    return {"status": "ok"}
```

- [ ] **Step 7: Write main.py**

```python
# speechify-server/main.py
from fastapi import FastAPI
from routers import health

app = FastAPI(title="Speechify Server")
app.include_router(health.router)
```

- [ ] **Step 8: Run test — must pass**

```bash
pytest tests/test_health.py -v
```
Expected: `PASSED tests/test_health.py::test_health_returns_ok`

- [ ] **Step 9: Smoke-test running server**

```bash
uvicorn main:app --reload --port 8765
# In another terminal:
curl http://localhost:8765/health
```
Expected: `{"status":"ok"}`

- [ ] **Step 10: Commit**

```bash
cd speechify-server
git add .
git commit -m "feat: scaffold FastAPI server with health endpoint"
```

---

## Task 2: Pydantic Schemas + Book Search Service

**Files:**
- Create: `speechify-server/models/schemas.py`
- Create: `speechify-server/services/book_search.py`
- Create: `speechify-server/routers/books.py`
- Modify: `speechify-server/main.py`
- Create: `speechify-server/tests/test_books.py`

- [ ] **Step 1: Write schemas.py**

```python
# speechify-server/models/schemas.py
from pydantic import BaseModel
from typing import Optional

class BookSearchRequest(BaseModel):
    title: str

class BookResult(BaseModel):
    found: bool
    title: Optional[str] = None
    author: Optional[str] = None
    cover_url: Optional[str] = None
    pdf_url: Optional[str] = None
    source: Optional[str] = None  # "gutenberg" | "openlibrary"

class DownloadRequest(BaseModel):
    pdf_url: str
    book_id: str

class Chapter(BaseModel):
    index: int
    title: str
    char_count: int

class DownloadResult(BaseModel):
    book_id: str
    chapters: list[Chapter]

class SynthesizeRequest(BaseModel):
    text: str
    voice_id: str = "en-US-GuyNeural"
    speed: float = 1.0
    book_id: str
    chapter_index: int

class OCRResult(BaseModel):
    text: str
```

- [ ] **Step 2: Write the failing test**

```python
# speechify-server/tests/test_books.py
from unittest.mock import patch, AsyncMock

def test_search_book_found(client):
    mock_result = {
        "found": True,
        "title": "Moby Dick",
        "author": "Herman Melville",
        "cover_url": "https://covers.openlibrary.org/b/id/8739161-M.jpg",
        "pdf_url": "https://www.gutenberg.org/files/2701/2701-0.txt",
        "source": "gutenberg"
    }
    with patch("services.book_search.search_book", new_callable=AsyncMock, return_value=mock_result):
        response = client.post("/search-book", json={"title": "Moby Dick"})
    assert response.status_code == 200
    data = response.json()
    assert data["found"] is True
    assert data["title"] == "Moby Dick"

def test_search_book_not_found(client):
    mock_result = {"found": False}
    with patch("services.book_search.search_book", new_callable=AsyncMock, return_value=mock_result):
        response = client.post("/search-book", json={"title": "zzznobookwiththisname999"})
    assert response.status_code == 200
    assert response.json()["found"] is False
```

- [ ] **Step 3: Run — confirm failure**

```bash
pytest tests/test_books.py -v
```
Expected: `ERROR` — `books` router not mounted

- [ ] **Step 4: Write book_search.py**

```python
# speechify-server/services/book_search.py
import httpx

OPENLIBRARY = "https://openlibrary.org/search.json"
GUTENDEX = "https://gutendex.com/books/"

async def search_book(title: str) -> dict:
    async with httpx.AsyncClient(timeout=10) as client:
        # Get cover from Open Library
        ol_resp = await client.get(OPENLIBRARY, params={"title": title, "limit": 3, "fields": "title,author_name,cover_i"})
        docs = ol_resp.json().get("docs", [])

        # Find PDF from Gutenberg
        pdf_url = await _gutenberg_pdf(title, client)
        if not pdf_url:
            return {"found": False}

        cover_url = None
        author = None
        ol_title = title
        if docs:
            doc = docs[0]
            cover_id = doc.get("cover_i")
            if cover_id:
                cover_url = f"https://covers.openlibrary.org/b/id/{cover_id}-M.jpg"
            author_list = doc.get("author_name", [])
            author = author_list[0] if author_list else None
            ol_title = doc.get("title", title)

        return {
            "found": True,
            "title": ol_title,
            "author": author,
            "cover_url": cover_url,
            "pdf_url": pdf_url,
            "source": "gutenberg",
        }

async def _gutenberg_pdf(title: str, client: httpx.AsyncClient) -> str | None:
    resp = await client.get(GUTENDEX, params={"search": title})
    for book in resp.json().get("results", []):
        formats = book.get("formats", {})
        for mime in ["application/pdf", "text/plain; charset=utf-8", "text/plain"]:
            if mime in formats:
                return formats[mime]
    return None
```

- [ ] **Step 5: Write books router (search + download stubs)**

```python
# speechify-server/routers/books.py
from fastapi import APIRouter, UploadFile, File, Form, HTTPException
import tempfile, os
from models.schemas import BookSearchRequest, BookResult, DownloadRequest, DownloadResult, Chapter
from services import book_search, pdf_processor

router = APIRouter()

@router.post("/search-book", response_model=BookResult)
async def search_book_endpoint(request: BookSearchRequest):
    return await book_search.search_book(request.title)

@router.post("/download-book", response_model=DownloadResult)
async def download_book(request: DownloadRequest):
    chapters = await pdf_processor.download_and_split(request.pdf_url, request.book_id)
    return DownloadResult(book_id=request.book_id, chapters=chapters)

@router.post("/upload-book", response_model=DownloadResult)
async def upload_book(pdf: UploadFile = File(...), book_id: str = Form(...)):
    contents = await pdf.read()
    with tempfile.NamedTemporaryFile(suffix=".pdf", delete=False) as tmp:
        tmp.write(contents)
        tmp_path = tmp.name
    try:
        text = pdf_processor.extract_text(tmp_path)
        chapters_raw = pdf_processor.split_text_into_chapters(text)
        pdf_processor.save_chapters(book_id, chapters_raw)
        chapters = [Chapter(index=i, title=ch["title"], char_count=len(ch["text"])) for i, ch in enumerate(chapters_raw)]
        return DownloadResult(book_id=book_id, chapters=chapters)
    finally:
        os.unlink(tmp_path)

@router.get("/chapter-text/{book_id}/{chapter_index}")
def get_chapter_text(book_id: str, chapter_index: int):
    try:
        text = pdf_processor.get_chapter_text(book_id, chapter_index)
        return {"text": text}
    except FileNotFoundError:
        raise HTTPException(status_code=404, detail="Chapter not found")
```

- [ ] **Step 6: Mount books router**

```python
# speechify-server/main.py
from fastapi import FastAPI
from routers import health, books

app = FastAPI(title="Speechify Server")
app.include_router(health.router)
app.include_router(books.router)
```

- [ ] **Step 7: Run tests — must pass**

```bash
pytest tests/test_books.py -v
```
Expected: both tests PASS

- [ ] **Step 8: Commit**

```bash
git add .
git commit -m "feat: add book search service and books router"
```

---

## Task 3: PDF Processor (Download + Chapter Split + Chapter Cache)

**Files:**
- Create: `speechify-server/services/pdf_processor.py`

- [ ] **Step 1: Write failing tests**

Add to `speechify-server/tests/test_books.py`:

```python
from services.pdf_processor import split_text_into_chapters, clean_text

def test_split_detects_chapter_headings():
    text = (
        "Chapter 1\nSome content that is long enough to count as a real chapter body. " * 5 + "\n"
        "Chapter 2\nMore content here that is also long enough to be valid. " * 5
    )
    chapters = split_text_into_chapters(text)
    assert len(chapters) == 2
    assert chapters[0]["title"] == "Chapter 1"
    assert chapters[1]["title"] == "Chapter 2"

def test_split_falls_back_to_single_chapter():
    text = "No headings here. Just some prose that goes on for a while. " * 10
    chapters = split_text_into_chapters(text)
    assert len(chapters) == 1
    assert chapters[0]["title"] == "Full Book"

def test_clean_text_removes_lone_page_numbers():
    text = "Real sentence here.\n\n42\n\nAnother real sentence."
    result = clean_text(text)
    assert "42" not in result
    assert "Real sentence here" in result
```

- [ ] **Step 2: Run — confirm failure**

```bash
pytest tests/test_books.py::test_split_detects_chapter_headings -v
```
Expected: `ImportError: cannot import name 'split_text_into_chapters'`

- [ ] **Step 3: Write pdf_processor.py**

```python
# speechify-server/services/pdf_processor.py
import re
import os
import httpx
import pypdf
from pathlib import Path
from models.schemas import Chapter

BOOKS_DIR = Path("cache/books")
BOOKS_DIR.mkdir(parents=True, exist_ok=True)

CHAPTER_RE = re.compile(
    r"^(?:CHAPTER|Chapter|PART|Part)\s+"
    r"(?:\d+|[IVXLC]+|One|Two|Three|Four|Five|Six|Seven|Eight|Nine|Ten"
    r"|Eleven|Twelve|Thirteen|Fourteen|Fifteen)\b",
    re.MULTILINE,
)

async def download_and_split(url: str, book_id: str) -> list[Chapter]:
    pdf_path = BOOKS_DIR / f"{book_id}.pdf"
    if not pdf_path.exists():
        async with httpx.AsyncClient(timeout=60, follow_redirects=True) as client:
            resp = await client.get(url)
            resp.raise_for_status()
            content_type = resp.headers.get("content-type", "")
            if "text/plain" in content_type:
                # Plain text from Gutenberg
                (BOOKS_DIR / f"{book_id}.txt").write_text(resp.text)
                text = resp.text
            else:
                pdf_path.write_bytes(resp.content)
                text = extract_text(str(pdf_path))
    else:
        text = extract_text(str(pdf_path))

    txt_path = BOOKS_DIR / f"{book_id}.txt"
    if txt_path.exists():
        text = txt_path.read_text()

    chapters_raw = split_text_into_chapters(text)
    save_chapters(book_id, chapters_raw)
    return [
        Chapter(index=i, title=ch["title"], char_count=len(ch["text"]))
        for i, ch in enumerate(chapters_raw)
    ]

def extract_text(path: str) -> str:
    reader = pypdf.PdfReader(path)
    return "\n".join(page.extract_text() or "" for page in reader.pages)

def split_text_into_chapters(full_text: str) -> list[dict]:
    matches = list(CHAPTER_RE.finditer(full_text))
    if len(matches) >= 2:
        chapters = []
        for i, m in enumerate(matches):
            end = matches[i + 1].start() if i + 1 < len(matches) else len(full_text)
            body = clean_text(full_text[m.start():end])
            if len(body) > 100:
                chapters.append({"title": m.group().strip(), "text": body})
        if chapters:
            return chapters
    return [{"title": "Full Book", "text": clean_text(full_text)}]

def save_chapters(book_id: str, chapters: list[dict]) -> None:
    out = BOOKS_DIR / book_id
    out.mkdir(exist_ok=True)
    for i, ch in enumerate(chapters):
        (out / f"{i:03d}.txt").write_text(ch["text"])

def get_chapter_text(book_id: str, chapter_index: int) -> str:
    path = BOOKS_DIR / book_id / f"{chapter_index:03d}.txt"
    if not path.exists():
        raise FileNotFoundError(f"Chapter {chapter_index} not found for book {book_id}")
    return path.read_text()

def clean_text(text: str) -> str:
    text = re.sub(r"^\s*\d+\s*$", "", text, flags=re.MULTILINE)
    text = re.sub(r"\n{3,}", "\n\n", text)
    text = re.sub(r"-\n(\w)", r"\1", text)
    text = re.sub(r"(?<!\n)\n(?!\n)", " ", text)
    return text.strip()
```

- [ ] **Step 4: Run all tests — must pass**

```bash
pytest tests/ -v
```
Expected: all 5 tests PASS

- [ ] **Step 5: Commit**

```bash
git add .
git commit -m "feat: add PDF download, text extraction, and chapter split service"
```

---

## Task 4: OCR Endpoint

**Files:**
- Create: `speechify-server/services/ocr.py`
- Create: `speechify-server/routers/ocr.py`
- Modify: `speechify-server/main.py`
- Create: `speechify-server/tests/test_ocr.py`

Prerequisite: `brew install tesseract` (already done in Task 1 Step 3).

- [ ] **Step 1: Write the failing test**

```python
# speechify-server/tests/test_ocr.py
from unittest.mock import patch
import io
from PIL import Image

def test_ocr_returns_extracted_text(client):
    img = Image.new("RGB", (300, 80), color="white")
    buf = io.BytesIO()
    img.save(buf, format="JPEG")
    buf.seek(0)
    with patch("services.ocr.extract_title", return_value="Moby Dick"):
        response = client.post("/ocr-title", files={"image": ("test.jpg", buf, "image/jpeg")})
    assert response.status_code == 200
    assert response.json()["text"] == "Moby Dick"

def test_ocr_returns_empty_string_on_blank_image(client):
    img = Image.new("RGB", (100, 30), color="white")
    buf = io.BytesIO()
    img.save(buf, format="JPEG")
    buf.seek(0)
    # No mock — real pytesseract on blank image returns ""
    response = client.post("/ocr-title", files={"image": ("blank.jpg", buf, "image/jpeg")})
    assert response.status_code == 200
    assert isinstance(response.json()["text"], str)
```

- [ ] **Step 2: Run — confirm failure**

```bash
pytest tests/test_ocr.py -v
```
Expected: `ERROR` — `/ocr-title` route not found

- [ ] **Step 3: Write services/ocr.py**

```python
# speechify-server/services/ocr.py
import pytesseract
from PIL import Image
import io

def extract_title(image_bytes: bytes) -> str:
    img = Image.open(io.BytesIO(image_bytes))
    text = pytesseract.image_to_string(img)
    lines = [line.strip() for line in text.splitlines() if line.strip()]
    return lines[0] if lines else ""
```

- [ ] **Step 4: Write routers/ocr.py**

```python
# speechify-server/routers/ocr.py
from fastapi import APIRouter, UploadFile, File
from services.ocr import extract_title

router = APIRouter()

@router.post("/ocr-title")
async def ocr_title(image: UploadFile = File(...)):
    contents = await image.read()
    text = extract_title(contents)
    return {"text": text}
```

- [ ] **Step 5: Mount router**

```python
# speechify-server/main.py
from fastapi import FastAPI
from routers import health, books, ocr

app = FastAPI(title="Speechify Server")
app.include_router(health.router)
app.include_router(books.router)
app.include_router(ocr.router)
```

- [ ] **Step 6: Run all tests — must pass**

```bash
pytest tests/ -v
```
Expected: all 7 tests PASS

- [ ] **Step 7: Commit**

```bash
git add .
git commit -m "feat: add OCR endpoint (pytesseract) for book title extraction"
```

---

## Task 5: Edge TTS Synthesis Endpoint + Audio Cache

**Files:**
- Create: `speechify-server/services/tts_edge.py`
- Create: `speechify-server/routers/synthesis.py`
- Modify: `speechify-server/main.py`
- Create: `speechify-server/tests/test_synthesis.py`

- [ ] **Step 1: Write failing tests**

```python
# speechify-server/tests/test_synthesis.py
from unittest.mock import patch, AsyncMock

def test_voices_returns_non_empty_list(client):
    response = client.get("/voices")
    assert response.status_code == 200
    data = response.json()
    assert isinstance(data, list)
    assert len(data) > 0
    assert "voice_id" in data[0]
    assert "name" in data[0]

def test_synthesize_returns_audio_bytes(client):
    with patch("services.tts_edge.synthesize_chapter", new_callable=AsyncMock, return_value=b"FAKE_MP3"):
        response = client.post("/synthesize", json={
            "text": "Hello world.",
            "voice_id": "en-US-GuyNeural",
            "speed": 1.0,
            "book_id": "test_book",
            "chapter_index": 0
        })
    assert response.status_code == 200
    assert response.headers["content-type"] == "audio/mpeg"
    assert response.content == b"FAKE_MP3"
```

- [ ] **Step 2: Run — confirm failure**

```bash
pytest tests/test_synthesis.py -v
```
Expected: `ERROR` — routes not found

- [ ] **Step 3: Write services/tts_edge.py**

```python
# speechify-server/services/tts_edge.py
import asyncio
import subprocess
from pathlib import Path
import edge_tts

AUDIO_CACHE = Path("cache/audio")
AUDIO_CACHE.mkdir(parents=True, exist_ok=True)

VOICES = [
    {"voice_id": "en-US-GuyNeural",    "name": "Guy (Default)"},
    {"voice_id": "en-US-BrianNeural",  "name": "Brian"},
    {"voice_id": "en-US-AvaNeural",    "name": "Ava"},
    {"voice_id": "en-US-JennyNeural",  "name": "Jenny"},
    {"voice_id": "en-US-AriaNeural",   "name": "Aria"},
]

def list_voices() -> list[dict]:
    return VOICES

async def synthesize_chapter(
    text: str,
    voice_id: str,
    speed: float,
    book_id: str,
    chapter_index: int,
) -> bytes:
    speed_key = str(speed).replace(".", "_")
    cache_path = AUDIO_CACHE / book_id / f"{chapter_index:03d}_{voice_id}_{speed_key}.mp3"
    cache_path.parent.mkdir(parents=True, exist_ok=True)

    if cache_path.exists():
        return cache_path.read_bytes()

    raw_path = cache_path.with_suffix(".raw.mp3")
    communicate = edge_tts.Communicate(text[:9000], voice_id)
    await communicate.save(str(raw_path))

    if speed == 1.0:
        raw_path.rename(cache_path)
    else:
        atempo = _atempo_filter(speed)
        subprocess.run(
            ["ffmpeg", "-y", "-i", str(raw_path), "-filter:a", atempo, str(cache_path)],
            check=True,
            capture_output=True,
        )
        raw_path.unlink(missing_ok=True)

    return cache_path.read_bytes()

def _atempo_filter(speed: float) -> str:
    # ffmpeg atempo range: 0.5–2.0; chain for values outside that range
    if 0.5 <= speed <= 2.0:
        return f"atempo={speed}"
    if speed < 0.5:
        return f"atempo=0.5,atempo={speed / 0.5:.4f}"
    return f"atempo=2.0,atempo={speed / 2.0:.4f}"
```

- [ ] **Step 4: Write routers/synthesis.py**

```python
# speechify-server/routers/synthesis.py
from fastapi import APIRouter
from fastapi.responses import Response
from models.schemas import SynthesizeRequest
from services import tts_edge

router = APIRouter()

@router.get("/voices")
def list_voices():
    return tts_edge.list_voices()

@router.post("/synthesize")
async def synthesize(request: SynthesizeRequest):
    audio = await tts_edge.synthesize_chapter(
        text=request.text,
        voice_id=request.voice_id,
        speed=request.speed,
        book_id=request.book_id,
        chapter_index=request.chapter_index,
    )
    return Response(content=audio, media_type="audio/mpeg")
```

- [ ] **Step 5: Mount synthesis router**

```python
# speechify-server/main.py
from fastapi import FastAPI
from routers import health, books, ocr, synthesis

app = FastAPI(title="Speechify Server")
app.include_router(health.router)
app.include_router(books.router)
app.include_router(ocr.router)
app.include_router(synthesis.router)
```

- [ ] **Step 6: Run all tests — must pass**

```bash
pytest tests/ -v
```
Expected: all 9 tests PASS

- [ ] **Step 7: Full server smoke test**

```bash
uvicorn main:app --reload --port 8765

# In another terminal:
curl http://localhost:8765/health
curl http://localhost:8765/voices
curl -X POST http://localhost:8765/search-book \
  -H "Content-Type: application/json" \
  -d '{"title": "Moby Dick"}'
```
Expected: health ok, voices list, Moby Dick result with pdf_url

- [ ] **Step 8: Commit**

```bash
git add .
git commit -m "feat: add Edge TTS synthesis endpoint with speed control and audio cache"
```

---

Server complete. All 9 tests passing. Now building the iOS app.

---

## Task 6: Expo App Scaffold + Navigation

**Files:**
- Create: `speechify-app/` (new Expo project)
- Create: `speechify-app/src/types/index.ts`
- Modify: `speechify-app/App.tsx`

- [ ] **Step 1: Create Expo project**

```bash
npx create-expo-app@latest speechify-app --template blank-typescript
cd speechify-app
```

- [ ] **Step 2: Install all dependencies at once**

```bash
npx expo install expo-av expo-camera expo-document-picker expo-file-system \
  @react-navigation/native @react-navigation/native-stack \
  react-native-screens react-native-safe-area-context \
  @react-native-async-storage/async-storage
```

- [ ] **Step 3: Write src/types/index.ts**

```typescript
// src/types/index.ts
export interface Book {
  id: string;
  title: string;
  author: string | null;
  coverUrl: string | null;
  chapters: Chapter[];
  addedAt: number;
}

export interface Chapter {
  index: number;
  title: string;
  charCount: number;
}

export interface BookProgress {
  bookId: string;
  chapterIndex: number;
  positionMs: number;
  updatedAt: number;
}

export interface Voice {
  voice_id: string;
  name: string;
}

export type RootStackParamList = {
  Library: undefined;
  AddBook: undefined;
  Player: { bookId: string };
  Settings: undefined;
};
```

- [ ] **Step 4: Write App.tsx**

```typescript
// App.tsx
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import LibraryScreen from './src/screens/LibraryScreen';
import AddBookScreen from './src/screens/AddBookScreen';
import PlayerScreen from './src/screens/PlayerScreen';
import SettingsScreen from './src/screens/SettingsScreen';
import { RootStackParamList } from './src/types';

const Stack = createNativeStackNavigator<RootStackParamList>();

export default function App() {
  return (
    <NavigationContainer>
      <Stack.Navigator initialRouteName="Library">
        <Stack.Screen name="Library"  component={LibraryScreen}  options={{ title: 'My Books' }} />
        <Stack.Screen name="AddBook"  component={AddBookScreen}  options={{ title: 'Add Book' }} />
        <Stack.Screen name="Player"   component={PlayerScreen}   options={{ title: '' }} />
        <Stack.Screen name="Settings" component={SettingsScreen} options={{ title: 'Settings' }} />
      </Stack.Navigator>
    </NavigationContainer>
  );
}
```

- [ ] **Step 5: Add temporary placeholder screens so app compiles**

```typescript
// src/screens/LibraryScreen.tsx  (placeholder — replaced in Task 9)
import { View, Text } from 'react-native';
export default function LibraryScreen() { return <View><Text>Library</Text></View>; }

// src/screens/AddBookScreen.tsx  (placeholder — replaced in Task 10)
import { View, Text } from 'react-native';
export default function AddBookScreen() { return <View><Text>Add Book</Text></View>; }

// src/screens/PlayerScreen.tsx   (placeholder — replaced in Task 11)
import { View, Text } from 'react-native';
export default function PlayerScreen() { return <View><Text>Player</Text></View>; }

// src/screens/SettingsScreen.tsx (placeholder — replaced in Task 8)
import { View, Text } from 'react-native';
export default function SettingsScreen() { return <View><Text>Settings</Text></View>; }
```

- [ ] **Step 6: Verify TypeScript compiles**

```bash
npx tsc --noEmit
```
Expected: no errors

- [ ] **Step 7: Run on iOS simulator to verify it launches**

```bash
npx expo start
```
Press `i`. Expected: app launches, shows "Library" text, no crashes.

- [ ] **Step 8: Commit**

```bash
git add .
git commit -m "feat: scaffold Expo app with navigation stack and placeholder screens"
```

---

## Task 7: API Client + Storage Service

**Files:**
- Create: `speechify-app/src/services/api.ts`
- Create: `speechify-app/src/services/storage.ts`

- [ ] **Step 1: Write src/services/storage.ts**

```typescript
// src/services/storage.ts
import AsyncStorage from '@react-native-async-storage/async-storage';
import { Book, BookProgress } from '../types';

const BOOKS_KEY = 'books_v1';
const PROGRESS_PREFIX = 'progress:';
const SERVER_URL_KEY = 'serverUrl';

export async function getServerUrl(): Promise<string | null> {
  return AsyncStorage.getItem(SERVER_URL_KEY);
}

export async function saveServerUrl(url: string): Promise<void> {
  await AsyncStorage.setItem(SERVER_URL_KEY, url.trim().replace(/\/$/, ''));
}

export async function getBooks(): Promise<Book[]> {
  const raw = await AsyncStorage.getItem(BOOKS_KEY);
  return raw ? JSON.parse(raw) : [];
}

export async function saveBook(book: Book): Promise<void> {
  const books = await getBooks();
  const idx = books.findIndex(b => b.id === book.id);
  if (idx >= 0) {
    books[idx] = book;
  } else {
    books.unshift(book);
  }
  await AsyncStorage.setItem(BOOKS_KEY, JSON.stringify(books));
}

export async function getBook(bookId: string): Promise<Book | null> {
  const books = await getBooks();
  return books.find(b => b.id === bookId) ?? null;
}

export async function saveProgress(progress: BookProgress): Promise<void> {
  await AsyncStorage.setItem(`${PROGRESS_PREFIX}${progress.bookId}`, JSON.stringify(progress));
}

export async function getProgress(bookId: string): Promise<BookProgress | null> {
  const raw = await AsyncStorage.getItem(`${PROGRESS_PREFIX}${bookId}`);
  return raw ? JSON.parse(raw) : null;
}
```

- [ ] **Step 2: Write src/services/api.ts**

```typescript
// src/services/api.ts
import { getServerUrl } from './storage';

async function baseUrl(): Promise<string> {
  const url = await getServerUrl();
  if (!url) throw new Error('Server URL not configured. Open Settings first.');
  return url;
}

export async function checkHealth(): Promise<boolean> {
  try {
    const url = await baseUrl();
    const res = await fetch(`${url}/health`, { signal: AbortSignal.timeout(5000) });
    return (await res.json()).status === 'ok';
  } catch {
    return false;
  }
}

export async function searchBook(title: string): Promise<any> {
  const url = await baseUrl();
  const res = await fetch(`${url}/search-book`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ title }),
  });
  return res.json();
}

export async function downloadBook(pdfUrl: string, bookId: string): Promise<any> {
  const url = await baseUrl();
  const res = await fetch(`${url}/download-book`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ pdf_url: pdfUrl, book_id: bookId }),
  });
  return res.json();
}

export async function uploadBook(fileUri: string, fileName: string, bookId: string): Promise<any> {
  const url = await baseUrl();
  const form = new FormData();
  form.append('pdf', { uri: fileUri, name: fileName, type: 'application/pdf' } as any);
  form.append('book_id', bookId);
  const res = await fetch(`${url}/upload-book`, { method: 'POST', body: form });
  return res.json();
}

export async function ocrTitle(imageUri: string): Promise<string> {
  const url = await baseUrl();
  const form = new FormData();
  form.append('image', { uri: imageUri, name: 'photo.jpg', type: 'image/jpeg' } as any);
  const res = await fetch(`${url}/ocr-title`, { method: 'POST', body: form });
  return (await res.json()).text ?? '';
}

export async function getVoices(): Promise<any[]> {
  const url = await baseUrl();
  const res = await fetch(`${url}/voices`);
  return res.json();
}

export async function getChapterText(bookId: string, chapterIndex: number): Promise<string> {
  const url = await baseUrl();
  const res = await fetch(`${url}/chapter-text/${bookId}/${chapterIndex}`);
  return (await res.json()).text ?? '';
}
```

- [ ] **Step 3: Verify TypeScript**

```bash
npx tsc --noEmit
```
Expected: no errors

- [ ] **Step 4: Commit**

```bash
git add .
git commit -m "feat: add API client and AsyncStorage service"
```

---

## Task 8: Settings Screen

**Files:**
- Modify: `speechify-app/src/screens/SettingsScreen.tsx`

- [ ] **Step 1: Replace placeholder with real SettingsScreen**

```typescript
// src/screens/SettingsScreen.tsx
import React, { useState, useEffect } from 'react';
import {
  View, Text, TextInput, TouchableOpacity,
  StyleSheet, ActivityIndicator
} from 'react-native';
import { saveServerUrl, getServerUrl } from '../services/storage';
import { checkHealth } from '../services/api';

export default function SettingsScreen() {
  const [url, setUrl] = useState('');
  const [status, setStatus] = useState<'idle' | 'ok' | 'error'>('idle');
  const [testing, setTesting] = useState(false);

  useEffect(() => {
    getServerUrl().then(saved => { if (saved) setUrl(saved); });
  }, []);

  async function handleSaveAndTest() {
    await saveServerUrl(url);
    setTesting(true);
    const ok = await checkHealth();
    setStatus(ok ? 'ok' : 'error');
    setTesting(false);
  }

  return (
    <View style={styles.container}>
      <Text style={styles.label}>Mac mini Tailscale address</Text>
      <TextInput
        style={styles.input}
        value={url}
        onChangeText={setUrl}
        placeholder="http://100.x.x.x:8765"
        autoCapitalize="none"
        keyboardType="url"
      />
      <TouchableOpacity style={styles.btn} onPress={handleSaveAndTest} disabled={testing}>
        {testing
          ? <ActivityIndicator color="#fff" />
          : <Text style={styles.btnText}>Save & Test Connection</Text>}
      </TouchableOpacity>
      {status !== 'idle' && (
        <View style={[styles.badge, status === 'ok' ? styles.badgeOk : styles.badgeErr]}>
          <Text style={styles.badgeText}>
            {status === 'ok' ? '● Connected' : '● Cannot reach server'}
          </Text>
        </View>
      )}
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, padding: 24, backgroundColor: '#fff' },
  label:     { fontSize: 15, fontWeight: '600', marginBottom: 8 },
  input:     { borderWidth: 1, borderColor: '#d1d5db', borderRadius: 8, padding: 12, fontSize: 15, marginBottom: 16 },
  btn:       { backgroundColor: '#3b82f6', borderRadius: 8, padding: 14, alignItems: 'center' },
  btnText:   { color: '#fff', fontSize: 16, fontWeight: '600' },
  badge:     { marginTop: 16, borderRadius: 8, padding: 12, alignItems: 'center' },
  badgeOk:   { backgroundColor: '#22c55e' },
  badgeErr:  { backgroundColor: '#ef4444' },
  badgeText: { color: '#fff', fontWeight: '600' },
});
```

- [ ] **Step 2: Run app, navigate to Settings, test with server running**

```bash
npx expo start
```
Enter `http://localhost:8765`, tap "Save & Test". Expected: green "Connected" badge.

- [ ] **Step 3: Commit**

```bash
git add .
git commit -m "feat: add settings screen with server URL and connection test"
```

---

## Task 9: Library Screen

**Files:**
- Create: `speechify-app/src/components/BookCard.tsx`
- Modify: `speechify-app/src/screens/LibraryScreen.tsx`

- [ ] **Step 1: Write BookCard component**

```typescript
// src/components/BookCard.tsx
import React from 'react';
import { View, Text, Image, TouchableOpacity, StyleSheet } from 'react-native';
import { Book } from '../types';

interface Props {
  book: Book;
  onPress: () => void;
}

export default function BookCard({ book, onPress }: Props) {
  return (
    <TouchableOpacity style={styles.card} onPress={onPress} activeOpacity={0.7}>
      {book.coverUrl ? (
        <Image source={{ uri: book.coverUrl }} style={styles.cover} />
      ) : (
        <View style={[styles.cover, styles.coverFallback]}>
          <Text style={styles.coverLetter}>{book.title[0]?.toUpperCase()}</Text>
        </View>
      )}
      <View style={styles.info}>
        <Text style={styles.title} numberOfLines={2}>{book.title}</Text>
        {book.author && <Text style={styles.author} numberOfLines={1}>{book.author}</Text>}
        <Text style={styles.chapters}>{book.chapters.length} chapter{book.chapters.length !== 1 ? 's' : ''}</Text>
      </View>
    </TouchableOpacity>
  );
}

const styles = StyleSheet.create({
  card:         { flexDirection: 'row', padding: 14, borderBottomWidth: 1, borderColor: '#f3f4f6' },
  cover:        { width: 56, height: 76, borderRadius: 4, flexShrink: 0 },
  coverFallback:{ backgroundColor: '#e5e7eb', alignItems: 'center', justifyContent: 'center' },
  coverLetter:  { fontSize: 22, fontWeight: '700', color: '#6b7280' },
  info:         { flex: 1, marginLeft: 14, justifyContent: 'center', gap: 4 },
  title:        { fontSize: 16, fontWeight: '600', color: '#111827' },
  author:       { fontSize: 14, color: '#6b7280' },
  chapters:     { fontSize: 13, color: '#9ca3af' },
});
```

- [ ] **Step 2: Replace LibraryScreen placeholder**

```typescript
// src/screens/LibraryScreen.tsx
import React, { useState, useCallback } from 'react';
import { View, FlatList, Text, TouchableOpacity, StyleSheet } from 'react-native';
import { useFocusEffect } from '@react-navigation/native';
import { NativeStackNavigationProp } from '@react-navigation/native-stack';
import { RootStackParamList, Book } from '../types';
import { getBooks } from '../services/storage';
import BookCard from '../components/BookCard';

type Props = { navigation: NativeStackNavigationProp<RootStackParamList, 'Library'> };

export default function LibraryScreen({ navigation }: Props) {
  const [books, setBooks] = useState<Book[]>([]);

  useFocusEffect(useCallback(() => {
    getBooks().then(setBooks);
  }, []));

  return (
    <View style={styles.container}>
      <View style={styles.header}>
        <TouchableOpacity onPress={() => navigation.navigate('Settings')}>
          <Text style={styles.gear}>⚙</Text>
        </TouchableOpacity>
        <TouchableOpacity style={styles.addBtn} onPress={() => navigation.navigate('AddBook')}>
          <Text style={styles.addBtnText}>+ Add Book</Text>
        </TouchableOpacity>
      </View>

      {books.length === 0 ? (
        <View style={styles.empty}>
          <Text style={styles.emptyTitle}>No books yet</Text>
          <Text style={styles.emptySub}>Tap "+ Add Book" to scan a title or upload a PDF.</Text>
        </View>
      ) : (
        <FlatList
          data={books}
          keyExtractor={b => b.id}
          renderItem={({ item }) => (
            <BookCard
              book={item}
              onPress={() => navigation.navigate('Player', { bookId: item.id })}
            />
          )}
        />
      )}
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#fff' },
  header:    { flexDirection: 'row', justifyContent: 'space-between', alignItems: 'center', padding: 16 },
  gear:      { fontSize: 24 },
  addBtn:    { backgroundColor: '#3b82f6', borderRadius: 8, paddingHorizontal: 16, paddingVertical: 8 },
  addBtnText:{ color: '#fff', fontWeight: '600', fontSize: 15 },
  empty:     { flex: 1, alignItems: 'center', justifyContent: 'center', padding: 40 },
  emptyTitle:{ fontSize: 20, fontWeight: '700', marginBottom: 8 },
  emptySub:  { fontSize: 15, color: '#6b7280', textAlign: 'center' },
});
```

- [ ] **Step 3: Run and verify library screen manually**

Expected: empty state with "+ Add Book" button and settings gear. Tapping settings navigates correctly.

- [ ] **Step 4: Commit**

```bash
git add .
git commit -m "feat: add library screen and book card component"
```

---

## Task 10: Add Book Screen (Upload + Camera Flows)

**Files:**
- Modify: `speechify-app/app.json` (camera permission)
- Modify: `speechify-app/src/screens/AddBookScreen.tsx`

- [ ] **Step 1: Add camera permission to app.json**

Inside the `"expo"` key in `app.json`:
```json
"plugins": [
  ["expo-camera", {
    "cameraPermission": "Allow Speechify to use your camera to scan book titles."
  }]
],
"ios": {
  "bundleIdentifier": "com.intermatrixnaut.speechify",
  "infoPlist": {
    "NSCameraUsageDescription": "Used to photograph book titles for automatic search."
  }
}
```

- [ ] **Step 2: Replace AddBookScreen placeholder**

```typescript
// src/screens/AddBookScreen.tsx
import React, { useState, useRef } from 'react';
import {
  View, Text, TouchableOpacity, StyleSheet,
  ActivityIndicator, Alert, Image
} from 'react-native';
import { CameraView, useCameraPermissions } from 'expo-camera';
import * as DocumentPicker from 'expo-document-picker';
import { NativeStackNavigationProp } from '@react-navigation/native-stack';
import { RootStackParamList, Book } from '../types';
import { searchBook, downloadBook, uploadBook, ocrTitle } from '../services/api';
import { saveBook } from '../services/storage';

type Props = { navigation: NativeStackNavigationProp<RootStackParamList, 'AddBook'> };
type Mode = 'menu' | 'camera' | 'confirm';

export default function AddBookScreen({ navigation }: Props) {
  const [mode, setMode] = useState<Mode>('menu');
  const [permission, requestPermission] = useCameraPermissions();
  const [result, setResult] = useState<any>(null);
  const [loading, setLoading] = useState(false);
  const camRef = useRef<CameraView>(null);

  async function handleScan() {
    if (!permission?.granted) await requestPermission();
    setMode('camera');
  }

  async function handleCapture() {
    if (!camRef.current) return;
    setLoading(true);
    try {
      const photo = await camRef.current.takePictureAsync({ quality: 0.7 });
      const title = await ocrTitle(photo!.uri);
      if (!title) {
        Alert.alert('Could not read title', 'Try again — hold steady and ensure the title is lit.', [
          { text: 'Try Again' },
        ]);
        setLoading(false);
        return;
      }
      const found = await searchBook(title);
      if (!found.found) {
        Alert.alert(
          'Book not found',
          `"${title}" wasn't found in free sources. Upload a PDF instead?`,
          [
            { text: 'Upload PDF', onPress: () => { setMode('menu'); handleUpload(); } },
            { text: 'Try Again', onPress: () => setMode('camera') },
            { text: 'Cancel', style: 'cancel', onPress: () => setMode('menu') },
          ],
        );
      } else {
        setResult(found);
        setMode('confirm');
      }
    } catch (e: any) {
      Alert.alert('Error', e.message);
    }
    setLoading(false);
  }

  async function handleUpload() {
    const picked = await DocumentPicker.getDocumentAsync({ type: 'application/pdf' });
    if (picked.canceled) return;
    const file = picked.assets[0];
    setLoading(true);
    try {
      const bookId = `upload_${Date.now()}`;
      const data = await uploadBook(file.uri, file.name, bookId);
      const book: Book = {
        id: bookId,
        title: file.name.replace(/\.pdf$/i, ''),
        author: null,
        coverUrl: null,
        chapters: data.chapters.map((c: any) => ({ index: c.index, title: c.title, charCount: c.char_count })),
        addedAt: Date.now(),
      };
      await saveBook(book);
      navigation.replace('Player', { bookId });
    } catch (e: any) {
      Alert.alert('Upload failed', e.message);
    }
    setLoading(false);
  }

  async function handleConfirm() {
    if (!result) return;
    setLoading(true);
    try {
      const bookId = `search_${Date.now()}`;
      const data = await downloadBook(result.pdf_url, bookId);
      const book: Book = {
        id: bookId,
        title: result.title,
        author: result.author ?? null,
        coverUrl: result.cover_url ?? null,
        chapters: data.chapters.map((c: any) => ({ index: c.index, title: c.title, charCount: c.char_count })),
        addedAt: Date.now(),
      };
      await saveBook(book);
      navigation.replace('Player', { bookId });
    } catch (e: any) {
      Alert.alert('Download failed', e.message);
    }
    setLoading(false);
  }

  if (mode === 'camera') {
    return (
      <View style={{ flex: 1 }}>
        <CameraView ref={camRef} style={{ flex: 1 }} facing="back" />
        <View style={styles.camControls}>
          <TouchableOpacity style={styles.shutter} onPress={handleCapture} disabled={loading}>
            {loading ? <ActivityIndicator color="#fff" /> : <View style={styles.shutterInner} />}
          </TouchableOpacity>
          <TouchableOpacity onPress={() => setMode('menu')}>
            <Text style={styles.camCancel}>Cancel</Text>
          </TouchableOpacity>
        </View>
      </View>
    );
  }

  if (mode === 'confirm' && result) {
    return (
      <View style={styles.center}>
        {result.cover_url && (
          <Image source={{ uri: result.cover_url }} style={styles.previewCover} />
        )}
        <Text style={styles.previewTitle}>{result.title}</Text>
        {result.author && <Text style={styles.previewAuthor}>by {result.author}</Text>}
        <TouchableOpacity style={styles.primary} onPress={handleConfirm} disabled={loading}>
          {loading ? <ActivityIndicator color="#fff" /> : <Text style={styles.primaryTxt}>Add This Book</Text>}
        </TouchableOpacity>
        <TouchableOpacity style={styles.secondary} onPress={() => setMode('menu')}>
          <Text style={styles.secondaryTxt}>Not the right book</Text>
        </TouchableOpacity>
      </View>
    );
  }

  return (
    <View style={styles.center}>
      <TouchableOpacity style={styles.primary} onPress={handleScan}>
        <Text style={styles.primaryTxt}>📷  Scan Book Title</Text>
      </TouchableOpacity>
      <Text style={styles.or}>— or —</Text>
      <TouchableOpacity style={styles.secondary} onPress={handleUpload} disabled={loading}>
        {loading ? <ActivityIndicator /> : <Text style={styles.secondaryTxt}>Upload PDF from Files</Text>}
      </TouchableOpacity>
    </View>
  );
}

const styles = StyleSheet.create({
  center:        { flex: 1, padding: 24, backgroundColor: '#fff', alignItems: 'center', justifyContent: 'center', gap: 12 },
  primary:       { backgroundColor: '#3b82f6', borderRadius: 12, padding: 16, width: '100%', alignItems: 'center' },
  primaryTxt:    { color: '#fff', fontSize: 17, fontWeight: '700' },
  secondary:     { borderWidth: 1, borderColor: '#d1d5db', borderRadius: 12, padding: 16, width: '100%', alignItems: 'center' },
  secondaryTxt:  { fontSize: 17, color: '#374151' },
  or:            { color: '#9ca3af', fontSize: 14 },
  camControls:   { position: 'absolute', bottom: 48, width: '100%', alignItems: 'center' },
  shutter:       { width: 72, height: 72, borderRadius: 36, backgroundColor: 'rgba(255,255,255,0.25)', alignItems: 'center', justifyContent: 'center', borderWidth: 3, borderColor: '#fff' },
  shutterInner:  { width: 56, height: 56, borderRadius: 28, backgroundColor: '#fff' },
  camCancel:     { color: '#fff', marginTop: 16, fontSize: 16 },
  previewCover:  { width: 120, height: 160, borderRadius: 8, marginBottom: 16 },
  previewTitle:  { fontSize: 22, fontWeight: '700', textAlign: 'center' },
  previewAuthor: { fontSize: 16, color: '#6b7280', marginBottom: 8 },
});
```

- [ ] **Step 3: Run on device/simulator and test both flows**

Flow A (upload): Add Book → Upload PDF → pick any PDF → navigates to Player with chapter list.
Flow B (scan): Add Book → Scan → point at book cover → capture → confirm screen → "Add This Book" → Player.

- [ ] **Step 4: Commit**

```bash
git add .
git commit -m "feat: add book screen with camera scan and PDF upload flows"
```

---

## Task 11: Player Screen + Background Audio

**Files:**
- Create: `speechify-app/src/services/player.ts`
- Create: `speechify-app/src/components/SpeedSelector.tsx`
- Create: `speechify-app/src/components/ChapterList.tsx`
- Modify: `speechify-app/src/screens/PlayerScreen.tsx`

- [ ] **Step 1: Write src/services/player.ts**

```typescript
// src/services/player.ts
import { Audio, AVPlaybackStatus } from 'expo-av';
import * as FileSystem from 'expo-file-system';
import { getServerUrl } from './storage';

let _sound: Audio.Sound | null = null;

export async function initAudioSession(): Promise<void> {
  await Audio.setAudioModeAsync({
    allowsRecordingIOS: false,
    staysActiveInBackground: true,
    playsInSilentModeIOS: true,
  });
}

export async function loadChapter(
  bookId: string,
  chapterIndex: number,
  voiceId: string,
  speed: number,
  startMs: number,
  onStatus: (s: AVPlaybackStatus) => void,
): Promise<void> {
  if (_sound) {
    await _sound.unloadAsync();
    _sound = null;
  }

  const serverUrl = await getServerUrl();
  if (!serverUrl) throw new Error('Server URL not configured');

  // Fetch chapter text
  const textRes = await fetch(`${serverUrl}/chapter-text/${bookId}/${chapterIndex}`);
  const { text } = await textRes.json();

  // Download synthesized audio to a temp file (expo-av cannot POST)
  const tmpUri = `${FileSystem.cacheDirectory}ch_${bookId}_${chapterIndex}_${speed}.mp3`;
  const { uri } = await FileSystem.downloadAsync(
    `${serverUrl}/synthesize`,
    tmpUri,
    {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        text,
        voice_id: voiceId,
        speed,
        book_id: bookId,
        chapter_index: chapterIndex,
      }),
    },
  );

  const { sound } = await Audio.Sound.createAsync(
    { uri },
    { shouldPlay: true, positionMillis: startMs },
    onStatus,
  );
  _sound = sound;
}

export async function pause(): Promise<void> { await _sound?.pauseAsync(); }
export async function resume(): Promise<void> { await _sound?.playAsync(); }
export async function seekBack30s(): Promise<void> {
  if (!_sound) return;
  const status = await _sound.getStatusAsync();
  if (status.isLoaded) {
    await _sound.setPositionAsync(Math.max(0, status.positionMillis - 30000));
  }
}
export async function getPositionMs(): Promise<number> {
  if (!_sound) return 0;
  const s = await _sound.getStatusAsync();
  return s.isLoaded ? s.positionMillis : 0;
}
export async function unload(): Promise<void> {
  await _sound?.unloadAsync();
  _sound = null;
}
```

- [ ] **Step 2: Write SpeedSelector component**

```typescript
// src/components/SpeedSelector.tsx
import React from 'react';
import { View, Text, TouchableOpacity, StyleSheet } from 'react-native';

const SPEEDS = [0.75, 1.0, 1.25, 1.5, 2.0];

interface Props { speed: number; onChange: (s: number) => void; }

export default function SpeedSelector({ speed, onChange }: Props) {
  return (
    <View style={styles.row}>
      {SPEEDS.map(s => (
        <TouchableOpacity
          key={s}
          style={[styles.btn, speed === s && styles.active]}
          onPress={() => onChange(s)}
        >
          <Text style={[styles.label, speed === s && styles.activeLabel]}>
            {s === 1.0 ? '1×' : `${s}×`}
          </Text>
        </TouchableOpacity>
      ))}
    </View>
  );
}

const styles = StyleSheet.create({
  row:         { flexDirection: 'row', gap: 8 },
  btn:         { paddingHorizontal: 12, paddingVertical: 6, borderRadius: 20, borderWidth: 1, borderColor: '#d1d5db' },
  active:      { backgroundColor: '#3b82f6', borderColor: '#3b82f6' },
  label:       { fontSize: 13, color: '#374151' },
  activeLabel: { color: '#fff', fontWeight: '700' },
});
```

- [ ] **Step 3: Write ChapterList component**

```typescript
// src/components/ChapterList.tsx
import React from 'react';
import { ScrollView, TouchableOpacity, Text, StyleSheet } from 'react-native';
import { Chapter } from '../types';

interface Props {
  chapters: Chapter[];
  currentIndex: number;
  onSelect: (index: number) => void;
}

export default function ChapterList({ chapters, currentIndex, onSelect }: Props) {
  return (
    <ScrollView
      horizontal
      showsHorizontalScrollIndicator={false}
      style={styles.scroll}
      contentContainerStyle={styles.content}
    >
      {chapters.map(ch => (
        <TouchableOpacity
          key={ch.index}
          style={[styles.chip, ch.index === currentIndex && styles.chipActive]}
          onPress={() => onSelect(ch.index)}
        >
          <Text
            style={[styles.text, ch.index === currentIndex && styles.textActive]}
            numberOfLines={1}
          >
            {ch.title}
          </Text>
        </TouchableOpacity>
      ))}
    </ScrollView>
  );
}

const styles = StyleSheet.create({
  scroll:     { maxHeight: 44, flexGrow: 0 },
  content:    { paddingHorizontal: 16, gap: 8, alignItems: 'center' },
  chip:       { paddingHorizontal: 14, paddingVertical: 6, borderRadius: 16, backgroundColor: '#f3f4f6', maxWidth: 180 },
  chipActive: { backgroundColor: '#3b82f6' },
  text:       { fontSize: 13, color: '#374151' },
  textActive: { color: '#fff', fontWeight: '600' },
});
```

- [ ] **Step 4: Replace PlayerScreen placeholder**

```typescript
// src/screens/PlayerScreen.tsx
import React, { useState, useEffect, useRef, useCallback } from 'react';
import {
  View, Text, Image, TouchableOpacity,
  StyleSheet, ActivityIndicator, Alert
} from 'react-native';
import { RouteProp } from '@react-navigation/native';
import { NativeStackNavigationProp } from '@react-navigation/native-stack';
import { AVPlaybackStatus } from 'expo-av';
import { RootStackParamList, Book, BookProgress } from '../types';
import { getBook, getProgress, saveProgress } from '../services/storage';
import * as player from '../services/player';
import SpeedSelector from '../components/SpeedSelector';
import ChapterList from '../components/ChapterList';

type Props = {
  navigation: NativeStackNavigationProp<RootStackParamList, 'Player'>;
  route: RouteProp<RootStackParamList, 'Player'>;
};

export default function PlayerScreen({ route }: Props) {
  const { bookId } = route.params;
  const [book, setBook]             = useState<Book | null>(null);
  const [chapterIdx, setChapterIdx] = useState(0);
  const [playing, setPlaying]       = useState(false);
  const [loading, setLoading]       = useState(false);
  const [speed, setSpeed]           = useState(1.0);
  const [voiceId]                   = useState('en-US-GuyNeural');
  const [posMs, setPosMs]           = useState(0);
  const [durMs, setDurMs]           = useState(0);
  const saveTimer = useRef<ReturnType<typeof setInterval> | null>(null);
  const currentChapter = useRef(0);

  useEffect(() => {
    player.initAudioSession();
    (async () => {
      const b = await getBook(bookId);
      if (!b) return;
      setBook(b);
      const prog = await getProgress(bookId);
      const ch = prog?.chapterIndex ?? 0;
      const pos = prog?.positionMs ?? 0;
      setChapterIdx(ch);
      setPosMs(pos);
    })();
    return () => {
      player.unload();
      if (saveTimer.current) clearInterval(saveTimer.current);
    };
  }, [bookId]);

  const onStatus = useCallback((status: AVPlaybackStatus) => {
    if (!status.isLoaded) return;
    setPosMs(status.positionMillis);
    if (status.durationMillis) setDurMs(status.durationMillis);
    if (status.didJustFinish) {
      setBook(b => {
        if (b && currentChapter.current < b.chapters.length - 1) {
          playChapter(currentChapter.current + 1, 0);
        } else {
          setPlaying(false);
        }
        return b;
      });
    }
  }, []);

  async function playChapter(idx: number, startMs: number) {
    setLoading(true);
    setChapterIdx(idx);
    currentChapter.current = idx;
    try {
      await player.loadChapter(bookId, idx, voiceId, speed, startMs, onStatus);
      setPlaying(true);
      startSaving(idx);
    } catch (e: any) {
      Alert.alert('Playback error', e.message);
    }
    setLoading(false);
  }

  function startSaving(idx: number) {
    if (saveTimer.current) clearInterval(saveTimer.current);
    saveTimer.current = setInterval(async () => {
      const pos = await player.getPositionMs();
      await saveProgress({ bookId, chapterIndex: idx, positionMs: pos, updatedAt: Date.now() });
    }, 10000);
  }

  async function handleSpeedChange(s: number) {
    const pos = await player.getPositionMs();
    setSpeed(s);
    await playChapter(chapterIdx, pos);
  }

  function fmt(ms: number) {
    const s = Math.floor(ms / 1000);
    return `${Math.floor(s / 60)}:${String(s % 60).padStart(2, '0')}`;
  }

  if (!book) return <ActivityIndicator style={{ flex: 1 }} size="large" />;

  return (
    <View style={styles.container}>
      {book.coverUrl ? (
        <Image source={{ uri: book.coverUrl }} style={styles.cover} />
      ) : (
        <View style={[styles.cover, styles.coverFb]}>
          <Text style={styles.coverLetter}>{book.title[0]}</Text>
        </View>
      )}

      <Text style={styles.title} numberOfLines={2}>{book.title}</Text>
      {book.author && <Text style={styles.author}>{book.author}</Text>}

      <ChapterList chapters={book.chapters} currentIndex={chapterIdx} onSelect={idx => playChapter(idx, 0)} />

      <View style={styles.times}>
        <Text style={styles.time}>{fmt(posMs)}</Text>
        <Text style={styles.time}>{fmt(durMs)}</Text>
      </View>

      <View style={styles.controls}>
        <TouchableOpacity onPress={() => chapterIdx > 0 && playChapter(chapterIdx - 1, 0)}>
          <Text style={styles.navBtn}>⏮</Text>
        </TouchableOpacity>

        <TouchableOpacity onPress={player.seekBack30s}>
          <Text style={styles.rewindBtn}>↺30</Text>
        </TouchableOpacity>

        <TouchableOpacity
          style={styles.playBtn}
          disabled={loading}
          onPress={() => {
            if (loading) return;
            if (playing) { player.pause(); setPlaying(false); }
            else if (posMs > 0 || chapterIdx > 0) { player.resume(); setPlaying(true); }
            else { playChapter(0, 0); }
          }}
        >
          {loading
            ? <ActivityIndicator color="#fff" />
            : <Text style={styles.playIcon}>{playing ? '⏸' : '▶'}</Text>}
        </TouchableOpacity>

        <TouchableOpacity onPress={() => book && chapterIdx < book.chapters.length - 1 && playChapter(chapterIdx + 1, 0)}>
          <Text style={styles.navBtn}>⏭</Text>
        </TouchableOpacity>
      </View>

      <SpeedSelector speed={speed} onChange={handleSpeedChange} />
    </View>
  );
}

const styles = StyleSheet.create({
  container:   { flex: 1, backgroundColor: '#fff', alignItems: 'center', padding: 20, gap: 10 },
  cover:       { width: 150, height: 200, borderRadius: 8, marginTop: 16 },
  coverFb:     { backgroundColor: '#e5e7eb', alignItems: 'center', justifyContent: 'center' },
  coverLetter: { fontSize: 60, color: '#6b7280' },
  title:       { fontSize: 20, fontWeight: '700', textAlign: 'center', paddingHorizontal: 16 },
  author:      { fontSize: 15, color: '#6b7280' },
  times:       { flexDirection: 'row', justifyContent: 'space-between', width: '100%', paddingHorizontal: 4 },
  time:        { fontSize: 13, color: '#9ca3af' },
  controls:    { flexDirection: 'row', alignItems: 'center', gap: 24 },
  navBtn:      { fontSize: 30 },
  rewindBtn:   { fontSize: 18, color: '#374151', fontWeight: '600' },
  playBtn:     { width: 68, height: 68, borderRadius: 34, backgroundColor: '#3b82f6', alignItems: 'center', justifyContent: 'center' },
  playIcon:    { fontSize: 26, color: '#fff' },
});
```

- [ ] **Step 5: End-to-end test (manual)**

With server running on `localhost:8765`:
1. Add Book → Upload a PDF
2. Navigate to Player → tap ▶ on chapter 1
3. Verify audio plays through device speaker
4. Change speed to 1.5× → verify audio restarts at faster pace
5. Tap ⏭ → verify chapter advances
6. Lock phone → verify audio continues playing
7. Unlock → verify player still shows correct position
8. Kill app → reopen → verify chapter and position are restored from where you left off

- [ ] **Step 6: Commit**

```bash
git add .
git commit -m "feat: add player screen with audio, chapter nav, speed control, background play, and resume"
```

---

## Task 12: EAS Build + Sideload Instructions

**Files:**
- Create: `speechify-app/eas.json`
- Modify: `speechify-app/app.json`

- [ ] **Step 1: Install EAS CLI and log in**

```bash
npm install -g eas-cli
eas login
```
(Creates a free Expo account if you don't have one — no credit card needed for preview builds.)

- [ ] **Step 2: Write eas.json**

```json
{
  "cli": { "version": ">= 12.0.0" },
  "build": {
    "preview": {
      "ios": {
        "simulator": false,
        "buildConfiguration": "Release"
      }
    }
  }
}
```

- [ ] **Step 3: Confirm bundle ID in app.json**

The `"ios"` block inside `"expo"` in `app.json` must include:
```json
"ios": {
  "bundleIdentifier": "com.intermatrixnaut.speechify",
  "buildNumber": "1",
  "infoPlist": {
    "NSCameraUsageDescription": "Used to photograph book titles for automatic search."
  }
}
```

- [ ] **Step 4: Build the IPA**

```bash
cd speechify-app
eas build --platform ios --profile preview
```
EAS builds in the cloud (~10–15 min). When done, it prints a download URL for the `.ipa` file.
Download it to your Mac mini.

- [ ] **Step 5: Install AltServer on Mac mini**

1. Download AltServer from `altstore.io/altserver` → move to Applications
2. Launch AltServer — it sits in the menu bar
3. AltServer will auto-refresh the IPA on connected phones when they are on the Tailscale network

- [ ] **Step 6: Install on your own iPhone first**

1. Install AltStore on your iPhone: open `altstore.io` in Safari on the iPhone → follow instructions → requires brief USB connection to Mac mini once
2. In AltStore on iPhone, tap **+** → select the `.ipa` file (share it to your phone via AirDrop or Files)
3. App installs. Open it, go to Settings, enter `http://127.0.0.1:8765` (or Tailscale IP), verify green dot.

- [ ] **Step 7: Set up Tailscale for your friend**

1. You: go to `login.tailscale.com` → Machines → Share → Invite by email (your friend's email)
2. Friend: install Tailscale from the App Store
3. Friend: accepts invite email → joins your network
4. Friend gets your Mac mini's Tailscale IP (shown in `login.tailscale.com` → Machines)
5. You: send friend the `.ipa` file via iMessage or Google Drive
6. Friend: install AltStore (one-time USB step, they need a computer), then sideload the IPA
7. Friend: opens app → Settings → enters your Mac mini's Tailscale IP → green dot

- [ ] **Step 8: Verify AltServer auto-refresh works**

With both phones on Tailscale and AltServer running on Mac mini:
- After 6 days, AltStore on each phone should silently renew without any action needed
- If it ever doesn't auto-renew: open AltStore → tap the app → "Refresh"

- [ ] **Step 9: Commit**

```bash
git add eas.json app.json
git commit -m "feat: add EAS build config and complete Phase 1"
```

---

## Self-Review

**Spec coverage check:**

| Requirement | Task |
|---|---|
| Camera → OCR → book title search | Task 4 (server OCR) + Task 10 (camera UI) |
| Not found → "not found" message + upload fallback | Task 10 `handleCapture` alert |
| PDF upload manual fallback | Task 10 `handleUpload` |
| Voice selection (Edge TTS, Phase 1) | Task 5 `/voices` endpoint + Task 11 `voiceId` state |
| Speed presets 0.75×/1×/1.25×/1.5×/2× | Task 5 `_atempo_filter` + Task 11 SpeedSelector |
| Chapter navigation | Task 11 ChapterList + PlayerScreen |
| Resume position (saves every 10s) | Task 11 `startSaving` interval + `getProgress` on load |
| Background playback | Task 11 `staysActiveInBackground: true` in `initAudioSession` |
| iOS AltStore sideload | Task 12 |
| AltServer auto-refresh | Task 12 Step 5 |
| Tailscale connectivity | Task 12 Step 7 |
| Settings screen + connection indicator | Task 8 |
| ⬜ Voice cloning (XTTS-v2) | → Phase 2 plan |
| ⬜ Lock screen playback controls | → Phase 2 plan |

**Placeholder scan:** None found.

**Type consistency:**
- `Chapter` interface uses `charCount` (camelCase) in TypeScript; server returns `char_count` (snake_case) — the mapping `c.char_count` is explicit in AddBookScreen and PlayerScreen wherever chapters are constructed from API responses. Consistent.
- `voice_id` passed as snake_case in JSON body to match server Pydantic field names. Consistent across `api.ts` and `player.ts`.
- `book_id` / `chapter_index` in POST bodies match server Pydantic `SynthesizeRequest`. Consistent.
