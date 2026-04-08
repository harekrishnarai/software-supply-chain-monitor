# Maven Central Jackson Typosquatting — First Sophisticated Malware on Maven Central

**Date:** December 2025
**Ecosystem:** Maven Central (Java/JVM)
**Severity:** Critical
**Type:** Typosquatting / Backdoor / RAT / Infostealer
**Sources:**
- [Aikido Security — Maven Central Jackson Typosquatting Malware](https://www.aikido.dev/blog/maven-central-jackson-typosquatting-malware)

---

## Summary

On December 25, 2025, Aikido Security discovered a sophisticated malicious package on Maven Central impersonating the Jackson Databind library — one of the most widely-used Java JSON processing libraries. The attack represents the first sophisticated malware ever found on Maven Central, a registry historically considered significantly more secure than npm or PyPI due to its signing requirements and manual review processes.

The attacker registered the fake Maven group ID `org.fasterxml.jackson.core` to impersonate the legitimate `com.fasterxml.jackson.core` namespace — a single character difference (`org.` vs `com.`) that is easily overlooked in `pom.xml` dependency declarations. The package contained a 6-stage execution chain that auto-executes via Spring Boot's autoconfiguration mechanism, contacts an attacker-controlled C2 to fetch an AES-encrypted payload URL, downloads platform-specific binaries for Windows and macOS, and deploys a full Remote Access Trojan. The malware also embedded Chinese-language prompt injection strings specifically designed to mislead LLM-based code analysis tools. Aikido reported the package to Maven Central and GoDaddy; it was taken down within approximately 1.5 hours.

---

## Compromised Artifacts

| Artifact | Malicious | Legitimate |
|---|---|---|
| Group ID | `org.fasterxml.jackson.core` | `com.fasterxml.jackson.core` |
| Artifact ID | `jackson-databind` | `jackson-databind` |
| C2 Domain | `m.fasterxml[.]org` | — |
| C2 Domain Registered | 2025-12-17 (GoDaddy) | — |
| C2 Domain Updated | 2025-12-22 | — |
| Payload Server IP | `103.127.243[.]82:8000` | — |

**Maven dependency declaration (malicious)**:
```xml
<dependency>
    <groupId>org.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>[malicious version]</version>
</dependency>
```

**Legitimate Jackson Databind**:
```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.x.x</version>
</dependency>
```

---

## How It Worked

### Entry Point — Namespace Typosquatting + Spring Boot Autoconfiguration

The malicious package uses the group ID `org.fasterxml.jackson.core` — differing from the legitimate `com.fasterxml.jackson.core` only in the top-level domain prefix. The package presents as a drop-in replacement for `jackson-databind` and includes a Spring Boot autoconfiguration class (`JacksonSpringAutoConfiguration`) registered in `META-INF/spring.factories`. This causes the malicious code to **auto-execute at Spring Boot application startup** without any explicit call or import from the application developer — the Spring Boot framework invokes it automatically.

### Stage 1 — Auto-Execution via Spring Autoconfiguration

At application startup, Spring Boot discovers `JacksonSpringAutoConfiguration` via `META-INF/spring.factories` and instantiates it. The `@Configuration` annotation causes Spring to process it as a configuration bean, triggering the malicious initialization logic with no developer action required.

### Stage 2 — Persistence Check

The malware checks for the presence of a `.idea.pid` file — mimicking an IntelliJ IDEA project lock file — to determine whether it has already run on the host. This prevents repeated executions and reduces noise in system logs.

### Stage 3 — OS Fingerprinting

```java
String os = System.getProperty("os.name").toLowerCase();
// Checks for: win, mac, darwin, nux, linux
```

The OS is identified to select the appropriate platform-specific payload.

### Stage 4 — C2 Contact and Config Retrieval

The malware contacts the C2 endpoint to retrieve an AES-encrypted configuration file:

```
http://m.fasterxml[.]org:51211/config.txt
```

The configuration is encrypted with AES-ECB using a hardcoded 16-byte key embedded in the malware:

```
9237527890923496
```

The decrypted config contains the platform-specific payload download URL.

### Stage 5 — Platform-Specific Payload Download

| Platform | Payload URL | Local Filename |
|---|---|---|
| Windows | `http://103.127.243[.]82:8000/http/192he23/svchosts.exe` | `payload.bin` (temp) |
| macOS | `http://103.127.243[.]82:8000/http/192he23/update` | `payload.bin` (temp) |

The Windows payload is named `svchosts.exe` — a deliberate misspelling of the legitimate Windows system process `svchost.exe` — to blend into process listings. The macOS payload is a generic binary named `update`.

### Stage 6 — Execution and Concealment

After download, the payload is:
1. Written to a temp directory
2. Made executable: `chmod +x` (macOS/Linux)
3. Executed with stdout and stderr redirected to `/dev/null` to suppress any terminal output

The executed binary is a full Remote Access Trojan (RAT) with capabilities including:

- **`RunFileContent`** — download arbitrary files from the compromised host
- **`CompressDownload`** — package and download entire directories as ZIP archives
- **Embedded operator panel** — Chinese-language HTML management interface for the attacker

### Anti-Analysis: LLM Prompt Injection

The malware source code contains embedded Chinese-language strings specifically crafted as **prompt injection payloads targeting LLM-based code analyzers**. Tools that use language models to assess package safety (such as AI-powered security scanners) may be misled by these strings into misclassifying the malicious code as benign. This represents a novel evasion technique targeting AI-augmented security tooling.

### C2 Infrastructure

| Endpoint | Role |
|---|---|
| `m.fasterxml[.]org:51211/config.txt` | AES-encrypted payload URL delivery |
| `103.127.243[.]82:8000/http/192he23/svchosts.exe` | Windows RAT binary |
| `103.127.243[.]82:8000/http/192he23/update` | macOS RAT binary |

The domain `fasterxml.org` was registered December 17, 2025 via GoDaddy (8 days before discovery) and last updated December 22, 2025.

---

## Timeline

| Date/Time (UTC) | Event |
|---|---|
| 2025-12-17 | C2 domain `fasterxml.org` registered via GoDaddy |
| 2025-12-22 | `fasterxml.org` domain updated (likely infrastructure configuration) |
| 2025-12-25 | Malicious `org.fasterxml.jackson.core:jackson-databind` detected on Maven Central by Aikido Security |
| 2025-12-25 | Aikido reports to Maven Central and GoDaddy |
| 2025-12-25 + ~1.5h | Package removed from Maven Central; GoDaddy moves to suspend domain |
| 2025-12-25 | Aikido publishes technical analysis |

---

## Detection

```bash
# Search for the malicious group ID in Maven project files
grep -r "org\.fasterxml\.jackson\.core" pom.xml */pom.xml 2>/dev/null

# Gradle projects
grep -r "org\.fasterxml\.jackson\.core" build.gradle build.gradle.kts **/*.gradle 2>/dev/null

# Check local Maven cache for the fake artifact
ls ~/.m2/repository/org/fasterxml/jackson/core/ 2>/dev/null
# LEGITIMATE cache path is: ~/.m2/repository/com/fasterxml/jackson/core/
# Any content under org/fasterxml/ is suspicious

# Search for the Spring autoconfiguration class name in installed JARs
find ~/.m2/repository/org/fasterxml/ -name "*.jar" -exec jar tf {} \; 2>/dev/null | \
  grep -i "JacksonSpringAutoConfiguration"

# Check for the persistence marker file
find ~ -name ".idea.pid" -not -path "*/.idea/*" 2>/dev/null
# Legitimate .idea.pid is only inside .idea/ project directories

# Check for payload download artifacts in temp directories
ls /tmp/payload.bin 2>/dev/null
ls $TMPDIR/payload.bin 2>/dev/null  # macOS
ls %TEMP%\payload.bin 2>/dev/null   # Windows

# Check for suspicious processes (Windows: svchosts.exe misspelling)
# Windows PowerShell:
Get-Process | Where-Object { $_.Name -like "svchosts*" }

# Check network logs for C2 communication
grep -E "fasterxml\.org|103\.127\.243\.82" /var/log/syslog /var/log/dns*.log 2>/dev/null
# macOS:
grep "fasterxml.org" /var/log/system.log 2>/dev/null
# Windows:
ipconfig /displaydns | findstr "fasterxml"

# Scan gradle cache as well
find ~/.gradle/caches -path "*org/fasterxml*" 2>/dev/null | grep -v "com/fasterxml"
```

---

## Remediation

1. **Audit all Maven/Gradle dependency declarations** for `org.fasterxml.jackson.core` and replace with the legitimate `com.fasterxml.jackson.core`:
   ```xml
   <!-- MALICIOUS — remove this -->
   <groupId>org.fasterxml.jackson.core</groupId>

   <!-- LEGITIMATE — use this -->
   <groupId>com.fasterxml.jackson.core</groupId>
   ```

2. **Purge the malicious artifact from your local Maven cache**:
   ```bash
   rm -rf ~/.m2/repository/org/fasterxml/
   ```

3. **Rebuild all projects** from clean dependency resolution to ensure no cached malicious JARs are used.

4. **Block C2 infrastructure at DNS and firewall levels**:
   - `fasterxml.org` and `*.fasterxml.org`
   - `103.127.243.82` (all ports)

5. **Check for persistence marker** — the `.idea.pid` file outside of `.idea/` directories indicates the payload ran:
   ```bash
   find ~ -name ".idea.pid" -not -path "*/.idea/*"
   ```

6. **Hunt for the RAT payload**:
   - Windows: Kill any `svchosts.exe` processes; check `%APPDATA%`, `%TEMP%`, and startup locations
   - macOS: Check `/tmp/`, `~/Library/LaunchAgents/`, and running processes for unsigned binaries named `update`
   - Both: Check for scheduled tasks or cron jobs pointing to unfamiliar binaries

7. **Assume full machine compromise** if the payload executed — rotate all credentials, API keys, SSH keys, and secrets accessible from the affected environment.

8. **Enable Maven artifact signature verification** in your build tooling. Maven Central supports PGP signing; configure your build to reject unsigned or improperly signed artifacts.

---

## Lessons Learned

- **Maven Central is not immune to supply chain attacks.** This was the first sophisticated malware discovered on Maven Central, shattering the assumption that the registry's signing and review processes make it inherently safer than npm or PyPI. Group ID namespace squatting (`org.` vs `com.`) is a viable and previously underexploited attack surface.
- **Spring Boot autoconfiguration is a silent code execution vector.** `META-INF/spring.factories` causes classes to be instantiated at startup without any developer action. Any malicious JAR that includes a `@Configuration` class via this mechanism auto-executes in Spring Boot applications.
- **AES-ECB with hardcoded keys provides obfuscation, not security.** The encryption conceals the payload URL from static analysis but is trivially reversible with the embedded key. Defenders should treat hardcoded decryption keys in third-party JARs as an immediate red flag.
- **LLM prompt injection in source code is an emerging evasion technique.** Embedding Chinese-language strings designed to mislead AI code analyzers represents a novel countermeasure against the growing use of LLMs in supply chain security tooling. Security teams should be aware that AI-based scanners can be deceived by adversarially crafted package content.
- **1.5-hour exposure can still cause significant harm.** Although Maven Central and GoDaddy responded quickly, any organization that pulled the dependency during that window — or still has it cached locally — remains vulnerable.
- **Group ID typosquatting is hard to catch visually.** `org.fasterxml.jackson.core` vs `com.fasterxml.jackson.core` is extremely easy to overlook in a `pom.xml` file. Automated dependency scanning tools that flag namespace anomalies are essential.

---

## Related Incidents

- [./2025-12-neoshadow-npm.md](./2025-12-neoshadow-npm.md) — npm typosquatting campaign with multi-stage modular backdoor and 0 VirusTotal detections
- [./2026-01-dydx-npm-pypi.md](./2026-01-dydx-npm-pypi.md) — Cross-ecosystem (npm + PyPI) compromise with RAT and wallet stealer
- [./2025-04-xrp-supply-chain.md](./2025-04-xrp-supply-chain.md) — Official crypto SDK backdoored with private key exfiltration
- [./2025-08-infostealer-campaign.md](./2025-08-infostealer-campaign.md) — npm infostealer campaign targeting Chrome credentials and crypto wallets
