# File Agent v2 Technical Specification

**Status:** Final — R1 + R2 + R3 findings addressed. Approved for implementation.
**Date:** 2026-04-10 (v4: R3 execution edge cases)
**Scope:** User-facing file manipulation for AiMe v3 (create, edit, move, copy, delete)
**Constraint:** All mutating file operations require user confirmation via fingerprinted plans. This module is NOT the sole write gateway — internal system writes (data stores, config checkpoints, retro projects) remain as-is. File Agent v2 governs user-directed file operations only.

---

## 1. Purpose and Scope Boundary

### What This Module Is

File Agent v2 handles **user-directed file operations** — when the user explicitly asks AiMe to create, edit, move, or organize files on their behalf. It provides a confirmation-gated, sandboxed execution path.

### What This Module Is NOT

It is NOT a universal file write gateway. The following existing write surfaces are outside its scope and remain unchanged:

| Existing Write Surface | Location | Policy |
|---|---|---|
| Retro project files | `data/retro_projects/` | `_retro_safe_path()` validation, user-initiated via UI |
| Chat UI settings | `data/chat_ui_settings.json` | Atomic write, UI-driven |
| Config checkpoints | `data/config_checkpoints/` | Admin route, SHA256 integrity hash |
| Image uploads | `static/images/` | Sanitized ID + whitelisted extension |
| Widget installation | `widget_store/` | ZIP extraction, manifest validation |
| Email settings | `email_settings.json` | OAuth flow, UI-driven |

**The safety claim is:** User-directed file operations that pass through the action dispatcher require confirmation. System-internal writes have their own (existing) safety mechanisms.

### Relationship to Existing file_agent.py

The existing read-only file agent (`features/cognitive_bridge_parts/file_agent.py`) is scoped to `AIME_PROJECT_ROOT` and provides pre-LM context injection. File Agent v2 operates on **user directories** (Documents, Desktop, Downloads) and uses a **different allowed-path configuration**. They share no state, but they share a single source of truth for path policy: the `file_policy` section of `lm_tuning.json`.

```
file_agent.py                        file_agent_writer.py
├── Scope: AIME_PROJECT_ROOT         ├── Scope: user-configured directories
├── Pre-LM, deterministic            ├── Post-intent, confirmation-gated
├── Read-only                         ├── Read + Write
├── Injects <<<FILE_CONTEXT>>>        ├── Executes confirmed plans
└── No SBA involvement                └── Full SBA spine involvement
```

**Coupling acknowledged:** In Phase 4, the read agent's recently-referenced files will inform write plan context. This is one-way data flow (read → write context), not bidirectional coupling.

---

## 2. Non-Negotiable Architecture Rules

1. **The response model is a pure narrator for file operations.** It does not emit REQUEST forms, structured file intents, plans, diffs, paths, tool calls, or execution directives. All user-directed file operations are handled by the **sovereign File Agent**, routed by `action_dispatcher`, following the same separation-of-concerns pattern as `email_agent`, `scheduler_agent`, and `coding_agent`. The response model only narrates already-produced confirmation text, status updates, and results returned by the File Agent.
2. **All user-directed file writes enter the SBA spine.** The File Agent proposes a plan; user confirms via fingerprinted plan; File Agent executes. Results flow back through the SBA spine for the response model to narrate.
3. **The File Agent is sovereign.** It owns planning, path resolution, validation, fingerprinting, confirmation staging, execution, undo journaling, and evidence logging. No part of this pipeline runs inside or is influenced by the response model.
4. **Sandbox is Windows-aware.** Path validation explicitly rejects known Windows path abuse vectors (see Section 5).
5. **Plans are fingerprinted and expire.** Confirmation is bound to a plan ID with per-file precondition hashes. Execution fails closed if filesystem state changed after approval.
6. **Undo is compensating actions, not transactions.** The system makes best-effort reversal. Partial failure is an explicit state, not hidden.
7. **Read operations are unchanged.** No modification to existing `file_agent.py`.

---

## 3. Architecture Overview

```text
User Request ("create a file...", "edit the config...", "move that to...")
  → Intent Engine classifies as "file_op"
  → Action Dispatcher routes to File Agent v2 (sovereign agent)
  → File Agent deterministic planner (agent-owned, no LM involvement):
      1. Parse user_text deterministically → operation type + path references
      2. Resolve paths against configured allowed directories
      3. Validate all paths against sandbox (Section 5)
      4. Compute preconditions: hash + mtime + size for all involved files
      5. For edits: read current content, compute diff deterministically
      6. Build FilePlan with fingerprint
      7. Render confirmation text via deterministic template
      8. Surface confirmation to user via SurfaceQueue
  → User confirms (references plan_id, explicitly or implicitly)
  → File Agent execution engine:
      1. Re-validate sandbox with fresh resolution (TOCTOU defense)
      2. Verify precondition fingerprints match current state (fail closed if changed)
      3. Create backup/journal entry BEFORE mutation
      4. Execute operation
      5. Log to evidence ledger (full audit)
      6. Push result text to SurfaceQueue
  → Response model narrates the result text (does not generate it)
```

**Actor boundary:** The response model receives confirmation/result text from the File Agent and narrates it to the user. It never generates operational content, paths, diffs, or action decisions. This is identical to how the email agent, scheduler agent, and coding agent already work.

---

## 4. Core Runtime Objects

### 4.1 `FileOperation`

```python
@dataclass
class FileOperation:
    op_type: str                    # "create", "edit", "move", "copy", "delete"
    target_path: Path               # Primary target (resolved absolute path)
    source_path: Path | None = None # For move/copy: original location (resolved)
    content: str = ""               # For create: full content. For edit: new content.
    description: str = ""           # Human-readable step description (deterministic template, NOT LM-generated)
    
    # Encoding (R3 #1 — Encoding Detection, NOT assumed UTF-8)
    # Detected via charset-normalizer at plan time for edits. Stored and reused for writes.
    # For creates: defaults to "utf-8". For edits: detected from existing file.
    encoding: str = "utf-8"
    encoding_confidence: float = 1.0  # charset-normalizer confidence (0.0-1.0)
    has_bom: bool = False             # Whether file starts with BOM (preserved on write)
    
    # Raw paths for double-resolution (R2 #7): stored as originally resolved,
    # re-resolved from scratch at execution time to detect junction attacks
    raw_target_path: str = ""       # Original path string before resolution
    raw_source_path: str = ""       # Original source path string (for move/copy)
    
    # File classification (determined by planner + content analysis, governs permission policy)
    # Classification tracks renames: if source was .exe and target is .txt, highest class applies (R3 #6)
    file_class: str = "text"        # "text", "config", "executable", "data" (extension-based)
    source_file_class: str = ""     # Class of source file (for move/copy/rename — tracks classification changes)
    content_class: str = "text"     # Result of content smuggling analysis (may override file_class)
    content_elevation_reason: str = ""  # Why content_class differs from file_class
    
    # Full-graph preconditions: ALL paths involved are fingerprinted (R2 #3)
    target_precondition: "FilePrecondition | None" = None   # State of target_path
    source_precondition: "FilePrecondition | None" = None   # State of source_path (for move/copy)
    parent_precondition: "DirPrecondition | None" = None    # State of target parent directory
    
    # Backup pointer (set after backup is created, before execution)
    backup_path: Path | None = None


@dataclass
class FilePrecondition:
    """
    Immutable snapshot of file state at plan time. Execution fails if mismatch.
    
    Tiered verification (R2 #5 — performance):
    - "text" files: metadata-only check (mtime_ns + size_bytes) — fast
    - "config" and "executable" files: full SHA256 hash — slower but necessary
    
    The tier is determined by the file_class of the operation.
    """
    exists: bool
    size_bytes: int = 0
    mtime_ns: int = 0               # os.stat().st_mtime_ns
    sha256: str = ""                # Hash of content (only computed for config/executable class)
    verification_tier: str = "metadata"  # "metadata" or "hash"
    
    def matches(self, current: "FilePrecondition") -> bool:
        """Returns True if filesystem state matches what we saw at plan time."""
        if self.exists != current.exists:
            return False
        if not self.exists:
            return True  # Both don't exist — match
        
        if self.verification_tier == "hash":
            # Full hash verification for config/executable
            return self.sha256 == current.sha256
        else:
            # Metadata verification for text files (fast path)
            return (self.mtime_ns == current.mtime_ns and 
                    self.size_bytes == current.size_bytes)


@dataclass
class DirPrecondition:
    """Snapshot of parent directory state. Detects if directory was replaced/junctioned."""
    exists: bool
    is_dir: bool = True
    is_reparse_point: bool = False  # Must remain False at execution time
    
    def matches(self, current: "DirPrecondition") -> bool:
        if not current.exists or not current.is_dir:
            return False
        if current.is_reparse_point and not self.is_reparse_point:
            return False  # Directory was replaced with a junction — fail closed
        return True
```

### 4.2 `FilePlan`

```python
@dataclass
class FilePlan:
    plan_id: str                          # UUID, referenced in confirmation
    session_id: str
    directive: str                        # Original user request
    operations: list[FileOperation]       # Ordered operations
    plan_summary: str                     # Human-readable summary for confirmation
    plan_fingerprint: str                 # SHA256(plan_id + sorted precondition hashes)
    created_at: float
    expires_at: float                     # created_at + TTL (default 300s)
    
    # State
    status: str = "pending"               # "pending", "confirmed", "executing", "completed", "failed", "expired"
    confirmed_at: float | None = None
    completed_at: float | None = None
    
    # Execution result
    operations_completed: int = 0
    error: str = ""
    
    def is_expired(self) -> bool:
        return time.time() > self.expires_at
```

