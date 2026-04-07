# Student Issues and Fixes

This document covers the issues I faced while developing and running the project. It focuses on the real failure points across the frontend, backend, MCP servers, and PowerPoint export flow, along with the cleanest fix I used for each one.

## 1. Windows virtual environment did not activate correctly

### Issue
The first blocker was the Python environment. The existing virtual environment was created in a Unix-style layout, so PowerShell on Windows could not use it correctly.

### Why it happened
The repo had been prepared in a different environment, so the path layout did not match the current machine.

### How I resolved it
I created a fresh Windows-native environment and used the correct activation path.

```powershell
py -0p
py -3.14 -m venv .venv
```

```powershell
.\.venv\Scripts\Activate.ps1
```

---

## 2. Python version mismatch blocked venv creation

### Issue
When I tried to create the environment with Python 3.12, the machine did not have that interpreter installed.

### Why it happened
The project required Python 3.11+, but the local machine only exposed a different version.

### How I resolved it
I listed the installed interpreters first, then used the one that actually existed.

```powershell
py -0p
```

That avoided guessing and let me build the environment with a valid runtime.

---

## 3. Backend dependencies were missing

### Issue
The backend would not start because FastAPI and the other Python packages were not installed in the active environment.

### Why it happened
The venv was not ready, so imports failed immediately.

### How I resolved it
I installed the backend requirements inside the new `.venv`.

```powershell
Set-Location 'd:\Calibo\assignment'
.\.venv\Scripts\python.exe -m pip install -r backend\requirements.txt
```

I then verified the environment with an import check.

```powershell
.\.venv\Scripts\python.exe -c "import fastapi, uvicorn, mcp; print('backend deps ok')"
```

---

## 4. Frontend Vite command was not found

### Issue
The frontend dev server failed with a Vite not found error.

### Why it happened
The frontend packages needed to be installed again for the current workspace.

### How I resolved it
I reinstalled the dependencies in the `frontend` folder and then ran the dev server from there.

```powershell
Set-Location 'd:\Calibo\assignment\frontend'
npm install
npm run dev
```

---

## 5. The app was started from the wrong folder

### Issue
Some commands failed when I ran them from the repo root instead of the `frontend` or project root folders.

### Why it happened
`package.json` is in `frontend`, and the Python backend expects to be launched from the project root.

### How I resolved it
I made a clear split:

- backend commands from `d:\Calibo\assignment`
- frontend commands from `d:\Calibo\assignment\frontend`

That removed path confusion and made the run steps repeatable.

---

## 6. LLM output was not always valid JSON

### Issue
The model sometimes returned the presentation plan wrapped in markdown fences or extra text.

### Why it happened
The prompt asked for JSON only, but LLM output is not perfectly deterministic.

### How I resolved it
I added a cleanup step before `json.loads()`.

```python
match = re.search(r"```(?:json)?\s*([\s\S]+?)```", raw)
if match:
    raw = match.group(1).strip()
else:
    start = raw.find("{")
    end = raw.rfind("}") + 1
    if start != -1 and end > start:
        raw = raw[start:end]

data = json.loads(raw)
```

---

## 7. Slide outlines sometimes came back incomplete

### Issue
The generated deck could be missing a proper outline or could return too few slides.

### Why it happened
The LLM was not always aligned with the requested number of slides.

### How I resolved it
I enforced the requested slide count in the generator and kept a fallback outline in the CLI path.

---

## 8. Bullets were sometimes empty

### Issue
Some slides, especially conclusions or closing slides, had `bullets: []`.

### Why it happened
The model did not always produce enough structured content.

### How I resolved it
I filled empty bullet lists with safe fallback content.

```python
if not slide.get("bullets"):
    slide["bullets"] = [f"Key point about {slide['title']}." for _ in range(4)]
```

---

## 9. Slide descriptions were sometimes too thin

### Issue
Some slides had a title and bullets but not enough explanatory context.

### Why it happened
The generation prompt did not always produce a strong description paragraph.

### How I resolved it
I tightened the prompt and treated description text as a required part of the slide schema.

---

## 10. Image queries were too generic

### Issue
The image search often used weak keywords and returned unrelated visuals.

### Why it happened
Topic-only queries were too broad for a stock image search API.

