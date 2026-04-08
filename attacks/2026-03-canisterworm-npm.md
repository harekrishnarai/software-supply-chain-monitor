# CanisterWorm — TeamPCP's Self-Propagating npm Worm & Kubernetes Wiper

**Date:** March 20–22, 2026
**Ecosystem:** npm / Kubernetes
**Severity:** Critical
**Type:** Self-propagating worm / ICP C2 backdoor / Kubernetes DaemonSet wiper
**Sources:**
- [Aikido — TeamPCP deploys CanisterWorm on NPM following Trivy compromise](https://www.aikido.dev/blog/teampcp-deploys-worm-npm-trivy-compromise)
- [Aikido — CanisterWorm Gets Teeth: TeamPCP's Kubernetes Wiper Targets Iran](https://www.aikido.dev/blog/teampcp-stage-payload-canisterworm-iran)

---

## Summary

On March 20, 2026, the TeamPCP threat actor — the same group behind the Trivy compromise — deployed a novel self-propagating npm worm dubbed CanisterWorm. The attack compromised 47+ packages across multiple npm scopes using stolen credentials, likely obtained from the Trivy CI/CD secret theft. CanisterWorm is notable for using an Internet Computer Protocol (ICP) canister as a censorship-resistant C2 dead-drop — the first observed use of this technique in an npm supply chain attack.

Two days later, Aikido discovered that CanisterWorm had evolved to include a Kubernetes-targeting payload with geopolitical dimensions: clusters identified as running in Iran receive a destructive wiper DaemonSet that bricks the entire cluster, while non-Iranian targets receive a persistent backdoor. The worm also gained self-propagation capability, harvesting npm tokens from infected machines to automatically publish itself to all packages the victim has access to.

---

## Compromised Artifacts

| Package / Scope | Count | Details |
|----------------|-------|---------|
| `@EmilGroup/*` | 28 | All packages in scope compromised in <60 seconds |
| `@opengov/*` | 16 | Full scope compromise |
| `@teale.io/eslint-config` | 1 | Self-propagating variant (Waves 3–4) |
| `@airtm/uuid-base32` | 1 | Compromised via stolen token |
| `@pypestream/floating-ui-dom` | 1 | Compromised via stolen token |

---

## How It Worked

### Three-Stage Architecture

**Stage 1 — Node.js Postinstall Loader:** The malicious `index.js` runs as a postinstall hook during `npm install`. It decodes a base64-embedded Python script, writes it to `~/.local/share/pgmon/service.py`, creates a systemd user service (`pgmon.service`) with `Restart=always`, and starts it immediately. No root required. All artifacts are named to resemble PostgreSQL tooling (`pgmon`, `pglog`, `.pg_state`) to blend in on developer machines.

**Stage 2 — Python ICP Backdoor:** The persistent Python service sleeps 5 minutes (sandbox evasion), then polls an ICP canister at `tdtqy-oyaaa-aaaae-af2dq-cai.raw.icp0.io` every ~50 minutes. The canister returns a URL pointing to the current binary payload. The binary is downloaded to `/tmp/pglog`, made executable, and launched. A kill switch exists: if the URL contains `youtube.com`, execution is skipped (dormant state).

**Stage 3 — Self-Propagating Worm (Waves 3–4):** The `deploy.js` worm script harvests npm tokens from `~/.npmrc`, project `.npmrc`, `/etc/npmrc`, environment variables (`NPM_TOKEN`, `NPM_TOKENS`), and `npm config`. For each token, it resolves the username, enumerates all publishable packages, bumps the patch version, preserves the original README, and publishes the infected payload with `--tag latest`.

### Kubernetes Wiper (March 22 evolution)

The ICP canister payload was updated to include a Kubernetes-targeting Python script with geopolitical targeting:

**Target Detection:**
- `is_k8s()` — checks for `/var/run/secrets/kubernetes.io/serviceaccount` or `KUBERNETES_SERVICE_HOST` env var
- `is_iran()` — checks timezone (`Asia/Tehran`), locale (`fa_IR`, `Persian`), and `LANG`/`LC_ALL` env vars

**Iranian Targets — Destructive Wiper (`host-provisioner-iran`):**
A DaemonSet mounts the host root filesystem at `/mnt/host` with `privileged: true` and `tolerations: [operator: Exists]`, ensuring it runs on every node including the control plane. It executes `rm -rf /mnt/host/*`, bricking the entire cluster.

**Non-Iranian Targets — Persistent Backdoor (`host-provisioner-std`):**
A DaemonSet writes the CanisterWorm backdoor to every node and registers it as a systemd service (`internal-monitor`), providing persistent access across the cluster.

---

## Timeline

| Date/Time (UTC) | Event |
|-----------------|-------|
| Mar 20, 2026 20:45 | Aikido detects initial CanisterWorm wave; 28 @EmilGroup packages compromised |
| Mar 20, 2026 ~20:45 | 16 @opengov packages compromised |
| Mar 20, 2026 21:16 | Wave 3: `@teale.io/eslint-config@1.8.11` published with self-propagating worm |
| Mar 20, 2026 21:21 | `@teale.io/eslint-config@1.8.12` — test payload (`hello123`) with full worm plumbing |
| Mar 21, 2026 | Wave 4: Final form — self-propagating + armed ICP backdoor |
| Mar 22, 2026 | Kubernetes wiper payload discovered targeting Iranian infrastructure |
| Mar 22, 2026 | Third iteration adds worm self-propagation to Kubernetes payload |

---

## Detection

```bash
# Check for the CanisterWorm systemd service
systemctl --user status pgmon.service
systemctl --user status internal-monitor

# Check for CanisterWorm filesystem artifacts
ls -la ~/.local/share/pgmon/service.py
ls -la ~/.config/systemd/user/pgmon.service
ls -la /tmp/pglog
ls -la /tmp/.pg_state
ls -la /var/lib/svc_internal/runner.py

# Check for Kubernetes DaemonSets deployed by CanisterWorm
kubectl get ds -n kube-system | grep -E "host-provisioner-(iran|std)"

# Check for processes masquerading as PostgreSQL
ps aux | grep -E "pgmon|pglog"

# Check for outbound connections to ICP canister
# Look for connections to icp0.io domains in network logs

# Check npm packages for compromised postinstall hooks
npm ls 2>/dev/null | grep -E "@emilgroup|@opengov|@teale.io/eslint-config|@airtm/uuid-base32|@pypestream/floating-ui-dom"

# Verify index.js hashes against known-malicious
sha256sum node_modules/*/index.js | grep -E "e9b1e069|61ff00a8|0c0d206d|c37c0ae9"
```

---

## Remediation

1. **Stop and disable the pgmon systemd service:**
   ```bash
   systemctl --user stop pgmon.service
   systemctl --user disable pgmon.service
   rm ~/.config/systemd/user/pgmon.service
   rm -rf ~/.local/share/pgmon/
   rm -f /tmp/pglog /tmp/.pg_state
   systemctl --user daemon-reload
   ```
2. **For Kubernetes clusters**, check for and remove unauthorized DaemonSets:
   ```bash
   kubectl delete ds host-provisioner-iran -n kube-system --ignore-not-found
   kubectl delete ds host-provisioner-std -n kube-system --ignore-not-found
   ```
3. **Rotate all npm tokens** — revoke and regenerate from npmjs.com settings
4. **Audit `~/.npmrc`** for any tokens that may have been harvested
5. **Check published packages** — verify no unexpected patch versions were published to your scopes
6. **Rotate all credentials** that may have been present on compromised machines

---

## Lessons Learned

- ICP canisters represent a new class of censorship-resistant C2 infrastructure; defenders cannot take down the dead-drop
- Self-propagating npm worms can compromise entire scopes in under 60 seconds using stolen tokens
- Geopolitically-targeted payloads (destructive for Iran, persistent for others) indicate a state-aligned or politically-motivated threat actor
- PostgreSQL-themed naming (pgmon, pglog) is effective camouflage on developer machines
- The kill switch (YouTube URL = dormant) allows the operator to arm/disarm the implant at will without modifying any packages

---

## Related Incidents

- [Trivy Second Compromise](./2026-03-trivy-second-compromise.md) — The credential theft that likely enabled the npm token acquisition
- [Trivy GitHub Actions Tag Compromise](./2026-03-trivy-github-actions.md) — Original Trivy compromise by the same actor
- [GlassWorm / CanisterWorm](./2026-03-glassworm-canisterworm.md) — Earlier CanisterWorm variant
- [Shai-Hulud Worm](./2025-late-shai-hulud-worm.md) — Prior self-propagating npm worm campaign
