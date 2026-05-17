# Parallel Subagent Bug Analysis

A technique for finding bugs in unfamiliar codebases by delegating analysis to parallel subagents, each focused on a different concern.

## When to Use

- User reports crashes with vague symptoms ("it crashes when I press X")
- You need to analyze a codebase you haven't seen before
- Multiple potential failure modes to investigate independently
- User wants code quality + crash fixes simultaneously

## Pattern

```python
# Step 1: Identify the independent concern areas to analyze
# Each subagent gets the SAME context but a DIFFERENT focus

tasks = [
    {
        "goal": "Analyze the [CAPTURE/CRASH/AREA] path in [APP]. "
               "Read ALL source files and identify EVERY bug, crash, "
               "and issue. Focus especially on: [specific concerns]. "
               "Return a list of EVERY bug you find with file path and line number.",
        "context": f"Full source code is at {project_path}/",
        "toolsets": ["file"],
    },
    {
        "goal": "Analyze the [GALLERY/OTHER/AREA] path in [APP]. "
               "Focus especially on: [specific concerns]. "
               "Return a list of EVERY bug you find with file path and line number.",
        "context": f"Full source code is at {project_path}/",
        "toolsets": ["file"],
    },
]

# Step 2: Run parallel analysis (up to 3 concurrent)
results = delegate_task(tasks=tasks)

# Step 3: Consolidate findings — each subagent returns a comprehensive bug list
# Prioritize: CRASH bugs > MODERATE bugs > MINOR issues
# Fix in order of crash likelihood

# Step 4: Fix all bugs, rebuild, deliver working product
```

## Key Principles

### 1. Overlapping Analysis Is Intentional
Give each subagent the SAME set of files but DIFFERENT focus areas. Some bugs will be found by both — that's confirmation. Each agent catches things the other missed.

### 2. Be Specific About Focus
Don't say "find bugs" — say:
- "Focus on lifecycle management, file I/O, and threading"
- "Focus on URI handling, content resolver, permission checks"
- "Focus on camera API usage, executor threading, Compose state"

### 3. Require File:Line References
Tell subagents to return bugs with exact file paths and line numbers. This makes consolidation and fixing much faster.

### 4. Categorize by Severity
Subagents return bugs as:
- **CRITICAL / CRASH-CAUSING** — will crash at runtime
- **MODERATE** — non-crash but real logic/data failures
- **MINOR** — code smells, dead code, unused imports

### 5. Fix in Priority Order
CRASH causes first, then moderate bugs, then minor cleanup.

### 6. Surgical Fix Phase (Critical)

After the analysis is complete, apply fixes **surgically**:

1. **Identify the minimal change** — what exact lines/expressions cause each crash?
2. **Use `patch`** for targeted edits. Only rewrite a file if the changes touch >50% of its lines.
3. **Preserve everything else** — imports, class structure, method signatures, variable names, the composable tree shape. If it wasn't causing the crash, don't touch it.
4. **Verify after each batch** — rebuild and confirm the app opens before fixing the next crash.

**Wrong approach:**
- Getting a bug list then rewriting all files at once
- "Cleaning up" code while fixing crashes
- Changing working patterns (e.g., `ctx as LifecycleOwner` → `LocalLifecycleOwner.current`) while fixing unrelated crash paths
- Adding new features or "improvements" during crash fixes

**Right approach:**
```python
# After subagents return bug list:
# For each unique crash bug:
#   1. patch(path, old_string=..., new_string=...)   # 2 lines changed
#   2. ./gradlew assembleDebug --no-daemon           # verify
#   3. Only then move to next bug
```

### 7. Crash Fix Verification Protocol

When the user says an existing (working) app crashes on specific actions:

1. **Build the original code** as-is. Confirm the app opens and the crash path reproduces.
2. **Identify crash causes** via parallel subagents.
3. **Fix ONE crash path at a time** — apply the minimal change, rebuild, verify app still opens.
4. **If the app stops opening** after a fix batch, REVERT IMMEDIATELY. The crash is not in the code you changed — look at what else you touched.
5. **DO NOT** modernize working patterns (`runBlocking` → `lifecycleScope.launch`, unsafe cast → `LocalLifecycleOwner.current`, etc.) during crash fixes. These are orthogonal concerns with their own risk of breaking launch.
6. **Preserve original import style, class structure, and file layout.** Using different imports or restructuring the class can introduce subtle runtime issues.

## Pitfalls

- Subagents can produce verbose output. The summary field in the result is auto-truncated to ~8k chars which is usually enough.
- Subagents don't have access to external context (past conversations, user preferences). Include ALL relevant context in the `context` field.
- Results are self-reported, not verified. After fixing, always rebuild to confirm.
- Max 3 concurrent subagents per delegation call.