### 4.3 `CompensatingAction`

```python
@dataclass
class CompensatingAction:
    """
    Best-effort reversal of a completed operation. NOT a transaction rollback.
    May fail if filesystem state has changed since execution.
    """
    entry_id: str
    plan_id: str
    operation_index: int                  # Which operation in the plan this reverses
    original_op: FileOperation            # What was done
    reversal_op: FileOperation            # What to do to undo it
    backup_path: Path | None              # Where the backup lives (for edits/deletes)
    
    # Preconditions for the undo itself
    undo_precondition: FilePrecondition   # Expected state before undo executes
    
    # State
    timestamp: float
    status: str = "available"             # "available", "executed", "failed", "expired"
    failure_reason: str = ""


@dataclass
class UndoJournal:
    """
    Journal of compensating actions. NOT a transaction log.
    Honest about what can and cannot be undone.
    """
    entries: list[CompensatingAction]
    
    # Retention
    retention_hours: int = 24             # Entries pruned after this
    backup_retention_hours: int = 24      # Backup files deleted after this
    trash_retention_days: int = 7         # Soft-deleted files purged after this
```

### 4.4 `FileResult`

```python
@dataclass
class FileResult:
    success: bool
    plan_id: str
    operations_completed: int
    operations_total: int
    files_affected: list[str]
    
    # Partial failure is explicit
    partial_failure: bool = False         # True if some ops succeeded, some failed
    failed_at_index: int = -1             # Which operation failed (-1 = none)
    error: str = ""
    
    # Undo availability
    undo_available: bool = False
    undo_entry_ids: list[str] = field(default_factory=list)
    
    # Compensating action status for partial failure
    rollback_attempted: bool = False
    rollback_succeeded: bool = False
    rollback_details: str = ""
```

---

## 5. Windows-Specific Sandbox Security

### 5.1 Threat Model

On Windows, the following path abuse vectors exist beyond simple `..` traversal:

| Vector | Example | Mitigation |
|--------|---------|-----------|
| Junction/reparse point | `mklink /J escape_dir \\server\share` | Reject paths containing junctions |
| UNC paths | `\\server\share\file.txt` | Reject any path starting with `\\` |
| Device paths | `\\.\COM1`, `\\?\C:\` | Reject paths starting with `\\.` or `\\?` |
| Alternate Data Streams | `file.txt:hidden_stream` | Reject paths containing `:` after drive letter |
| Short name aliases | `PROGRA~1` instead of `Program Files` | Resolve from scratch at BOTH plan and execution time (never reuse cached resolution) |
| Drive-relative paths | `C:file.txt` (resolves to CWD of C:) | Reject paths matching `X:` without `X:\` — must be fully absolute |
| Hard links | `mklink /H link.txt target.txt` | Not preventable; accept as known limitation |
| TOCTOU race | Path validated, then target changed before write | Re-validate + re-resolve immediately before execution |
| Path length abuse | Paths >260 chars with `\\?\` prefix | Reject paths exceeding MAX_PATH |
| Reserved names | `CON`, `PRN`, `AUX`, `NUL`, `COM1`-`COM9` | Reject reserved device names |
| Trailing dots/spaces | `file.txt.` or `file.txt ` | REJECT as ambiguous (Section 6.8) — never normalize |

### 5.2 Path Validation Implementation

```python
import ctypes
import os
import re
from pathlib import Path

# Windows reserved device names
_RESERVED_NAMES = frozenset({
    "CON", "PRN", "AUX", "NUL",
    "COM1", "COM2", "COM3", "COM4", "COM5", "COM6", "COM7", "COM8", "COM9",
    "LPT1", "LPT2", "LPT3", "LPT4", "LPT5", "LPT6", "LPT7", "LPT8", "LPT9",
})

# Regex: detects Alternate Data Streams (colon after drive letter position)
_ADS_RE = re.compile(r"^[A-Za-z]:[/\\].+:.+$")

# Regex: detects UNC/device paths
_UNC_RE = re.compile(r"^\\\\")


def validate_path(
    path: Path,
    allowed_dirs: list[Path],
    blocked_patterns: list[str],
) -> tuple[bool, str]:
    """
    Windows-aware path validation. Returns (is_valid, error_reason).
    Designed to reject ambiguous or dangerous paths rather than try to sanitize them.
    
    IMPORTANT (R2 #7 — Double Resolution):
    This function ALWAYS resolves from the raw path. It NEVER accepts a pre-resolved
    Path from a cached plan. Both plan-time and execution-time validation must call
    this with the ORIGINAL user-supplied path string to detect junctions/short-names
    that changed between plan and execution.
    """
    raw = str(path)
    
    # --- Reject unsupported path classes outright ---
    
    # UNC paths (\\server\share or \\.\device or \\?\extended)
    if _UNC_RE.match(raw):
        return False, "UNC/device paths are not supported"
    
    # Alternate Data Streams (file.txt:stream)
    if _ADS_RE.match(raw):
        return False, "Alternate Data Streams are not supported"
    
    # Drive-relative paths (R2 #2): "C:file.txt" resolves to CWD of C: drive
    # Must reject any path that is "X:" without "X:\" — not fully absolute
    _DRIVE_RELATIVE_RE = re.compile(r"^[A-Za-z]:[^/\\]")
    if _DRIVE_RELATIVE_RE.match(raw):
        return False, "Drive-relative paths are not supported (must be fully absolute: C:\\...)"
    
    # Require fully absolute paths: must start with drive letter + separator
    abspath = os.path.abspath(raw)
    if not re.match(r"^[A-Za-z]:[/\\]", abspath):
        return False, "Path must be fully absolute (drive letter + backslash)"
    
    # Path length > MAX_PATH (260)
    if len(abspath) > 260:
        return False, "Path exceeds MAX_PATH (260 characters)"
    
    # Reserved device names (check all path components, not just the final stem)
    for part in Path(abspath).parts:
        part_stem = Path(part).stem.upper()
        if part_stem in _RESERVED_NAMES:
            return False, f"Reserved device name in path: {part_stem}"
    
    # --- Resolve to canonical form (fresh resolution, never cached) ---
    
    # Convert short names (PROGRA~1) to long names via Windows API
    try:
        resolved = _resolve_long_path(Path(abspath))
    except OSError:
        # If path doesn't exist yet, resolve parent
        parent = Path(abspath).parent
        if parent.exists():
            resolved = _resolve_long_path(parent) / Path(abspath).name
        else:
            resolved = Path(abspath)
    
    # REJECT trailing dots/spaces (R3 #5: don't normalize, reject as ambiguous)
    for part in resolved.parts:
        valid, err = validate_filename(part)
        if not valid:
            return False, err
    
    # --- Check against allowed directories ---
    
    if not any(_is_strict_subpath(resolved, d) for d in allowed_dirs):
        return False, f"Path is not within any allowed directory"
    
    # --- Check for junction/reparse points in path components ---
    
    # Walk up from target to allowed dir root, checking each component
    for ancestor in _path_ancestors(resolved):
        if ancestor.exists() and _is_reparse_point(ancestor):
            return False, f"Path contains junction/reparse point: {ancestor}"
    
    # --- Check blocked patterns ---
    
    for pattern in blocked_patterns:
        if resolved.match(pattern):
            return False, f"Path matches blocked pattern: {pattern}"
    
    return True, ""


def _resolve_long_path(path: Path) -> Path:
    """Convert short name (8.3) path to long name using Windows API."""
    if os.name != "nt":
        return path.resolve()
    try:
        buf = ctypes.create_unicode_buffer(260)
        ctypes.windll.kernel32.GetLongPathNameW(str(path), buf, 260)
        if buf.value:
            return Path(buf.value).resolve()
    except Exception:
        pass
    return path.resolve()


def _is_reparse_point(path: Path) -> bool:
    """Check if a path is a junction or other reparse point (Windows)."""
    if os.name != "nt":
        return path.is_symlink()
    try:
        attrs = ctypes.windll.kernel32.GetFileAttributesW(str(path))
        if attrs == -1:  # INVALID_FILE_ATTRIBUTES
            return False
        FILE_ATTRIBUTE_REPARSE_POINT = 0x0400
        return bool(attrs & FILE_ATTRIBUTE_REPARSE_POINT)
    except Exception:
        return path.is_symlink()  # Fallback


def _is_strict_subpath(child: Path, parent: Path) -> bool:
    """
    True if child is strictly under parent.
    Both must be resolved (absolute, no symlinks in parent).
    """
    try:
        child.relative_to(parent)
        return child != parent  # Must be strictly inside, not equal
    except ValueError:
        return False


def _path_ancestors(path: Path) -> list[Path]:
    """Return all ancestor directories from path up to root."""
    ancestors = []
    current = path.parent
    while current != current.parent:
        ancestors.append(current)
        current = current.parent
    return ancestors
