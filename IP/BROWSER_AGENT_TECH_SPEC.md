# Browser Agent Technical Specification

**Status:** Draft — awaiting R1 review
**Date:** 2026-04-10
**Scope:** Chrome DevTools Protocol (CDP) based browser control for AiMe v3
**Constraint:** All browser actions pass through the SBA spine. Plan→Confirm→Execute safety model.

---

## 1. Purpose

This document specifies a **Browser Agent** for AiMe v3 that enables direct, programmatic control of the Chrome browser via the Chrome DevTools Protocol (CDP).

The current Desktop Agent uses pyautogui (screenshot + pixel-coordinate clicks), which is general-purpose but fragile for web tasks — it can't read DOM content, breaks on scroll position changes, and requires expensive vision calls for every action.

The Browser Agent provides:
- **Structured DOM access** — click elements by CSS selector, read page text, fill forms
- **Navigation** — go to URLs, manage tabs, handle page loads
- **Content extraction** — read page content as structured data, not screenshots
- **JavaScript execution** — run scripts in-page for complex interactions
- **Hybrid fallback** — fall back to vision-based (screenshot + coordinate) mode when DOM selection fails

---

## 2. Non-Negotiable Architecture Rules

1. **All browser actions enter the SBA spine.** The LM does not autonomously browse. It produces a plan; user approves; system executes.
2. **Plan→Confirm→Execute** safety model — identical pattern to Desktop Agent.
3. **No credential auto-fill.** The system will never type passwords or fill credential fields without explicit per-field user approval.
4. **No autonomous navigation to new domains.** If execution requires navigating to a domain not in the approved plan, execution pauses for re-confirmation.
5. **Rate-limited.** Maximum 30 CDP actions per approved plan. Exceeding this requires re-approval.
6. **Read-only default.** Content extraction (reading page text, taking screenshots) does not require approval. Only mutating actions (clicking, typing, navigating) require the safety gate.
7. **Session isolation.** Browser Agent connects to user's existing Chrome via debug port. It does NOT store cookies, history, or session data independently.

---

## 3. Architecture Overview

```text
User Request
  → Intent Engine classifies as "browse" / "browser_read" / "browser_fill" / "browser_click"
  → Action Dispatcher routes to Browser Agent
  → Phase 1 (ANALYZE): 
      Connect to Chrome via CDP
      → Read current page state (URL, title, visible text)
      → LLM produces action plan with steps
      → Plan surfaced to user for approval
  → Phase 2 (EXECUTE):
      User approves → execute steps sequentially
      → Each step: CDP action → verify result → next step
      → Push result summary to SurfaceQueue
```

### 3.1 Relationship to Desktop Agent

| Capability | Desktop Agent | Browser Agent |
|---|---|---|
| Target | Any visible application | Chrome browser specifically |
| Method | pyautogui (pixels) | CDP (DOM selectors) |
| Reliability | Fragile (depends on pixel positions) | Robust (semantic selectors) |
| Speed | Slow (screenshot per action) | Fast (direct DOM access) |
| Content reading | OCR from screenshots | Direct text extraction |
| Fallback | N/A | Can invoke Desktop Agent for non-DOM elements |

The two agents coexist. Intent routing decides which to invoke based on context:
- Explicit web/browser references → Browser Agent
- General desktop tasks → Desktop Agent
- Browser Agent can delegate to Desktop Agent for browser chrome elements (address bar, extensions) that aren't accessible via CDP

---

## 4. Core Runtime Objects

### 4.1 `BrowserConnection`

```python
@dataclass
class BrowserConnection:
    """Manages CDP WebSocket connection to Chrome."""
    ws_url: str              # WebSocket debugger URL
    connected: bool = False
    tabs: list[TabInfo] = field(default_factory=list)
    active_tab_id: str = ""
```

### 4.2 `BrowserAction`

```python
@dataclass
class BrowserAction:
    action_type: str         # "navigate", "click", "type", "read", "scroll", "wait", "js_eval"
    selector: str = ""       # CSS selector or XPath
    value: str = ""          # For type actions: text to type. For navigate: URL.
    description: str = ""    # Human-readable step description
    timeout_ms: int = 5000   # Per-action timeout
```

