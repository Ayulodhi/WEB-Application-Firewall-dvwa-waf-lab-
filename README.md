# Homelab: Attacking DVWA Through a Self-Hosted WAF

A small home-lab project simulating a real attacker vs. defended-target scenario — Kali Linux attacking a deliberately vulnerable web app (DVWA), first with no protection, then again through **SafeLine WAF**, to see exactly what a Web Application Firewall catches and what it misses.

## Architecture

Two isolated environments talking over a bridged network:

![Architecture](screenshots/01_architecture.png)

- **Attacker side:** Kali Linux running as a Docker container on the host machine
- **Target side:** Ubuntu VM (VirtualBox), running DVWA in Docker, fronted by SafeLine WAF

## Traffic Flow

Every request from Kali hits SafeLine first, not DVWA directly:

![Traffic flow](screenshots/02_traffic_flow.png)

```
Kali → SafeLine WAF (:8881, inspects) → allow → DVWA (:80, Docker)
                                       → block/challenge → attacker
```

DVWA itself only listens on `127.0.0.1:80` — unreachable directly from the network — so every request is forced through the WAF. No bypass path.

## Setup Summary

| Component | Tool |
|---|---|
| Attacker OS | Kali Linux (Docker container) |
| Target OS | Ubuntu 22.04 (VirtualBox VM) |
| Vulnerable app | DVWA (Docker) |
| WAF | [SafeLine](https://waf.chaitin.com/) (Docker Compose) |
| Attack tools | `curl`, Apache Bench (`ab`), `hping3`, `nmap` |

WAF sits in front of DVWA as a reverse proxy — attacker traffic hits SafeLine's port (`8881`), which forwards clean requests to DVWA's internal port `80`:

![WAF reverse proxy config](screenshots/03_waf_config.png)

## Experiments & Results

### 1. SYN flood (`hping3 --flood`)
Sent thousands of raw SYN packets at the target.

![SYN flood attempt](screenshots/04_syn_flood_test.png)

**Result: invisible to the WAF.** SafeLine operates at the HTTP layer — raw SYN packets never complete a TCP handshake, so they never register as "requests." Dashboard stayed at 0.

### 2. HTTP flood (`ab -n 2000 -c 100`)
Same idea, but with real, completed HTTP requests.

![Flood traffic stats](screenshots/05_flood_stats.png)

**Result: caught immediately.** 4,000 requests logged, 99.48% error rate — SafeLine's rate-limiting kicked in and started rejecting/challenging the flood mid-attack.

![Rate limit block](screenshots/06_rate_limiting_block.png)

The attacking IP tripped a **"100 requests in 10 seconds"** rule and got hit with a 60-minute Anti-Bot Challenge — automatically, no manual intervention.

### 3. SQL Injection (`' OR '1'='1`)
Sent the classic injection payload against DVWA's SQLi lab, once direct to port `80`, once through SafeLine on `8881`.

**Result:** payload executed cleanly on the unprotected port — but was flagged and blocked as a distinct attack event when routed through the WAF:

![Attack events log](screenshots/07_attack_events.png)

## Key Takeaways

- **A WAF is a traffic filter, not a code fix.** DVWA's vulnerabilities (raw SQL concatenation, unescaped output) are still there — SafeLine just blocks the exploit *in transit*. The real fix is parameterized queries, output encoding, etc.
- **Protocol layer matters.** Attacks that never complete a full HTTP request (like a raw SYN flood) can sail past an HTTP-layer WAF completely — flood protection isn't a silver bullet against every kind of DoS.
- **Rate limiting + signature detection work well together.** Volume attacks got caught by request-rate rules; payload-based attacks (SQLi) got caught by pattern/signature matching — two different detection mechanisms doing two different jobs.
- **Defense in depth beats a single layer.** The best setup here is app-layer fixes *and* the WAF running together, not one instead of the other.

## Disclaimer

This is a personal, fully isolated lab (bridged network between two of my own VMs). DVWA is intentionally vulnerable software — never expose it to a real network or the internet.
