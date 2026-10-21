# ClCoChBa: Claude Code Chat Backup

## 0. Purpose and terms

Back up the user's Claude Code chats to a git repository, keeping only the substantive conversation: what the user said, and what the AI said back to the user.

Terms used throughout:

- **You**: the AI agent executing this prompt.
- **This session**: the Claude Code session in which you are executing this prompt.
- **REPO**: the local clone of the git repository that holds the backup.
- **SCRATCH**: your temporary working folder (§1.2).
- **SOURCE**: a Claude Code data folder on this machine (by default, `~/.claude` of a given environment), which holds the chat transcripts.
- **Chat**: one Claude Code session. **Chat ID**: its session ID.
- **Entry**: one item in a chat's `transcript` list (§7.3).

This prompt cannot foresee everything. Where it is silent or ambiguous, use your best judgment of intent, in good faith; when the choice matters, ask.

## 1. Ground rules

These apply at every step.

### 1.1 Diligence and honesty

- Do all of the work yourself; do not delegate to subagents.
- Never fabricate, guess silently, or paper over gaps. If something is uncertain, say so; if it matters, ask.
- Claude Code's storage layout and transcript format change between versions. §6.1 describes the concepts; §6.2 records what was observed in one version. Verify against the actual data rather than assuming, and consult your own knowledge and the Claude Code documentation when something does not match.

### 1.2 File system

You MUST NOT create, edit, rename, delete, or otherwise touch anything on the user's file systems, directly or indirectly, except for these four exceptions:

1. Caches and similar files that are created automatically (e.g., `__pycache__`), and residue that your harness creates on its own. You need not clean these up.
2. SCRATCH: exactly one folder that you create in the system temp location, named `clcochba-` followed by `secrets.token_hex(16)`. You may work freely inside it. You MUST delete it before every response to the user, i.e., whenever you return control to the user, whether the task is done, you have a question, you hit a blocker, or otherwise. If you need a scratch folder again afterwards, create a new one.
3. The config file `~/.clcochba` (§3), and only after the user explicitly grants permission for that specific write.
4. REPO: `git fetch` as allowed in §4, and otherwise only as described in §11, after the user approves.

Reading is allowed where this task needs it, with these limits:

- You MUST NOT read credential files, including but not necessarily limited to `.credentials.json` inside SOURCEs.
- From Claude Code settings files, read only the single key described in §10, e.g., by searching for that line, not the rest of the file.

You MUST NOT start, stop, or otherwise change the state of WSL distributions, virtual machines, or containers without the user's permission. Note that on Windows, accessing a stopped WSL distribution's files starts it.

### 1.3 Tools

- Your main tools are Python 3 (§2) and the system shell(s).
- You MUST NOT install anything: no packages, programs, Python modules, or otherwise.
- Anything that is already installed MAY be used. You MAY probe the system for common tools and libraries, e.g., with `command -v` or by trying an import.

### 1.4 Git

- You MUST NOT change any git configuration: global, system, or repository. For one-off settings, use per-command `git -c <key>=<value>`.
- Commit only what the user approved (§11).
- You MUST NOT push unless the user explicitly grants permission for that push. After committing, ask the user whether they would like you to push.

## 2. Preflight

1. Find a usable Python 3.8 or newer that you can run with `-c`. Try `python3`, then `python`, and on Windows also `py -3`; confirm with `-c "import sys; print(sys.version)"`. If none works, stop and ask the user to install Python 3. You MUST NOT proceed further without it.
2. Confirm that `git --version` works. If it does not, stop and ask the user to install git.

## 3. Config file: `~/.clcochba`

`~` is the home folder of the environment you are running in. WSL and Windows have different home folders, so each keeps its own config.

The file contains JSON (UTF-8, 4-space indentation):

```json
{
    "repo": "<absolute path to REPO, in this environment's native form>",
    "sources": [
        {
            "path": "<absolute path to a SOURCE, in this environment's native form>",
            "label": "<the SOURCE's label (§5), e.g., wsl:Ubuntu>"
        }
    ],
    "suppress": {
        "cleanup_period_suggestion": false,
        "image_tool_suggestion": false
    }
}
```

