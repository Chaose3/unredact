# Solve Result Pagination Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Paginate solve results so the frontend receives at most 200 results at a time, with a "Load more" button to fetch subsequent pages.

**Architecture:** Stream first 200 results via SSE (preserving real-time UX), continue draining the generator into a server-side buffer, then expose a REST endpoint for fetching additional pages. Frontend shows a "Load more (showing N of M)" button when more results are available.

**Tech Stack:** Python/FastAPI backend, vanilla JS frontend, SSE streaming, httpx AsyncClient for tests.

---

### Task 1: Add server-side result buffer and PAGE_SIZE constant

**Files:**
- Modify: `unredact/app.py:273-274`

**Step 1: Add the constant and buffer dict**

Add right after the `_active_solves` line (line 274):

```python
PAGE_SIZE = 200

# Buffered solve results for pagination (keyed by solve_id)
_solve_results: dict[str, list[dict]] = {}
```

**Step 2: Verify no syntax errors**

Run: `python -c "from unredact.app import app"`
Expected: No errors

**Step 3: Commit**

```bash
git add unredact/app.py
git commit -m "feat: add PAGE_SIZE constant and solve results buffer"
```

---

### Task 2: Modify SSE generator to cap streamed results and buffer the rest

**Files:**
- Modify: `unredact/app.py:356-515` (the `solve` function and its `event_generator`)

**Step 1: Write the failing test**

Add to `tests/test_app.py`:

```python
@pytest.mark.anyio
async def test_solve_paginates_results():
    """Solve should stream at most PAGE_SIZE results and buffer the rest."""
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        resp = await client.get("/api/fonts")
        fonts = resp.json()["fonts"]
        font = next(f for f in fonts if f["available"])

        # Use word mode with full vocab and wide tolerance to get many results
        resp = await client.post("/api/solve", json={
            "font_id": font["id"],
            "font_size": 40,
            "gap_width_px": 80.0,
            "tolerance_px": 10.0,
            "left_context": "",
            "right_context": "",
            "hints": {"charset": "lowercase"},
            "mode": "word",
            "vocab_size": 0,
        })
        assert resp.status_code == 200

        # Parse SSE events from the response
        text = resp.text
        events = []
        for line in text.split("\n"):
            if line.startswith("data: "):
                try:
                    events.append(json.loads(line[6:]))
                except (json.JSONDecodeError, ValueError):
                    pass

        matches = [e for e in events if e.get("status") == "match"]
        done_events = [e for e in events if e.get("status") == "done"]
        page_events = [e for e in events if e.get("status") == "page_complete"]

        assert len(done_events) == 1
        total = done_events[0]["total_found"]

        if total > 200:
            # Should have capped SSE matches at PAGE_SIZE
            assert len(matches) == 200
            # Should have a page_complete event
            assert len(page_events) == 1
            assert page_events[0]["sent"] == 200
        else:
            # Under PAGE_SIZE — all sent, no page_complete
            assert len(matches) == total
            assert len(page_events) == 0
```

**Step 2: Run test to verify it fails**

Run: `pytest tests/test_app.py::test_solve_paginates_results -v`
Expected: FAIL (currently all results are streamed, no page_complete event)

**Step 3: Implement the pagination cap in the event generator**

Modify the `solve` function in `unredact/app.py`. The key change is in `event_generator()`:

1. Initialize a counter `sent_count = 0` and register the buffer `_solve_results[solve_id] = []`.
2. For each match result dict, always append to `_solve_results[solve_id]`.
3. Only `yield` the SSE event if `sent_count < PAGE_SIZE`. Increment `sent_count` after yielding.
4. When `sent_count` hits `PAGE_SIZE`, yield a `page_complete` event once.
5. The `done` event at the end remains unchanged (already sends `total_found`).
6. In the `finally` block, also pop `solve_id` from `_active_solves` (already done) but do NOT remove from `_solve_results` (needed for pagination).
7. At the start of the function (before `event_generator`), clean up any previous solve buffer: `_solve_results.clear()`.

