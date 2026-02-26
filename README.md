# Security Sentinel

Autonomous security scanning agent for VPS and web infrastructure. Runs 60–80+ automated security checks across your entire stack — VPS, local machine, API endpoints, dependencies, TLS, secrets, and compliance — with AI-powered analysis.

> Drop this repo into Claude Code and say "set up Security Sentinel for my infrastructure" to get started.

---

## Quick Start

```bash
# 1. Clone and install
git clone https://github.com/tk-47/claudebot-security-sentinel.git
cd claudebot-security-sentinel
cp .env.example .env
# Edit .env with your VPS URL, SSH details, etc.

# 2. Install CLI tools (optional — gracefully skipped if missing)
brew install nmap nuclei testssl trufflehog trivy
ollama pull qwen3:8b

# 3. Run your first scan
bun run scan
```

## Scan Modes

| Mode | Checks | AI Review | Duration |
|------|--------|-----------|----------|
| `scan` | ~55 core checks | None | ~15s |
| `hourly` | ~65 checks + recon | Ollama (local) | ~20s |
| `daily` | ~75 checks + CLI tools | Ollama (local) | ~3 min |
| `deep` | ~80+ all checks | Claude API | ~5 min |

```bash
bun run scan           # Quick deterministic scan
bun run hourly         # + external recon + AI review
bun run daily          # + nmap/nuclei/trufflehog/trivy
bun run deep           # + testssl.sh + Claude API analysis
bun run compliance     # Compliance report from last scan
bun run install-check  # Verify tool installation
```

Every scan produces a JSON report in `logs/security/` with diff tracking against the previous scan. New critical/high findings trigger a Telegram alert (if configured). All findings map to SOC2, HIPAA, and PCI-DSS compliance controls.

---

## Configuration

All configuration is via environment variables. Copy `.env.example` to `.env` and fill in your values. Only `VPS_URL` is required — everything else has sensible defaults.

### Key Settings

| Variable | Required | Description |
|----------|----------|-------------|
| `VPS_URL` | Yes | Public URL of your VPS |
| `VPS_SSH_HOST` | Yes | VPS IP address for SSH + nmap |
| `VPS_SSH_USER` | No | SSH username (default: `deploy`) |
| `VPS_SSH_KEY` | No | SSH key path (default: `~/.ssh/id_ed25519`) |
| `VPS_PROJECT_DIR` | No | Project directory on VPS |
| `DOMAIN` | No | Your domain (defaults to VPS_URL hostname) |
| `WEBHOOK_ENDPOINTS` | No | Comma-separated webhook paths to test auth on |
| `PM2_PROCESS_NAME` | No | PM2 process name to check health |
| `TELEGRAM_BOT_TOKEN` | No | For alert notifications |
| `ANTHROPIC_API_KEY` | No | For Claude deep scan analysis |

See `.env.example` for the full list.

---

## What It Checks

### Authentication Verification
Sends unsigned/invalid requests to every webhook endpoint. Verifies rejection with proper HTTP status codes.

### Rate Limiting
Fires rapid requests to confirm rate limiter activates at threshold.

### Information Disclosure
Checks for leaked server info, framework identifiers, CORS misconfiguration, and sensitive data in responses.

### Security Headers (OWASP)
Verifies X-Frame-Options, X-Content-Type-Options, Referrer-Policy, Permissions-Policy, and Content-Security-Policy.

### TLS / SSL
Certificate validity, expiry warning (30 days), HSTS header. Deep mode adds full testssl.sh audit (protocol versions, cipher suites, known vulnerabilities).

### Web Application Testing
HTTP method rejection, sensitive path exposure (`.env`, `.git`, `node_modules`), stack trace leakage.

### Input Validation / Fuzzing
Path traversal, SQL injection patterns, oversized payloads.

### VPS Hardening (via SSH)
Root login, password auth, firewall, .env permissions, process isolation, PM2 health, disk/memory/CPU, fail2ban, CrowdSec, patch status, runtime versions.

### Local Security
Subprocess permission checks, .env permissions, SSH key age, gateway secret entropy.

### API Credential Validation
Tests Telegram, Anthropic, Convex, MS365, and OpenAI tokens against their respective APIs.

### External Recon (hourly+)
Shodan InternetDB, certificate transparency (crt.sh), DNS record integrity, OSV.dev dependency CVEs.

### CLI Security Tools (daily+)
nmap port scanning, Nuclei vulnerability templates (11k+), TruffleHog verified secrets, Trivy filesystem scanning, testssl.sh deep TLS audit.

---

## Compliance Mapping

Every finding maps to compliance controls:

| Framework | Controls Covered |
|-----------|-----------------|
| **SOC2** | CC6.1, CC6.2, CC6.6, CC7.1, CC7.2, CC8.1 |
| **HIPAA** | 164.312(a)(c)(d)(e), 164.308(a)(1)(5) |
| **PCI-DSS v4.0** | 1.3, 2.2, 6.4, 7.1, 8.3, 8.6, 10.2, 10.3, 11.3, 11.4 |

