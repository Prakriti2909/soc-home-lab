# Attack-to-Detection Map

Attacker: Kali `192.168.56.102`. Target: Juice Shop behind ModSecurity WAF at `192.168.56.105:8080`, SSH on `192.168.56.105:22`.

| # | Attack | Command / payload | ATT&CK | WAF / log source | Wazuh rule(s) | Result |
|---|---|---|---|---|---|---|
| 1 | Reflected XSS | `curl -i "http://192.168.56.105:8080/?q=<script>alert(1)</script>"` | T1190 | CRS 941 (XSS), 403 | 100204 (lvl 10) | Detected, blocked |
| 2 | SQL injection | `curl -i "http://192.168.56.105:8080/rest/products/search?q=%27%20OR%201%3D1--"` | T1190 | CRS 942 (SQLi), 403 | 100203 (lvl 12) | Detected, blocked |
| 3 | Scanner user-agent | `curl -i -A "sqlmap/1.7" "http://192.168.56.105:8080/"` | T1595.002 | CRS 913 (scanner detection), 403 | 100202 (lvl 10) | Detected, blocked |
| 4 | Path traversal (query string) | `curl -i "http://192.168.56.105:8080/?file=../../../../etc/passwd"` | T1190 | CRS 930 (LFI), 403 | 100205 (lvl 10) | Detected, blocked |
| 5 | SSH brute force (non-existent user) | `hydra -l admin -P /usr/share/wordlists/fasttrack.txt ssh://192.168.56.105 -t 4` (259 attempts) | T1110.001 | sshd via systemd journal | 5710, 5503, 2502, 5758, 5712 (built-in) | Detected |
| 6 | Path traversal (raw path) | `curl -i --path-as-is "http://192.168.56.105:8080/../../etc/passwd"` | n/a | nginx returned 400 before ModSecurity | none | **Not detected (gap)**: request rejected upstream of the WAF, so no log entry exists |
| 7 | Port scan (SYN) | `nmap -sS -p 1-1000 192.168.56.105` | T1046 | Suricata custom rule sid 9000001 (SYN threshold: 20 in 5 s) | 100300 (lvl 10) | Detected (after tuning out one false positive, see below) |

## Custom rule chain

```
100200  ModSecurity JSON event (level 3)
  └─ 100201  WAF returned 403 (level 8, T1190)
        ├─ 100202  CRS 913 scanner detection (level 10, T1595.002)
        ├─ 100203  CRS 942 SQL injection (level 12, T1190)
        ├─ 100204  CRS 941 XSS (level 10, T1190)
        └─ 100205  CRS 930 LFI / path traversal (level 10, T1190)

Suricata sid 9000001  SYN-rate threshold (custom, detections/suricata/local.rules)
86601   Suricata alert (built-in Wazuh rule, eve.json)
  └─ 100300  "LAB Possible Nmap SYN scan" (level 10, T1046)
```

## Tuning

| Rule | Problem | Change | Remaining gap |
|---|---|---|---|
| Suricata sid 9000001 | False positive: browser traffic from the Windows host (`192.168.56.1`) exceeded 20 SYNs in 5 s | Source changed from `any` to `!192.168.56.1` | A real scan from `192.168.56.1` would not alert; slow scans below the threshold also evade it |

Details: [docs/lessons-learned.md](../docs/lessons-learned.md), item 8.

## Not yet tested

- SSH brute force against a valid account (built-in rule 5763).
- Evasion of the WAF (double encoding, case switching, comment splitting).
- WAF in detection-only mode (current rules depend on the 403).