The modified `event_generator` inner function (replacing the existing one at lines 367-513):

```python
    async def event_generator():
        # Clean up previous solve buffers (single user)
        _solve_results.clear()
        _solve_results[solve_id] = []

        try:
            found_texts = set()
            sent_count = 0
            page_complete_sent = False
            charset_name = req.hints.get("charset", "lowercase")

            def _emit(result_dict):
                """Append to buffer, return whether to yield as SSE."""
                nonlocal sent_count, page_complete_sent
                _solve_results[solve_id].append(result_dict)
                sent_count += 1
                return sent_count <= PAGE_SIZE

            # Name mode
            if req.mode == "name" and not _active_solves.get(solve_id):
                name_results = solve_name_dictionary(
                    font, req.gap_width_px, req.tolerance_px,
                    req.left_context, req.right_context,
                    casing=charset_name,
                    known_start=req.known_start,
                    known_end=req.known_end,
                )
                for r in name_results:
                    if _active_solves.get(solve_id):
                        break
                    if r.text in found_texts:
                        continue
                    found_texts.add(r.text)
                    d = {
                        "status": "match",
                        "text": r.text,
                        "width_px": round(r.width, 2),
                        "error_px": round(r.error, 2),
                        "source": "names",
                    }
                    if _emit(d):
                        yield json.dumps(d)
                    elif not page_complete_sent:
                        page_complete_sent = True
                        yield json.dumps({"status": "page_complete", "sent": PAGE_SIZE, "solve_id": solve_id})

            # Full name mode
            if req.mode == "full_name" and not _active_solves.get(solve_id):
                fn_results = solve_full_name_dictionary(
                    font, req.gap_width_px, req.tolerance_px,
                    req.left_context, req.right_context,
                    casing=charset_name,
                    known_start=req.known_start,
                    known_end=req.known_end,
                )
                for r in fn_results:
                    if _active_solves.get(solve_id):
                        break
                    if r.text in found_texts:
                        continue
                    found_texts.add(r.text)
                    d = {
                        "status": "match",
                        "text": r.text,
                        "width_px": round(r.width, 2),
                        "error_px": round(r.error, 2),
                        "source": "names",
                    }
                    if _emit(d):
                        yield json.dumps(d)
                    elif not page_complete_sent:
                        page_complete_sent = True
                        yield json.dumps({"status": "page_complete", "sent": PAGE_SIZE, "solve_id": solve_id})

            # Email mode
            if req.mode == "email" and not _active_solves.get(solve_id):
                entries = _get_emails()
                if entries:
                    email_results = solve_dictionary(
                        font, entries, req.gap_width_px, req.tolerance_px,
                        req.left_context, req.right_context,
                    )
                    for r in email_results:
                        if _active_solves.get(solve_id):
                            break
                        found_texts.add(r.text)
                        d = {
                            "status": "match",
                            "text": r.text,
                            "width_px": round(r.width, 2),
                            "error_px": round(r.error, 2),
                            "source": "emails",
                        }
                        if _emit(d):
                            yield json.dumps(d)
                        elif not page_complete_sent:
                            page_complete_sent = True
                            yield json.dumps({"status": "page_complete", "sent": PAGE_SIZE, "solve_id": solve_id})

            # Word mode
            if req.mode == "word" and not _active_solves.get(solve_id):
                for r in solve_word_dictionary(
                    font, req.gap_width_px, req.tolerance_px,
                    req.left_context, req.right_context,
                    casing=charset_name,
                    known_start=req.known_start,
                    known_end=req.known_end,
                    ensure_plural=req.ensure_plural,
                    vocab_size=req.vocab_size,
                ):
                    if _active_solves.get(solve_id):
                        break
                    if r.text in found_texts:
                        continue
                    found_texts.add(r.text)
                    d = {
                        "status": "match",
                        "text": r.text,
                        "width_px": round(r.width, 2),
                        "error_px": round(r.error, 2),
                        "source": "words",
                    }
                    if _emit(d):
                        yield json.dumps(d)
                    elif not page_complete_sent:
                        page_complete_sent = True
                        yield json.dumps({"status": "page_complete", "sent": PAGE_SIZE, "solve_id": solve_id})

            # Enumerate mode
            if req.mode == "enumerate" and not _active_solves.get(solve_id):
                charset = CHARSETS.get(charset_name, charset_name)
                constraint = None
                if charset_name == "capitalized":
                    charset = CHARSETS["alpha"] + " "
                    constraint = build_constraint(charset_name, charset)
                payload = _build_rust_solve_payload(
                    font, charset, req.gap_width_px, req.tolerance_px,
                    req.left_context, req.right_context, constraint,
                )
                payload["filter"] = req.word_filter
                payload["filter_prefix"] = req.known_start
                payload["filter_suffix"] = req.known_end
                url = f"{SOLVER_URL}/solve"

                async with httpx.AsyncClient(timeout=None) as client:
                    async with client.stream("POST", url, json=payload) as resp:
                        buf = ""
                        async for chunk in resp.aiter_text():
                            if _active_solves.get(solve_id):
                                break
                            buf += chunk
                            while "\n" in buf:
                                line, buf = buf.split("\n", 1)
                                line = line.strip()
                                if not line.startswith("data: "):
                                    continue
                                try:
                                    r = json.loads(line[6:])
                                except (json.JSONDecodeError, ValueError):
                                    continue
                                if r.get("done"):
                                    break
                                text = r.get("text", "")
                                if text in found_texts:
                                    continue
                                found_texts.add(text)
                                d = {
                                    "status": "match",
                                    "text": text,
                                    "width_px": round(r["width"], 2),
                                    "error_px": round(r["error"], 2),
                                    "source": "enumerate",
                                }
                                if _emit(d):
                                    yield json.dumps(d)
                                elif not page_complete_sent:
                                    page_complete_sent = True
                                    yield json.dumps({"status": "page_complete", "sent": PAGE_SIZE, "solve_id": solve_id})

            yield json.dumps({
                "status": "done",
                "total_found": len(found_texts),
            })
        finally:
            _active_solves.pop(solve_id, None)
```