```

### 5.3 TOCTOU Defense + Double Resolution (R2 #7)

Path validation happens twice, and **each time resolves from scratch**:

1. **At plan time** — resolve raw path → validate → store the RAW path (not the resolved one) in the plan.
2. **Immediately before execution** — resolve the same raw path AGAIN from scratch → validate again → compare resolved result with plan-time resolution. If they differ (e.g., a short name now resolves differently because a junction was created), fail closed.

**Critical rule:** The `FilePlan` stores the **original unresolved path string** alongside the resolved path. At execution time, `validate_path()` is called again with the original string. This detects short-name aliasing attacks where the resolution target changes between plan and execution.

```python
def pre_execution_check(plan: FilePlan) -> tuple[bool, str]:
    """
    Full pre-execution verification. Returns (safe_to_proceed, error).
    Re-resolves all paths from scratch. Never trusts cached resolutions.
    """
    for op in plan.operations:
        # Re-resolve from original raw path (stored at plan time)
        valid, err = validate_path(Path(op.raw_target_path), allowed_dirs, blocked_patterns)
        if not valid:
            return False, f"Path validation failed at execution time: {err}"
        
        # Check that resolution matches what we showed the user
        fresh_resolved = _resolve_long_path(Path(os.path.abspath(op.raw_target_path)))
        if fresh_resolved != op.target_path:
            return False, (
                f"Path resolution changed since plan was created. "
                f"Plan-time: {op.target_path}, Now: {fresh_resolved}. "
                f"This may indicate a junction or symlink was created. Aborting."
            )
        
        # Verify preconditions (full-graph: target, source, parent)
        if op.target_precondition:
            current = compute_precondition(op.target_path)
            if not op.target_precondition.matches(current):
                return False, f"Target file changed after approval: {op.target_path.name}"
        
        if op.source_precondition and op.source_path:
            current = compute_precondition(op.source_path)
            if not op.source_precondition.matches(current):
                return False, f"Source file changed after approval: {op.source_path.name}"
        
        if op.parent_precondition:
            current_dir = compute_dir_precondition(op.target_path.parent)
            if not op.parent_precondition.matches(current_dir):
                return False, f"Parent directory changed (possible junction attack): {op.target_path.parent}"
    
    return True, ""
```

Additionally, precondition fingerprints are verified for ALL paths in the operation graph (target, source, parent directory). If anything changed after the user approved the plan, execution fails closed:

```
"Plan expired: [target file/source file/parent directory] was modified after 
 you approved the plan. Please re-request the operation to get an updated plan."
```

### 5.4 Known Limitations (Accepted)

- **Hard links:** Cannot prevent a hard link from being created to a file we wrote. This is a filesystem-level concern outside our control.
- **Admin-created junctions:** If an admin creates a junction inside an allowed directory pointing outside, we detect it at execution time but cannot prevent its creation. We DO detect it and abort.
- **Race window:** Between the final re-validation and the actual write, a microsecond race exists. This is inherent to non-transactional filesystems and is accepted.
- **Metadata-tier false negatives:** For "text" class files using metadata-only preconditions, a file rewritten with identical size at the same nanosecond timestamp would not be detected. This is accepted as astronomically unlikely for user-facing files.

---

## 6. Permissions Matrix

Operations are governed by **file class** and **operation type**. File class is determined by BOTH extension AND content analysis (see Section 6.2 for content smuggling defense):

| | text | config | executable | data (binary) |
|---|---|---|---|---|
| **create** | Confirm | Elevated confirm | Elevated confirm + warning | Reject |
| **edit** | Confirm + diff | Elevated confirm + diff | Reject | Reject |
| **move** | Confirm | Elevated confirm | Elevated confirm | Confirm |
| **copy** | Confirm | Confirm | Elevated confirm | Confirm |
| **delete** | Confirm (soft) | Elevated confirm (soft) | Elevated confirm (soft) | Confirm (soft) |

**"Elevated confirm"** means the confirmation message explicitly names the risk:
- Config: "This is a configuration file. Changes may affect application behavior."
- Executable: "This will create/modify an executable file that could run code on your system."

**"Reject"** means the operation is refused outright — not a harder confirmation.

### 6.1 File Class Determination (Extension-Based)

```python
_EXECUTABLE_EXT = frozenset({".exe", ".bat", ".cmd", ".ps1", ".sh", ".com", ".msi", ".vbs", ".wsf"})
_CONFIG_EXT = frozenset({".conf", ".ini", ".cfg", ".yaml", ".yml", ".toml", ".env", ".json"})
_BINARY_EXT = frozenset({
    ".png", ".jpg", ".jpeg", ".gif", ".bmp", ".ico", ".webp",
    ".pdf", ".zip", ".tar", ".gz", ".7z", ".rar",
    ".exe", ".dll", ".so", ".bin", ".dat",
    ".db", ".sqlite", ".sqlite3",
    ".mp3", ".mp4", ".wav", ".avi", ".mov",
})

def classify_file_by_extension(path: Path) -> str:
    ext = path.suffix.lower()
    if ext in _EXECUTABLE_EXT:
        return "executable"
    if ext in _CONFIG_EXT:
        return "config"
    if ext in _BINARY_EXT:
        return "data"
    return "text"
```

### 6.2 Content Smuggling Defense (R2 #1)

**The Attack:** A malicious user crafts input that causes the content assembler (Section 7.3) to produce content that looks harmless in the deterministic plan summary but contains shell commands or executable code. The user confirms the innocent-looking summary without reading the full content preview.

**The Defense:** After the planner assembles content, a **content analysis pass** inspects the actual bytes that will be written. If a "text" file contains executable patterns, its `content_class` is elevated, forcing the appropriate risk warning regardless of what the plan summary says.

```python
import re

# Patterns that indicate executable/script content
_SCRIPT_INDICATORS = [
    # Shell scripts
    re.compile(r"^#!\s*/(?:bin|usr/bin)/", re.MULTILINE),           # Shebang
    re.compile(r"\b(?:rm\s+-rf|sudo\s+|chmod\s+|chown\s+)", re.MULTILINE),
    
    # Windows batch/PowerShell
    re.compile(r"^@echo\s+off", re.MULTILINE | re.IGNORECASE),
    re.compile(r"\b(?:del\s+/[fqs]|rmdir\s+/s|format\s+[a-z]:)", re.MULTILINE | re.IGNORECASE),
    re.compile(r"\bInvoke-(?:Expression|WebRequest|Command)\b", re.MULTILINE),
    re.compile(r"\bNew-Object\s+System\.Net", re.MULTILINE),
    re.compile(r"\b(?:Start-Process|Set-ExecutionPolicy)\b", re.MULTILINE),
    
    # JavaScript/VBScript execution
    re.compile(r"\bWScript\.Shell\b", re.MULTILINE),
    re.compile(r"\beval\s*\(", re.MULTILINE),
    re.compile(r"\bnew\s+ActiveXObject\b", re.MULTILINE),
    
    # Registry manipulation
    re.compile(r"\b(?:reg\s+add|reg\s+delete|HKLM\\|HKCU\\)", re.MULTILINE | re.IGNORECASE),
    
    # Network/download commands
    re.compile(r"\b(?:curl|wget|Invoke-WebRequest|certutil\s+-urlcache)", re.MULTILINE | re.IGNORECASE),
    re.compile(r"\bnet\s+(?:user|localgroup|use)\b", re.MULTILINE | re.IGNORECASE),
]

# Binary content indicators
_BINARY_INDICATORS = [
    b"\x00",                    # Null bytes
    b"\x4d\x5a",               # MZ header (PE executable)
    b"\x7f\x45\x4c\x46",      # ELF header
    b"\x50\x4b\x03\x04",      # ZIP/DOCX header
]


def analyze_content_class(content: str, declared_class: str) -> tuple[str, str]:
    """
    Second-pass content analysis. Returns (effective_class, reason).
    If content contains executable patterns, elevates class regardless of extension.
    
    This prevents semantic smuggling: plan says "meeting notes" but content is a script.
    """
    if declared_class in ("executable", "data"):
        return declared_class, ""  # Already elevated — no further analysis needed
    
    # Check for binary content
    content_bytes = content.encode("utf-8", errors="replace")
    for indicator in _BINARY_INDICATORS:
        if indicator in content_bytes[:512]:  # Check first 512 bytes
            return "data", f"Binary content detected (header: {indicator.hex()})"
    
    # Check for script/executable patterns
    matches = []
    for pattern in _SCRIPT_INDICATORS:
        match = pattern.search(content)
        if match:
            matches.append(match.group(0)[:60])
    
    if len(matches) >= 2:
        # Two or more script indicators → elevate to executable class
        return "executable", f"Script content detected: {'; '.join(matches[:3])}"
    elif len(matches) == 1:
        # Single indicator → elevate to config (warning, not rejection)
        return "config", f"Possible script content: {matches[0]}"
    
    return declared_class, ""  # No elevation needed


def classify_file_complete(path: Path, content: str) -> tuple[str, str]:
    """
    Full classification: extension-based + content analysis.
    Returns (effective_class, elevation_reason).
    The higher-risk class wins.
    """
    ext_class = classify_file_by_extension(path)
    content_class, reason = analyze_content_class(content, ext_class)
    
    # Take the more restrictive class
    CLASS_PRIORITY = {"text": 0, "config": 1, "executable": 2, "data": 3}
    if CLASS_PRIORITY.get(content_class, 0) > CLASS_PRIORITY.get(ext_class, 0):
        return content_class, reason
    return ext_class, ""