### 4.3 `BrowserPlan`

```python
@dataclass
class BrowserPlan:
    plan_id: str
    session_id: str
    directive: str                  # Original user request
    current_url: str
    current_title: str
    steps: list[BrowserAction]
    plan_summary: str              # Human-readable summary for confirmation
    domains_involved: list[str]    # All domains the plan will touch
    created_at: float
    confirmed: bool = False
    executed: bool = False
```

### 4.4 `BrowserResult`

```python
@dataclass
class BrowserResult:
    success: bool
    steps_completed: int
    steps_total: int
    extracted_content: str = ""    # Page content if read action
    screenshot_b64: str = ""       # Final state screenshot
    error: str = ""
    final_url: str = ""
```

---

## 5. CDP Connection Strategy

### 5.1 Connecting to User's Chrome

Chrome must be launched with remote debugging enabled:
```
chrome.exe --remote-debugging-port=9222
```

The Browser Agent:
1. Connects to `http://localhost:9222/json` to discover available tabs
2. Establishes WebSocket connection to the target tab's `webSocketDebuggerUrl`
3. Uses CDP domains: `Page`, `DOM`, `Runtime`, `Input`, `Network` (read-only)

### 5.2 Chrome Launch Helper

If Chrome is not running with debug port, the agent can:
1. Detect if Chrome is running (process check)
2. If not running: launch with `--remote-debugging-port=9222`
3. If running without debug port: notify user to restart Chrome with the flag, OR use a separate Chrome profile with debug port

### 5.3 Connection Lifecycle

```python
class CDPClient:
    """Low-level CDP WebSocket client."""
    
    async def connect(self, tab_ws_url: str) -> None: ...
    async def send(self, method: str, params: dict = None) -> dict: ...
    async def disconnect(self) -> None: ...
    
    # High-level helpers
    async def navigate(self, url: str) -> None: ...
    async def click(self, selector: str) -> None: ...
    async def type_text(self, selector: str, text: str) -> None: ...
    async def read_text(self, selector: str = "body") -> str: ...
    async def evaluate_js(self, expression: str) -> Any: ...
    async def screenshot(self) -> bytes: ...
    async def get_page_info(self) -> dict: ...  # url, title, visible text summary
    async def wait_for_selector(self, selector: str, timeout_ms: int = 5000) -> bool: ...
```

---

## 6. Action Types

| Action | CDP Method | Requires Approval | Description |
|--------|-----------|-------------------|-------------|
| `navigate` | `Page.navigate` | Yes | Go to URL |
| `click` | `DOM.querySelector` + `Input.dispatchMouseEvent` | Yes | Click element |
| `type` | `Input.dispatchKeyEvent` | Yes | Type into focused element |
| `read` | `Runtime.evaluate` (innerText) | No (read-only) | Extract page text |
| `scroll` | `Input.dispatchMouseEvent` (wheel) | Yes | Scroll page |
| `wait` | `Runtime.evaluate` (poll) | No | Wait for element/condition |
| `js_eval` | `Runtime.evaluate` | Yes | Execute arbitrary JS |
| `screenshot` | `Page.captureScreenshot` | No (read-only) | Capture current state |
| `tab_switch` | Target domain | Yes | Switch active tab |
| `tab_open` | `Target.createTarget` | Yes | Open new tab |

---

## 7. Safety Model

### 7.1 Credential Protection

```python
_CREDENTIAL_SELECTORS = [
    'input[type="password"]',
    'input[name*="pass"]',
    'input[name*="secret"]',
    'input[name*="token"]',
    'input[autocomplete="current-password"]',
    'input[autocomplete="new-password"]',
]
```

Before any `type` action, the agent checks if the target element matches credential selectors. If so, execution halts and requires **per-field explicit approval** with the field name shown to the user.

### 7.2 Domain Boundary Enforcement

The plan declares `domains_involved` at confirmation time. During execution:
- If a navigation would go to an unlisted domain (e.g., redirect), execution pauses
- User sees: "Plan is navigating to [new-domain.com] which wasn't in the original plan. Allow?"