- Every key is optional. When you rewrite the file, preserve keys you do not recognize.
- `sources`: if present, use exactly these SOURCEs, with these labels, instead of discovering them and asking (§5). Still verify that each exists; if one does not, tell the user. For a SOURCE that is reachable only through a command (§5), e.g., another WSL distribution from inside WSL, `path` is its path inside that environment, which `label` identifies.
- `suppress`: `true` hides the corresponding suggestion (§10).

If the file exists, parse it. At the very top of your first response, state what it contains, that you are using it for this session, and that the user may ask to change it. If it cannot be parsed, stop and ask.

If the file does not exist or lacks `repo`, ask the user for the path to a local clone of their backup git repository, validate it (§4), and then ask whether they would like it remembered in `~/.clcochba`, so they need not provide it again. Create or write the file only if the user explicitly says yes. The same applies to:

- `sources`, after §5;
- suppressions, whenever the user asks, in natural language, to stop showing a suggestion: stop showing it for this session, and offer to remember it.

## 4. Validate REPO

Paths:

- Paths may contain spaces and other special characters. Always pass a path as a single argument, e.g., in a Python `subprocess` argument list or properly quoted in the shell; never let it be split.
- If the user gives a path in another environment's form (e.g., `C:\...` while you are in WSL, or `/mnt/c/...` while you are on Windows), convert it (e.g., with `wslpath`), and confirm the converted path with the user.

All of these checks MUST pass; if one fails, explain and ask:

1. `git -C <path> rev-parse --show-toplevel` succeeds and refers to the same folder as `<path>` (compare after resolving symlinks, and case-insensitively on case-insensitive file systems). A subfolder of a repository does not count.
2. A branch is checked out (not a detached HEAD); in an empty repository, the unborn initial branch counts. Your commits go to that branch.
3. The working tree is clean: `git status --porcelain` prints nothing. Otherwise, stop and ask, so the user's uncommitted changes never get mixed into backup commits.
4. If the branch has an upstream, you MAY run `git fetch`, non-interactively, so that it cannot hang on or open a credential prompt (e.g., with the environment variables `GIT_TERMINAL_PROMPT=0` and `GCM_INTERACTIVE=never`); if fetching fails (e.g., no network or no stored credentials), say so and continue. If the branch is behind its upstream, stop and tell the user, since the diff would be computed against a stale backup; do not pull unless the user asks.

An empty repository (no commits, no `raw/`) is valid; every chat is then new.

## 5. Discover SOURCEs

Find every Claude Code data folder on this machine that the user wants backed up. Remote machines, SSH hosts, and cloud or web sessions (e.g., claude.ai/code) are out of scope.

If the config sets `sources`, use those instead (§3), and skip steps 1 to 4 below and their questions.

1. Current environment: the SOURCE is `$CLAUDE_CONFIG_DIR` if that variable is set, else `~/.claude`. It is always included.
2. Determine which kind of environment you are in: native Windows, WSL, macOS, or other Linux.
3. Windows and WSL:
   - On native Windows: run `wsl -l -v` to learn whether WSL is installed and which distributions exist and are running. Its output is often UTF-16LE; decode it accordingly, or set the environment variable `WSL_UTF8=1` for that command. If any distribution exists, ask whether the user uses Claude Code inside it and would like it included. A distribution's files are reachable at `\\wsl.localhost\<distribution>\` (older Windows: `\\wsl$\<distribution>\`); reaching a stopped distribution starts it, so ask first (§1.2).
   - On WSL: the Windows side's SOURCE is usually `.claude` in the Windows user profile, e.g., `/mnt/c/Users/<user>/.claude`; find the profile with, e.g., `cmd.exe /c echo %USERPROFILE%` and `wslpath`. If it exists, ask whether to include it. Also run `wsl.exe -l -v` (same encoding note); if other distributions exist, ask about them too, and read any that the user includes through, e.g., `wsl.exe -d <distribution> -- …`, which starts a stopped distribution (§1.2).
4. Virtual machines and containers: probe for the existence (only) of common tools, including but not necessarily limited to Docker, Podman, Lima (`limactl`), OrbStack (`orb`), Multipass, Vagrant, VirtualBox (`VBoxManage`), libvirt (`virsh`), Distrobox, and Toolbox, plus their Windows `.exe` counterparts when in WSL. If any is found, ask whether the user runs Claude Code inside it and, if so, how to reach its SOURCE. Do not start anything (§1.2).