**Step 4: Run test to verify it passes**

Run: `pytest tests/test_app.py::test_solve_paginates_results -v`
Expected: PASS

**Step 5: Run existing solve tests to verify no regressions**

Run: `pytest tests/test_app.py -v -k solve`
Expected: All PASS

**Step 6: Commit**

```bash
git add unredact/app.py tests/test_app.py
git commit -m "feat: cap SSE stream at PAGE_SIZE and buffer remaining results"
```

---

### Task 3: Add GET pagination endpoint

**Files:**
- Modify: `unredact/app.py:518-523` (after the cancel endpoint)
- Modify: `tests/test_app.py`

**Step 1: Write the failing test**

Add to `tests/test_app.py`:

```python
@pytest.mark.anyio
async def test_solve_pagination_endpoint():
    """GET /api/solve/{id}/results should return paginated buffered results."""
    from unredact.app import _solve_results

    # Seed a fake buffer
    fake_id = "test123"
    _solve_results[fake_id] = [
        {"text": f"word{i}", "width_px": 50.0, "error_px": 0.1, "source": "words"}
        for i in range(500)
    ]

    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        # First page
        resp = await client.get(f"/api/solve/{fake_id}/results?offset=0&limit=200")
        assert resp.status_code == 200
        data = resp.json()
        assert len(data["results"]) == 200
        assert data["total"] == 500
        assert data["offset"] == 0
        assert data["limit"] == 200
        assert data["complete"] is True

        # Second page
        resp = await client.get(f"/api/solve/{fake_id}/results?offset=200&limit=200")
        data = resp.json()
        assert len(data["results"]) == 200
        assert data["results"][0]["text"] == "word200"

        # Third page (partial)
        resp = await client.get(f"/api/solve/{fake_id}/results?offset=400&limit=200")
        data = resp.json()
        assert len(data["results"]) == 100

        # Unknown solve_id
        resp = await client.get("/api/solve/nonexistent/results")
        assert resp.status_code == 404

    # Clean up
    _solve_results.pop(fake_id, None)
```