### 7.3 Action Limit

Hard cap of 30 actions per confirmed plan. If the LLM plan has >30 steps, it must be broken into multiple plans, each separately confirmed.

### 7.4 Timeout and Abort

- Per-action timeout: 5 seconds default (configurable per action)
- Total plan timeout: 120 seconds
- pyautogui.FAILSAFE remains active (mouse to corner aborts everything)
- User can say "stop" / "cancel" / "abort" at any time

---

## 8. Intent Integration

### 8.1 New Intents for Action Dispatcher

```python
# In modules/action_dispatcher.py — new elif branches:

elif intent == "browse":
    # Full browser interaction plan
    return await _dispatch_browse(session_id, user_text, context)

elif intent == "browser_read":
    # Read-only: extract current page content (no approval needed)
    return await _dispatch_browser_read(session_id, user_text, context)

elif intent == "browser_fill":
    # Fill form fields on current page
    return await _dispatch_browser_fill(session_id, user_text, context)

elif intent == "browser_click":
    # Click specific element on current page
    return await _dispatch_browser_click(session_id, user_text, context)
```

### 8.2 Intent Detection Patterns

```python
_BROWSE_RE = re.compile(
    r"\b(go to|open|navigate to|visit|browse to|pull up)\b.{0,30}\b(website|site|page|url|http|\.com|\.org|\.io)\b",
    re.IGNORECASE,
)

_BROWSER_READ_RE = re.compile(
    r"\b(read|what does|what's on|show me|get the|extract|scrape)\b.{0,20}\b(page|tab|website|site|browser)\b",
    re.IGNORECASE,
)

_BROWSER_FILL_RE = re.compile(
    r"\b(fill|enter|type|put|write)\b.{0,30}\b(form|field|input|box|search)\b.{0,20}\b(on the|in the|on that)?\s*(page|site|browser)?\b",
    re.IGNORECASE,
)
```

### 8.3 Feature Flag

```json
{
  "agents": {
    "browser_agent_enabled": true
  }
}
```

---

## 9. Daemon Architecture

### 9.1 Service: `browser_daemon.py`

Port: **8770** (next available after desktop at 8769)

Follows identical HTTP daemon pattern as `desktop_daemon.py`:

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/status` | GET | Connection status, active tab info |
| `/submit` | POST | Submit new browse directive → produces plan |
| `/confirm` | POST | User approves plan → begins execution |
| `/cancel` | POST | Cancel pending plan or abort execution |
| `/read` | POST | Read-only page content extraction (no plan needed) |
| `/screenshot` | GET | Current page screenshot (no plan needed) |
| `/task/{id}` | GET | Poll task status |

### 9.2 Worker Thread Model

```
Main Thread: HTTP server (port 8770)
Worker Thread: CDP connection + action execution
  - Maintains persistent WebSocket to Chrome
  - Executes confirmed plans step-by-step
  - Reports progress via task status updates
```

### 9.3 File Structure

```
modules/agents/browser_service/
├── __init__.py
├── browser_daemon.py        # HTTP daemon (port 8770)
├── cdp_client.py            # Low-level CDP WebSocket client
├── browser_worker.py        # Plan execution engine
├── page_reader.py           # Content extraction utilities
└── safety_checks.py         # Credential detection, domain boundary
```

---

## 10. Content Extraction (Read Mode)

For `browser_read` intent — no approval needed, purely informational:

```python
async def extract_page_content(cdp: CDPClient, selector: str = "body") -> PageContent:
    """Extract structured content from current page."""
    return PageContent(
        url=await cdp.evaluate_js("window.location.href"),
        title=await cdp.evaluate_js("document.title"),
        text=await cdp.read_text(selector),
        links=await cdp.evaluate_js("""
            Array.from(document.querySelectorAll('a[href]'))
                .slice(0, 50)
                .map(a => ({text: a.innerText.trim(), href: a.href}))
        """),
        forms=await cdp.evaluate_js("""
            Array.from(document.querySelectorAll('form'))
                .map(f => ({
                    action: f.action,
                    fields: Array.from(f.querySelectorAll('input,select,textarea'))
                        .map(el => ({name: el.name, type: el.type, value: el.value}))
                }))
        """),
    )
