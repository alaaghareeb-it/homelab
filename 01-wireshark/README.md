# Lab 01 — Network Traffic Analysis with Wireshark

**Goal:** Capture live traffic and analyze DNS resolution, the TCP three-way handshake, the start of a TLS 1.3 session, and how DHCP hands out an IP address.

**Tools:** Wireshark 4.6, Windows 11 (Wi-Fi), Windows Server 2022 `pktmon`

> Source/destination addresses are redacted in the screenshots for privacy.

---

## Part 1 — DNS Query and Response

**Steps**
1. Started a capture on the Wi-Fi interface.
2. Browsed to `www.wikipedia.org` to generate DNS traffic.
3. Applied the display filter: `dns.qry.name contains "wikipedia"`

![DNS query and response](dns-query-response.jpg)

**Findings**
- The client sent two queries for each lookup: **A** (IPv4) and **AAAA** (IPv6).
- The response shows `www.wikipedia.org` is a **CNAME** (alias) for `dyna.wikimedia.org`.
- The A record resolved to `185.15.58.224`; the AAAA record resolved to an IPv6 address in `2a02:ec80:600::/48`.
- The transaction ID (e.g. `0xbcf7`) links each response to its query.

## Part 2 — TCP Three-Way Handshake and TLS

**Steps**
1. Applied `tcp.flags.syn==1` to locate the connection setup.
2. Followed the connection with `tcp.port==51610`.

![TCP handshake and TLS Client Hello](tcp-handshake-tls.jpg)

**Findings**

| Packet | Flags | Meaning |
|---|---|---|
| 287 | `SYN` | Client requests a connection (Seq=0) |
| 288 | `SYN, ACK` | Server accepts and acknowledges (Ack=1) |
| 289 | `ACK` | Client confirms — connection established |
| 291 | TLS `Client Hello` | Encryption negotiation begins (TLS 1.3) |
| 295 | TLS `Server Hello` | Server answers; traffic is encrypted from here |

- Even though the session is encrypted, the **SNI** field in the Client Hello reveals the destination hostname (`www.wikipedia.org`). This is useful for detecting suspicious domains in a SOC.
- Round-trip time from SYN to SYN-ACK was about **58 ms**.

## Part 3 — DHCP DORA (Discover, Offer, Request, Acknowledge)

**Setup:** Windows Server 2022 DHCP server on `DC01` (10.0.2.15) and a domain client `CLIENT01` on the same lab network (VirtualBox NAT Network `LabNet`). VirtualBox's built-in DHCP was turned off so the Windows server is the only one answering.

**Steps**
1. Started a capture on DC01 with the built-in Windows packet monitor, filtered to UDP 67/68:
   ```powershell
   pktmon filter add DHCP-Server -t UDP -p 67
   pktmon filter add DHCP-Client -t UDP -p 68
   pktmon start --capture --pkt-size 0 -f C:\Install\dhcp.etl
   ```
2. On CLIENT01 ran `ipconfig /release` then `ipconfig /renew`.
3. Stopped the capture, converted it with `pktmon etl2pcap`, and opened the `.pcapng` in Wireshark on my laptop.
4. Filtered one conversation: `dhcp.id == 0xf9533216 || dhcp.option.dhcp == 7`

```
No.  Time        Source      Destination      Info
17   808.713287  10.0.2.100  10.0.2.15        DHCP Release  - Transaction ID 0x41ef9f36
19   810.526483  0.0.0.0     255.255.255.255  DHCP Discover - Transaction ID 0xf9533216
21   810.528125  10.0.2.15   255.255.255.255  DHCP Offer    - Transaction ID 0xf9533216
23   810.544795  0.0.0.0     255.255.255.255  DHCP Request  - Transaction ID 0xf9533216
25   810.546011  10.0.2.15   255.255.255.255  DHCP ACK      - Transaction ID 0xf9533216
```

**Findings**

| Packet | Message | From → To | What it means |
|---|---|---|---|
| 17 | `RELEASE` | 10.0.2.100 → 10.0.2.15 | Client gives its address back (unicast, because it still has an IP) |
| 19 | `DISCOVER` | 0.0.0.0 → 255.255.255.255 | Client has no IP, so it broadcasts and asks for its old address (option 50 = 10.0.2.100) |
| 21 | `OFFER` | 10.0.2.15 → broadcast | Server offers 10.0.2.100 |
| 23 | `REQUEST` | 0.0.0.0 → broadcast | Client accepts the offer from server 10.0.2.15 (option 54) |
| 25 | `ACK` | 10.0.2.15 → broadcast | Server confirms the lease; client configures its IP |

- All four packets share **one Transaction ID** (`0xf9533216`). This is how the client matches answers to its own request.
- The full DORA took about **20 ms** (810.5265 s → 810.5460 s).
- The ACK carries the settings the client needs:

| Option | Value |
|---|---|
| 1 Subnet Mask | 255.255.255.0 |
| 3 Router | 10.0.2.1 |
| 6 DNS Server | 10.0.2.15 (the domain controller) |
| 15 Domain Name | lab.local |
| 51 Lease Time | 8 days |
| 58 Renewal (T1) | 4 days (50%) |
| 59 Rebinding (T2) | 7 days (87.5%) |

- After each reboot the client sent only `REQUEST` + `ACK` (no Discover). It already had a valid lease, so it just asked to keep it (INIT-REBOOT).
- The very first DORA in the capture came from the hostname `WIN-KLVT7IO8LUL`. That was the random name Windows Setup gave the machine before it was renamed to CLIENT01.
- Every packet appears twice because `pktmon` logs it at two points in the network stack. In a normal Wireshark capture you see it once.

> The raw `.pcapng` is not published.

---

## What I Learned
- A single page load triggers both A and AAAA lookups, and the browser may connect over **IPv6** even when an IPv4 address is returned. My first filter on the IPv4 address returned nothing for this reason.
- How to narrow thousands of packets down to one conversation using display filters.
- What an analyst can still see in encrypted traffic: DNS queries, SNI, timing, and packet sizes.
- The order of the TCP handshake and where TLS negotiation starts.
- How a client gets an IP with DORA, why the first messages are broadcasts, and which options (router, DNS, domain) arrive in the ACK.
- How to capture on a Windows server without installing Wireshark there (`pktmon`).

## Useful Filters
```
dhcp
dhcp.option.dhcp == 1      # Discover (2 Offer, 3 Request, 5 ACK, 7 Release)
dhcp.id == 0xf9533216
dns
dns.qry.name contains "example"
tcp.flags.syn==1
tcp.flags.syn==1 && tcp.flags.ack==1
tcp.port==443
tls.handshake.type==1
```
