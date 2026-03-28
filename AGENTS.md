# AGENTS.md — MoneyPrinterTurbo

## Project Overview

MoneyPrinterTurbo is a Python-based AI video generation tool that automatically generates short videos from a text subject — including video material search, TTS voiceover, subtitle generation, and video composition.

- **Language**: Python 3.11
- **Framework**: Streamlit (WebUI) + FastAPI (REST API)
- **Testing**: Python built-in `unittest` (no pytest)
- **Key deps**: moviepy, edge_tts, fastapi, loguru, pydantic, openai

## Project Structure

```
MoneyPrinterTurbo/
├── app/
│   ├── asgi.py              # FastAPI app factory
│   ├── router.py            # API router aggregation
│   ├── config/              # Config loading (TOML-based)
│   ├── controllers/v1/      # API endpoint handlers
│   ├── models/
│   │   ├── schema.py        # Pydantic request/response models
│   │   └── ...
│   ├── services/
│   │   ├── voice.py         # TTS (Edge TTS, Azure, SiliconFlow)
│   │   ├── video.py         # Video generation with moviepy
│   │   ├── task.py          # Task orchestration
│   │   ├── llm.py           # LLM integration
│   │   ├── material.py       # Video material search
│   │   └── subtitle.py      # Subtitle generation
│   └── utils/
├── webui/
│   └── Main.py              # Streamlit WebUI entry point
├── test/
│   ├── README.md
│   ├── resources/            # Test fixture files
│   └── services/
│       ├── test_video.py
│       ├── test_task.py
│       └── test_voice.py
├── main.py                  # FastAPI entry point
├── requirements.txt
└── Dockerfile
```

## Build / Run / Test Commands

### Run the WebUI
```bash
streamlit run ./webui/Main.py --browser.serverAddress=127.0.0.1 \
  --server.enableCORS=True --browser.gatherUsageStats=False
```

### Run the API server
```bash
uvicorn main:app --host 0.0.0.0 --port 8080 --reload
```

### Run all tests
```bash
python -m unittest discover -s test
```

### Run a specific test file
```bash
python -m unittest test/services/test_video.py
```

### Run a specific test class
```bash
python -m unittest test.services.test_video.TestVideoService
```

### Run a single test method
```bash
python -m unittest test.services.test_video.TestVideoService.test_preprocess_video
```

### Docker build & run (WebUI)
```bash
docker build -t moneyprinterturbo .
docker run -v $(pwd)/config.toml:/MoneyPrinterTurbo/config.toml \
  -v $(pwd)/storage:/MoneyPrinterTurbo/storage -p 8501:8501 moneyprinterturbo
```

### Docker run (API)
```bash
docker run -v $(pwd)/config.toml:/MoneyPrinterTurbo/config.toml \
  -v $(pwd)/storage:/MoneyPrinterTurbo/storage -p 8080:8080 moneyprinterturbo \
  uvicorn main:app --host 0.0.0.0 --port 8080
```

---

## Code Style Guidelines

### Python Version
- Target **Python 3.11**
- No Python 3.10-or-older features (use `|` for union types only where supported)

### Formatting
- Use **4 spaces** for indentation (no tabs)
- Keep lines under **120 characters**
- Use blank lines to separate logical sections within functions
- No enforced formatter — write clean, readable code manually

### Imports
- **Standard library first**, then third-party, then project-local
- Never use `from app import xxx` in services — use **relative imports**:
  ```python
  # ✅ Correct
  from app.services import voice
  from app.models.schema import VideoParams

  # ✅ Correct (within app/services/)
  from ..models.schema import VideoParams

  # ❌ Avoid
  from app.services.voice import something
  ```
- Add `sys.path` manipulation only in test entry points:
  ```python
  sys.path.insert(0, str(Path(__file__).parent.parent.parent))
  ```

### Type Annotations
- Pydantic models (`app/models/schema.py`) use **Pydantic v2** style:
  ```python
  class VideoParams(BaseModel):
      video_subject: str
      video_script: str = ""
      video_aspect: Optional[VideoAspect] = VideoAspect.portrait.value
  ```
- Avoid complex generic types in public schemas — use `Any` or nested models instead
- Internal code does **not** require type annotations everywhere, but public method signatures should be annotated

