# ClCoChBa: Claude Code Chat Backup

ClCoChBa is a prompt that has a Claude Code session back up your Claude Code chats to a git repository you own. It keeps only the conversation itself, i.e., what you said and what Claude said back to you, and drops tool calls, tool output, and injected context. Each chat becomes one JSON file, organized by project and date.

The prompt is in [PROMPT.md](PROMPT.md).

## Usage

You need Python 3.8 or newer, git, and a local clone of the git repository that will hold the backup.

In a Claude Code session, ask Claude to fetch the prompt and follow it:

> Fetch `https://raw.githubusercontent.com/HACKERALERT/ClCoChBa/main/PROMPT.md` verbatim, e.g., with `curl`, read it in full, and follow it.

Ask for a verbatim fetch, since some web-fetch tools summarize pages rather than returning them in full.

Claude builds the backup in its own temporary folder, shows you what changed, commits only what you approve, and pushes only with your permission. With your permission, it remembers settings such as the backup repository's path in `~/.clcochba`.

## Backup layout

```
raw/<project path>/@chats/<YYYY>/<MM>/<DD>/<chat ID>.json
img/<image hash>.jpg
```

Images are stored once each, converted to JPEG at quality 80, at most 1920×1920, with metadata such as location removed. Animated images keep their original format.

## About

ClCoChBa was conceived by a human. Claude Opus 5.5 helped make it concrete through dry runs on real chat data. It was written and tested in Claude Code v2.1.304.
