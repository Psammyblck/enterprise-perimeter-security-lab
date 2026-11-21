# Lab 1 — Build the Lab and Map the Attack Surface

## Objective

The objective of this lab was to build the foundational virtual network, configure an OPNsense firewall (`fw01`) with WAN and LAN interfaces, and map its attack surface from two perspectives:

- **External testing:** Kali Linux on the simulated WAN network.
- **Internal testing:** Ubuntu Desktop on the LAN network.

The tests were used to examine which services were reachable and whether the firewall's default-deny behaviour was working as expected.

## Network topology

| System | Role | IP address |
|---|---|---|
| Kali Linux VM | External testing host | `203.0.113.50/24` |
| OPNsense WAN | External firewall interface | `203.0.113.1/24` |
| OPNsense LAN | Internal firewall interface | `10.10.10.1/24` |
| Ubuntu Desktop VM (`lan01`) | Internal testing host | `10.10.10.10/24` |

- Simulated WAN network: `203.0.113.0/24`
- Internal LAN network: `10.10.10.0/24`
- Ubuntu's default gateway: `10.10.10.1`
- Kali was configured without a default gateway.

## Configuration

### OPNsense firewall

- WAN assigned to `em0` with address `203.0.113.1/24`.
- LAN assigned to `em1` with address `10.10.10.1/24`.
- The WAN options **Block RFC1918 private networks** and **Block bogon networks** were disabled for this simulated lab network.
- Outbound NAT was configured in Hybrid mode to translate traffic from `10.10.10.0/24` to the WAN address.

### Test hosts

- Kali Linux: static address `203.0.113.50/24`.
- Ubuntu Desktop: static address `10.10.10.10/24`, gateway `10.10.10.1`.

## Testing and results

### 1. External WAN scan

A full TCP SYN scan was run from Kali against the firewall WAN address:

```bash
nmap -n -sS -p- -T4 -oN lab1-wan-scan.txt 203.0.113.1
```

**Observed result:** Nmap reported the host as up and all 65,535 scanned TCP ports as filtered (no response).

This indicates that the probes did not receive responses from the scanned TCP ports. The result is consistent with the firewall dropping unsolicited probes on the WAN interface.

### 2. Internal LAN scan

The supplied LAN scan output shows:

| Port | State | Service | Detection |
|---|---|---|---|
| 53/tcp | Open | DNS | Unbound 1.25.1 |
| 80/tcp | Open | HTTP | OPNsense |
| Other 998 scanned TCP ports | Filtered | — | No response |

**Evidence limitation:** The original report also mentions TCP 443 as open, but the supplied `lab1-lan-scan.txt` output confirms only TCP 53 and TCP 80. No separate scan evidence confirming TCP 443 was supplied, so it is not recorded here as a verified scan finding.

### 3. WAN reachability and firewall logs

The original report includes a Kali ping test showing 100% packet loss when testing the WAN address, and an OPNsense Live View screenshot showing blocked traffic associated with the Kali host. These observations were documented in the original report.

## Attack surface summary

| Interface | Address | Verified service / result | Assessment |
|---|---|---|---|
| WAN | `203.0.113.1` | All 65,535 scanned TCP ports filtered | No TCP ports were shown as open in the supplied scan. Maintain default-deny inbound policy. |
| LAN | `10.10.10.1` | TCP 53 (DNS) and TCP 80 (HTTP) open | These services are reachable from the internal test network. Restrict management access appropriately. |
| LAN | `10.10.10.1` | TCP 443 | Mentioned in the original report, but not confirmed by the supplied scan output. |

## Questions and analysis

### Why did Nmap report the firewall as up even though it did not answer ping?

The original lab explanation was that Nmap can use ARP for host discovery on a directly connected local network. ARP operates at Layer 2 and does not route across the public internet. Therefore, a local scanner may discover a host through ARP even when the host does not respond to an ICMP echo request.

### What is the difference between filtered and closed ports?

- **Filtered:** A filtering device or rule prevents Nmap from determining whether the port is open, often because probes are dropped.
- **Closed:** The target responds, typically with a TCP reset (RST), indicating that no service is listening on that port.

### Why was the bogon-blocking option disabled on the WAN?

The lab used the documentation address range `203.0.113.0/24`. The original report records that the bogon-blocking option was disabled so the simulated WAN traffic could be tested. This was a lab-specific configuration and should not be copied blindly to a production firewall.

### Which services were found on the LAN interface?

The supplied scan confirms DNS on TCP 53 and the OPNsense HTTP service on TCP 80. The original report associated exposed DNS with possible DNS tunnelling and exposed web management services with risks such as exploitation of public-facing applications or brute-force attempts. These are potential risks, not evidence that an attack occurred in this lab.

### Why could the LAN host initiate traffic while unsolicited WAN traffic was blocked?

The original report attributes this behaviour to outbound NAT and stateful firewall handling: outbound connections initiated from the LAN can receive return traffic associated with those connections, while unsolicited inbound WAN traffic is blocked by the default-deny policy.

## Challenges and reflection

During the lab, the WAN and LAN interfaces were initially assigned in the wrong order. This prevented the expected connectivity until the interfaces were reassigned through the OPNsense console.

The original report also describes an Ubuntu-to-Kali connectivity issue related to NAT behaviour. The firewall Live View was used to inspect whether traffic was being passed or blocked.

## Conclusion

The lab established a virtual network with an OPNsense firewall between a simulated external network and an internal LAN. The supplied WAN scan showed all scanned TCP ports as filtered. The supplied LAN scan identified DNS and HTTP services on the firewall's LAN address. The results provided an initial view of the firewall's exposed attack surface and the importance of default-deny inbound filtering.

## Evidence files

- `evidence/lab1-wan-scan.txt` — supplied full TCP scan from Kali.
- `evidence/lab1-lan-scan.txt` — supplied internal scan from Ubuntu.

Screenshots are retained in the original submitted report and can be added separately to this repository.
