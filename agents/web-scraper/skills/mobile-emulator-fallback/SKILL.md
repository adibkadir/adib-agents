---
name: mobile-emulator-fallback
description: "Tier 5 — Last resort: mobile app automation via Android device + ADB for data only available in native apps or when all web approaches fail. Uses Python + uiautomator2 for scripted automation, mobile-mcp for Claude-driven control, scrcpy-mcp for fast screenshots, or raw ADB commands. Requires physical Android device with USB debugging."
---

# Mobile App Emulator Fallback (Tier 5)

The absolute last resort. Use only when ALL web-based approaches (Tiers 1-4) have failed and the data is exclusively available in a native mobile app, or when mobile apps have significantly less bot detection than web equivalents.

## When to Escalate to Tier 5

- Data is ONLY available in a native mobile app (no web version)
- Mobile app has significantly less bot detection than web
- All web approaches (fetch, Playwright, Crawlee, Firecrawl) are blocked
- User has explicitly approved mobile automation setup

## Before Using This Tier

**Always ask the user first.** This tier requires:
1. An Android device (physical or emulator)
2. USB debugging enabled (Settings → Developer Options → USB Debugging)
3. ADB installed (`brew install android-platform-tools` on macOS)
4. The target app installed and logged in on the device
5. Device connected via USB and authorized (`adb devices` should list it)

Tell the user: "I've exhausted all web-based approaches. The data appears to only be available in the [App Name] mobile app. I'd need you to set up an Android device with USB debugging and install the app. Would you like to proceed?"

## Option 1: Python + uiautomator2 (Recommended)

The simplest path. Pure Python script, no AI agent needed.

**Setup:**
```bash
pip install uiautomator2
# Ensure ADB is installed and device is connected
adb devices  # Should show your device
```

**Basic Script:**
```python
import uiautomator2 as u2
import time
import json
from pathlib import Path

# Connect to device via ADB
d = u2.connect()

# Launch the target app
d.app_start("com.example.app")
time.sleep(3)  # Wait for app to load

# Navigate to the data screen
d.click(0.5, 0.3)  # Tap center-top (relative coordinates)
time.sleep(1)

# Screenshot for verification
d.screenshot("screenshots/step_001.png")

# Scroll through content
d.swipe(0.5, 0.8, 0.5, 0.2, duration=0.5)  # Swipe up
time.sleep(1)

# Extract text from screen
elements = d(className="android.widget.TextView")
texts = [e.get_text() for e in elements]
print(json.dumps(texts, indent=2))
```

**Deterministic Loop Pattern (e.g., capturing stories/posts):**
```python
import uiautomator2 as u2
import time
from pathlib import Path

d = u2.connect()
d.app_start("com.example.app")
time.sleep(3)

output_dir = Path("screenshots")
output_dir.mkdir(exist_ok=True)

# Capture 20 items by tapping through
for i in range(20):
    # Screenshot current screen
    d.screenshot(str(output_dir / f"item_{i:03d}.png"))

    # Tap right side to go to next item
    d.click(0.8, 0.5)  # 80% x, 50% y
    time.sleep(1.5)  # Wait for next item to load

print(f"Captured {20} screenshots to {output_dir}")
```

**Pros:** Pure Python, easy to script, direct screenshot saving, relative coordinate tapping
**Cons:** ADB screenshots ~500ms each (fine for most use cases)

## Option 2: mobile-mcp (Claude-Driven)

Lets Claude see the phone screen and make decisions about what to tap.

**Setup:**
```bash
npx -y @mobilenext/mobile-mcp@latest
# Add to Claude Code MCP config
```

**Use when:** The app navigation is complex or varies between runs (e.g., different layouts, A/B tests). Claude can adapt to what it sees.

**Pros:** Claude can see the screen and make decisions, cross-platform (iOS too)
**Cons:** Requires MCP client, more moving parts, slower than scripted approach