Ask all of these questions in a single message.

Label each SOURCE with its environment, e.g., `windows`, `wsl:<distribution>`, `macos`, `linux`. The label is used in §7.1 and in the report.

## 6. Read chats

### 6.1 Concepts

A SOURCE stores each chat as an append-only transcript file with one JSON record per line, named by chat ID and grouped into per-project folders. Records include, but are not necessarily limited to:

- messages the user sent;
- the AI's replies, possibly split across several records, each holding content blocks such as user-facing text, thinking or progress summaries, and tool calls;
- tool results (stored as user-role records);
- context the harness injects (system reminders, environment information, instructions files, skill bodies, file snapshots, and so on);
- harness events and metadata (titles, modes, costs, copies of API requests, compaction markers, command runs, queue operations, and so on).

Files stored next to transcripts are not chats and are not backed up, including but not necessarily limited to subagent transcripts, saved tool outputs, and memory files.

A per-project folder's name is a lossy encoding of the project path. Take the project path from the working directory recorded in the records, not from the folder name.

### 6.2 Observed in Claude Code v2.1.304 (October 2026)

This is a starting point, not a specification; verify it against the actual data.

- Transcripts are at `<SOURCE>/projects/<encoded project path>/<chat ID>.jsonl`. Subagent transcripts are under `<chat ID>/subagents/` and saved tool outputs under `<chat ID>/tool-results/`; some project folders also have a `memory/` folder.
- Each record has a `type`. `user` and `assistant` records carry `message.content`: a string, or a list of blocks typed `text`, `image`, `thinking`, `tool_use`, or `tool_result`. Other types include `attachment`, `system`, `queue-operation`, `ai-title`, `api-request`, `api-request-blob`, `mode`, `permission-mode`, `last-prompt`, and `cost-state`.
- Message records carry `cwd`, `timestamp` (ISO 8601, UTC), `sessionId`, and `isSidechain`. Injected user records have `isMeta: true`; tool results have `toolUseResult`; compaction summaries have `isCompactSummary`. User records may carry `origin.kind`, e.g., `human`, `task-notification`, `peer`, or `auto-continuation`.
- Messages that the user sent while the AI was working are stored once each, as `attachment` records whose attachment type is `queued_command` with `origin.kind` `human`. Messages sent after a turn ended are regular `user` records.
- Slash commands appear as user records containing `<command-name>…</command-name>` and `<command-args>…</command-args>` markup; their output appears as `<local-command-stdout>…`. Some, e.g., `/compact` with or without instructions, appear instead as plain text starting with `/`, on user records that lack the `origin` that typed messages carry.
- Interrupt markers, e.g., `[Request interrupted by user]` and `[Request interrupted by user for tool use]`, are user records that also lack `origin`; they are entries nonetheless (§6.3).
- `api-request-blob` records hold copies of messages; never take entries from them, or entries would be duplicated.
- Raw thinking is stored empty. Non-empty `thinking` blocks are the short, first-person progress summaries that the user sees while the AI works.
- Error replies carry `isApiErrorMessage`; placeholder replies have the model `<synthetic>`.
- `ai-title` records hold Claude Code's auto-generated chat title, in `aiTitle`. A transcript may contain several; the last one in file order is current.
- This session's chat ID is in the environment variable `CLAUDE_CODE_SESSION_ID`.

### 6.3 What counts as an entry

Process records in file order, which keeps entries from branches the user rewound past as well. Skip records that belong to subagents or sidechains.

User entries (`"from": "user"`) include, but are not necessarily limited to:

- everything the user typed or pasted, whether sent normally or queued while the AI was working, each exactly once; pasted blocks are kept verbatim, including their wrapper markup;
- images that the user attached or pasted;
- interrupt markers, e.g., `[Request interrupted by user]`.

AI entries (`"from": "ai"`) include, but are not necessarily limited to:

- all user-facing text: intermediate updates between tool calls and the final response of each turn;
- user-visible progress summaries (§6.2). If a version ever stores raw chain-of-thought, exclude it, and tell the user.

Excluded, including but not necessarily limited to:

- tool calls and their inputs, tool results, file edits and diffs, web search and fetch output;
- harness-injected context and notifications: system reminders, environment, instructions, and skill content, background-task notifications, messages from other sessions or subagents, auto-continuation messages;
- compaction summaries;
- slash commands and other commands run directly through the harness (e.g., `!` shell commands), however they are recorded, and their output;
- error and placeholder replies;
- copies of messages held in metadata records.

When unsure whether something is user-facing, sample the records, and ask if it matters.

### 6.4 Which chats to back up

- Always skip this session. Identify it by `CLAUDE_CODE_SESSION_ID` if available; otherwise, it is the most recently modified transcript in the current working directory's project that contains this prompt; if still uncertain, ask.
- Skip chats with no AI entries, e.g., sessions in which the user only ran `/model` and `/exit`.
- Other chats may still be in progress. Back up what exists now; the next sync picks up the rest.
- Check that each transcript's file name matches the chat ID recorded inside it; report any mismatch.
- If a chat appears to duplicate another chat's content (e.g., a resumed or forked session that copied history), report it rather than silently deduplicating.

## 7. Output format

### 7.1 `project_dir`

Start from the working directory recorded on the chat's first record that has one, and apply these steps in order:

1. Replace every `\` with `/`.
2. If the SOURCE is a WSL distribution and the path is `/mnt/<letter>` or starts with `/mnt/<letter>/`, where `<letter>` is a single ASCII letter, rewrite it to `<letter>:/…`. (`/mnt` is WSL's default drive mount root; if the distribution's `/etc/wsl.conf` sets a different `automount` `root`, use that root instead.)
3. Normalize lexically: collapse repeated `/` (except exactly two leading slashes, which mark a UNC path and stay), remove `.` segments, resolve `..` segments, and remove any trailing `/` unless the path is a root (`/` or `<letter>:/`). In Python, for a path that starts with `<letter>:`, this is `path[:2] + posixpath.normpath("/" + path[2:].lstrip("/"))`; otherwise, `posixpath.normpath(path)`.
4. Lowercase the whole path (in Python, `str.lower()`).

| Recorded working directory | SOURCE | `project_dir` |
|---|---|---|
| `C:\Users\Alice\Code\MyProj` | `windows` | `c:/users/alice/code/myproj` |
| `/mnt/c/Users/Alice/My Files/./x/` | `wsl:Ubuntu` | `c:/users/alice/my files/x` |
| `/Users/alice/proj/../Proj2` | `macos` | `/users/alice/proj2` |

Lowercasing can merge distinct folders on case-sensitive file systems; this is accepted.

### 7.2 Location in REPO

Each chat is stored at:

`REPO/raw/<segments>/@chats/<YYYY>/<MM>/<DD>/<chat ID>.json`

- `<segments>`: `project_dir` split on `/`, with empty parts dropped and a drive letter's colon removed. For example, `c:/users/alice` gives `c`, `users`, `alice`; `/home/alice` gives `home`, `alice`; `//server/share/x` gives `server`, `share`, `x`.
- Each segment is made safe on every operating system, in this order:
  1. Replace each of `<` `>` `:` `"` `\` `|` `?` `*` and each control character U+0000 to U+001F with `_`.
  2. Replace each character of the run of spaces and `.` at the end of the segment, if any, with `_`; e.g., `a. .` gives `a___`.
  3. If the part before the first `.` (the whole segment, if it has no `.`), ignoring trailing spaces, equals, case-insensitively, a Windows reserved name, append `_` to that part; e.g., `con` gives `con_`, and `con.txt` gives `con_.txt`. The reserved names are: `CON`, `PRN`, `AUX`, `NUL`, `CONIN$`, `CONOUT$`, `COM0` to `COM9`, `COM¹`, `COM²`, `COM³`, `LPT0` to `LPT9`, `LPT¹`, `LPT²`, `LPT³`.
  4. If the segment equals, case-insensitively, `.git` or `git~1`, which git refuses as folder names, append `_`.
- `@chats` separates the project path from the date folders, so a project subfolder can never be mistaken for a date folder.
- `<YYYY>/<MM>/<DD>`: the UTC date of the earliest timestamp on any of the chat's message records, zero-padded.
- Folder paths are for browsing only. The `project_dir` inside the file is authoritative, and chats are always matched by chat ID, never by path.
- If a resulting path, prefixed with REPO, would exceed 260 characters (Windows' default path length limit), warn the user in the report.

### 7.3 File contents

```json
{
    "project_dir": "c:/users/alice/code/myproj",
    "title": "…",
    "transcript": [
        {
            "from": "user",
            "text": "…",
            "images": [
                "img/0123456789abcdef0123456789abcdef.jpg"
            ]
        },
        {
            "from": "ai",
            "text": "…"
        }
    ]
}
```

Formatting is exact, so that reruns produce identical bytes:

- UTF-8 without a byte order mark; non-ASCII characters written as-is; 4-space indentation; keys in the order shown; LF line endings; exactly one trailing newline. In Python: `json.dumps(doc, indent=4, ensure_ascii=False) + "\n"`, written with `newline="\n"`.
- `title`: the chat's title. Use a name the user gave the chat, if the transcript records one; otherwise, Claude Code's auto-generated title, the last one in file order. Omit the key if there is neither.
- One entry per user message. One entry per AI text block or progress summary; do not merge consecutive entries.
- If a user message has several text parts, join them with `\n`.
- `text`: strip leading and trailing whitespace; keep everything else verbatim, except that any unpaired UTF-16 surrogate, which cannot be encoded as UTF-8, is replaced with U+FFFD. Drop entries left with empty text and no images.
- `images`: present only when non-empty, in the order the images appear in the message. Paths are relative to REPO and use `/`.

### 7.4 Images

- Images are stored in `REPO/img/`. A file's name is the first 32 hexadecimal digits (lowercase) of the SHA-256 of the original image bytes, plus the stored file's extension, e.g., `.jpg`. Identical images are stored once.
- Convert every image, JPEGs included, to JPEG at quality 80, scaled to fit within 1920×1920 with the aspect ratio kept (never upscaled), with transparency flattened onto white, and EXIF orientation applied and then all metadata, such as location, stripped.
- Animated images (e.g., animated GIF, WebP, or PNG) are the exception: they are not converted.
- Use whatever is already available, including but not necessarily limited to: Pillow in an available Python, ImageMagick (`magick`, or `convert` in version 6), GraphicsMagick (`gm`), libvips (`vips`), ffmpeg, macOS's `sips`, and Windows PowerShell with .NET's `System.Drawing`. If nothing works, store the originals unconverted, and see §10.
- From every image stored unconverted, strip the metadata without re-encoding the image, if `exiftool` or a similar tool is available, including but not necessarily limited to `exiv2` and, for JPEGs only, `jpegtran`. Make sure the image still displays upright, e.g., by keeping the orientation tag, as in `exiftool -all= -tagsfromfile @ -Orientation`. If no such tool is available, store the image with its metadata, and see §10.
- An unconverted image's extension follows its media type: `.png`, `.jpg`, `.gif`, or `.webp` for `image/png`, `image/jpeg`, `image/gif`, or `image/webp`; for any other type, derive it from the media type's subtype.
- If `REPO/img/` already holds a file whose name starts with the same 32 hexadecimal digits, whatever its extension, reference that file as-is; never convert it again or overwrite it.
- Images that a transcript references only by path, rather than embedding, cannot be backed up; count them and mention it in the report.

## 8. Build the candidate in SCRATCH

Write the complete candidate backup (its `raw/` and `img/`) into SCRATCH, never directly into REPO. Keep your scripts in SCRATCH as well.

## 9. Compare with REPO

1. Index REPO's chats by chat ID: every `REPO/raw/**/*.json`, where the chat ID is the file name without `.json`. Index SCRATCH the same way. If a chat ID appears more than once in REPO, report it. If a chat ID appears in more than one SOURCE (e.g., because a data folder was copied between environments), report it and ask which to use.
2. Classify each candidate chat:
   - **New**: not in REPO.
   - **Identical**: same path and same bytes, treating CRLF and LF as equal (git may convert line endings on Windows checkouts).
   - **Non-substantive**: different path or bytes, but the same `project_dir` and the same sequence of entries when entries are compared by `from`, by `text` with every run of whitespace collapsed to a single space, and by `images`. `title` is not compared, so a difference only in `title` is non-substantive. This happens when, e.g., formatting or layout rules change.
   - **Substantive**: everything else.
3. Chats that are only in REPO ("backup-only") MUST NOT be deleted. Claude Code deletes old transcripts locally (§10), so these are expected.
4. Images: each image that a candidate chat references is either already in `REPO/img/` (reused as-is, §7.4) or new. Images that are only in REPO are kept.
5. Investigate:
   - For every substantive change, read every changed entry yourself. Transcripts are append-only, so expect only appended entries. Edits or removals of earlier entries are abnormal, e.g., from a change in rules, in the transcript format, or corruption; explain them.
   - For new chats, you need not read every entry; identify each by its title (if any), its first user message, and its counts.
   - Because of harness changes, non-determinism, and so on, a diff classified as substantive may turn out to be trivial on inspection. If so, reclassify it as non-substantive and say why.
   - REPO may be pushed to a remote, so scan the entries of new and substantively changed chats for likely secrets, including but not necessarily limited to API keys, access tokens, private keys, and passwords, using simple pattern matching. Report each finding by item number and entry position, without repeating the secret itself.

## 10. Report

Present, in this order:

1. If the config file exists and you have not yet stated its contents in this session: its contents, as described in §3.
2. **Material changes**: a numbered list of new chats and substantive changes, numbered continuously so the user can reference items. For each: the UTC date or date range, the SOURCE label, the project (the last segment of `project_dir`, or more segments if needed to disambiguate), the title, the entry counts, and for substantive changes, what changed (e.g., "2 entries appended", plus a short summary of them). For long lists, e.g., a first sync, group items by project while keeping the numbering continuous.
3. Counts of identical chats, backup-only chats, and new images, plus any warnings from earlier steps, including but not necessarily limited to those of §6.4, §7.2, §7.4, and §9, such as likely secrets.
4. **Retention suggestion**: for each SOURCE, read only the `cleanupPeriodDays` key of `<SOURCE>/settings.json`. If it is absent (the default has been 30 days) or 30 or lower, and `suppress.cleanup_period_suggestion` is not `true`, suggest raising it: Claude Code deletes transcripts older than that, and the backup can only capture what still exists. Mention that the user can ask you to stop showing this.
5. **Image tool suggestion**: if an image could not be converted, or an image stored unconverted kept its metadata (§7.4), and `suppress.image_tool_suggestion` is not `true`, lightly suggest installing a suitable tool (e.g., Pillow for the system Python 3, ImageMagick, or exiftool), and mention that the user can ask you to stop showing this.
6. If there are non-substantive differences: their count, and a single question asking whether the user would like them migrated (rewritten to the current rules) in a separate commit. If the user declines, leave those files untouched, and do not present their diffs.
7. Ask whether the user grants permission to sync. The user may approve everything, some items by number, or nothing. If there is nothing to sync, say that the backup is up to date instead.

Delete SCRATCH before responding (§1.2).

## 11. Sync

Only after the user approves:

1. Repeat the checks of §4.
2. SCRATCH was deleted before your response, so rebuild the candidate (§8) and compare again (§9). Sync exactly what the user approved:
   - If an approved chat has gained appended entries since the report, include them, and mention this afterwards.
   - If an approved item differs from the report in any other way, report the difference and ask again before syncing that item.
   - Chats that became new or changed after the report, and so were not part of it, are left for the next run; mention them.
3. Git identity:
   - If git has `user.name` and `user.email` configured in any scope, commit normally.
   - Otherwise, if REPO has commits, reuse the most recent commit's author name and email for your commits only, via `git -c user.name=<name> -c user.email=<email>`.
   - Otherwise, ask the user how to proceed.
4. If the migration was approved, commit it first, on its own: rewrite or move the non-substantive chats to the current rules, using `git mv` for moves.
5. Then commit the approved material changes, in one commit: add the approved new chats and the new images they reference, and update the approved changed chats. If a changed chat's path differs under the current rules and a migration was declined, update it at its existing path; otherwise, move it with `git mv` and update it. Never delete backup-only chats or images.
6. Commit messages are short and factual, e.g., the numbers of new and updated chats and new images, and the SOURCE labels.
7. Verify: the working tree is clean, every committed chat file parses as JSON in the format of §7.3, and every image reference resolves to a file in `REPO/img/`.
8. Delete SCRATCH, report the commit(s), and ask whether the user would like you to push. Push only with the user's explicit permission.
