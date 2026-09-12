---
title: "recall: Bring Your Terminal Back When a Long Job Finishes"
date: 2026-09-12 00:00:00 -0700
categories: [Announcement]
tags: [macos, cli, claude-code, productivity, tools]
lang: en
---

I published a small macOS CLI tool called [recall](https://github.com/hanyuliu/recall). Here is what it is, what it does, and how to turn it on.

## What it is

`recall` is a one-line command you drop at the end of anything that takes a while. When it runs, it posts a native macOS notification, and when you click that notification, it brings the exact terminal window that ran the command back to the front — even if that window is minimized.

It works with whatever terminal you actually use: iTerm2, Terminal.app, Wave, Hyper, VS Code's integrated terminal, and others. There is nothing to configure per-terminal; `recall` figures out which app to focus on its own.

It also plugs directly into **Claude Code**: wired up as a hook, it notifies you the moment a session finishes a turn or is blocked waiting on your input, so you don't have to keep a tab open just to check.

## What it does

The core idea is small on purpose:

```bash
recall                            # "Script finished" · "Terminal"
recall "Build done"               # custom message
recall "Build done" "My Project"  # custom message + title
sleep 60 && recall "Done"         # notify after a long command
npm run build && recall "Build done" "Frontend"
```

Under the hood, `recall` walks up its own process tree looking for the first PID that macOS recognizes as a GUI app — that's the terminal that launched it, regardless of which one it is. It writes a small AppleScript click-handler that unminimizes every window belonging to that app (using `AXWindows`/`AXMinimized`, since System Events' plain `windows` property skips minimized ones) and activates it. A gentle "Tink" sound plays when the notification lands, and the terminal app's own icon shows up as the notification's thumbnail.

For Claude Code specifically, the same mechanism drives two hooks:

- **Stop** — fires when Claude finishes responding, so a long agentic run doesn't leave you staring at an idle tab wondering if it's done.
- **Notification** — fires when Claude is waiting on you (a permission prompt, a clarifying question), so you can step away without missing the moment it needs you.

## How to enable it

**Requirements:** macOS 12+, [Homebrew](https://brew.sh), and [terminal-notifier](https://github.com/julienXX/terminal-notifier) (macOS will prompt for Accessibility permission the first time it runs).

The fastest path is the guided installer — it installs the dependency, drops `recall` on your `PATH`, and optionally wires up the Claude Code hooks for you:

```bash
curl -fsSL https://raw.githubusercontent.com/hanyuliu/recall/main/install.sh | bash
```

Prefer to do it by hand:

```bash
brew install terminal-notifier

curl -fsSL https://raw.githubusercontent.com/hanyuliu/recall/main/recall.sh \
  -o /usr/local/bin/recall && chmod +x /usr/local/bin/recall
```

### If you use Claude Code

The cleanest option is the plugin marketplace — add this to `~/.claude/settings.json` and `/recall` becomes available in every session:

```json
{
  "extraKnownMarketplaces": {
    "recall": {
      "source": {
        "source": "github",
        "repo": "hanyuliu/recall"
      }
    }
  },
  "enabledPlugins": {
    "recall@recall": true
  }
}
```

From there, `/recall install`, `/recall enable hooks`, `/recall disable hooks`, and `/recall uninstall` manage everything without touching a config file by hand.

Or wire the hooks in yourself:

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "recall 'Claude finished' 'Claude Code'",
            "async": true
          }
        ]
      }
    ],
    "Notification": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "recall 'Claude needs your input' 'Claude Code'",
            "async": true
          }
        ]
      }
    ]
  }
}
```

Restart Claude Code (or open `/hooks` once) and you're done. Full source is at [github.com/hanyuliu/recall](https://github.com/hanyuliu/recall).