```

**How this integrates with the confirmation UX:**

If content analysis elevates a file from "text" to "executable," the confirmation message changes:

```
AiMe: ⚠️ Content Warning — I'll create a file:
  Path: C:/Users/jvcan/Documents/meeting_notes.md
  
  WARNING: This file contains executable script patterns:
    - "Invoke-WebRequest" (PowerShell download command)
    - "@echo off" (batch script header)
  
  Despite the .md extension, this content could execute code if renamed or piped.
  
  [Plan FA-7x2k | expires in 5:00]
  Say "go ahead" only if you trust this content.
```

The user sees the ACTUAL risk, not just the plan summary text.

**"Reject"** means the operation is refused outright with an explanation, not just a harder confirmation.

---

## 6.3 Classification Persistence for Move/Rename (R3 #6)

A file's highest-risk classification persists across operations within a single plan. This prevents the "rename executable to .txt, move to Startup, rename back" attack:

```python
def classify_operation_effective(op: FileOperation) -> str:
    """
    Effective class for permission lookup.
    Takes the HIGHEST risk class across: source extension, target extension, and content analysis.
    """
    CLASS_PRIORITY = {"text": 0, "config": 1, "executable": 2, "data": 3}
    
    candidates = [op.file_class, op.content_class]
    if op.source_file_class:
        candidates.append(op.source_file_class)
    
    # If the operation involves a classification CHANGE (e.g., .exe → .txt or .txt → .exe),
    # always use the highest class and force elevated confirmation
    return max(candidates, key=lambda c: CLASS_PRIORITY.get(c, 0))
```

If a move/copy/rename causes a classification change (source class != target class), the confirmation message explicitly warns:

```
⚠️  FILE TYPE CHANGE: This moves an executable file (.exe) to a text file path (.txt).
    The file retains its executable nature regardless of extension.
    Effective security class: EXECUTABLE (elevated confirmation required).
```

---

## 6.4 Encoding Detection (R3 #1)

The File Agent does NOT assume UTF-8. For edit operations, encoding is detected from the existing file and preserved on write.

```python
from charset_normalizer import from_bytes

def detect_encoding(file_path: Path) -> tuple[str, float, bool]:
    """
    Detect file encoding using charset-normalizer.
    Returns (encoding_name, confidence, has_bom).
    
    Rules:
    - If confidence < 0.7: REFUSE the edit ("Encoding is ambiguous, cannot safely edit")
    - If BOM detected: preserve it on write
    - If file is binary: REFUSE ("Binary file detected")
    """
    raw = file_path.read_bytes()
    
    # Check for BOM markers
    has_bom = False
    if raw.startswith(b'\xff\xfe') or raw.startswith(b'\xfe\xff'):
        has_bom = True  # UTF-16 BOM
    elif raw.startswith(b'\xef\xbb\xbf'):
        has_bom = True  # UTF-8 BOM
    
    # Detect encoding
    result = from_bytes(raw).best()
    if result is None:
        return "", 0.0, False  # Undetectable — refuse
    
    return result.encoding, result.encoding_confidence, has_bom


def safe_read_file(file_path: Path) -> tuple[str, str, float, bool]:
    """
    Read file with detected encoding. Returns (content, encoding, confidence, has_bom).
    Raises EncodingRefusal if encoding cannot be determined.
    """
    encoding, confidence, has_bom = detect_encoding(file_path)
    
    if confidence < 0.7:
        raise EncodingRefusal(
            f"Cannot safely edit '{file_path.name}': encoding detection confidence "
            f"is {confidence:.0%} ({encoding}). File may be corrupted by editing. "
            f"Please specify the encoding explicitly or edit the file manually."
        )
    
    content = file_path.read_bytes()
    # Strip BOM for internal processing (re-added on write)
    if has_bom:
        if content.startswith(b'\xef\xbb\xbf'):
            content = content[3:]
        elif content.startswith(b'\xff\xfe'):
            content = content[2:]
        elif content.startswith(b'\xfe\xff'):
            content = content[2:]
    
    return content.decode(encoding), encoding, confidence, has_bom


def safe_write_file(file_path: Path, content: str, encoding: str, has_bom: bool) -> None:
    """Write file preserving original encoding and BOM."""
    encoded = content.encode(encoding)
    
    # Re-add BOM if original had one
    if has_bom:
        if encoding.lower().startswith("utf-8"):
            encoded = b'\xef\xbb\xbf' + encoded
        elif encoding.lower() in ("utf-16-le", "utf-16"):
            encoded = b'\xff\xfe' + encoded
        elif encoding.lower() == "utf-16-be":
            encoded = b'\xfe\xff' + encoded
    
    file_path.write_bytes(encoded)
```

**Integration with planner:** When building a plan for an edit operation, the planner calls `detect_encoding()` and stores the result in `FileOperation.encoding`, `encoding_confidence`, and `has_bom`. If confidence is below threshold, the plan is rejected at creation time with a clear message to the user.

**For create operations:** Default to UTF-8 without BOM. User can specify encoding explicitly.

---

## 6.5 Pre-Flight Resource Checks (R3 #3)

Before executing ANY plan, the executor performs resource validation:

```python
import shutil

def preflight_check(plan: FilePlan) -> tuple[bool, str]:
    """
    Verify system resources are sufficient for plan execution.
    Called AFTER confirmation, BEFORE any mutation.
    """
    # 1. Calculate total space needed for backups
    backup_bytes_needed = 0
    for op in plan.operations:
        if op.op_type in ("edit", "delete"):
            # Need to back up the existing file
            if op.target_path.exists():
                backup_bytes_needed += op.target_path.stat().st_size
        if op.op_type == "create":
            # Need space for the new file
            backup_bytes_needed += len(op.content.encode(op.encoding))
    
    # 2. Add safety margin (2x for backup + execution headroom)
    total_needed = backup_bytes_needed * 2
    
    # 3. Check available disk space on target drive
    target_drives = set()
    for op in plan.operations:
        target_drives.add(op.target_path.drive or op.target_path.anchor)
    
    for drive in target_drives:
        usage = shutil.disk_usage(drive)
        if usage.free < total_needed + (1024 * 1024):  # 1MB minimum headroom
            return False, (
                f"Insufficient disk space on {drive}. "
                f"Need {total_needed // 1024}KB for backups + execution, "
                f"only {usage.free // 1024}KB available. "
                f"Free up space before proceeding."
            )
    
    # 4. Check backup directory is writable
    backup_dir = Path("C:/AiMe_v3/data/file_agent/backups")
    backup_dir.mkdir(parents=True, exist_ok=True)
    try:
        test_file = backup_dir / ".write_test"
        test_file.write_text("test")
        test_file.unlink()
    except (OSError, PermissionError) as e:
        return False, f"Backup directory is not writable: {e}"
    
    return True, ""
```

If the pre-flight check fails, the plan is rejected with a clear message BEFORE any filesystem mutation occurs. The user's data remains untouched.

---

## 6.6 Directory Expansion Safety (R3 #2)

When the user requests organization of a directory, the planner must scan for files to operate on. This scan is bounded:

```python
import time

_MAX_SCAN_DEPTH = 3           # Maximum directory recursion depth
_MAX_SCAN_FILES = 500         # Maximum files to enumerate before stopping
_MAX_SCAN_TIME_S = 2.0        # Maximum wall-clock time for scan
_SKIP_DIRS = frozenset({      # Never descend into these
    "node_modules", ".git", "__pycache__", ".venv", "venv",
    ".tox", "dist", "build", ".cache", ".npm", ".yarn",
})


def scan_directory_bounded(
    root: Path,
    max_depth: int = _MAX_SCAN_DEPTH,
    max_files: int = _MAX_SCAN_FILES,
    max_time_s: float = _MAX_SCAN_TIME_S,
) -> tuple[list[Path], bool]:
    """
    Non-recursive bounded directory scan.
    Returns (files_found, was_truncated).
    
    Uses iterative BFS with depth tracking. Never follows junctions/symlinks.
    Skips known-heavy directories (node_modules, .git, etc.).
    """
    start = time.monotonic()
    files: list[Path] = []
    truncated = False
    
    # BFS queue: (path, depth)
    queue: list[tuple[Path, int]] = [(root, 0)]
    
    while queue:
        # Time limit check
        if time.monotonic() - start > max_time_s:
            truncated = True
            break
        
        # File count limit
        if len(files) >= max_files:
            truncated = True
            break
        
        current, depth = queue.pop(0)
        
        # Depth limit
        if depth > max_depth:
            continue
        
        try:
            for entry in current.iterdir():
                # Never follow reparse points/junctions
                if _is_reparse_point(entry):
                    continue
                
                if entry.is_file():
                    files.append(entry)
                    if len(files) >= max_files:
                        truncated = True
                        break
                elif entry.is_dir() and entry.name not in _SKIP_DIRS:
                    if depth < max_depth:
                        queue.append((entry, depth + 1))
        except PermissionError:
            continue  # Skip directories we can't read
    
    return files, truncated
```

**When truncated:** The planner reports to the user:

```
"Found 500+ files in Documents (scan limited to depth 3, 2 seconds).
 Showing the top 20 candidates for organization. 
 To operate on deeper files, specify a more specific subdirectory."
```

The 20-file plan limit (Section 11) applies AFTER the bounded scan selects candidates.

---

## 6.7 Link-Type Prohibition (R3 #4)

The File Agent NEVER creates symbolic links, junctions, or hard links. All file operations produce regular files only.

```python
import os

