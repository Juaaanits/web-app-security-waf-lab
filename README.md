# DVWA + SafeLine WAF Security Lab

A hands-on web application security lab demonstrating how a Web Application Firewall (WAF) can detect, block, and log common web attacks against a deliberately vulnerable application.

The project uses **DVWA**, **SafeLine WAF**, **Apache**, and **Kali Linux** in an isolated homelab environment.

---

## Architecture

```text
Kali Linux
   │
   │ HTTPS :8443
   ▼
SafeLine WAF
   │
   │ Reverse Proxy
   ▼
127.0.0.1:8080
   │
   ▼
Apache / DVWA
```

The Ubuntu host also runs **Traefik** for the existing homelab:

```text
443   → Traefik
8443  → SafeLine WAF
9443  → SafeLine Management
8080  → DVWA backend (loopback only)
```

### Backend Hardening

DVWA originally listened on:

```text
*:8080
```

which allowed clients to bypass the WAF directly.

Apache was hardened to listen only on:

```text
127.0.0.1:8080
```

SafeLine now proxies requests internally to:

```text
http://127.0.0.1:8080
```

Direct LAN access to the DVWA backend is no longer possible.

---

## Security Controls Tested

### SQL Injection Protection

A SQL injection payload was submitted through the protected endpoint:

```text
https://dvwa.local:8443
```

SafeLine detected the request as **SQL Injection** and blocked it before it reached DVWA.

Verified through:

- SafeLine block page
- Attack logs
- Source IP logging
- SQL Injection classification

---

### HTTP Flood / Rate Limiting

SafeLine was configured with a controlled access threshold.

ApacheBench generated:

```text
50 requests
5 concurrent clients
~82 requests/second
```

SafeLine triggered its Anti-Bot / rate-limiting protection and challenged subsequent requests.

Normal access returned automatically after the configured penalty period.

---

### Authentication Gateway

SafeLine authentication was enabled in front of DVWA.

Request flow:

```text
Client
  ↓
SafeLine Authentication
  ↓
SafeLine WAF
  ↓
DVWA Authentication
```

Only authorized SafeLine users can reach the protected application.

---

### Custom IP Deny Rule

A deny rule was created for the Kali test client.

SafeLine successfully blocked the source IP before the request reached DVWA.

---

## WAF Bypass Remediation

Before hardening:

```text
Kali
  ↓
192.168.1.17:8080
  ↓
DVWA
```

This bypassed SafeLine completely.

After hardening:

```text
Kali → 192.168.1.17:8080  ✗

Kali
  ↓
SafeLine :8443
  ↓
127.0.0.1:8080
  ↓
DVWA
```

Validation:

```bash
# Backend works locally on Ubuntu
curl -I http://127.0.0.1:8080/DVWA/login.php

# Protected route remains available
curl -k -I https://dvwa.local:8443/DVWA/login.php

# Direct LAN bypass fails
curl -I --connect-timeout 5 \
  http://192.168.1.17:8080/DVWA/login.php
```

---

## Root Cause: SafeLine vs Traefik Port Conflict

SafeLine was initially configured on HTTPS port `443`.

However, `443` was already being used by Traefik.

Troubleshooting with:

```bash
curl -k -v https://dvwa.local/DVWA/login.php
```

revealed:

```text
subject: CN=TRAEFIK DEFAULT CERT
HTTP/2 404
```

This proved the request was reaching **Traefik instead of SafeLine**.

SafeLine was moved to:

```text
8443/HTTPS
```

Afterward:

```text
CN=dvwa.local
HTTP/1.1 200 OK
Set-Cookie: sl-session=...
```

confirmed that the request was reaching SafeLine correctly.

---

## Technologies

- SafeLine WAF 9.4.1
- Damn Vulnerable Web Application (DVWA)
- Apache HTTP Server
- Kali Linux
- Ubuntu Server
- Docker / Docker Compose
- Traefik
- VMware Workstation
- curl
- ApacheBench

---

## Key Takeaways

This project demonstrates:

- Reverse proxy and WAF architecture
- TLS and port troubleshooting
- SQL injection detection
- HTTP flood protection
- Authentication and authorization controls
- IP-based access control
- WAF logging and attack investigation
- Backend isolation
- Prevention of direct WAF bypass
- Layer-by-layer troubleshooting using `curl`

A major lesson from the lab was that an HTTP status code alone does not identify which component generated the response. Inspecting the TLS certificate with `curl -v` exposed the Traefik port conflict and led directly to the root cause.

---

## Future Improvements

- Configure Traefik TLS passthrough:

```text
Client
  ↓
Traefik :443
  ↓
SafeLine :8443
  ↓
DVWA
```

This would allow:

```text
https://dvwa.local
```

without exposing the non-standard port.

- Test additional DVWA vulnerabilities:
  - XSS
  - Command Injection
  - File Inclusion
  - CSRF
- Forward SafeLine logs to a SIEM
- Integrate IDS/IPS monitoring
- Add dashboards and alerting
- Deploy additional vulnerable applications such as OWASP Juice Shop

---

## Disclaimer

This project was built exclusively in an isolated homelab using **DVWA**, an intentionally vulnerable application designed for cybersecurity training.

All security testing was performed against systems owned and controlled by the lab operator.
