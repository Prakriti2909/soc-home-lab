# Lessons Learned and Troubleshooting

Problems I hit while building the lab, what caused them, and how I fixed them.

## 1. Alerts showed the wrong attacker IP

* **Symptom:** WAF alerts listed `192.168.56.1` (my Windows host) as the source, though I ran the attacks from Kali (`192.168.56.102`).
* **Diagnosis:** `ip route get 192.168.56.105` on Kali showed traffic leaving via `eth0` (NAT) with source `10.0.2.15`. VirtualBox NAT rewrote it to the host's address. To confirm, I sent a uniquely tagged payload (`kali1`) and grepped the WAF audit log for its `client_ip`.
* **Cause:** Kali had one NetworkManager profile ("Wired connection 1"). Running `nmcli device connect` for each adapter moved that single profile between `eth0` and `eth1`, so the host-only interface never got a proper route for the lab network.
* **Fix:** one profile per adapter (`lab-hostonly` on eth1 with a saved route, `nat-internet` on eth0), then verified `ip route get` shows `dev eth1 src 192.168.56.102` and the WAF log shows `192.168.56.102`.
* **Takeaway:** attribution depends on network design. Verify the source IP your sensors see before trusting alerts.
* **Evidence:** `screenshots/08-kali-routing-fix.png`

## 2. Wazuh failed on Ubuntu 26.04

* **Symptom:** the indexer would not start; the kernel logged `BUG: soft lockup - CPU#1 stuck for 261s`, and the installer rolled back.
* **Cause:** Ubuntu 26.04 is not on Wazuh's supported list (the installer warned about this).
* **Fix:** reinstalled the VM on Ubuntu Server 22.04 LTS.

## 3. Wazuh dashboard install failed with "disk full"

* **Symptom:** manager and indexer installed, but the dashboard package failed and the installer rolled back. `dpkg` reported a disk full error.
* **Cause:** the Ubuntu installer's LVM default used only 24 GB of the 50 GB virtual disk.
* **Fix:** `sudo lvextend -l +100%FREE -r /dev/mapper/ubuntu--vg-ubuntu--lv`, then re-ran the installer.

## 4. WAF container would not start

* **Cause:** typo in the image name (`owasp/mosecurity-crs` instead of `owasp/modsecurity-crs`).
* **Fix:** corrected the image name. Lesson: read `docker: invalid reference format` as "bad image name".

## 5. `curl: Could not resolve host`

* **Cause:** a period instead of a colon before the port (`192.168.56.105.8080`), so curl treated the whole string as a hostname.

## 6. Raw path traversal returned 400 and produced no alert

* **Cause:** nginx rejects a malformed path like `/../../etc/passwd` before ModSecurity inspects it, so nothing is logged for Wazuh to alert on.
* **Fix / finding:** the traversal payload in a query string (`?file=../../..`) is inspected and blocked by CRS. Documented as a detection gap: the WAF log cannot be the only source for attacks nginx drops early. Monitoring nginx's own access/error log would close it.

## 7. SSH events were already collected

* I planned to add `/var/log/auth.log` to `ossec.conf`, but Wazuh 4.14 on Ubuntu 22.04 already reads the systemd journal. Adding the file would have created duplicate alerts. Confirmed by running Hydra and checking that rules 5710/5503/2502/5758/5712 fired.

## 8. Nmap rule false positive on my own Windows host

* **Symptom:** after writing the Suricata port-scan rule (sid 9000001), Wazuh rule 100300 fired for two sources: Kali (`192.168.56.102`), the real scan, and my Windows host (`192.168.56.1`), which was only browsing lab services.
* **Original rule:**
  `alert tcp any any -> $HOME_NET any (msg:"LAB Possible Nmap SYN scan"; flags:S; threshold:type both, track by_src, count 20, seconds 5; classtype:attempted-recon; sid:9000001; rev:1;)`
* **Cause:** the rule counts SYN packets per source IP and alerts at 20 in 5 seconds. A browser opens many parallel connections when loading a page, so normal use from a busy client can reach the same rate as a slow scan.
* **Fix:** excluded my Windows host from the source address:
  `alert tcp !192.168.56.1 any -> $HOME_NET any (msg:"LAB Possible Nmap SYN scan"; flags:S; threshold:type both, track by_src, count 20, seconds 5; classtype:attempted-recon; sid:9000001; rev:1;)`
* **Verification:** after `suricata -T` passed and Suricata restarted, a new Nmap scan from Kali still produced a `rule.id: 100300` alert, and browsing Juice Shop from Windows produced none.
* **Trade-off:** excluding an IP is a blind spot. A real scan from `192.168.56.1` would now go undetected. In production I would detect scans by the number of distinct destination ports contacted in a short window rather than raw SYN rate, which separates scanners from busy clients without whitelisting hosts.
* **Takeaway:** threshold rules need testing against normal traffic, not just against the attack, and every exclusion should be documented as a known gap.
* **Evidence:** `screenshots/10-nmap-false-positive.png`

## 9. Wazuh dashboard unreachable after a VM reboot

* **Symptom:** the dashboard showed `ERR_TIMED_OUT` after restarting the VM. The journal showed `wazuh-manager.service: start operation timed out`.
* **Cause:** with 5-6 GB of RAM, all Wazuh services starting at once can push the manager past systemd's start timeout.
* **Fix:** restarted the services in order (`wazuh-indexer`, `wazuh-manager`, `wazuh-dashboard`) and waited a minute before reloading.
* **Takeaway:** after booting the lab, give Wazuh a few minutes, and check `systemctl is-active` on all three services before assuming the dashboard is broken.
