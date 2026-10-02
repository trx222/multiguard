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

## Is a tunnel still up?

No root needed:

```bash
wg show interfaces          # e.g. utun10
ls /var/run/wireguard/      # mg_<id>.name -> config that owns it
```

Quitting the app does not close tunnels; see [Architecture](multiguard-architecture.md#tunnel-lifetime).
