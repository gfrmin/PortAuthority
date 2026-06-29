# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Port Authority is a fast Android network scanner (LAN/WAN host discovery, port scanning, MAC vendor lookup, DNS lookups, Wake-on-LAN). Pure **Java**, no Kotlin. GPLv3. Published on F-Droid and Google Play. Package: `com.aaronjwood.portauthority`.

- Default branch is **`development`** — open PRs against it, not `master`/`main`.
- No tracking/ads/analytics by design.

## Commands

Gradle 7.3.3 (wrapper), AGP 7.2.2. compileSdk/targetSdk **31**, minSdk **19**, Java 8. Two product flavors (`free`, `donate`) × two build types (`debug`, `release`).

```bash
./gradlew assembleDebug            # all flavors
./gradlew assembleFreeDebug        # single flavor+type
./gradlew assembleRelease          # minified (R8 + shrinkResources)
./gradlew lintDebug                # lint (abortOnError is false, so it won't fail the build)
./gradlew test                     # JUnit/Mockito unit tests (none exist yet)
./gradlew connectedAndroidTest     # instrumented tests (none exist yet)
```

There are currently **no tests** in the repo, though JUnit 4.13 and Mockito 1.10.19 are declared. Release signing is **not** configured in `build.gradle` (must be supplied locally). SonarQube is wired to an internal host in `app/build.gradle`.

## Native code (critical to host discovery)

`app/src/main/c/` builds `libjniportal.so` via `ndkBuild` (`Android.mk`) during a normal gradle build. `ipneigh.c` exposes `nativeIPNeigh(int fd)`, which reads the kernel **neighbor (ARP) table over RTNETLINK** and writes `<IP> <MAC> <state>` lines to a pipe fd handed in from Java. Most other `*_ntop.c` / `ll_map.c` / `libnetlink.c` files are vendored netlink helpers.

**Android 12/13 / SELinux:** newer SELinux policy blocks the `nlmsg_getneigh` netlink query, breaking host scans. The current mitigation is **targeting SDK 31** (see commit `89d7213`). Keep this constraint in mind before bumping `targetSdkVersion`.

## Architecture

Activity-based (no Fragments) with an `AsyncTask` + `ExecutorService` concurrency model. Async callbacks are delivered through **delegate interfaces** held by **`WeakReference`** to avoid leaking Activities. Source under `app/src/main/java/com/aaronjwood/portauthority/`:

- **`activity/`** — `MainActivity` (launcher: LAN discovery + network info), abstract `HostActivity`, `LanHostActivity` / `WanHostActivity` (port scan UIs), `DnsActivity`, `PreferencesActivity`.
- **`async/`** — orchestrators that spawn executor tasks: `ScanHostsAsyncTask`, `ScanPortsAsyncTask`, `WanIpAsyncTask`, `DnsLookupAsyncTask`, `Download{Ouis,PortData}AsyncTask`, `WolAsyncTask`.
- **`runnable/`** — the actual socket work: `ScanHostsRunnable` (TCP-connects across the subnet to populate the ARP cache), `ScanPortsRunnable` (per-port TCP connect; parses SSH banners on 22 and HTTP `Server` headers on 80/443/8080).
- **`network/`** — `Host` (Serializable model: IP/MAC/hostname/vendor), `Wireless` (Wi-Fi/connectivity info with Android 6/10/11 fallbacks), `MDNSResolver` + `NetBIOSResolver` (hostname resolution: mDNS first, NetBIOS fallback).
- **`response/`** — callback interfaces (`MainAsyncResponse`, `HostAsyncResponse`, `LanHostAsyncResponse`, `WanHostAsyncResponse`, `DnsAsyncResponse`, `ErrorAsyncResponse`).
- **`db/`** — `Database` (SQLite singleton; `ouis` MAC-vendor + `ports` description tables, lazily downloaded on first run).
- **`parser/`** — `OuiParser` (IEEE OUI), `PortParser` (IANA registry).
- **`adapter/`**, **`listener/`**, **`utils/`** — `HostAdapter`, click listeners, and helpers (`Constants`, `UserPreference`, `Errors`).

### Host discovery flow
`MainActivity` → `ScanHostsAsyncTask`: computes IP range from the subnet CIDR, spawns one `ScanHostsRunnable` per host bit to TCP-connect across the range (intentionally failing connects, purely to populate the kernel ARP cache). Then `nativeIPNeigh()` reads the populated neighbor table, entries are parsed and filtered (drops INCOMPLETE/FAILED/loopback/link-local), and each IP+MAC is enriched with vendor (DB), reverse DNS, mDNS, then NetBIOS, delivered back via `MainAsyncResponse.processFinish(Host, AtomicInteger)`.

### Port scanning flow
`Host.scanPorts(...)` → `ScanPortsAsyncTask` chunks the port range across N threads (default 500, staggered via `ScheduledThreadPoolExecutor`), each `ScanPortsRunnable` TCP-connects per port and reports open ports + banners through `HostAsyncResponse.processFinish(SparseArray<String>)`.

## Conventions

- Thread count and socket timeouts are user-configurable via `UserPreference` (Preferences screen) — surfaced to handle OOM on low-end devices and carrier throttling on WAN scans.
- Heavy version-gated branching in `Wireless`/`MainActivity` for Android 6/8/10/11 permission and API changes (SSID needs location permission; MAC access is restricted on 11+). Match the existing feature-detection pattern when touching network/permission code.
- OkHttp is pinned to **3.14.9** deliberately for Android 4 compatibility — do not upgrade casually.
- No enforced formatter (no editorconfig/spotless/ktlint/checkstyle); follow the surrounding Java/Javadoc style.