def safe_move(source: Path, target: Path) -> None:
    """
    Move a file. Refuses to create any type of link.
    Uses copy-then-delete for cross-volume moves (rename() can't cross volumes).
    """
    # Refuse if source is a reparse point
    if _is_reparse_point(source):
        raise LinkRefusal(f"Refusing to move a junction/symlink: {source}")
    
    # Verify target parent is not a reparse point
    if _is_reparse_point(target.parent):
        raise LinkRefusal(f"Refusing to write into a junction: {target.parent}")
    
    # Attempt rename (same volume)
    try:
        source.rename(target)
    except OSError:
        # Cross-volume: copy then delete
        safe_copy(source, target)
        source.unlink()


def safe_copy(source: Path, target: Path) -> None:
    """
    Copy a file. Never creates links. Never follows links.
    """
    # Refuse if source is a link
    if _is_reparse_point(source):
        raise LinkRefusal(f"Refusing to copy a junction/symlink: {source}")
    
    # Refuse if target parent is a link
    if _is_reparse_point(target.parent):
        raise LinkRefusal(f"Refusing to write into a junction: {target.parent}")
    
    # Open source without following links (Windows: FILE_FLAG_OPEN_REPARSE_POINT is NOT set,
    # but we've already verified it's not a reparse point above)
    # Copy as regular file — never use shutil.copy2 which preserves link status
    with open(source, "rb") as src:
        with open(target, "wb") as dst:
            while chunk := src.read(65536):
                dst.write(chunk)
    
    # Copy metadata (times only, not link attributes)
    stat = source.stat()
    os.utime(target, (stat.st_atime, stat.st_mtime))


def safe_create(target: Path, content: str, encoding: str, has_bom: bool) -> None:
    """
    Create a file. Verifies we're not writing through a junction.
    Uses os.open with O_CREAT|O_EXCL to guarantee no overwrite and no link follow.
    """
    if _is_reparse_point(target.parent):
        raise LinkRefusal(f"Refusing to create file in a junction: {target.parent}")
    
    # O_EXCL: fail if file already exists (prevents race where file was created after plan)
    # O_CREAT: create the file
    fd = os.open(str(target), os.O_WRONLY | os.O_CREAT | os.O_EXCL)
    try:
        safe_write_file(target, content, encoding, has_bom)
    finally:
        os.close(fd)
```

**The executor uses ONLY these safe_* functions.** Direct use of `Path.rename()`, `shutil.copy2()`, or `Path.write_text()` is prohibited in the executor module.

---

## 6.8 Filename Sanitization (R3 #5)

Instead of normalizing trailing dots/spaces (which assumes intent), the sandbox REJECTS them:

```python
_UNSAFE_FILENAME_RE = re.compile(r"[\s.]+$")  # Trailing spaces or dots

def validate_filename(name: str) -> tuple[bool, str]:
    """
    Reject ambiguous filenames that Windows handles inconsistently.
    """
    if _UNSAFE_FILENAME_RE.search(name):
        return False, (
            f"Filename '{name}' ends with spaces or dots. "
            f"Windows handles these inconsistently. "
            f"Please specify a clean filename."
        )
    
    # Also reject filenames with embedded null or control characters
    if any(ord(c) < 32 for c in name) or '\x00' in name:
        return False, f"Filename contains control characters"
    
    # Reject Windows-forbidden characters
    if re.search(r'[<>:"/\\|?*]', name):
        return False, f"Filename contains forbidden characters"
    
    return True, ""
```

This is called during path validation (Section 5.2) for EVERY path component, not just the final filename.

---

## 7. File Agent Planning Pipeline (Agent-Owned)

The response model does NOT participate in file operation planning. The File Agent is a sovereign agent that owns the entire planning pipeline internally. The response model only narrates results.

### 7.1 Agent-Owned Planning (No LM Involvement)

The File Agent receives a routed request from `action_dispatcher` containing:
- `user_text` — the raw user input string
- `session_id` — current session
- `context` — session context (recently referenced files, active directory, etc.)

The File Agent's **deterministic planner** extracts operation intent from `user_text` using the same rule-based parsing approach used by the existing agents:

```python
@dataclass
class FileRequest:
    """
    Parsed from user_text by the File Agent's own deterministic parser.
    NOT produced by the response model.
    """
    action: str                     # "create", "edit", "move", "copy", "delete", "organize"
    path_references: list[str]      # File/dir names extracted from user_text
    location_references: list[str]  # Location hints ("Documents", "Downloads", etc.)
    content_directives: list[str]   # Content-related phrases from user_text
    edit_directives: list[str]      # Edit-related phrases ("change X to Y", "add a line")
    move_destination: str = ""      # Destination hint for moves


def parse_file_request(user_text: str, context: dict) -> FileRequest | None:
    """
    Deterministic extraction of file operation intent from user text.
    Uses regex + keyword matching — same approach as email/scheduler agents.
    Returns None if user_text does not contain a file operation request.
    """
    # 1. Identify action type from verb patterns
    action = _classify_action(user_text)
    if action is None:
        return None
    
    # 2. Extract path references (filenames, extensions, directory names)
    paths = _extract_path_references(user_text)
    
    # 3. Extract location references (known directories: Documents, Desktop, etc.)
    locations = _extract_locations(user_text, context)
    
    # 4. Extract content/edit directives
    content = _extract_content_directives(user_text) if action == "create" else []
    edits = _extract_edit_directives(user_text) if action == "edit" else []
    
    # 5. Extract move destination
    dest = _extract_destination(user_text) if action in ("move", "copy") else ""
    
    return FileRequest(
        action=action,
        path_references=paths,
        location_references=locations,
        content_directives=content,
        edit_directives=edits,
        move_destination=dest,
    )
```

### 7.2 Deterministic Planner: FileRequest → FilePlan

The planner resolves the parsed `FileRequest` into concrete operations:

1. Resolves `path_references` + `location_references` → concrete absolute path
2. Validates the resolved path against sandbox (full Windows validation, Section 5)
3. Stores the **raw unresolved path string** (`raw_target_path`) alongside the resolved path for double-resolution at execution time
4. For edits: reads current file, applies `edit_directives` as deterministic text operations (find/replace, append, insert at line)
5. For creates: assembles content from `content_directives` using deterministic templates
6. **Runs content smuggling analysis** (Section 6.2) on assembled content → may elevate file_class
7. Computes full-graph preconditions (target, source, parent directory)
8. Builds the `FilePlan` with fingerprint
9. Renders confirmation using deterministic template (Section 8.4)

**Critical:** The response model never sees or generates paths, content, or operation types. The File Agent extracts all operational information from `user_text` using its own deterministic parser.

### 7.3 Content Assembly for Create Operations

For file creation, the File Agent assembles content from the user's natural language:

```python
def assemble_content(request: FileRequest, context: dict) -> str:
    """
    Deterministic content assembly. For complex content that cannot be
    deterministically assembled from user_text alone, the File Agent
    may invoke a SEPARATE, sandboxed content-generation call with
    strict output constraints (no paths, no action directives, text only).
    
    This is NOT the response model. It is a File-Agent-owned utility call
    with its own isolated prompt that cannot influence operation planning.
    """
    # Simple cases: user provided literal content
    if has_quoted_content(request.content_directives):
        return extract_quoted_content(request.content_directives)
    
    # Template cases: known file types with standard structure
    if request.path_references and has_known_template(request.path_references[0]):
        return apply_template(request.path_references[0], request.content_directives)
    
    # Complex cases: use isolated content-generation utility
    # This utility ONLY generates text content. It cannot:
    # - Emit paths, action directives, or operational schemas
    # - Influence which file is written or where
    # - Override the file_class or confirmation flow
    return generate_content_isolated(request.content_directives, context)
```

### 7.4 Edit Operations: Deterministic Diff

For edits, the File Agent parses `edit_directives` from user_text and applies them deterministically:

```python
def resolve_edit(target_path: Path, edit_directives: list[str]) -> tuple[str, str]:
    """
    Parse edit directives and apply to current file content.
    Returns (new_content, unified_diff).
    
    Supported directive patterns (extracted from user_text by parser):
    - "change X to Y" → find/replace
    - "add ... at the end" → append
    - "insert ... after ..." → insert_after
    - "remove/delete the line ..." → delete matching line
    """
    current = target_path.read_text(encoding="utf-8")
    new_content = current
    
    for directive in edit_directives:
        op = _parse_edit_directive(directive)
        if op["type"] == "replace":
            new_content = new_content.replace(op["find"], op["replace_with"])
        elif op["type"] == "append":
            new_content += "\n" + op["content"]
        elif op["type"] == "insert_after":
            # Insert content after the matching line
            new_content = _insert_after_line(new_content, op["after"], op["content"])
        elif op["type"] == "delete_matching":
            new_content = _delete_matching_lines(new_content, op["pattern"])
    
    # Compute unified diff for confirmation display
    diff = unified_diff(
        current.splitlines(keepends=True),
        new_content.splitlines(keepends=True),
        fromfile=f"a/{target_path.name}",
        tofile=f"b/{target_path.name}",
    )
    
    return new_content, "".join(diff)
