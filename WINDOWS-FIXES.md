# Windows fixes for `/watch`

This fork carries three fixes that make [`claude-video`](https://github.com/bradautomates/claude-video)
run on Windows 11 with App Control (WDAC / Smart App Control) enabled, and on
ffmpeg 9. Everything else is upstream, unchanged.

Nothing here is Windows-only in spirit: the ffmpeg 9 fix affects every platform
that already shipped ffmpeg 9, and each fix falls back instead of aborting, so
none of them change behaviour on a machine where the original code worked.

| # | Symptom you actually see | Root cause | Files |
|---|---|---|---|
| 1 | `/watch` dies during frame extraction: `Unrecognized option 'vsync'` | ffmpeg 9 removed `-vsync`; the replacement is `-fps_mode` | `skills/watch/scripts/frames.py` |
| 2 | Run aborts with `ffprobe failed`, even though ffmpeg itself works | App Control blocks `ffprobe.exe` while allowing `ffmpeg.exe` | `skills/watch/scripts/frames.py`, `skills/watch/scripts/whisper.py` |
| 3 | Run dies inside `subprocess` with `OSError WinError 4551` right after the "yt-dlp found" check passes | App Control blocks the unsigned `yt-dlp.exe` shim at *execution* time, while `shutil.which()` still returns its path | `skills/watch/scripts/download.py` |

Plus one false alarm removed:

- **Fake "config file is world-readable (644)" warning.** POSIX mode bits are
  emulated on Windows: `stat` always reports `0o666` and `chmod` is a no-op, so
  the `mode & 0o044` check was a guaranteed false positive on every Windows
  machine. It now reads the real NTFS ACL via `Get-Acl` and compares against
  well-known SIDs, and `hooks/scripts/check-setup.sh` skips the `stat`-based
  check under MSYS / Git Bash for the same reason.
  (`skills/watch/scripts/setup.py`, `hooks/scripts/check-setup.sh`)

## The shape of the bug, if you are porting something else

Two of the three are the same defect: **a presence check that is not an
execution check.** `shutil.which("yt-dlp")` and `shutil.which("ffprobe")` both
return a valid path for a binary that App Control will refuse to launch. The
guard passes, the run dies later, and the error surfaces from deep inside
`subprocess` where it reads like a bug in the video pipeline.

The fix in both cases is to treat "found" as a hypothesis and keep a fallback
path: `python -m yt_dlp` when the `yt-dlp.exe` shim is blocked, and parsing the
`ffmpeg -i` banner when `ffprobe` is blocked.

## Install this fork

Claude Code:

```
/plugin marketplace add vgrosetti-maker/claude-video
/plugin install watch@claude-video
```

Codex, Cursor, Copilot, Gemini CLI, and other Agent Skills hosts:

```bash
npx skills add vgrosetti-maker/claude-video -g
```

Note: this fork keeps the upstream marketplace name (`claude-video`), so remove
the upstream marketplace first if you have it added.

## Upstream status

- Fix 1 is submitted upstream as [PR #125](https://github.com/bradautomates/claude-video/pull/125), open and unreviewed.
- Fixes 2 and 3 are not submitted.

Use the fork if you need `/watch` working on Windows today. If upstream merges,
this fork goes away.

MIT, same as upstream. Original project and credit: [Bradley Bonanno](https://github.com/bradautomates/claude-video).
