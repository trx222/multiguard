---
title: MultiGuard Architecture
tags: [multiguard, architecture, swiftui, xpc]
summary: Internal structure of the MultiGuard app and its privileged helper.
---

# MultiGuard Architecture

## Targets

| Target | Type | Role |
|--------|------|------|
| `MultiGuard` | SwiftUI executable | Main app UI and business logic. |
| `MultiGuardHelper` | XPC command-line tool | Privileged helper that runs `wg-quick` as root. |

## App components

- **`ConfigParser`** — Parses WireGuard `.conf` files into `WireGuardConfig`.
- **`ConflictDetector`** — Compares `Address` and `AllowedIPs` CIDRs across configs to detect overlaps.
- **`TunnelStore`** — Copies imported configs to `~/Library/Application Support/MultiGuard/Configs/` and persists tunnel metadata.
- **`TunnelManager`** — Routes connect/disconnect requests to the privileged helper, falling back to `osascript` for unsigned dev builds.
- **`HelperManager`** — Installs the helper via `SMAppService` and manages the XPC connection.
- **`TunnelDetailsFetcher`** — Queries `wg show <interface>` for live RX/TX stats.
- **`ContentView` / `MenuBarView`** — SwiftUI frontends.

## Privileged helper

The helper (`com.multiguard.helper`) is registered once via `SMAppService.daemon`. Its launchd plist (`Resources/com.multiguard.helper.plist`) uses `BundleProgram` = `Contents/Library/LaunchServices/MultiGuardHelper`, so launchd runs the binary straight out of the app bundle — moving or deleting `MultiGuard.app` breaks the helper.

- **Client check:** the helper reads the Team ID from its *own* signature and only accepts XPC clients matching `identifier "com.multiguard.app" and anchor apple generic and certificate leaf[subject.OU] = "<team>"`. No Team ID is compiled in; an ad-hoc helper (no team) rejects everyone.
- **Approval:** the first registration needs the user to allow the background item in *System Settings → General → Login Items & Extensions*; the app opens that pane when `SMAppService` reports `.requiresApproval`. Afterwards `wg-quick up/down` runs without prompts.
- **PATH:** launchd starts the helper with `PATH=/usr/bin:/bin:/usr/sbin:/sbin`; the helper prepends `/opt/homebrew/bin:/usr/local/bin` for the processes it spawns so `wg-quick` finds `wg` and Bash 4+.

### App side (`HelperManager`)

- Ad-hoc builds (no Team ID in the app's own signature) never register the helper and unregister any stale registration, then go straight to the `osascript` fallback.
- The XPC `ping` uses `remoteObjectProxyWithErrorHandler` and a 3 s timeout. Without these, a helper that never starts left the call pending forever and the fallback was never reached.

## Tunnel lifetime

Tunnels are owned by the system (`wg-quick` creates the `utun` device), not by the app. Quitting MultiGuard leaves them up; on launch `TunnelManager.runningInterfaces` adopts them by matching each `utun` address against the configs.

## Multiple interfaces

Each tunnel gets a unique config file name (`mg_<12-hex-chars>.conf`). `wg-quick up <file>` creates a separate `utun` device per tunnel, so traffic is routed independently rather than forcing all peers through one shared interface.
