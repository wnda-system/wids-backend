# Alerts and rules in the proof of concept

Source: edge repo `POC_Mo/mo_app/monitor.py`, `analysis/detector.py`,
`detection_rules.md` (Thai, full detail).

## Detectors

| Detector | What it is | Says which attack? |
|---|---|---|
| **A** | Isolation Forest (300 trees, 256 samples each) over all 36 features; alert when the score is above the 99.5th percentile of the training scores | No — "unusual combination (type unknown)", plus the 3 most deviating features as hints |
| **B** | Per-feature rules, thresholds recomputed from the baseline at every refit | Yes |
| **A+B** | Both fired in the same second | Yes (from B) |

Both are refitted every 10 minutes on the stored per-second features.

## Rule types (detector B)

| Rule | Features | Fires when |
|---|---|---|
| First occurrence | `new_bssid_known_ssid` (evil twin), `deauth_seq_out_of_order` (spoofed deauth), `arp_ip_mac_flips_10s` (ARP spoofing, while the baseline has none) | value > 0 |
| Flood | `deauth`, `disassoc`, `auth`, `assoc_req` (floor 10/s), `deauth_to_one_client_10s` (5), `tsf_offset_switches` (3), `arp_ips_per_mac_60s` (3) | value > max(baseline max × 1.2, floor) |
| Above baseline | every other feature | robust z-score above the largest seen in the baseline × 1.2 (only increases count) |

Eight general features (`signal_mean`, `len_mean`, `mgmt`, `ctrl`, `data`,
`protected_ratio`, `n_dest_mac`, `tcp`) only produce an **info** notice, not
an alarm: a phone next to the sensor or a big download looks the same.

"Protected" SSIDs = seen in ≥ 90 % of 5-minute chunks; only those can trigger
evil-twin alerts.

## Alert event (one line of `alerts.jsonl`)

```json
{"start": 1792825316, "end": 1792825316, "seconds": 1, "detector": "B",
 "types": ["ARP spoofing (one MAC claiming many IPs)"],
 "features": {"arp_ips_per_mac_60s": 4.0},
 "explain": ["arp_ips_per_mac_60s=4.00↑ (normal≈0.00, z=+12.9) → ARP spoofing (one MAC claiming many IPs)"],
 "peak_score": 0.455, "confidence": "rule"}
```

- **Merging:** a second joins the previous event if it is ≤ 3 s after its end
  and has the same detector and confidence; `types` are unioned, `peak_score`
  is the max. A newly seen BSSID always starts its own event (with
  `details.new_bssids`).
- **confidence:** `rule` (an attack-specific rule fired), `info` (only general
  features), `normal` / `low (still learning)` (model A alone, after / before
  it is trusted).

## Keeping the baseline clean

- Seconds (±5 s) flagged by a rule, or by model A once trusted, are never
  trained on. Model A's alerts while still learning are not excluded (it is
  noisy then and would exclude almost everything).
- A new BSSID announcing a protected SSID raises **one** alert and goes to a
  pending list; it is never learned as known until a person approves it.
  A campus AP shows up as ~5 BSSIDs (one per SSID) with consecutive MACs.
- Training data can be limited to the last N days / chosen days, with
  excluded periods and a "pause learning" switch for attack tests.

## Trust and false-alarm rate

- False alarms are measured on *unseen* data only: seconds judged by a model
  fitted before they existed; the rate is the median of hourly rates over 24 h.
  `info` notices and A+B events are not counted.
- Model A is trusted at Brain Power ≥ 50 and ≤ 5 alarms/hour (≥ 20 h
  measured), and loses trust only after failing for 24 h straight, so a real
  attack burst cannot knock it back into learning mode.
- Brain Power = 50 % week coverage (each hour-of-week slot full at 30 min of
  data) + 25 % threshold stability + 25 % false-alarm score (≤ 1/h = 100 %,
  ≥ 30/h = 0 %).

## What the backend could keep

- Rules as a third alert source next to the WIDS and performance models: they
  are the only detector for ARP spoofing / mDNS floods, cheap and explainable.
- Event merging, confidence levels and acknowledgement in the `alerts` table.
- Per-class enable switches for learned models (Model C needed three classes
  switched off on our traffic).
