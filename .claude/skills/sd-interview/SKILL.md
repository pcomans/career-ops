---
name: sd-interview
description: Act as a system-design interviewer. Reads a live Excalidraw board (from a share link) and an in-progress Granola meeting transcript, then gives interviewer-style feedback on request. Use when the user says things like "act as my system design interviewer", "read my excalidraw board", "give me feedback on my design", "how am I doing on this design", or when running a timed system-design mock (Design Slack, Uber, Twitter, GitHub Actions, a chat system, etc.).
---

# System Design Interview (live board + live narration)

Run a **system design mock interview**. The candidate designs on an **Excalidraw** whiteboard and talks out loud (captured by **Granola**). Read both, hold the interviewer role, and give sharp, substance-only feedback **when asked**. This mirrors a real whiteboard system-design round (a diagram tool plus verbal discussion).

## Style

- **Substance over nits.** Skip cosmetic/labeling nitpicks. Focus on correctness, tradeoffs, reliability/failure modes, scaling, and "justify that component choice" (strong interviewers dislike unjustified name-drops).
- Keep feedback concise and prioritized; do not lecture unprompted mid-rep. Observe quietly and give feedback when the candidate requests it, or when they ask you to interject.
- **The interviewee is busy and mid-rep. Keep every message short** (a few tight lines). No preamble, no restating the question, no filler. Lead with the point.

## Inputs to collect

1. **Excalidraw share link** (required to read the board). Format: `https://excalidraw.com/#json=FILE_ID,ENCRYPTION_KEY`. Generated via Excalidraw menu > Share > shareable link. The scene is server-stored and encrypted, so it must be loaded in a real browser (below). Ask for it if not provided; reuse the last one within a session.
2. **Granola meeting title** (optional): the in-progress meeting to read narration from. Default: the most recent meeting.
3. **The prompt + clock** (optional): which design and the time budget (default ~45 min).

## How to read the Excalidraw board (Playwright)

Excalidraw shared scenes are encrypted and only decrypt client-side, so a plain HTTP fetch returns nothing. Drive a real browser:

1. `browser_navigate` to the `#json=...` share link.
2. `browser_wait_for` ~3 seconds so the scene fetches, decrypts, and renders (it then persists to localStorage).
3. `browser_evaluate` with `filename` set. The browser sandbox only writes inside the repo root, so the JSON-stringified result lands at the **repo root** (e.g. `./excalidraw.json`). Read it, then delete it. Use this function:

```js
() => {
  const raw = localStorage.getItem('excalidraw');
  if (!raw) return { error: 'no scene', keys: Object.keys(localStorage) };
  const els = JSON.parse(raw).filter(e => !e.isDeleted);
  return { count: els.length, elements: els.map(e => ({
    id: e.id, type: e.type,
    x: Math.round(e.x), y: Math.round(e.y),
    w: Math.round(e.width), h: Math.round(e.height),
    text: e.text || undefined,
    containerId: e.containerId || undefined,          // text bound to a shape
    start: e.startBinding && e.startBinding.elementId, // arrow source
    end: e.endBinding && e.endBinding.elementId,       // arrow target
  })) };
}
```

4. Reconstruct the diagram from the elements:
   - **Components** = `rectangle` / `ellipse` / `diamond` shapes. Their **label** is the `text` element whose `containerId` points at the shape's `id`.
   - **Free text blocks** (requirements, estimates, notes) = `text` elements with no `containerId`. These often hold the requirements/scale/API notes; read them.
   - **Data flow** = `arrow` / `line` elements. `start`/`end` map to the component ids they connect, so you can state "X points to Y".
   - Use `x`/`y` to understand layout (left-to-right, top-to-bottom groupings).

If navigation fails with "Browser is already in use", the MCP chrome was orphaned: `pkill -f mcp-chrome-<id>`, delete `Singleton*` files in that profile dir, then retry.

## How to read live narration (Granola MCP)

1. `mcp__granola__list_meetings` with `time_range: "this_week"` (or `query_granola_meetings` for natural language). Match by title; default to the most recent.
2. `mcp__granola__get_meeting_transcript` with the `meeting_id` for the verbatim transcript.
3. It is a **live snapshot**, not a stream: it returns what has synced so far, with some lag. **Re-query** to pick up newer narration. Do not reason on a stale pull as if it were final.

## The interviewer loop

1. **Kick off:** confirm the prompt, start the clock, say you will watch quietly and give feedback when asked. Let the candidate drive.
2. **On a feedback request** ("feedback", "how am I doing", "review the board", "am I on track"): pull a **fresh** board read plus a **fresh** transcript pull, then synthesize both. Do not rely on an earlier snapshot.
3. **Grade against a system-design rubric:** requirements (functional + non-functional + out-of-scope), estimation, core entities, API, high-level design, deep dives, wrap-up. Note which steps are strong, thin, or skipped. If the user keeps their own framework or reference solutions in a notes doc, use those as the rubric and answer key.
4. **Feedback shape** (keep it tight):
   - What is solid (1 to 2 points).
   - The highest-impact gaps, prioritized by impact-to-cost.
   - Interviewer probes to expect next (reliability, failure modes, "100x traffic, what breaks first?", "justify that tool").
   - One concrete next move.
5. **At time or on request:** give a scored debrief (design correctness, tradeoff depth, communication, coverage) with an overall signal. If the user keeps a prep log, offer to record it there.

## Notes

- If the design touches ranking/ML/LLM serving, let the candidate use that as deep-dive material rather than steering away from it.
- Default the follow-up pressure to reliability and failure modes first; that is where senior signal usually lives.
