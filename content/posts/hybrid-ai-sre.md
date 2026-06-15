---
title: "Hybrid AI for SRE: Keeping LLMs Out of Decision Loops"
date: 2026-06-15
tags: ["sre", "llm", "automation", "homelab"]
description: "Three real agents that use LLMs for narrative and Python for decisions — and why that split matters."
showToc: true
---

Somewhere between "no AI at all" and "LLM decides everything," there's a pattern that actually works in production: let Python make the call, then let the LLM explain it. I've been running three autonomous SRE agents on local Ollama models for several months, and every one of them follows this structure. Here's what it looks like and why it holds up.

## The Temptation

When you first start wiring LLMs into operational tooling, the obvious move is to hand the model a pile of metrics and ask it to decide what's wrong:

```python
# what you think you want
response = await llm(f"Here are my metrics: {metrics}. Is anything critical?")
severity = parse_response(response)
send_alert(severity)
```

It feels like the right call — LLMs are good at synthesis, and you're offloading cognition to a system that can "understand" context. The problem is that you've now put your alerting pipeline on the critical path of an inference call that can be slow, expensive, and wrong in ways that are hard to detect.

**Four failure modes that show up fast:**

1. **LLM is down, alerting is down.** If your Ollama node is unreachable, OOM'd, or mid-restart, you get silence instead of a page. Operational correctness should never depend on LLM availability.

2. **Hallucinated severity.** A model might look at 95% RAM and say "this appears to be within normal operating parameters" because it's drawing on general knowledge about servers. Your threshold is *not* general — it's specific to your fleet.

3. **Parsing fragility.** The model returns `"CRITICAL"`, `"critical!"`, `"Critical."`, or `"The system appears to be in a critical state."` You have to handle all of them, and you'll miss one eventually.

4. **Auditing is impossible.** When an alert fires at 3am, you want a straight line from observed value → threshold → severity. "The LLM said so" is not a root cause.

## The Pattern: Detect → Decide → Explain

The structure I landed on separates these three concerns cleanly:

1. **Detect with code** — collect metrics from monitoring APIs
2. **Decide with code** — compare against hardcoded thresholds, produce structured `Alert` or `Issue` objects with `severity` already set
3. **Explain with LLM** — take the already-decided list and generate a one-line summary for the notification title

The LLM never sees raw metrics. It only sees a pre-filtered, pre-scored list of things Python already decided were problems. Its job is cosmetic: turn `[CRITICAL] dionysus RAM: 95.2%; [WARNING] archive storage: 98% full` into `"RAM critical on dionysus, archive at limit"` for the ntfy title.

If the LLM is down, the alert still fires — it just uses a fallback string. The LLM's output cannot affect whether or when an alert is sent.

---

## Example 1: SRE Patrol — Thresholds Without a Net

`agent-sre-patrol` monitors two Unraid NAS nodes. It checks array state, disk health, RAM usage, and UPS battery. All decisions happen in `_check_thresholds()`:

```python
def _check_thresholds(metrics: dict) -> tuple[list[Alert], list[str]]:
    """Pure Python threshold checks — no LLM required for severity."""
    alerts: list[Alert] = []
    healthy: list[str] = []

    for node_key in ("dionysus", "archive"):
        n = metrics[node_key]
        name = n["node"]

        # Array state
        state = n.get("array_state", "unknown").upper()
        if state not in ("STARTED", "NORMAL"):
            alerts.append(Alert(
                severity="critical", component=f"{name} array",
                message="Array state is not Started",
                observed_value=state,
                threshold="STARTED",
            ))

        # RAM
        ram_pct = n.get("ram_used_pct", 0)
        if ram_pct > 95:
            alerts.append(Alert(
                severity="critical", component=f"{name} RAM",
                message="RAM critically high",
                observed_value=f"{ram_pct}%",
                threshold=">95%",
            ))
        elif ram_pct > 90:
            alerts.append(Alert(
                severity="warning", component=f"{name} RAM",
                message="RAM usage high",
                observed_value=f"{ram_pct}%",
                threshold=">90%",
            ))

        # UPS runtime
        ups_runtime = n.get("ups_runtime_min")
        if isinstance(ups_runtime, (int, float)):
            if ups_runtime < 10:
                alerts.append(Alert(severity="critical", component=f"{name} UPS",
                                    message="UPS runtime critically low",
                                    observed_value=f"{ups_runtime} min", threshold="<10 min"))
            elif ups_runtime < 15:
                alerts.append(Alert(severity="warning", component=f"{name} UPS",
                                    message="UPS runtime low",
                                    observed_value=f"{ups_runtime} min", threshold="<15 min"))

    return alerts, healthy
```

