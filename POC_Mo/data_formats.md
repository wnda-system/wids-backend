# Data formats in the proof of concept

## Per-second features (`FEATURE_VERSION` 2)

One row per second, computed on the sensor by `analysis/features.py:windows()`.
No MAC or IP addresses leave this stage.

| Group | Features | From raw fields |
|---|---|---|
| Frame counts | `frames`, `mgmt`, `ctrl`, `data`, `beacon`, `probe_req`, `auth`, `assoc_req`, `deauth`, `disassoc`, `rts`, `cts` | `wlan.fc.type`, `wlan.fc.subtype` (deauth/disassoc retransmissions removed with `wlan.ta`, `wlan.ra`, `wlan.seq`) |
| Ratios / radio | `retry_ratio`, `protected_ratio`, `signal_mean`, `signal_std`, `len_mean` | `wlan.fc.retry`, `wlan.fc.protected`, `wlan_radio.signal_dbm`, `frame.len` |
| Distinct addresses | `n_ssid`, `n_bssid`, `n_sa`, `n_dest_mac` | `wlan.ssid`, `wlan.bssid`, `wlan.sa`, `wlan.ra` |
| Upper layers (open networks) | `arp`, `arp_reply_ratio`, `dhcp`, `mdns`, `ssdp`, `tcp` | `arp.opcode`, `*.present`, `tcp.flags.syn` |
| Evil twin | `max_bssid_per_ssid`, `new_bssid_known_ssid`, `tsf_offset_switches`, `bssid_signal_range`, `bssid_beacon_rate` | beacons: `wlan.ssid`, `wlan.bssid`, `wlan.fixed.timestamp`, `radiotap.mactime`, signal |
| Deauth / ARP spoofing | `deauth_seq_out_of_order`, `deauth_to_one_client_10s`, `arp_ip_mac_flips_10s`, `arp_ips_per_mac_60s` | `wlan.seq`, `wlan.ta`, `wlan.ra`, `arp.src.hw_mac`, `arp.src.proto_ipv4` |

`new_bssid_known_ssid` and `arp_*` need state learned by the sensor (known
SSID/BSSID pairs, IP→MAC bindings), so they cannot be recomputed on the server
from the features alone.

## Capture sidecar (`capture_<time>.json`, next to every raw CSV)

```json
{"source": "capture", "profile": "field", "experiment": "test",
 "iface": "wlx1cbfce71ebe7", "channel": "100", "width": "HT40+",
 "channel_info": {"channel": 100, "freq_mhz": 5500, "width_mhz": 40, "center1_mhz": 5510},
 "raw_schema_version": 3, "feature_version": 2,
 "git_revision": "ad551df0c7ac4d8315f052251a298bf2e704e973",
 "llc_snap_dispatch": true,
 "buffer": "one continuous file per session; each second written once complete (1 s grace)",
 "ground_truth": "capture_20261024_150614_ground_truth.csv",
 "started_at": 1792829174.12, "ended_at": 1792829294.18,
 "first_frame": 1792829174, "last_frame": 1792829294, "frames": 13932}
```

Monitor chunks (`source: "monitor"`, 5 minutes each) have the same fields
minus `profile`/`experiment`/`ground_truth`, with `first_second` /
`last_second`. `channel_info` is read back from `iw dev <iface> info`.

Raw schema: 2 = missing values are empty cells; 3 = field profile, `wlan.da`
follows the DS bits like Wireshark. Every other column is the same.

## Ground truth (`capture_<time>_ground_truth.csv`, field profile)

```csv
start_epoch,end_epoch,attack,notes
1792598471.808,1792598473.828,Deauth,PMF off; aireplay -0 10
```

Times are the sensor's clock (same as `frame.time_epoch`); `attack` uses the
AWID3 class names; a file with only the header means the whole capture is
normal traffic.