```

If the deterministic edit directives cannot be applied cleanly (e.g., the find text doesn't exist in the file), the plan fails at creation time, not at execution time. The user is told: "Could not find the text you want to change in that file."

---

## 8. Confirmation Protocol

### 8.1 Plan Fingerprinting

Every plan gets a fingerprint computed from all precondition data (tiered):
```python
def compute_plan_fingerprint(plan_id: str, operations: list[FileOperation], created_at: float) -> str:
    """
    Fingerprint incorporates ALL precondition data for ALL paths.
    Uses tiered approach: hash for config/executable, metadata for text.
    """
    parts = [plan_id]
    for op in operations:
        # Target precondition
        if op.target_precondition:
            tp = op.target_precondition
            if tp.verification_tier == "hash":
                parts.append(f"t:{tp.sha256}")
            else:
                parts.append(f"t:{tp.mtime_ns}:{tp.size_bytes}")
        # Source precondition (R2 #3)
        if op.source_precondition:
            sp = op.source_precondition
            if sp.verification_tier == "hash":
                parts.append(f"s:{sp.sha256}")
            else:
                parts.append(f"s:{sp.mtime_ns}:{sp.size_bytes}")
        # Parent directory precondition
        if op.parent_precondition:
            pp = op.parent_precondition
            parts.append(f"p:{pp.is_reparse_point}")
    
    parts.append(str(created_at))
    return sha256("|".join(sorted(parts)).encode()).hexdigest()[:16]
```

### 8.2 Confirmation Binding

The user confirms a specific plan by its ID. Confirmation is valid only if:
1. `plan_id` matches a pending plan
2. Plan has not expired (default TTL: 5 minutes)
3. Plan fingerprint matches (no tampering)
4. Session ID matches (no cross-session replay)

### 8.3 Pre-Execution Verification (Full-Graph Preconditions — R2 #3)

After confirmation, before writing. Verifies ALL paths in the operation graph:

```python
def verify_preconditions(plan: FilePlan) -> tuple[bool, str]:
    """
    Fail closed if ANYTHING changed since plan was created.
    Checks: target path, source path, and parent directory for each operation.
    """
    for op in plan.operations:
        # Check target file precondition
        if op.target_precondition:
            current = compute_precondition(op.target_path)
            if not op.target_precondition.matches(current):
                return False, (
                    f"Target file '{op.target_path.name}' was modified after you approved the plan. "
                    f"Please re-request to get a fresh plan."
                )
        
        # Check source file precondition (R2 #3: move/copy source could be modified)
        if op.source_precondition and op.source_path:
            current = compute_precondition(op.source_path)
            if not op.source_precondition.matches(current):
                return False, (
                    f"Source file '{op.source_path.name}' was modified after you approved the plan. "
                    f"The file content may have changed. Please re-request to get a fresh plan."
                )
        
        # Check parent directory precondition (detects junction insertion)
        if op.parent_precondition:
            current_dir = compute_dir_precondition(op.target_path.parent)
            if not op.parent_precondition.matches(current_dir):
                return False, (
                    f"Parent directory of '{op.target_path.name}' was modified. "
                    f"Possible junction/reparse point inserted. Aborting for safety."
                )
    
    return True, ""
```

Note: This is called AFTER the double-resolution check in Section 5.3. The execution sequence is:
1. Re-resolve all raw paths from scratch (Section 5.3 `pre_execution_check`)
2. Verify full-graph preconditions (this section)
3. Create backups
4. Execute

### 8.4 Confirmation UX — Deterministic Templates (R2 #6)

**Critical rule:** The human-readable confirmation message is generated by **deterministic templates** based on `op_type`. The response model does not generate or narrate mutating action descriptions. This prevents any component from downplaying a dangerous operation with friendly language.

```python
_CONFIRMATION_TEMPLATES = {
    "create": "I will CREATE the following file:\n  Path: {path}\n  Size: {size}\n  Class: {file_class}",
    "edit": "I will MODIFY the following file:\n  Path: {path}\n  Changes: {change_count} line(s)",
    "move": "I will MOVE a file:\n  From: {source}\n  To: {target}",
    "copy": "I will COPY a file:\n  From: {source}\n  To: {target}",
    "delete": "I will DELETE the following file (moved to trash):\n  Path: {path}\n  Size: {size}",
}

_ELEVATED_TEMPLATES = {
    "config": "\n  ⚠️  CONFIG FILE — changes may affect application behavior.",
    "executable": "\n  ⚠️  EXECUTABLE FILE — this could run code on your system.",
    "content_elevated": "\n  ⚠️  CONTENT WARNING — file contains script patterns: {reason}",
}

def render_confirmation(plan: FilePlan) -> str:
    """
    Render confirmation message from deterministic template.
    The plan summary is shown as context, but the ACTION description is template-driven.
    """
    lines = []
    for i, op in enumerate(plan.operations):
        template = _CONFIRMATION_TEMPLATES[op.op_type]
        line = template.format(
            path=op.target_path,
            source=op.source_path or "",
            target=op.target_path,
            size=format_size(op),
            file_class=op.file_class,
            change_count=count_diff_lines(op),
        )
        lines.append(f"  {i+1}. {line}")
        
        # Add elevation warnings
        if op.content_class != op.file_class:
            lines.append(_ELEVATED_TEMPLATES["content_elevated"].format(
                reason=op.content_elevation_reason))
        elif op.file_class in _ELEVATED_TEMPLATES:
            lines.append(_ELEVATED_TEMPLATES[op.file_class])
    
    # Show diff for edit operations
    for op in plan.operations:
        if op.op_type == "edit":
            lines.append(f"\n  Diff preview:")
            lines.append(f"  {op.diff_preview}")
    
    lines.append(f"\n  [Plan {plan.plan_id[:7]} | expires in {format_ttl(plan)}]")
    lines.append(f"  Say \"go ahead\" to proceed.")
    
    return "\n".join(lines)
```

**Example outputs:**

Single file create:
```
AiMe: 
  1. I will CREATE the following file:
    Path: C:/Users/jvcan/Documents/meeting_notes.md
    Size: 1.2 KB (23 lines)
    Class: text

  Content preview:
    # Meeting Notes - April 10
    ## Attendees
    - John
    ...

  [Plan FA-7x2k | expires in 5:00]
  Say "go ahead" to proceed.
```

Content smuggling detected:
```
AiMe:
  1. I will CREATE the following file:
    Path: C:/Users/jvcan/Documents/helper_script.md
    Size: 0.8 KB (12 lines)
    Class: text
    ⚠️  CONTENT WARNING — file contains script patterns: "@echo off; Invoke-WebRequest"
    
  Despite the .md extension, this content contains executable patterns.
  
  [Plan FA-3k9m | expires in 5:00]
  Say "go ahead" only if you trust this content.
```

Batch delete:
```
AiMe:
  1. I will DELETE the following file (moved to trash):
    Path: C:/Users/jvcan/Downloads/old_report.pdf
    Size: 2.4 MB
  2. I will DELETE the following file (moved to trash):
    Path: C:/Users/jvcan/Downloads/temp_notes.txt
    Size: 0.3 KB
  3. I will MOVE a file:
    From: C:/Users/jvcan/Downloads/final_draft.docx
    To: C:/Users/jvcan/Documents/Projects/final_draft.docx

  [Plan FA-8p2q | expires in 5:00]
  Say "go ahead" to proceed.
```

### 8.5 Expiration and Stale Plans

- Plans expire after 5 minutes (configurable)
- Only one plan per session can be pending at a time (new plan cancels old)
- Expired plans are logged as "expired" in evidence, not silently dropped

---

## 9. Compensating Actions (Undo)

### 9.1 Honest Semantics

Undo is **compensating actions**, not database transactions. The system makes best-effort reversal with explicit failure modes:

| Scenario | Behavior |
|----------|----------|
| File created, user undoes | Move to trash (best-effort) |
| File edited, user undoes | Restore from backup if backup exists and target unchanged |
| File moved, user undoes | Move back if target still exists at destination and source path is free |
| File copied, user undoes | Delete the copy if it still exists unchanged |
| File deleted (in trash), user undoes | Restore from trash if still there and original path is free |
| File edited by someone else after our edit | Offer **Conflict Resolution** (see below) |
| Backup expired (>24h) | Undo REFUSES: "Backup no longer available." |
| Target path occupied (e.g., user created new file there) | Undo REFUSES: "Original path is now occupied." |

### 9.1.1 Conflict Resolution Mode (R3 #7)

When undo preconditions fail because the file was modified after AiMe's edit, the system does NOT simply refuse. It offers the user a choice:

```
AiMe: Cannot auto-undo — the file has been modified since my edit (23 hours ago).

  Current file: config.toml (modified 2 hours ago, 3.2 KB)
  My backup:    config.toml.backup (from my edit 23 hours ago, 2.8 KB)
  
  Options:
  1. "Show diff" — I'll show you the differences between current and my backup
  2. "Restore backup" — Overwrite current file with my backup (DESTRUCTIVE)
  3. "Save backup as" — Save my backup copy next to the current file for manual merge
  4. "Cancel" — Keep current file, discard undo request
  
  What would you like?
```

This prevents the user from being locked out of their backup. The File Agent:
- Shows the diff between current state and backup (read-only, no approval needed)
- Can restore the backup ONLY with explicit "restore backup" confirmation (elevated — warns about overwriting)
- Can save the backup as a separate file (e.g., `config.toml.aime_backup`) for manual merge
- Never silently overwrites a modified file

### 9.2 Partial Failure in Batch Operations

Batch operations do NOT guarantee atomicity. Instead:

```python
@dataclass
class RollbackReport:
    """Measured state after rollback attempt. Never guesses — always measures."""
    all_succeeded: bool
    per_operation: list[dict]  # [{index, status, measured_state}]
    summary: str