The `severity` field on every `Alert` is determined by an `if/elif` chain. No model touches it. The thresholds (`>95%`, `<10 min`) are visible in the code, reviewable in a PR, and version-controlled.

After `_check_thresholds()` runs, then and only then does the LLM run:

```python
async def _summarise(alerts: list[Alert], healthy: list[str]) -> str:
    if not alerts:
        return "All systems nominal"
    client = AsyncOpenAI(base_url=OLLAMA_URL, api_key="ollama")
    alert_text = "; ".join(
        f"[{a.severity.upper()}] {a.component}: {a.observed_value}"
        for a in alerts
    )
    response = await client.chat.completions.create(
        model=PATROL_MODEL,
        messages=[
            {"role": "system", "content": SUMMARY_PROMPT},
            {"role": "user",   "content": f"Alerts: {alert_text}"},
        ],
        temperature=0.1,
        max_tokens=60,
    )
    raw = (response.choices[0].message.content or "").strip().strip('"').strip("'")
    return raw[:120] or "Patrol complete — see alerts"
```

The LLM input is structured: `"[CRITICAL] dionysus RAM: 95.2%; [WARNING] archive storage: 14 disks, max 98%"`. It has 60 tokens to produce a title. If it returns nothing, the fallback `"Patrol complete — see alerts"` fires. The `ntfy.sh` call uses `Alert.severity` to set notification priority — that value came from `_check_thresholds()`, not from anything the model said.

---

## Example 2: Media Health — Same Pattern, Different Domain

`agent-media-health` checks NZBGet, Radarr, Sonarr, Lidarr, and Plex/Tautulli. The same split applies. Here's the Lidarr check:

```python
def _check_lidarr(queue: dict, wanted: dict) -> list[Issue]:
    issues: list[Issue] = _check_arr_queue("Lidarr", queue)

    total = wanted.get("totalRecords", 0) if isinstance(wanted, dict) else 0
    if total > 100:
        records = wanted.get("records", [])
        titles = ", ".join(r.get("title", "?")[:40] for r in records[:3])
        issues.append(Issue(severity="critical", component="Lidarr wanted",
                            message=f"{total} album(s) missing/wanted",
                            detail=titles + ("..." if total > 3 else "")))
    elif total > 20:
        issues.append(Issue(severity="warning", component="Lidarr wanted",
                            message=f"{total} album(s) missing/wanted",
                            detail=""))

    return issues
```

`>100` missing albums is critical. `>20` is a warning. Those are operational judgements baked into code — not something I want a model re-evaluating on every run.

The LLM's role in this agent is identical to SRE patrol: summarize the pre-scored issue list into a notification title. It sees `"[CRITICAL] Lidarr wanted: 150 album(s) missing/wanted"` and writes something human-readable. The alert fires (or doesn't) based on `Issue.severity`.

---

## Example 3: Network Sentinel — No LLM Required

`agent-network-sentinel` watches for unknown MAC addresses on the UniFi network. The decision is binary — either a MAC is in `known-devices.json` or it isn't. There's no threshold to second-guess, no nuance to summarize. So there's no LLM:

