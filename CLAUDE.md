# openclaw-tools

## Screenshot Storage

Save every screenshot to `~/Desktop/screenshots/{project-name}/`.
Use the git repo name for `{project-name}`:

```bash
export PROJECT_SCREENSHOT_DIR=~/Desktop/screenshots/$(basename "$(git rev-parse --show-toplevel)")
mkdir -p "$PROJECT_SCREENSHOT_DIR"
```

Do not keep proof or review screenshots in `/tmp`.

## What This Is
`openclaw-tools` is an interactive shell/TUI utility for managing OpenClaw installs, multi-user setups, backups, migrations, and uninstall flows.

## Tech Stack
Shell scripts, gum-based TUI, macOS/Linux system-management workflows

## Key Commands
- `./openclaw-tools`

## Conventions
- Keep the product zero-friction: launch straight into the TUI rather than requiring command memorization.
- System-management operations should stay explicit and reversible.
- Preserve cross-user and home-directory migration safety.

## Architecture Notes
- This repo touches real local installs and user data, so operational safety is part of the product contract.

## Do Not
- Hide destructive operations behind ambiguous UI copy.
- Assume a single-user machine when the tool explicitly supports multi-user setups.
