# SOC Home Lab — Wazuh SIEM + Sysmon + RDP Attack Detection

A self-built Security Operations Center (SOC) home lab. Simulates a small monitored network: a Wazuh SIEM manager, a Windows endpoint with Sysmon, and a Kali attacker box. Ends with a real RDP brute-force attack detected end-to-end in Wazuh, mapped to MITRE ATT&CK.

> Lab environment only. All targets are VMs I own and control. No production systems or third-party assets involved.

## Architecture

```
                         VirtualBox NAT Network
        ┌──────────────────────────────────────────────┐
        │                                                │
   ┌────┴─────┐        ┌───────────────┐        ┌───────┴──────┐
   │  Ubuntu   │        │   Windows     │        │     Kali      │
   │  (Wazuh   │◄──────►│  (Endpoint +  │◄──────►│  (Attacker)   │
   │  Manager) │  logs  │  Sysmon +     │  RDP   │  nmap, hydra  │
   │           │        │  Wazuh Agent) │  3389  │               │
   └───────────┘        └───────────────┘        └───────────────┘
```

| Role | OS | Tools |
|---|---|---|
| SIEM Manager | Ubuntu Server | Wazuh Manager + Indexer + Dashboard |
| Monitored Endpoint | Windows | Sysmon, Wazuh Agent, RDP enabled |
| Attacker | Kali Linux | nmap, Hydra |

All three VMs sit on the same VirtualBox **NAT Network** (not the default NAT adapter, and not `vboxnet0` host-only) so they can route to each other while still reaching the internet for package installs.

## Lab Story

1. Stand up an isolated virtual network for all three VMs.
2. Install and configure Wazuh manager on Ubuntu.
3. Install Sysmon on the Windows endpoint for rich process/network telemetry.
4. Enroll the Windows endpoint as a Wazuh agent.
5. From Kali, scan the Windows box and run a brute-force attack against RDP.
6. Detect the attack in Wazuh: failed logons, the successful logon, and MITRE ATT&CK technique mapping.

## Contents

| Doc | Covers |
|---|---|
| [docs/01-network-setup.md](docs/01-network-setup.md) | VirtualBox NAT Network for 3 VMs, static/DHCP IPs, connectivity tests |
| [docs/02-wazuh-install.md](docs/02-wazuh-install.md) | Installing Wazuh manager (all-in-one) on Ubuntu |
| [docs/03-sysmon-windows-agent.md](docs/03-sysmon-windows-agent.md) | Installing Sysmon on Windows with a SwiftOnSecurity-style config |
| [docs/04-agent-enrollment.md](docs/04-agent-enrollment.md) | Enrolling the Windows endpoint into Wazuh, troubleshooting a wrong manager IP |
| [docs/05-attack-simulation.md](docs/05-attack-simulation.md) | Recon with nmap, building a wordlist, RDP brute force with Hydra |
| [docs/06-detection-and-mitre-mapping.md](docs/06-detection-and-mitre-mapping.md) | Finding the attack in Wazuh Discover, rule IDs, MITRE ATT&CK mapping |
| [docs/lessons-learned.md](docs/lessons-learned.md) | Mistakes hit along the way and how they were fixed |

Screenshots for each phase live in `screenshots/<phase-folder>/`, referenced inline in each doc.

## Detection Summary

| Event | Wazuh Rule ID | Rule Level | MITRE Technique | Tactic |
|---|---|---|---|---|
| RDP logon failure (brute force attempts) | 60122 | 5 | T1110 / T1531 | Credential Access / Impact |
| RDP logon success | 60106 | 3 | T1078 (Valid Accounts) | Defense Evasion / Persistence |

Full detail, screenshots, and hit counts (11 failed / 39 success events matched in Discover) are in [docs/06-detection-and-mitre-mapping.md](docs/06-detection-and-mitre-mapping.md).

## Disclaimer

All activity performed against VMs owned and operated by the lab author, on an isolated virtual network. Screenshots have been redacted of real credentials, hostnames, and IPs where they appeared. This repository is for learning and portfolio purposes — not a guide for attacking systems you don't own or have written authorization to test.

## License

MIT — see [LICENSE](LICENSE).