```python
def _check(clients: list[dict], known: dict, state: dict, now: str) -> list[dict]:
    """Return list of newly-alerted unknown devices."""
    unknown_state = state.setdefault("unknown", {})
    newly_alerted = []

    for c in clients:
        mac = c.get("mac", "").lower()
        if not mac:
            continue

        if mac in known:         # allowlist check — simple, fast, auditable
            continue

        if mac not in unknown_state:
            unknown_state[mac] = {
                "first_seen": now,
                "last_seen":  now,
                "hostname":   c.get("hostname", ""),
                "ip":         c.get("ip", ""),
                "alerted":    False,
            }

        entry = unknown_state[mac]
        if not entry["alerted"]:    # fire once per device
            entry["alerted"] = True
            newly_alerted.append({**entry, "mac": mac})

    return newly_alerted
```

Two decisions, both deterministic: `if mac in known` (line 120) and `if not entry["alerted"]` (line 140). The alert body is a plain table of MAC / hostname / IP. Asking an LLM whether an unknown MAC address is "suspicious" would add latency, cost, and hallucination risk for zero benefit — the policy is already clear.

This is worth making explicit: **the right question isn't "can the LLM help here?" — it's "does the LLM add value that Python can't provide?"** For a binary allowlist check, it doesn't.

---

## Decision Matrix

| Check | Decision Type | Where Logic Lives | Why Not LLM |
|---|---|---|---|
| Array state (STARTED/NORMAL) | Binary | Python `if state not in (...)` | Unambiguous; must be fast |
| RAM usage >95% | Threshold | Python `if ram_pct > 95` | Threshold is operational fact |
| UPS runtime <10 min | Threshold | Python `if ups_runtime < 10` | Safety-critical; no tolerance for drift |
| Lidarr missing albums >100 | Threshold | Python `if total > 100` | Known acceptable backlog |
| Unknown MAC on network | Allowlist | Python `if mac in known` | Binary; allowlist is source of truth |
| Alert title for ntfy | Narrative | LLM (temperature=0.1) | Cosmetic; failure doesn't break alerting |

---

## What "LLM Decides" Actually Looks Like

For contrast, here's what the anti-pattern looks like in practice:

```python
# don't do this
async def check_ram(ram_pct: float) -> str:
    response = await llm(
        f"RAM usage is {ram_pct}%. Is this critical, warning, or ok?"
    )
    # response might be "critical", "Critical!", "I'd call this critical",
    # "warning", "This seems fine for a busy server", or None
    if "critical" in response.lower():
        return "critical"
    elif "warning" in response.lower():
        return "warning"
    return "ok"
```

You're now parsing natural language to get a severity enum. The model might be right most of the time, but "most of the time" is not a property you want in your alerting system. You've also made your alert delivery dependent on LLM latency (5-30 seconds per call vs. <1ms for a Python comparison) and LLM availability.

The moment the model says "This level of memory usage, while elevated, may be within acceptable parameters depending on the workload profile" — and it will — you have a silent failure with no alert.

---

## Why This Pattern Holds Up

After running these agents in production:

- **Fast.** Python threshold checks take microseconds. The LLM only runs after decisions are made and only to generate one line of text. End-to-end latency is dominated by API calls to monitoring systems, not inference.

- **Auditable.** Every alert has `observed_value` and `threshold` fields set in code. When I get a page at 2am, I can trace it: `ram_pct=96.1 → >95 → critical`. No model judgment to unpack.

- **Resilient.** The LLM summarizer has a hardcoded fallback in every agent. If Ollama is down, the alert still fires with "Patrol complete — see alerts" or equivalent. The LLM is enhancement, not infrastructure.

- **Cheap.** I'm generating one 60-token completion per patrol run, after all decisions are made. Not one per metric, not one per alert — one per run, on a local 8B model.

- **Swappable.** The model name is a constant at the top of each file. I've swapped between `hermes3:8b`, `qwen2.5-coder:7b`, and others without touching any decision logic. The narrative quality changes slightly; nothing breaks.

The core constraint is simple: **if the LLM's output can prevent an alert from firing, it's in the decision loop.** Keep it out of there. Let it write sentences.

---

*Source: [homelab-tools](https://github.com/wcollani/homelab-tools) — agent-sre-patrol, agent-media-health, agent-network-sentinel.*
