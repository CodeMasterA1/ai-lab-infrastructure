# Security Posture — AI Lab Infrastructure

**Last updated:** 2026-04-06 | **Environment:** Hetzner VPS, Ubuntu 24.04

  ---

  ## Philosophy: Defense-in-Depth

  This infrastructure applies **defense-in-depth** — multiple independent security layers so that
  no single failure compromises the system. Each layer assumes the previous one can fail.

  The key architectural advantage: **the VPS is not publicly accessible**. All management access
  goes through Tailscale (Zero Trust mesh VPN) before reaching any service. This means a CVE in
  an internal container is significantly less dangerous than in a publicly exposed one.

  ---

  ## The 8 Security Layers

  | Layer | Tool | What it protects | Acts automatically |
  |---|---|---|---|
  | 1 | Hetzner Cloud Firewall | Network perimeter — blocks unwanted ports at the provider level | Yes |
  | 2 | UFW (Linux firewall) | OS-level packet filtering — second line if Hetzner firewall fails | Yes |
  | 3 | Fail2ban | SSH — bans IPs with repeated failed login attempts | Yes |
  | 4 | CrowdSec | Network — blocks IPs flagged by global threat intelligence community | Yes |
  | 5 | Tailscale | Remote access — Zero Trust VPN, only authorized devices can connect | Yes |
  | 6 | SSH keys | Authentication — password login disabled, key-only access | Yes |
  | 7 | Docker network isolation | Lateral movement — containers only talk to who they need to | Yes |
  | 8 | Trivy | Supply chain — daily CVE scan of all Docker images | Yes (alerts only) |

  ---

  ## Layer-by-Layer: What it means in business terms

  ### Layer 1 — Hetzner Cloud Firewall
  Blocks all inbound traffic at the provider level except 3 ports.
  **Business translation:** The front door only has 3 entry points. Everything else is a wall.

  ### Layer 2 — UFW
  Independent OS-level firewall. Even if the cloud provider firewall is misconfigured, UFW applies
  the same rules at the server level.
  **Business translation:** Two locks on the front door from different manufacturers.

  ### Layer 3 — Fail2ban
  After N failed SSH login attempts from the same IP, that IP is
  banned for a configurable period. Generates ~10-100 bans/day from automated bots.
  **Business translation:** Automatic lockout after repeated failed attempts — same as your bank card.

  ### Layer 4 — CrowdSec
  Extends Fail2ban with community threat intelligence. Blocks IPs that have been flagged as
  malicious by thousands of servers globally, before they even attempt a login.
  **Business translation:** Shared intelligence network — if an IP attacked someone in Frankfurt
  yesterday, it's already blocked here today.

  ### Layer 5 — Tailscale (Zero Trust)
  All legitimate management access goes through Tailscale. No service needs to be directly exposed
  to the internet. Implements the Zero Trust principle: "never trust, always verify."
  **Business translation:** No VPN = no access, regardless of knowing the IP or SSH port.

  ### Layer 6 — SSH key authentication
  Password-based SSH login is disabled. Only an ed25519 private key grants access.
  **Business translation:** The physical key analogy — knowing the address isn't enough.

  ### Layer 7 — Docker network isolation
  Each service only connects to the networks it explicitly needs. n8n talks to PostgreSQL and Nginx.
  Pi-hole runs isolated. A compromised container can't freely reach other containers.
  **Business translation:** Compartmentalization — a fire in one room doesn't spread to others.

  ### Layer 8 — Trivy CVE scanning
  Runs daily at 2AM UTC. Scans all Docker images against the CVE database. Sends a Telegram alert
  only if CRITICAL or HIGH severity vulnerabilities are found with an available fix.
  **Business translation:** Daily security audit of every software component, automated.

  ---

  ## Automated Monitoring Stack

  Beyond the security layers, a monitoring system runs continuously:

  | Script | Schedule | Purpose |
  |---|---|---|
  | `daily-report.sh` | 7AM UTC daily | Consolidated report: Trivy, Fail2ban bans, container status, resource usage |
  | `resource-monitor.sh` | Every 15 minutes | Alerts on CPU >80%, RAM >85%, Disk >80%, container crashes |
  | `weekly-log-analyzer.sh` | Sunday 9AM UTC | Auth log analysis: successful logins, new users, Docker error patterns |
  | `docker-update.sh` | First Sunday of month 3AM UTC | Pulls updated Docker images to patch known vulnerabilities |

  All alerts go to Telegram. All logs rotate with 30-day retention in `/var/log/vps-security/`.

  ---

  ## Threat Model

  | Threat | Likelihood | Mitigated by |
  |---|---|---|
  | SSH brute force | High (constant) | Fail2ban + CrowdSec + ed25519 keys |
  | Compromised Docker image (CVE) | Medium | Trivy scanning + weekly image updates |
  | Lateral movement between containers | Low | Docker network isolation |
  | Unauthorized remote access | Low | Tailscale Zero Trust |
  | Data loss (DB corruption) | Low | Daily PostgreSQL backups, 7-day retention |
  | DDoS on public endpoints | Medium | Hetzner + UFW rate limiting |

  ---

  ## What I would do differently at enterprise scale

  1. **Offsite backup replication** — current backups live on the same server. At scale: S3/GCS.
  2. **SIEM integration** — Fail2ban and CrowdSec logs would feed into a SIEM (Splunk, Elastic).
  3. **Secrets management** — API keys in environment variables. At scale: HashiCorp Vault or AWS Secrets Manager.
  4. **IDS/IPS** — Suricata or Snort for deeper packet inspection beyond IP reputation.
  5. **Multi-region redundancy** — single AZ is a single point of failure for availability.

  ---
