# Hermes on Windows: Fixing the MSYS2 Phantom `C:\c\` Path Bug

**Author:** MilkyWay008  
**Date:** 2026-07-21  
**Affects:** Hermes Agent v0.19 and earlier — Windows Desktop App users  
**Status:** Active workaround (partial fix + guardrails)

---

## Table of Contents

1. [The Problem](#the-problem)
2. [Root Cause](#root-cause)
3. [Three-Layer Solution](#three-layer-solution)
   - [Layer 1: Partial Fix — MSYS2 Environment Variables](#layer-1-partial-fix--msys2-environment-variables)
   - [Layer 2: Guardrail — Deny-Write ACL on `C:\c`](#layer-2-guardrail--deny-write-acl-on-cc)
   - [Layer 3: Recovery — Agent Error Handling](#layer-3-recovery--agent-error-handling)
4. [Verification](#verification)
5. [Reversal](#reversal)
6. [Future Outlook](#future-outlook)

---

## The Problem

Hermes Agent runs inside a **git-bash (MSYS2)** terminal environment on Windows. This creates a persistent path-translation issue:

When the agent or a tool constructs a path using MSYS2 conventions like `/c/Projects/data.txt`, **native Windows tools** (Python's `open()`, the `write_file` tool, `FileSystem` MCP operations) interpret the path **literally** — resulting in `C:\c\Projects\data.txt` instead of the intended `C:\Projects\data.txt`.

### Symptoms

- Files "disappear" — written to a phantom `C:\c\...` directory instead of the expected location
- Agents report "path not found" or "file created" but the file isn't where expected
- A phantom `C:\c\` directory appears on the system, accumulating orphaned files
- Subagents lose work because they write to the wrong location
- Build outputs end up in unexpected places, wasting development time

### Why This Matters

Hermes Desktop for Windows uses **git-bash** as its terminal backend, which is built on **MSYS2**. MSYS2 provides a POSIX-compatible layer that includes automatic path conversion — it translates `/c/` to `C:\` when passing arguments to native Windows programs. However, this conversion only works when the path goes **through** MSYS2's argument processing. When a path string is embedded directly in a Python script, MCP tool call, or Hermes native tool (like `write_file` or `FileSystem`), no MSYS2 conversion occurs — the literal `/c/` prefix becomes a directory named `c` under the root `C:\`.

This issue has affected Hermes Agent v0.19 and all prior versions on Windows. Future versions may address this at the framework level, but this workaround remains applicable until then.

---

## Root Cause

The root cause is a **path-format mismatch** between two execution contexts:

| Context | Path Format | Resolution | Works? |
|---------|------------|------------|--------|
| MSYS2 terminal (bash, mkdir, git) | `/c/Projects/...` | Mount table resolves to `C:\Projects\...` | ✅ |
| Hermes native tools (write_file, FileSystem, read_file) | `/c/Projects/...` | Interpreted literally as `C:\c\Projects\...` | ❌ |
| Native Windows tools (python.exe, cmd.exe) | `/c/Projects/...` | Interpreted literally as `C:\c\Projects\...` | ❌ |

The bug occurs when a POSIX-style path (`/c/...`) is passed to a non-MSYS2 context. The MSYS2 argument-conversion layer (`MSYS2_ARG_CONV_EXCL`) only processes arguments passed to subprocesses at the shell level — it does not affect how Python's `os.path` module or Windows API calls interpret strings.

---

## Three-Layer Solution

We implement **three independent layers** of defense. Each layer catches failures the previous one misses.

### Layer 1: Partial Fix — MSYS2 Environment Variables

**What it does:** Prevents MSYS2 from converting `/c/...` paths when spawning native Windows subprocesses from bash.

**Files to edit:** `~/.bashrc` (or equivalent git-bash profile)

**Add these lines:**

```bash
# ===== MSYS2 PATH FIX =====
# Prevent MSYS2 from mangling /c/... paths into C:\c\...
# Without this, /c/Projects can be misinterpreted as C:\c\Projects
# causing phantom directories and lost files.
export MSYS2_ARG_CONV_EXCL="*"
export MSYS_NO_PATHCONV=1
```

**What these do:**

| Variable | Value | Effect |
|----------|-------|--------|
| `MSYS2_ARG_CONV_EXCL` | `*` | Exclude ALL arguments from MSYS2's path conversion (wildcard matches everything) |
| `MSYS_NO_PATHCONV` | `1` | Disable MSYS2's automatic path conversion entirely |

**Why "partial":** This only affects how MSYS2 processes arguments when spawning subprocesses. MSYS2-native binaries (bash, mkdir, git, cp, etc.) continue to resolve `/c/...` correctly through their own mount table. However, this does **not** fix the case where a path string is embedded directly in Python code or a Hermes tool argument — those contexts never went through MSYS2 argument processing to begin with.

**After editing**, restart git-bash or source the profile:

```bash
source ~/.bashrc
```

Always create a backup before editing:

```bash
cp ~/.bashrc ~/.bashrc.bak
```

---

### Layer 2: Guardrail — Deny-Write ACL on `C:\c`

**What it does:** Pre-creates the `C:\c` directory with a **Deny Write** access control entry. If Layer 1 fails or a path slips through, any attempt to write into `C:\c\...` is blocked with an explicit **Access Denied** error instead of silently creating a phantom directory.

**Implementation (run as Administrator in PowerShell):**

```powershell
# Create the phantom directory if it doesn't exist
if (-not (Test-Path "C:\c")) {
    New-Item -Path "C:\c" -ItemType Directory -Force
}

# Set Deny Write + Deny Delete Child for Everyone
icacls C:\c /deny "Everyone:(W,DC)"

# Verify
icacls C:\c
```

**What the ACL does:**

| Permission | Effect |
|-----------|--------|
| `W` (Write) | Blocks creating new files/subdirectories inside `C:\c` |
| `DC` (Delete Child) | Blocks deleting contained items |

**Test that it works:**

```powershell
# This should fail with "Access Denied"
New-Item -Path "C:\c\test-block.txt" -ItemType File
```

**Why this works:** The Access Denied error is impossible to ignore. When a tool or subagent gets this error, it's a clear signal that something is wrong with the path. Compare this to the original behavior — silent file creation in the wrong location — which could go unnoticed for hours.

---

### Layer 3: Recovery — Agent Error Handling

**What it does:** Trains the agent (via persistent memory — MEMORY.md or SOUL.md) to recognize the `Access Denied` signal and take corrective action.

**Memory/SOUL entry to add:**

```
TRIGGER (MSYS2 phantom path): When ANY tool or command errors with
"Access Denied", "Permission denied", "No such file or directory",
or "file not found" on a path containing `C:\c\...`, `C:/c/...`,
or `/c/c/...` — this is the MSYS2 path conversion bug.

IMMEDIATE FIX:
1. Replace the MSYS2-style prefix `/c/...` or the phantom `C:\c\...`
   with the correct Windows-native prefix `C:\` or `C:/`
2. Retry the operation with the corrected path
3. Do NOT manually delete or modify the C:\c directory — it is the
   guardrail and must remain in place

The root cause is passing a POSIX /c/ path to a native Windows tool
or Hermes native tool (write_file, FileSystem, read_file) that doesn't
understand MSYS2 mount table resolution. The Deny ACL on C:\c exists
specifically to catch this with a visible error.

NEXT ACTION: Always retry with a C:/ path after seeing this error.
```

**Agent workflow when the error fires:**

```
Tool call → "Access Denied on C:\c\Projects\..."
         → Agent memory triggers recognition
         → Agent identifies: path contains C:\c\ — MSYS2 phantom bug
         → Agent rewrites path: "/c/Projects/..." → "C:/Projects/..."
         → Agent retries with corrected path
         → Operation succeeds
```

---

## Verification

After implementing all three layers, run these tests:

### Test 1: Terminal path resolution (Layer 1)

```bash
mkdir -p /c/Projects/msys2-verify
echo "test" > /c/Projects/msys2-verify/test.txt
ls -la /c/Projects/msys2-verify/
# Should show test.txt
```

### Test 2: Cleanup test artifacts

```bash
rm -rf /c/Projects/msys2-verify
```

### Test 3: Phantom path is blocked (Layer 2)

```powershell
# Should fail: "Access Denied"
New-Item -Path "C:\c\verify-test.txt" -ItemType File
```

### Test 4: No phantom files exist

```powershell
Get-ChildItem "C:\c" -Force
# Should return empty (or nothing)
```

---

## Reversal

To undo Layer 1 (env vars), remove the two lines from `~/.bashrc` and restart git-bash.

To undo Layer 2 (guardrail ACL):

```powershell
icacls C:\c /remove:d Everyone
icacls C:\c /grant "Everyone:(W)"
# Or remove the directory entirely (after confirming it's empty):
rmdir C:\c
```

To undo Layer 3 (agent memory), remove or comment out the trigger entry from MEMORY.md / SOUL.md.

---

## Future Outlook

This issue originates from Hermes Agent's use of **git-bash (MSYS2)** as the Windows terminal backend. In a future version of Hermes, the framework may:

- Switch to a native Windows terminal (PowerShell Core or Windows Terminal) that doesn't use MSYS2 path conversion
- Automatically normalize `/c/...` paths to `C:/...` before passing them to native tools
- Provide a built-in path validator that catches MSYS2 paths in tool arguments

Until then, the three-layer solution documented here provides a reliable defense against the `C:\c\` phantom path bug. The Deny-Write guardrail (Layer 2) in particular is a **system-level protection** — it works regardless of whether the agent remembers the fix, because the operating system enforces it.

---

## Acknowledgments

This fix was developed through iterative debugging on a Windows 10 system running Hermes Agent v0.19. The three-layer approach (partial fix + guardrail + recovery) was designed after experiencing repeated file loss across multiple development sessions. The guardrail pattern — "make the system block it, don't just write a note to self" — was the key insight that made the solution robust.