Run `bun run compliance` to generate a formatted compliance report from the last scan.

---

## AI Review

### Hourly/Daily — Ollama (Local, Free)
Uses a local AI model (default: qwen3:8b) to analyze failures, identify attack chains, and recommend remediation. No data leaves your machine.

### Deep — Claude API (On-Demand, ~$0.05-0.10/scan)
Full expert analysis including risk assessment, attack chain analysis, compliance gap assessment, remediation plan, and baseline comparison.

---

## Scheduling (macOS launchd)

Create launchd plists for automated scanning:

**Hourly** (`~/Library/LaunchAgents/com.sentinel.security-hourly.plist`):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key><string>com.sentinel.security-hourly</string>
  <key>ProgramArguments</key>
  <array>
    <string>/path/to/bun</string>
    <string>run</string>
    <string>src/index.ts</string>
    <string>hourly</string>
  </array>
  <key>WorkingDirectory</key><string>/path/to/claudebot-security-sentinel</string>
  <key>StartCalendarInterval</key>
  <dict><key>Minute</key><integer>0</integer></dict>
  <key>StandardOutPath</key><string>/tmp/sentinel-hourly.log</string>
  <key>StandardErrorPath</key><string>/tmp/sentinel-hourly.log</string>
</dict>
</plist>
```

---

## Architecture

```
src/
  index.ts          # Main orchestrator + CLI entry point
  config.ts         # Configuration from environment variables
  lib/
    scanner.ts      # Core deterministic checks (~55 checks)
    recon.ts        # External recon APIs (Shodan, crt.sh, DNS, OSV.dev)
    tools.ts        # CLI tool wrappers (nmap, nuclei, testssl, trufflehog, trivy)
logs/
  security/         # One JSON report per scan, timestamped
```

### Data Flow

```
                    ┌──────────────┐
  CLI / Scheduler   │  index.ts    │  ← Orchestrator
                    └──┬───┬───┬──┘
                       │   │   │
          ┌────────────┘   │   └────────────┐
          ▼                ▼                ▼
  ┌───────────────┐ ┌──────────────┐ ┌──────────────┐
  │ scanner.ts    │ │ recon.ts     │ │ tools.ts     │
  │ 55 checks     │ │ Shodan       │ │ nmap         │
  │ Auth, TLS,    │ │ crt.sh       │ │ Nuclei       │
  │ VPS, Local,   │ │ DNS (DoH)    │ │ testssl.sh   │
  │ Headers, etc. │ │ OSV.dev      │ │ TruffleHog   │
  └───────┬───────┘ └──────┬───────┘ │ Trivy        │
          │                │         └──────┬───────┘
          └────────┬───────┘                │
                   ▼                        │
           ┌──────────────┐                 │
           │  Ollama      │◄────────────────┘
           │  (local AI)  │  hourly/daily review
           └──────────────┘
           ┌──────────────┐
           │  Claude API  │  deep scan only
           └──────────────┘
                   │
                   ▼
           ┌──────────────┐
           │  Telegram    │  alerts on critical/high
           └──────────────┘
```

---

## Customization

### Adding a New Check

Every check follows this pattern in `scanner.ts`:

```typescript
results.push(ok(
  "unique_check_id",     // Unique identifier for diff tracking
  "CATEGORY",            // Category for grouping
  "Human-readable name", // What the check tests
  true_or_false,         // Pass/fail boolean
  "medium",              // Severity: critical, high, medium, low, info
  "Details about result", // Explanation
  ["SOC2:CC6.1"],        // Compliance control tags
  "optional evidence"     // Raw evidence string
));
```

### Configuring Webhook Endpoints

Set `WEBHOOK_ENDPOINTS` in `.env`:
```bash
# Format: /path:METHOD:auth_type (comma-separated)
WEBHOOK_ENDPOINTS=/telegram:POST:signature,/deploy:POST:signature,/api/messages:POST:signature
```

### Switching AI Models

```bash
SECURITY_OLLAMA_MODEL=llama3:8b        # Any Ollama model
SECURITY_CLAUDE_MODEL=claude-sonnet-4-5-20250929  # Any Anthropic model
```

---

## Data Sharing & Privacy

| Data | Destination | Can Disable? |
|------|------------|-------------|
| VPS IP | Shodan InternetDB | Yes — remove from recon |
| Domain | crt.sh, Google DNS | Yes — remove from recon |
| Package versions | OSV.dev (Google) | Yes — remove from recon |
| Scan results | Anthropic API (deep only) | Yes — don't use `deep` mode |

**Never shared:** Source code, .env contents, API keys, SSH credentials, file contents.

---

## License

MIT
