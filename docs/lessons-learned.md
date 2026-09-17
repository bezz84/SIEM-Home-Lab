# Lessons Learned / Mistakes Along the Way

Keeping this section honest rather than pretending everything worked first try — the mistakes taught me more than the parts that went smoothly.

1. **VirtualBox Host-only vs NAT Network.** Spent time configuring `vboxnet0` (Host-only Network Manager) before realizing I needed File → Preferences → Network → **NAT Networks** instead. Host-only gives inter-VM connectivity but no internet; NAT Network gives both. Easy to confuse at a glance.

2. **Wazuh admin password only printed once.** The all-in-one installer prints the auto-generated dashboard password exactly once at the end of the install log. Didn't lose it this time, but this is a "screenshot it immediately or regenerate later with `wazuh-passwords-tool`" situation.

3. **Wrong manager IP in the agent install.** Typo'd the `WAZUH_MANAGER` value during the Windows agent MSI install. The install completed and the service started with zero errors — MSI doesn't validate that the manager is actually reachable. Looked "done" but the agent never showed up on the manager side. Lesson: a clean install log is not the same as a working connection. Always confirm with `ossec.log` (`(4102): Connected to the server`) and check the manager's **Endpoints** page, not just "did the installer exit 0."

4. **Sysmon Event ID 3 came back empty when I expected network events.** Assumed Sysmon would log the RDP connections from Kali by default. The SwiftOnSecurity baseline config deliberately keeps `NetworkConnect` scoped to an include/exclude allow-list to avoid drowning in noise — it's not a bug, it's a tuning choice, and RDP logon telemetry ended up coming from the Windows Security log instead. Good reminder to actually read a community config rather than assuming it logs everything.

5. **nmap under-reported open ports.** A `-sV` scan showed only 2 open ports and 998 filtered — RDP (3389) didn't show up at all, even though it was open and workable. Windows Firewall was simply not responding to the probe style nmap was using by default. Would need `-p-` (full port range), possibly `-Pn` (skip host discovery ping) and patience, to get a fuller picture in a real assessment instead of trusting a default scan's port list at face value.

6. **Hydra's RDP module is flagged experimental for a reason.** Kept hitting `freerdp: The connection failed to establish` errors even against a live, correctly-configured target. Dropping concurrency to `-t 4` reduced but didn't eliminate the noise. Bruteforcing RDP with Hydra works, but expect it to be flaky, and don't assume every "error" line in the output means the whole attempt failed.

7. **The built-in MITRE tag on rule 60122 isn't a perfect fit.** Wazuh mapped generic Windows logon-failure events to T1531 (Account Access Removal) instead of T1110 (Brute Force). The underlying detection (repeated 4625s) is correct and useful — but the out-of-the-box technique tag is a generalization, not a precise classification for this specific pattern. Worth remembering that SIEM default rulesets get you most of the way, but a real SOC still needs custom rules/correlation for the last mile of precision. This is on my list for a v2 of this lab (see below).

## Ideas for extending this lab further

- Custom Wazuh rule that correlates N × rule 60122 hits against the same `dstuser` within a short window and fires a dedicated "possible RDP brute force" alert tagged T1110, instead of relying on the generic per-event mapping.
- Active response: auto-block the source IP via `active-response` after N failed RDP attempts (Wazuh ships an example `firewall-drop` script for exactly this).
- Add a second Windows event source: enable RDP-specific logging (`Microsoft-Windows-TerminalServices-RemoteConnectionManager/Operational`) for cleaner RDP-specific detection instead of relying solely on the generic Security log.
- Sysmon config: widen the `NetworkConnect` include rule to capture inbound RDP connections directly, and compare against the Security-log-based detection built here.
- Add a second attacker technique (e.g., a simple Mimikatz-style credential dump simulation, or lateral movement to the Ubuntu box) to broaden the MITRE coverage of the lab.
