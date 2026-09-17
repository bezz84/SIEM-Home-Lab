# 01 — Building the Lab Network

Before touching Wazuh or Sysmon, I needed all three VMs (Ubuntu, Windows, Kali) talking to each other but isolated from my home LAN. VirtualBox gives you a few options here and I burned some time picking the wrong one first, so I'm documenting that too.

## Why NAT Network, not Host-Only or plain NAT

- **Plain NAT** (per-VM default): each VM gets its own private NAT instance — they can reach the internet but **cannot see each other**. No good for a SIEM lab.
- **Host-only (`vboxnet0`)**: VMs can see each other and the host, but have **no internet access** — can't `apt install` or `yum install` anything.
- **NAT Network**: a single virtual switch shared by every VM attached to it. VMs can reach each other *and* the internet through NAT. This is the one I wanted.

My first attempt at configuring this went to the wrong panel — I was editing `vboxnet0` (Host-only Network Manager) instead of File → Preferences → Network → **NAT Networks**. Looks similar at a glance, completely different feature.

![Wrong panel — vboxnet0 instead of NAT Network](../screenshots/01-network-setup/01_virtualbox_natnetwork_wrong_panel_vboxnet0.png)

### Creating the NAT Network correctly

In VirtualBox Manager:

```
File → Preferences → Network → NAT Networks → Add (the + icon)
```

I named it `NatNetwork` (default), gave it CIDR `10.0.2.0/24`, and made sure DHCP was disabled since I wanted predictable addressing per VM rather than VirtualBox handing out random leases.

![NAT Network created, DHCP disabled, ready](../screenshots/01-network-setup/02_virtualbox_natnetwork_dhcp_disabled_ready.png)

I later went back and enabled IPv6 + DHCP on the network object itself (this only controls whether the network *offers* DHCP — each VM still needs to be told to actually use it, see below):

![NAT Network with IPv6/DHCP enabled](../screenshots/01-network-setup/28_virtualbox_natnetwork_ipv6_dhcp_enabled.png)

## Attaching all three VMs to the same NAT Network

For **each** VM (Windows, Kali, Ubuntu) — Settings → Network → Adapter 1:

- Attached to: **NAT Network**
- Name: **NatNetwork** (the one created above)

Windows:
![Windows adapter attached to NatNetwork](../screenshots/01-network-setup/29_windows_vm_settings_adapter1_natnetwork.png)

Kali:
![Kali adapter attached to NatNetwork](../screenshots/01-network-setup/30_kali_vm_settings_adapter1_natnetwork.png)

Ubuntu:
![Ubuntu adapter attached to NatNetwork](../screenshots/01-network-setup/31_ubuntu_vm_settings_adapter1_natnetwork.png)

Once all three were booted on the same network:

![All 3 VMs running in VirtualBox Manager](../screenshots/01-network-setup/32_virtualbox_manager_all3_vms_running.png)

## Getting Ubuntu its address

Ubuntu came up with a DHCP-assigned address first, confirmed with:

```bash
ip a
```

![ip a showing DHCP address on enp0s3](../screenshots/01-network-setup/03_ubuntu_ip_a_dhcp_address.png)

Since the Ubuntu box is the Wazuh manager, I wanted its IP to stop changing (every Windows agent config points at this address), so I switched it to a static IP via Netplan:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: no
      addresses: [10.0.2.15/24]
      routes:
        - to: default
          via: 10.0.2.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

![Netplan yaml edited in nano](../screenshots/01-network-setup/04_ubuntu_netplan_yaml_edit.png)

Apply it:

```bash
sudo netplan apply
```

> Real addresses are blanked out in the screenshot above and replaced with placeholders in this doc — use whatever range your NAT Network's CIDR actually is (check File → Preferences → Network → NAT Networks → your network → Network CIDR).

## Confirming everyone can see everyone

Basic ICMP checks from all three directions:

**Windows → Ubuntu / Kali:**
```
ping <ubuntu-ip>
ping <kali-ip>
```
![Windows ping test success](../screenshots/01-network-setup/33_windows_ping_test_success.png)

**Kali → Windows / Ubuntu:**
```bash
ping -c 4 <windows-ip>
ping -c 4 <ubuntu-ip>
```
![Kali ping test success to Windows and Ubuntu](../screenshots/01-network-setup/34_kali_ping_test_windows_ubuntu_success.png)

**Ubuntu → Windows / Kali:**
```bash
ping -c 4 <windows-ip>
ping -c 4 <kali-ip>
```
![Ubuntu ping test success to Windows and Kali](../screenshots/01-network-setup/35_ubuntu_ping_test_windows_kali_success.png)

All green. Network layer done — moved on to installing Wazuh.

**Next:** [02 — Wazuh Install](02-wazuh-install.md)