### How I resolved it
I combined the main topic with the slide focus before sending the query.

```python
clean_topic = ' '.join(
    w for w in topic.split()
    if w.lower() not in ('a', 'an', 'the', 'of', 'for', 'about', 'on', 'create', 'presentation', 'slide')
).strip()

slide["image_query"] = f"{clean_topic[:20]} {suggested}".strip()
```

---

## 11. Pexels returned duplicate images

### Issue
Two nearby slides sometimes ended up with the same image.

### Why it happened
The API naturally returned the same top result for similar search terms.

### How I resolved it
I tracked previously used image URLs and skipped them when downloading slide images.

---

## 12. Missing Pexels key caused blank image results

### Issue
If the `PEXELS_API_KEY` was missing, image search returned empty results.

### Why it happened
The backend depended on the API key being present in `.env`.

### How I resolved it
I made the search server return a safe empty string when the key is missing, and the slide renderer handles the no-image case cleanly.

---

## 13. Missing Hugging Face token blocked generation

### Issue
The LLM call would fail if the Hugging Face token or model config was missing.

### Why it happened
The generator depends on environment variables for authentication and model selection.

### How I resolved it
I documented the expected `.env` variables and confirmed they are loaded before the model call.

---

## 14. Exported PPT did not match the preview

### Issue
The screenshot-based export path produced blank or partial slides.

### Why it happened
React state timing and CORS issues made `html2canvas` unreliable for this use case.

### How I resolved it
I moved export to the backend and generated the `.pptx` directly from the structured slide data.

```python
@app.post("/export")
async def export(req: ExportRequest):
    pptx_bytes = await _build_pptx_via_mcp(req.presentation, req.fetch_images)
    return Response(
        content=pptx_bytes,
        media_type="application/vnd.openxmlformats-officedocument.presentationml.presentation",
    )
```

---

## 15. React state updates caused timing issues

### Issue
The app could try to export or preview before the presentation had fully re-rendered.

### Why it happened
`setState` is asynchronous.

### How I resolved it
I added a short delay before export and cleaned up the modal reset flow so state transitions were more predictable.

---

## 16. MCP server responsibilities could blur together

### Issue
It was easy to accidentally mix image search logic with slide rendering logic.

### Why it mattered
That would make the system harder to debug and less reusable.

### How I resolved it
I kept the roles separate:

- `mcp_server.py` renders slides
- `web_search_mcp.py` searches for images
- FastAPI coordinates both

---

## 17. MCP stdio startup was confusing at first

### Issue
The MCP servers are launched as subprocesses over stdio, which was harder to reason about at first than a normal HTTP server.

### Why it mattered
If the process wiring is wrong, the tools appear to fail even when the slide logic itself is fine.

### How I resolved it
I verified the `StdioServerParameters`, then tested the tool calls in small steps before relying on the full flow.

---

## 18. Presentation history needed collision handling

### Issue
Generated `.json` and `.pptx` files could overwrite each other if the same title was reused.

### Why it happened
Multiple presentations can share the same title or cleaned filename.

### How I resolved it
I added suffix-based collision handling so each export and history record gets a unique name.

---

## 19. The generation modal kept stale topic state

### Issue
Reopening the modal sometimes reused the previous example text.

### Why it mattered
It made the UI feel inconsistent.

### How I resolved it
I reset the draft state on close unless an example topic was intentionally passed in.

```jsx
const closeModal = () => {
    setShowModal(false)
    setDraftTopic('')
}
```

---

## 20. The layout had to be fixed for smaller screens and final build validation

### Issue
After the dark redesign, I still had to check that the UI stayed usable on smaller screens and that the app built cleanly.

### Why it mattered
A nice-looking interface is not useful if it breaks at different widths or fails the production build.

### How I resolved it
I added responsive CSS rules for the hero, history grid, and modal, then verified the project with a production build.

```powershell
Set-Location 'd:\Calibo\assignment\frontend'
npm run build
```

---

## Summary

The issues I faced covered the full project lifecycle, not just startup problems. They included environment setup, dependency installation, LLM parsing, slide content quality, image search quality, MCP orchestration, export consistency, state management, naming collisions, and responsive UI validation.

Fixing them in layers made the project much more stable and easier to explain in a report.