async def execute_batch(plan: FilePlan) -> FileResult:
    """
    Execute operations in order. On failure:
    1. Stop executing remaining operations
    2. Take a POST-FAILURE SNAPSHOT of current filesystem state (R2 #4)
    3. Attempt compensating actions for completed operations
    4. Take a POST-ROLLBACK SNAPSHOT
    5. Report: original plan vs. partial success state vs. rollback result
    """
    completed: list[tuple[int, FileOperation, CompensatingAction]] = []
    
    for i, op in enumerate(plan.operations):
        # Verify full-graph preconditions for THIS operation
        if op.target_precondition and not op.target_precondition.matches(
                compute_precondition(op.target_path)):
            post_failure_snapshot = measure_filesystem_state(plan, completed)
            rollback_result = await attempt_rollback_with_measurement(completed)
            return FileResult(
                success=False,
                partial_failure=True,
                failed_at_index=i,
                operations_completed=i,
                operations_total=len(plan.operations),
                rollback_attempted=True,
                rollback_succeeded=rollback_result.all_succeeded,
                rollback_details=rollback_result.summary,
                error=f"Precondition failed at operation {i+1}: file changed",
            )
        
        if op.source_precondition and op.source_path:
            if not op.source_precondition.matches(compute_precondition(op.source_path)):
                post_failure_snapshot = measure_filesystem_state(plan, completed)
                rollback_result = await attempt_rollback_with_measurement(completed)
                return FileResult(
                    success=False,
                    partial_failure=True,
                    failed_at_index=i,
                    operations_completed=i,
                    operations_total=len(plan.operations),
                    rollback_attempted=True,
                    rollback_succeeded=rollback_result.all_succeeded,
                    rollback_details=rollback_result.summary,
                    error=f"Source file changed at operation {i+1}: {op.source_path.name}",
                )
        
        # Create backup, execute, record compensating action
        backup = await create_backup(op)
        result = await execute_single(op)
        if not result.success:
            post_failure_snapshot = measure_filesystem_state(plan, completed)
            rollback_result = await attempt_rollback_with_measurement(completed)
            return FileResult(
                success=False,
                partial_failure=True,
                failed_at_index=i,
                operations_completed=i,
                operations_total=len(plan.operations),
                rollback_attempted=True,
                rollback_succeeded=rollback_result.all_succeeded,
                rollback_details=rollback_result.summary,
                error=result.error,
            )
        
        compensating = build_compensating_action(op, backup)
        completed.append((i, op, compensating))
    
    # All succeeded — journal compensating actions for future undo
    for i, op, comp in completed:
        journal.store(comp)
    
    return FileResult(success=True, ...)


async def attempt_rollback_with_measurement(
    completed: list[tuple[int, FileOperation, CompensatingAction]]
) -> RollbackReport:
    """
    R2 #4: Pre-Rollback Snapshots.
    
    1. Measure current state BEFORE attempting rollback
    2. Attempt each compensating action in reverse order
    3. Measure state AFTER each compensating action
    4. If a compensating action fails, record failure and continue with remaining
    5. Final report compares: planned state → actual state → post-rollback state
    """
    report_entries = []
    
    # Process in reverse order (undo last operation first)
    for i, op, comp in reversed(completed):
        # Measure state before this rollback step
        pre_state = measure_single_file(comp.reversal_op.target_path)
        
        try:
            # Verify undo precondition (the file is in the state we expect)
            current_pre = compute_precondition(comp.reversal_op.target_path)
            if not comp.undo_precondition.matches(current_pre):
                report_entries.append({
                    "index": i,
                    "status": "skipped_precondition_mismatch",
                    "measured_state": pre_state,
                    "reason": "File state doesn't match expected post-operation state",
                })
                continue
            
            # Attempt the compensating action
            result = await execute_single(comp.reversal_op)
            post_state = measure_single_file(comp.reversal_op.target_path)
            
            report_entries.append({
                "index": i,
                "status": "rolled_back" if result.success else "rollback_failed",
                "measured_state": post_state,
                "error": result.error if not result.success else "",
            })
        except Exception as e:
            post_state = measure_single_file(comp.reversal_op.target_path)
            report_entries.append({
                "index": i,
                "status": "rollback_exception",
                "measured_state": post_state,
                "reason": str(e),
            })
    
    all_ok = all(e["status"] == "rolled_back" for e in report_entries)
    summary_parts = []
    for e in report_entries:
        summary_parts.append(f"Op {e['index']}: {e['status']}")
    
    return RollbackReport(
        all_succeeded=all_ok,
        per_operation=report_entries,
        summary="; ".join(summary_parts),
    )


def measure_filesystem_state(plan: FilePlan, completed: list) -> dict:
    """
    Take a complete snapshot of all files involved in the plan.
    Returns measured state — never guesses, always reads from filesystem.
    """
    state = {}
    for op in plan.operations:
        state[str(op.target_path)] = measure_single_file(op.target_path)
        if op.source_path:
            state[str(op.source_path)] = measure_single_file(op.source_path)
    return state


def measure_single_file(path: Path) -> dict:
    """Measure actual current state of a file. No caching, no assumptions."""
    if not path.exists():
        return {"exists": False}
    stat = path.stat()
    return {
        "exists": True,
        "size": stat.st_size,
        "mtime_ns": stat.st_mtime_ns,
        "is_file": path.is_file(),
        "is_dir": path.is_dir(),
    }
```

### 9.3 Rollback Honesty

If rollback itself fails (e.g., a moved file was immediately opened by another process):
- The system **measures** the actual filesystem state (does NOT guess based on which operations "should have" worked)
- Reports a three-way comparison: Original State → Post-Failure State → Post-Rollback State
- Each operation's rollback status is independently measured and reported
- It does NOT retry silently or leave the user guessing
- Evidence ledger records all three states with measured hashes

### 9.4 Journal Storage

```sql
-- data/file_agent/undo_journal.db

CREATE TABLE compensating_actions (
    entry_id TEXT PRIMARY KEY,
    plan_id TEXT NOT NULL,
    operation_index INTEGER NOT NULL,
    original_op_json TEXT NOT NULL,
    reversal_op_json TEXT NOT NULL,
    backup_path TEXT,
    undo_precondition_json TEXT NOT NULL,
    created_at REAL NOT NULL,
    expires_at REAL NOT NULL,  -- created_at + retention_hours * 3600
    status TEXT NOT NULL DEFAULT 'available',
    failure_reason TEXT DEFAULT '',
    executed_at REAL
);

CREATE INDEX idx_plan_id ON compensating_actions(plan_id);
CREATE INDEX idx_status_expires ON compensating_actions(status, expires_at);
```

---

## 10. Intent Integration

### 10.1 Single Intent, Agent-Owned Classification

We use **one intent** (`file_op`). The `action_dispatcher` routes to the File Agent, which **internally** classifies the operation type using its own deterministic parser (Section 7.1). The response model plays no role in operation classification.

```python
# In action_dispatcher.py — single new branch:

elif intent == "file_op":
    if not _file_agent_write_enabled():
        return ContextPack.empty(intent="file_op", error="File write agent disabled")
    return await _dispatch_file_op(session_id, user_text, context)
```

The `file_op` intent is detected by the existing intent engine (not regex). Once routed, the File Agent's `parse_file_request()` (Section 7.1) determines the operation type from `user_text` — this is agent-owned deterministic parsing, not response-model output.

### 10.2 Intent Detection

A single lexical override in the dispatcher catches high-confidence cases:

```python
_FILE_OP_RE = re.compile(
    r"\b(create|make|write|save|edit|modify|move|rename|copy|delete|remove|organize|undo)\b"
    r".{0,40}"
    r"\b(file|document|folder|note)\b",
    re.IGNORECASE,
)
```

This is a **hint** to the intent classifier, not the sole routing mechanism. The intent model remains the primary classifier; this regex is a safety net for obvious cases.

### 10.3 Feature Flags

```json
{
  "agents": {
    "file_agent_write_enabled": true
  }
}
```

Single flag. If false, the `file_op` intent is never dispatched.

---

## 11. Limits (Precisely Defined)

| Limit | Value | Unit | Enforced At |
|-------|-------|------|-------------|
| Max single file size | 102,400 | bytes on disk (UTF-8 encoded) | Plan creation AND before execution |
| Max files per plan | 20 | files (after directory expansion) | Plan creation |
| Max total write bytes per plan | 512,000 | bytes on disk | Plan creation |
| Plan TTL | 300 | seconds | Confirmation check |
| Max pending plans per session | 1 | plans | Plan creation (new cancels old) |
| Undo retention | 24 | hours | Background cleanup |
| Backup retention | 24 | hours | Background cleanup |
| Trash retention | 7 | days | Background cleanup |
| Max path length | 260 | characters | Path validation |

**Directory expansion:** If the user says "organize my Downloads folder," the planner lists individual files and counts them against the 20-file limit. If >20 files need operations, the plan is split into multiple plans, each requiring separate confirmation.

---

## 12. Evidence Logging (Full Audit)

Evidence is logged at each lifecycle stage with immutable identifiers:

```python
# 1. Plan created
ledger.append({
    "event": "file_plan_created",
    "plan_id": plan.plan_id,
    "plan_fingerprint": plan.plan_fingerprint,
    "session_id": plan.session_id,
    "directive": plan.directive,
    "operations": [serialize_op(op) for op in plan.operations],
    "preconditions": {str(op.target_path): op.precondition.sha256 for op in plan.operations},
    "timestamp": time.time(),
})

