![Cover](Cover.png)

# Claude Code — macOS Notifications Setup

Get native macOS notifications when Claude Code finishes a task or requests your approval, with a custom icon.

---

## Prerequisites

- macOS
- [Homebrew](https://brew.sh) installed
- Claude Code CLI installed

---

## Step 1 — Install terminal-notifier

```bash
brew install terminal-notifier
```

`terminal-notifier` is a command-line tool that sends native macOS notifications. It registers as its own app, which means macOS will prompt you for notification permissions properly.

---

## Step 2 — Replace the app icon

The included `claude-character-icon.icns` is the custom icon used in the notification. Replace the default `terminal-notifier` icon with it.

```bash
sudo cp claude-character-icon.icns \
  /opt/homebrew/opt/terminal-notifier/terminal-notifier.app/Contents/Resources/Terminal.icns
```

Then clear the macOS icon cache so it picks up the change:

```bash
sudo rm -rf /Library/Caches/com.apple.iconservices.store \
  && sudo killall -9 iconservicesd 2>/dev/null; \
  killall NotificationCenter 2>/dev/null; \
  killall Dock
```

Your Dock will briefly disappear and restart — that's normal.

---

## Step 3 — Configure Claude Code hooks

Add the following hooks to `~/.claude/settings.json`. If the file already exists, merge the `hooks` block into it.

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "terminal-notifier -title 'Claude Code' -message 'Claude has finished the task.' -sound Glass",
            "async": true
          }
        ]
      }
    ],
    "PermissionRequest": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "terminal-notifier -title 'Claude Code' -message 'Claude is requesting your approval.' -sound Ping",
            "async": true
          }
        ]
      }
    ]
  }
}
```

- **Stop** — fires when Claude finishes responding
- **PermissionRequest** — fires when Claude needs your approval to run a tool

---

## Step 4 — Grant notification permissions

The first time `terminal-notifier` runs, macOS will prompt you to allow notifications. Accept it. You can verify in:

**System Settings → Notifications → terminal-notifier**

Set it to **Alerts** so notifications persist until dismissed.

---

## Step 5 — Test it

Fire a test notification from your terminal:

```bash
terminal-notifier -title 'Claude Code' -message 'Claude has finished the task.' -sound Glass
```

---

## Files in this directory

| File | Description |
|------|-------------|
| `claude-character-icon.icns` | The converted icon file — drop this into the terminal-notifier app bundle |
| `claude-character-icon-square.png` | The square (114x114) padded PNG used to generate the `.icns` |
| `README.md` | This file |

---

## How the icon was made

The original PNG was 114x77 (non-square). To avoid stretching when converting to `.icns` (which requires square dimensions), it was padded to 114x114 with a transparent background using Python's Pillow library, then converted using macOS's built-in `iconutil`.

```python
from PIL import Image

src = "claude-character-icon.png"  # your original PNG
out = "claude-character-icon-square.png"

img = Image.open(src).convert("RGBA")
w, h = img.size
size = max(w, h)

square = Image.new("RGBA", (size, size), (0, 0, 0, 0))
square.paste(img, ((size - w) // 2, (size - h) // 2))
square.save(out)
```

Then convert to `.icns`:

```bash
mkdir -p claude-icon.iconset

for size in 16 32 128 256 512; do
  sips -z $size $size claude-character-icon-square.png \
    --out claude-icon.iconset/icon_${size}x${size}.png
  double=$((size * 2))
  sips -z $double $double claude-character-icon-square.png \
    --out claude-icon.iconset/icon_${size}x${size}@2x.png
done

iconutil -c icns claude-icon.iconset -o claude-character-icon.icns
```