**Step 2: Run test to verify it fails**

Run: `pytest tests/test_app.py::test_solve_pagination_endpoint -v`
Expected: FAIL (endpoint doesn't exist yet)

**Step 3: Add the pagination endpoint**

Add after the `cancel_solve` function in `unredact/app.py` (after line 523):

```python
@app.get("/api/solve/{solve_id}/results")
async def get_solve_results(solve_id: str, offset: int = 0, limit: int = 200):
    if solve_id not in _solve_results:
        return JSONResponse({"error": "solve not found"}, status_code=404)
    buf = _solve_results[solve_id]
    page = buf[offset:offset + limit]
    # complete=True means the solve generator has finished (solve_id no longer in _active_solves)
    complete = solve_id not in _active_solves
    return {
        "results": page,
        "total": len(buf),
        "offset": offset,
        "limit": limit,
        "complete": complete,
    }
```

**Step 4: Run test to verify it passes**

Run: `pytest tests/test_app.py::test_solve_pagination_endpoint -v`
Expected: PASS

**Step 5: Commit**

```bash
git add unredact/app.py tests/test_app.py
git commit -m "feat: add GET /api/solve/{id}/results pagination endpoint"
```

---

### Task 4: Clean up solve buffer on cancel and new solve

**Files:**
- Modify: `unredact/app.py` (cancel endpoint)
- Modify: `tests/test_app.py`

**Step 1: Write the failing test**

Add to `tests/test_app.py`:

```python
@pytest.mark.anyio
async def test_cancel_clears_buffer():
    """DELETE /api/solve/{id} should also clean up the results buffer."""
    from unredact.app import _solve_results, _active_solves

    fake_id = "cancel_test"
    _active_solves[fake_id] = False
    _solve_results[fake_id] = [{"text": "test"}]

    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        resp = await client.delete(f"/api/solve/{fake_id}")
        assert resp.status_code == 200

    assert fake_id not in _solve_results
    # Clean up
    _active_solves.pop(fake_id, None)
```

**Step 2: Run test to verify it fails**

Run: `pytest tests/test_app.py::test_cancel_clears_buffer -v`
Expected: FAIL (cancel doesn't clear `_solve_results`)

**Step 3: Update the cancel endpoint**

Modify `cancel_solve` in `unredact/app.py` (line 518-523):

```python
@app.delete("/api/solve/{solve_id}")
async def cancel_solve(solve_id: str):
    if solve_id in _active_solves:
        _active_solves[solve_id] = True
        _solve_results.pop(solve_id, None)
        return {"status": "cancelled"}
    return JSONResponse({"error": "solve not found"}, status_code=404)
```

**Step 4: Run test to verify it passes**

Run: `pytest tests/test_app.py::test_cancel_clears_buffer -v`
Expected: PASS

**Step 5: Commit**

```bash
git add unredact/app.py tests/test_app.py
git commit -m "feat: clean up solve buffer on cancel"
```

---

### Task 5: Frontend — handle page_complete and done events for pagination

**Files:**
- Modify: `unredact/static/solver.js:107-195` (handleSolveEvent function)
- Modify: `unredact/static/dom.js` (add solveLoadMore element reference)
- Modify: `unredact/static/index.html` (add load-more button to DOM)

**Step 1: Add the Load More button to the HTML**

Find the `solve-results` container in `index.html` and add a button right after it:

```html
<button id="solve-load-more" class="btn btn-sm" hidden>Load more</button>
```

**Step 2: Add the DOM reference**

Add to `unredact/static/dom.js` after the `solveResults` line:

```javascript
export const solveLoadMore = document.getElementById("solve-load-more");
```

**Step 3: Update solver.js imports and module state**

Add `solveLoadMore` to the import from `./dom.js`. Add module-level state:

```javascript
let currentSolveId = null;
let displayedCount = 0;
let totalFound = 0;
```

**Step 4: Update handleSolveEvent to handle page_complete and done**

In `handleSolveEvent`:

- On `match` events: increment `displayedCount`.
- On `page_complete`: store `currentSolveId = data.solve_id`. (Don't show button yet — wait for `done` to know total.)
- On `done`: if `data.total_found > displayedCount`, show the load-more button with text "Load more (showing {displayedCount} of {data.total_found})". Store `totalFound = data.total_found`.

```javascript
} else if (data.status === "page_complete") {
    currentSolveId = data.solve_id;
} else if (data.status === "done") {
    totalFound = data.total_found;
    if (totalFound > displayedCount) {
      solveLoadMore.textContent = `Load more (showing ${displayedCount} of ${totalFound})`;
      solveLoadMore.hidden = false;
    }
    solveStatus.textContent = `Done. ${data.total_found} total matches.`;
    solveStart.hidden = false;
    solveStop.hidden = true;
    activeEventSource = null;
}
```

In the `match` handler, after appending/inserting the div, add:

```javascript
displayedCount++;
```

**Step 5: Add loadMore function**

```javascript
async function loadMore() {
  if (!currentSolveId) return;
  solveLoadMore.disabled = true;
  solveLoadMore.textContent = "Loading...";

  try {
    const resp = await fetch(
      `/api/solve/${currentSolveId}/results?offset=${displayedCount}&limit=200`
    );
    if (!resp.ok) throw new Error("Failed to load results");
    const data = await resp.json();

    const redactionId = state.activeRedaction;
    for (const item of data.results) {
      handleSolveEvent({ status: "match", ...item }, redactionId);
    }

    if (displayedCount >= totalFound) {
      solveLoadMore.hidden = true;
    } else {
      solveLoadMore.textContent = `Load more (showing ${displayedCount} of ${totalFound})`;
      solveLoadMore.disabled = false;
    }
  } catch (err) {
    solveLoadMore.textContent = "Error loading — click to retry";
    solveLoadMore.disabled = false;
  }
}
```

**Step 6: Reset state in startSolve**

In the `startSolve` function, after `solveResults.innerHTML = ""` (line 34), add:

```javascript
solveLoadMore.hidden = true;
currentSolveId = null;
displayedCount = 0;
totalFound = 0;
```

**Step 7: Wire up the button in initSolver**

In the `initSolver` function, add:

```javascript
solveLoadMore.addEventListener("click", loadMore);
```

**Step 8: Manual test**

1. Run `make dev`
2. Upload a document, select a redaction
3. Set tolerance to 10px, word mode, full vocabulary
4. Click Solve — verify only 200 results appear
5. Verify "Load more (showing 200 of N)" button appears
6. Click Load more — verify 200 more results appear, button updates
7. Repeat until all results shown, button disappears
8. Test with a narrow tolerance that produces <200 results — verify no button

**Step 9: Commit**

```bash
git add unredact/static/solver.js unredact/static/dom.js unredact/static/index.html
git commit -m "feat: add Load More button for paginated solve results"
```

---

### Task 6: Update status text during buffering

**Files:**
- Modify: `unredact/static/solver.js`

Currently after `page_complete`, the status still says "Found 200 matches" while the backend silently buffers. We should update it to indicate work is ongoing.

**Step 1: Update page_complete handler**

In the `page_complete` handler, add:

```javascript
solveStatus.textContent = `Found ${displayedCount} matches, searching for more...`;
```

**Step 2: Manual test**

Verify status says "Found 200 matches, searching for more..." while backend finishes, then switches to "Done. N total matches."

**Step 3: Commit**

```bash
git add unredact/static/solver.js
git commit -m "feat: show 'searching for more' status during buffering"
```