```

This gives AiMe rich context about what's on the page without needing vision/OCR.

---

## 11. LLM Plan Generation Prompt

When the user requests a browser action, the system captures current page state and prompts the LLM:

```
You are planning browser actions for the user. Given the current page state and user request, produce a step-by-step plan.

CURRENT PAGE:
  URL: {url}
  Title: {title}
  Visible text (truncated): {text_summary}
  Forms: {forms_json}
  Links (relevant): {filtered_links}

USER REQUEST: {directive}

Produce your plan in this exact format:
PLAN_SUMMARY: <one-line description of what you'll do>
DOMAINS: <comma-separated list of all domains the plan will visit>
STEPS:
1. ACTION:navigate URL:{url}
2. ACTION:wait SELECTOR:{selector}
3. ACTION:click SELECTOR:{selector} DESCRIPTION:{what this does}
4. ACTION:type SELECTOR:{selector} TEXT:{text} DESCRIPTION:{what this fills}
5. ACTION:read SELECTOR:{selector} DESCRIPTION:{what we're extracting}

Only use action types: navigate, click, type, read, scroll, wait, js_eval, screenshot, tab_switch, tab_open
```

---

## 12. Implementation Phases

### Phase 1: CDP Foundation + Read-Only
- `cdp_client.py` — WebSocket connection, basic CDP commands
- `page_reader.py` — Content extraction
- `/status`, `/read`, `/screenshot` endpoints
- Feature flag in `lm_tuning.json`
- `browser_read` intent in action_dispatcher
- **Tests:** Connection lifecycle, page reading, content extraction

### Phase 2: Plan→Confirm→Execute for Mutations
- `browser_worker.py` — Plan execution engine
- `safety_checks.py` — Credential detection, domain boundaries
- `/submit`, `/confirm`, `/cancel` endpoints
- `browse`, `browser_click`, `browser_fill` intents
- LLM plan generation prompt
- **Tests:** Plan parsing, safety gate, action execution

### Phase 3: Robustness + Hybrid Fallback
- Retry logic for flaky selectors (try alternate selectors)
- Vision fallback: if selector fails, take screenshot → use Desktop Agent's vision approach for that step
- Tab management (multi-tab workflows)
- Network-idle detection (wait for page to finish loading)
- **Tests:** Fallback paths, timeout handling, multi-tab

### Phase 4: Advanced Features
- Form auto-detection (scan page, present fill-able fields to user)
- Download management (detect and manage file downloads)
- Cookie/session awareness (warn user about login state)
- Integration with AiMe's web_search to open results directly
- **Tests:** End-to-end workflows

---

## 13. Dependencies

```
# New in requirements.txt
websockets>=12.0          # CDP WebSocket communication
```

No Selenium, no Playwright — raw CDP over WebSocket keeps it lightweight and gives full control. The only external dependency is `websockets` for the WebSocket client.

Chrome must be available with `--remote-debugging-port=9222`. The daemon will provide clear error messages if Chrome isn't accessible.

---

## 14. Integration Points

| System | Integration |
|--------|------------|
| Action Dispatcher | New `browse`/`browser_*` intents |
| SurfaceQueue | Push results for async workflows |
| Desktop Agent | Fallback for non-DOM browser chrome elements |
| Web Search | Can open search results directly in browser |
| Living Memory | Browser interactions logged as evidence (URL + action + outcome) |
| SBA Spine | Plans enter as TurnInput; results flow through ResponseSynthesizer |

---

## 15. Review Checklist

- [ ] Does CDP connection handle Chrome not running gracefully?
- [ ] Is credential detection comprehensive enough?
- [ ] Are domain boundaries enforced on redirects?
- [ ] Does the action limit prevent runaway execution?
- [ ] Is the plan format parseable and unambiguous?
- [ ] Does read-only mode truly skip the safety gate?
- [ ] Is the daemon start/stop lifecycle clean (no orphan WebSockets)?
- [ ] Does fallback to Desktop Agent work when selectors fail?
