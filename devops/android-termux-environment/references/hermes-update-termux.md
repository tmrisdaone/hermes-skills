# Updating Hermes on Android/Termux

## Workflow

```bash
cd ~/hermes-agent
source venv/bin/activate         # or .venv/bin/activate
git pull origin main
pip install --no-deps -e .       # MUST use --no-deps — psutil won't build on Android
rm -f ~/.hermes/.update_check    # clear stale "N commits behind" cache
hermes --version                 # verify: should show "Up to date"
```

## Why --no-deps?

`psutil` (an optional dependency) explicitly refuses to build on Android:

```
platform android is not supported
```

This is not a real problem — psutil is used in ~11 files but always inside `try/except` blocks. The memory monitor (`gateway/memory_monitor.py`) tries `resource.getrusage()` from stdlib first, only falls back to psutil when `resource` isn't available (Windows). Android has `resource`, so everything works fine without psutil.

Using `--no-deps` skips all dependency resolution and just installs the local checkout as an editable package. All existing dependencies remain in the venv from the previous install.

## Stale Update Cache

After `git pull` + `pip install`, `hermes --version` may still report:
```
Update available: 42 commits behind — run 'hermes update'
```

This is a cached result in `~/.hermes/.update_check`. The version check code caches the "commits behind" count so it doesn't run `git fetch` every CLI invocation. After a successful update, deleting this file forces a fresh check.

Verify there are actually 0 commits behind:
```bash
git rev-list --count HEAD..origin/main   # should print 0
```

## Verifying

```bash
hermes --version
# → Hermes Agent v0.14.0 (2026.5.16)
# → Up to date
```

The `hermes` binary lives at `venv/bin/hermes` (or `.venv/bin/hermes`). After the editable install it points directly at the local checkout.