## Option 3: scrcpy-mcp (Fastest Screenshots)

Similar to mobile-mcp but uses scrcpy for ~33ms screenshots (vs ~500ms for raw ADB). 34 tools available.

**Setup:**
```bash
brew install scrcpy  # On macOS
npx scrcpy-mcp
```

**Use when:** Speed matters — scraping hundreds of screens where screenshot latency adds up.

**Pros:** Much faster screenshots, comprehensive toolset
**Cons:** Extra dependency (scrcpy)

## Option 4: Raw ADB Shell Commands (Zero Dependencies)

No libraries beyond ADB itself. Works everywhere.

```bash
# Screenshot
adb shell screencap -p /sdcard/screen.png
adb pull /sdcard/screen.png ./screenshots/

# Tap at coordinates
adb shell input tap 720 640

# Swipe (scroll)
adb shell input swipe 500 1500 500 500 300

# Type text
adb shell input text "search query"

# Press keys
adb shell input keyevent KEYCODE_ENTER
adb shell input keyevent KEYCODE_BACK

# Launch app
adb shell am start -n com.example.app/.MainActivity

# List running activities
adb shell dumpsys activity activities | grep "mResumedActivity"
```

**Batch Script:**
```bash
#!/bin/bash
# capture_items.sh — capture 20 screens from a list/story view

OUTPUT_DIR="screenshots"
mkdir -p "$OUTPUT_DIR"

for i in $(seq -w 1 20); do
  adb shell screencap -p /sdcard/screen.png
  adb pull /sdcard/screen.png "$OUTPUT_DIR/item_${i}.png"

  # Tap right side to advance
  adb shell input tap 900 640
  sleep 1.5
done

echo "Captured 20 screenshots to $OUTPUT_DIR"
```

## Data Extraction from Screenshots

After capturing screenshots, extract text with OCR or visual analysis:

```python
# Using pytesseract for OCR
import pytesseract
from PIL import Image

image = Image.open("screenshots/item_001.png")
text = pytesseract.image_to_string(image)
print(text)
```

Or use an LLM with vision to extract structured data from screenshots:

```javascript
// Node.js — send screenshot to OpenRouter/GPT-4V for structured extraction
import { readFileSync } from 'fs'

const image = readFileSync('screenshots/item_001.png')
const base64 = image.toString('base64')

const response = await fetch('https://openrouter.ai/api/v1/chat/completions', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${process.env.OPENROUTER_API_KEY}`,
  },
  body: JSON.stringify({
    model: 'anthropic/claude-sonnet-4',
    messages: [{
      role: 'user',
      content: [
        { type: 'image_url', image_url: { url: `data:image/png;base64,${base64}` } },
        { type: 'text', text: 'Extract the following from this screenshot: title, description, price, rating. Return as JSON.' },
      ],
    }],
  }),
})
```

## Decision Tree

```
Can you get the data from a web browser?
  ├── YES → Use Tiers 1-4 (never reach Tier 5)
  └── NO → Is there a mobile app with the data?
      ├── NO → Data is not accessible. Tell the user.
      └── YES → Is the navigation deterministic (same taps every time)?
          ├── YES → Use Option 1 (uiautomator2) — simple Python script
          └── NO → Does the layout vary between runs?
              ├── YES → Use Option 2 (mobile-mcp) — Claude adapts
              └── NO → Use Option 4 (raw ADB) — zero dependencies
```

## Important Notes

- Always get explicit user approval before suggesting mobile automation
- The user must physically set up the device and install the app
- ADB screenshots are slow (~500ms) — factor this into timing
- Coordinate taps use absolute pixels or relative fractions
- Test on the actual device — emulators may have different layouts
- Some apps detect rooted/emulated devices and refuse to run
- Keep the device awake: `adb shell svc power stayon true`
- Disable screen rotation: `adb shell content insert --uri content://settings/system --bind name:s:accelerometer_rotation --bind value:i:0`