### Naming Conventions

| Thing | Convention | Example |
|-------|-----------|---------|
| Classes | PascalCase | `VideoParams`, `TaskResponse` |
| Functions / methods | snake_case | `generate_video()`, `siliconflow_tts()` |
| Variables | snake_case | `video_subject`, `task_id` |
| Constants | UPPER_SNAKE | `MAX_RETRIES = 3` |
| Private helpers | _leading_underscore | `_wrap_text()` |
| Enum values | lowercase or PascalCase | `VideoAspect.landscape = "16:9"` |

### Pydantic Models
- All API request/response models inherit from `pydantic.BaseModel`
- Use `Optional[T] = None` for optional fields (not `T | None` outside of annotations)
- Use `Union[T1, T2]` for multiple types (not `T1 | T2`)
- Wrap Pydantic warnings to suppress field-name shadowing warnings:
  ```python
  warnings.filterwarnings("ignore", category=UserWarning, message="Field name.*shadows.*")
  ```

### Error Handling
- **Log errors with loguru**, do not print:
  ```python
  from loguru import logger
  logger.error(f"failed to generate audio: {e}")
  ```
- Use logger levels correctly: `logger.info()` for flow steps, `logger.warning()` for recoverable issues, `logger.error()` for failures
- Never swallow exceptions silently — always log or re-raise
- API endpoints return structured responses:
  ```python
  return {"status": 200, "message": "success", "data": {...}}
  ```

### Logging (loguru)
- Use `logger` from `loguru` — already configured globally in the app
- Never use `print()` in production code — use `logger.info/warning/error`
- Log format: include key context (task_id, voice_name, video_path) so traces are searchable

### File Paths
- Use `os.path` or `pathlib.Path` for cross-platform paths
- Use `str(Path(...))` when passing to functions expecting strings
- Task output directories: `utils.task_dir(task_id)` — do not hardcode paths

### Configuration
- Config is TOML-based, loaded via `app.config.config.load_config()`
- Do **not** use `.env` files for app config — use `config.toml`
- Environment variables can be used for secrets in `.env` but the app reads from `config.toml`

### API Design
- All endpoints return `{"status": int, "message": str, "data": Any}`
- POST endpoints accept JSON body (`application/json`)
- File upload endpoints use `multipart/form-data`
- Task operations are async — `POST /videos` returns `task_id`, clients poll `GET /tasks/{task_id}`

### Video / Subtitle Processing
- Video generation uses **moviepy 2.x**:
  - `TextClip` for subtitles: specify `text_align='center'` for horizontal alignment
  - Use `.with_start()`, `.with_end()`, `.with_duration()` to set clip timing
  - Use `.with_position(("center", y))` for horizontal centering at specific y
- Subtitle text wrapping: use the existing `wrap_text()` function in `video.py` — do not manually split lines

### Testing
- Use Python `unittest.TestCase` as base class
- Test file naming: `test_<module>.py`
- Test method naming: `test_<descriptive_name>`
- Add `sys.path.insert(0, ...)` at top of test files to resolve imports
- Test fixtures go in `test/resources/`
- Mock external dependencies (API calls, file I/O) where possible

### Docker Conventions
- `PYTHONPATH` is set to `/MoneyPrinterTurbo` in the container
- Project root is mounted at `/MoneyPrinterTurbo` in docker-compose
- Changes to local files are reflected in running containers immediately (volume mount)
- Rebuild only needed for dependency changes

### Git Commit Style
- Use imperative mood: `feat:`, `fix:`, `refactor:`, `docs:`, `test:`
- Example: `fix: center-align multi-line subtitles in TextClip`
- Branch naming: `feat/<feature-name>`, `fix/<bug-name>`

---

## Recent Changes (Feature Branch)

**Branch**: `feat/siliconflow-cloned-voice`

Key changes made on this branch:
- `app/services/voice.py` — SiliconFlow TTS now accepts `speech:`-prefixed cloned voice URIs
- `webui/Main.py` — Added "SiliconFlow Cloned Voice URI" text input; fixed Play Voice ordering bug
- `app/services/video.py` — Added `text_align='center'` to subtitle TextClip for multi-line centering
