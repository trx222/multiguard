---
title: MultiGuard Troubleshooting
tags: [multiguard, troubleshooting, helper, launchd]
summary: Diagnosing a Connect click that does nothing and other helper problems.
---

# MultiGuard Troubleshooting

## Connect does nothing, no password prompt (fixed 2026-10-02)

Two independent bugs, see [Design Decisions](multiguard-decisions.md):

1. **Hanging XPC ping.** An ad-hoc build had registered the helper; its plist pointed at the non-existent `/Library/PrivilegedHelperTools/com.multiguard.helper`. launchd kept the job in `spawn scheduled`, the ping reply never came and the `osascript` fallback was never reached. Check with:
   ```bash
   launchctl print system/com.multiguard.helper | grep -E "state|program"
   ```
2. **Localized Bash version string.** On a German system `bash --version` prints `GNU bash, Version 5.3…` (capital V). The case-sensitive parser treated Bash 5 as too old and aborted before the prompt. Both app (`BashPaths`) and helper now match case-insensitively.

## Disconnect spins forever, "Disconnecting…" never ends (fixed 2026-10-05)

Seen after a tunnel had silently died (its `utun*.sock` was gone, the `mg_<id>.name` file still there). The app polls `wg show <iface> dump` through the helper every 2 s; against a hung wireguard-go that call never returns. The helper handled one request at a time and the app waited without a timeout, so the disconnect queued behind the stuck stats call indefinitely. The main thread stayed idle (`sample <pid>` shows it in the run loop), only the tunnel's state was stuck.

Fix: helper runs requests concurrently with per-process timeouts, the app times out every XPC call, and a dead tunnel is detected and shown as failed. See [Architecture](multiguard-architecture.md#app-side-helpermanager).

**After rebuilding, restart the helper** — launchd keeps the old binary running:
```bash
sudo launchctl kickstart -k system/com.multiguard.helper
```

## Is a tunnel still up?

No root needed:

```bash
wg show interfaces          # e.g. utun10
ls /var/run/wireguard/      # mg_<id>.name -> config that owns it
```

Quitting the app does not close tunnels; see [Architecture](multiguard-architecture.md#tunnel-lifetime).