# 2. Plan confirmed (or expired/cancelled)
ledger.append({
    "event": "file_plan_confirmed",  # or "file_plan_expired" / "file_plan_cancelled"
    "plan_id": plan.plan_id,
    "plan_fingerprint": plan.plan_fingerprint,
    "confirmed_by_session": session_id,
    "timestamp": time.time(),
})

# 3. Execution result (per operation)
ledger.append({
    "event": "file_op_executed",
    "plan_id": plan.plan_id,
    "operation_index": i,
    "op_type": op.op_type,
    "target_path": str(op.target_path),
    "precondition_verified": True,
    "result_hash": sha256(new_content),  # What was actually written
    "backup_path": str(op.backup_path) if op.backup_path else None,
    "success": True,
    "timestamp": time.time(),
})

# 4. Execution failure
ledger.append({
    "event": "file_op_failed",
    "plan_id": plan.plan_id,
    "operation_index": i,
    "op_type": op.op_type,
    "target_path": str(op.target_path),
    "failure_reason": "precondition_mismatch | permission_denied | disk_full | ...",
    "precondition_expected": op.precondition.sha256,
    "precondition_actual": current.sha256,
    "rollback_attempted": True,
    "rollback_result": "success | partial | failed",
    "timestamp": time.time(),
})

# 5. Undo executed
ledger.append({
    "event": "file_undo_executed",
    "entry_id": comp.entry_id,
    "plan_id": comp.plan_id,
    "operation_index": comp.operation_index,
    "success": True,
    "timestamp": time.time(),
})
```

---

## 13. Module Structure

```
modules/agents/file_service/
├── __init__.py
├── request_parser.py       # Deterministic user_text → FileRequest (Section 7.1)
├── planner.py              # Deterministic planner: FileRequest → FilePlan (agent-owned, no LM)
├── executor.py             # Execute confirmed plans (uses ONLY safe_* functions)
├── safe_io.py              # safe_move, safe_copy, safe_create, safe_write_file (Section 6.7)
├── sandbox.py              # Windows-aware path validation (Section 5) + filename sanitization (6.8)
├── permissions.py          # File classification + content smuggling + classification persistence (Section 6)
├── encoding.py             # Encoding detection via charset-normalizer (Section 6.4)
├── preflight.py            # Pre-flight disk space + resource checks (Section 6.5)
├── dir_scanner.py          # Bounded directory expansion (Section 6.6)
├── content_assembler.py    # Deterministic content assembly for creates (Section 7.3)
├── undo.py                 # Compensating actions + conflict resolution (Section 9.1.1)
├── evidence.py             # Audit trail integration with ledger
└── schema.py               # All dataclasses (Section 4)

features/cognitive_bridge_parts/
├── file_agent.py           # UNCHANGED — existing read-only agent
└── file_agent_writer.py    # Bridge: action_dispatcher → file_service (routing only)
```

---

## 14. Implementation Phases

### Phase 1: Sandbox + Single File Create/Edit + Confirmation Protocol
- `schema.py` — All dataclasses with full-graph preconditions, raw path storage, encoding fields, content elevation
- `sandbox.py` — Full Windows-aware path validation (Section 5), drive-relative rejection, filename sanitization (6.8), double-resolution
- `permissions.py` — File classification (extension + content smuggling + classification persistence)
- `encoding.py` — Encoding detection via charset-normalizer, BOM handling (Section 6.4)
- `safe_io.py` — safe_create, safe_write_file with link-type prohibition (Section 6.7)
- `preflight.py` — Disk space check before execution (Section 6.5)
- `request_parser.py` — Deterministic user_text → FileRequest parser (agent-owned, no response model)
- `planner.py` — Resolve FileRequest → FilePlan with fingerprints, encoding detection, content analysis, deterministic templates
- `executor.py` — Create/edit only, with preflight + double-resolution + full-graph precondition verification
- `undo.py` — Backup creation + basic compensating action journal
- `evidence.py` — Full audit logging from day one (all lifecycle events)
- `file_agent_writer.py` — Bridge to action dispatcher
- Single `file_op` intent registered
- Feature flag
- Install `charset-normalizer>=3.0` dependency
- **Tests:** Sandbox escape (all vectors from Section 5.1 including drive-relative), filename sanitization (trailing dots/spaces rejected), double-resolution attack detection, content smuggling (script in .md), encoding detection (UTF-16 file edit preserves encoding, low-confidence encoding refused, BOM preserved), preflight disk space check (fails before mutation on full disk), link-type prohibition in safe_create, precondition verification (target + source + parent), plan expiry, fingerprint tampering, deterministic template rendering, create/edit happy path, undo for edits, evidence trail completeness

### Phase 2: Move/Copy/Delete + Full Undo
- `safe_io.py` — Add safe_move, safe_copy with link-type prohibition and cross-volume handling
- `executor.py` — Add move, copy, delete operations (all via safe_io only)
- `undo.py` — Compensating actions for all types + conflict resolution mode (Section 9.1.1)
- Classification persistence for move/rename operations (Section 6.3)
- Trash directory management with retention cleanup
- **Tests:** Cross-volume move, link-type prohibition (refuses to create junctions), classification persistence (exe→txt stays elevated), undo conflict resolution (modified-after-edit → show diff, offer choices), trash cleanup, partial failure reporting

### Phase 3: Batch Operations
- `dir_scanner.py` — Bounded directory expansion (depth 3, 500 files, 2s timeout, skips node_modules/.git)
- `planner.py` — Multi-operation plans from organize intent with bounded scan
- `executor.py` — Batch execution with per-operation precondition checks + measured rollback (post-failure snapshots, three-way comparison)
- Elevated confirmation for config/executable files in batch
- **Tests:** Bounded scan (verify depth/time/count limits, verify skip-dirs), batch rollback with measurement, rollback-of-rollback failure (locked file), partial failure reporting with accurate state, 20-file limit after expansion

### Phase 4: Integration + Polish
- Read agent context → write planner (one-way: recently-referenced files inform suggestions)
- Evidence integration with Living Memory
- Background cleanup (journal pruning, trash purging, backup deletion)
- Health metrics: write-latency, rollback-failure rate (feed to cortex health report)
- **Tests:** End-to-end workflows, retention enforcement, cleanup scheduling, encoding edge cases (UTF-16 files, ANSI files, BOM preservation)

---

## 15. Dependencies

One new external dependency:
- `charset-normalizer>=3.0` — encoding detection for edit operations (R3 #1)

Stdlib:
- `pathlib` — path manipulation
- `shutil` — disk usage checks, fallback copy operations
- `difflib` — unified diff generation
- `sqlite3` — undo journal
- `hashlib` — SHA256 for fingerprints and preconditions
- `ctypes` — Windows API calls for path resolution and reparse point detection
- `json` — structured schema serialization
- `time` — bounded scan timing

---

## 16. Review Checklist

### Security (P0)
- [ ] Does sandbox reject all vectors from Section 5.1 threat model?
- [ ] Does sandbox reject drive-relative paths (`C:file.txt`)?
- [ ] Does sandbox REJECT (not normalize) trailing dots/spaces in filenames?
- [ ] Are paths re-resolved from scratch at execution time (never cached)?
- [ ] Does double-resolution detect junction/short-name changes between plan and execution?
- [ ] Does content smuggling analysis catch script patterns in "text" files?
- [ ] Is the confirmation message generated by deterministic template (not response model)?
- [ ] Does the executor NEVER create symlinks, junctions, or hard links?
- [ ] Does classification persist across renames (exe→txt still elevated)?

### Correctness (P1)
- [ ] Does precondition verification check ALL paths (target, source, parent)?
- [ ] Does source-path precondition catch modifications to move/copy sources after approval?
- [ ] Is the confirmation bound to plan_id + fingerprint + session?
- [ ] Does plan expiry prevent stale confirmations from executing?
- [ ] Does batch rollback MEASURE state (not guess) after each compensating action?
- [ ] Does rollback report three-way comparison (original → partial → post-rollback)?
- [ ] Does undo offer conflict resolution when file was modified (not just refuse)?
- [ ] Is the permissions matrix enforced for copy/move (not just create/edit)?
- [ ] Does encoding detection refuse edits when confidence < 0.7?
- [ ] Is BOM preserved on write for files that originally had BOM?
- [ ] Does pre-flight disk space check run BEFORE any mutation?

### Stability (P2)
- [ ] Is directory expansion bounded (depth 3, 500 files, 2 seconds)?
- [ ] Are known-heavy directories (node_modules, .git) skipped in scans?
- [ ] Does pre-flight check verify backup directory is writable?
- [ ] Does the executor use ONLY safe_* functions (never raw Path.rename/shutil.copy2)?

### Architecture (P3)
- [ ] Is the File Agent's request parser deterministic and unambiguous?
- [ ] Does the response model never generate paths, content, or operation types?
- [ ] Are all lifecycle events logged to evidence with immutable IDs?
- [ ] Are limits enforced at plan creation AND before execution?
- [ ] Is the single `file_op` intent sufficient for routing?
- [ ] Are tiered preconditions (metadata vs. hash) applied correctly by file class?
- [ ] Is the `content_elevation_reason` always shown to the user when triggered?
