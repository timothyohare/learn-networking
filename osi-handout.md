# The OSI Model: A Reference Handout

A companion to the slide deck
([view it online](https://timothyohare.github.io/learn-networking/osi-slides.html)).
Read it at your own pace. It
follows the same example as the slides, then adds reference tables, a
troubleshooting checklist and a glossary.

---

## 1. What the OSI model is, and why you'll use it

The **OSI (Open Systems Interconnection) model** splits the job of moving data
between two computers into **seven layers**. Each layer has one job, relies on
the layer below it, and serves the layer above it.

Nobody runs "OSI" as software. The internet actually runs on the simpler
**TCP/IP** model (section 6). OSI survives because it's the shared vocabulary
of the industry. When an engineer says *"it's a Layer 1 problem"*, everyone
knows to check the cables before touching the configuration.

Think of a building site. The electrician doesn't redesign the footings, and the
concreter doesn't care what colour the walls are. Each trade works to an agreed
interface with the trades around it. Networking layers are the same: you can
swap Wi-Fi for fibre without the web browser knowing or caring.

**Mnemonic (top-down, 7 → 1):** *All People Seem To Need Data Processing*

---

## 2. The seven layers at a glance

| # | Layer | Its job | Unit (PDU) | Examples | Devices | Freight analogy |
| --- | --- | --- | --- | --- | --- | --- |
| 7 | Application | What the user actually wants | Data | HTTP, DNS, SMTP, SSH | — | The project brief |
| 6 | Presentation | Format, compress, encrypt | Data | TLS, UTF-8, JPEG | — | Sealed, tamper-evident envelope |
| 5 | Session | Open, keep alive and close a conversation | Data | TLS sessions, RPC | — | The contract period |
| 4 | Transport | Reliable delivery to the right program | Segment | TCP, UDP | Firewalls (ports) | Delivery docket and loading dock number |
| 3 | Network | Addresses and routes across networks | Packet | IPv4, IPv6, ICMP | Routers | Street addresses and the road network |
| 2 | Data Link | Delivery to the next box on this link | Frame | Ethernet, Wi-Fi (802.11), ARP | Switches, access points | One leg, depot to depot |
| 1 | Physical | Bits as voltage, light or radio | Bits | Copper, fibre, radio | Cables, hubs, repeaters | The road surface itself |

**PDU** means *protocol data unit*: the name for "the bundle" at that layer.
Using the right word (segment vs packet vs frame) instantly tells another
engineer which layer you mean.

---

## 3. Encapsulation: the worked example

**Scenario:** Alex's laptop loads `https://example.com/`.

| Thing | Value |
| --- | --- |
| Laptop IP / MAC / port | `192.168.1.10` / `a4:83:e7:12:34:56` / `51514` |
| Home router (gateway) LAN IP / MAC | `192.168.1.1` / `3c:22:fb:aa:bb:cc` |
| Home router public IP | `203.0.113.7` |
| Web server IP / port | `198.51.100.80` / `443` (HTTPS) |

The addresses are illustrative, taken from ranges reserved for documentation.
Header sizes are the standard minimums, with no optional extras.

Going **down** the stack, each layer wraps what it was given. Each step below
shows the bundle with the new part in **CAPITALS**:

```text
L7  [HTTP 75]                                                     =  75 B  Data
L6  [TLS 5][http 75, encrypted][TLS TAG 17]                       =  97 B  Data
L5  (no new bytes: manages the session)                           =  97 B  Data
L4  [TCP 20][tls 5][http 75][tls 17]                              = 117 B  Segment
L3  [IP 20][tcp 20][tls 5][http 75][tls 17]                       = 137 B  Packet
L2  [ETH 14][ip 20][tcp 20][tls 5][http 75][tls 17][FCS 4]        = 155 B  Frame
L1  [PRE+SFD 8][eth 14][ip ...                ... ][fcs 4]        = 163 B  Bits
```

The useful payload is 75 bytes, but **88 bytes** of wrapping go on the wire.
For small messages, overhead often outweighs the data. That's why tunnel
overhead (section 9) matters so much.

### Step by step

1. **L7 Application.** The browser writes the HTTP request (75 bytes, counting
   the hidden line-break characters):

   ```http
   GET / HTTP/1.1
   Host: example.com
   User-Agent: Mozilla/5.0
   Accept: */*

   ```

   Before this, **DNS** (also Layer 7) has already looked up `example.com` and
   got back `198.51.100.80`.
2. **L6 Presentation.** TLS encrypts the request. It adds a 5-byte record header
   in front and a 17-byte trailer behind it (a hidden 1-byte type and a 16-byte
   integrity tag that reveals any tampering). The 75 bytes are now unreadable to
   anyone in between.
3. **L5 Session.** No new bytes. In the real world this job is shared by TCP
   (opening and closing the connection) and TLS (the secure session and its keys).
4. **L4 Transport.** TCP adds 20 bytes. Ports say which program sends and
   receives (`51514` → `443`), and sequence numbers let lost or out-of-order
   pieces be fixed.
5. **L3 Network.** IP adds 20 bytes: source `192.168.1.10`, destination
   `198.51.100.80`, TTL `64`. The packet is now 137 bytes.
6. **L2 Data Link.** Ethernet adds a 14-byte header and a 4-byte FCS trailer.
   The destination MAC is the **router's** (`3c:22:fb:aa:bb:cc`), *not* the
   server's. The laptop learned it using ARP.
7. **L1 Physical.** The network card adds an 8-byte preamble and start delimiter,
   then sends 163 bytes (1,304 bits) as signals: about 1.3 µs at 1 Gb/s. A
   12-byte silent gap must follow before the next frame.

> **The single most important idea:** the **IP address is the final
> destination** and stays the same end to end. The **MAC address is only the
> next hop** and is replaced at every router.

---

## 4. Header field reference

### TLS record header (Layer 6): 5 bytes, plus a 17-byte trailer

| Field | Size | Example | Meaning |
| --- | --- | --- | --- |
| Content type | 1 B | `23` | Application data |
| Version | 2 B | `0x0303` | Legacy label kept for compatibility (also used by TLS 1.3) |
| Length | 2 B | `92` | Encrypted bytes that follow (75 + 1 + 16) |
| Trailer: inner type + auth tag | 1 + 16 B | — | Hidden real type and tamper-evident seal |

### TCP header (Layer 4): 20 bytes

| Field | Size | Example | Meaning |
| --- | --- | --- | --- |
| Source port | 2 B | `51514` | Temporary port picked by the laptop |
| Destination port | 2 B | `443` | HTTPS service on the server |
| Sequence number | 4 B | `1001` | Byte position of this data (simplified, relative) |
| Acknowledgement number | 4 B | `5001` | "I've received everything up to here" |
| Header length + flags | 2 B | `5`, PSH+ACK | 5 × 4 = 20 B header; flags SYN, ACK, PSH, FIN, RST |
| Window | 2 B | `64240` | How much more data the sender of this segment can accept |
| Checksum | 2 B | (calculated) | Detects corruption |
| Urgent pointer | 2 B | `0` | Rarely used |

**UDP** is the lightweight alternative: an **8-byte** header (source port,
destination port, length, checksum). It has no handshake and never re-sends.
It's used for voice, video, gaming and DNS.

### IPv4 header (Layer 3): 20 bytes

| Field | Size | Example | Meaning |
| --- | --- | --- | --- |
| Version + header length | 1 B | `4`, `5` | IPv4, 5 × 4 = 20 B header |
| DSCP / ECN | 1 B | `0` | Priority marking, used heavily by QoS and SD-WAN |
| Total length | 2 B | `137` | Whole packet in bytes |
| ID + flags + fragment offset | 4 B | `0x1c46`, DF | Used if the packet has to be split |
| Time to live (TTL) | 1 B | `64` | Decremented by each router; at 0 the packet is dropped |
| Protocol | 1 B | `6` | 6 = TCP, 17 = UDP, 1 = ICMP |
| Header checksum | 2 B | `0x31f3` | Detects a corrupted header |
| Source address | 4 B | `192.168.1.10` | Sender |
| Destination address | 4 B | `198.51.100.80` | Final destination |

### Ethernet frame (Layer 2): 14-byte header, 4-byte trailer

| Field | Size | Example | Meaning |
| --- | --- | --- | --- |
| Destination MAC | 6 B | `3c:22:fb:aa:bb:cc` | Next hop (the router) |
| Source MAC | 6 B | `a4:83:e7:12:34:56` | This device |
| EtherType | 2 B | `0x0800` | IPv4 inside (`0x86DD` = IPv6, `0x0806` = ARP) |
| *payload* | 46–1,500 B | the IP packet | — |
| Frame check sequence | 4 B | CRC-32 | Damaged frames are silently dropped |

### Physical layer extras (Layer 1)

| Field | Size | Pattern | Meaning |
| --- | --- | --- | --- |
| Preamble | 7 B | `10101010` × 7 | Receiver locks onto the timing |
| Start frame delimiter | 1 B | `10101011` | "Frame starts now" |
| Inter-frame gap | 12 B | silence | Required gap before the next frame |

---

## 5. What a router changes as the packet crosses it

| Field | Laptop → router | Router → internet | Why |
| --- | --- | --- | --- |
| Destination MAC | `3c:22:fb:aa:bb:cc` | ISP's next device | New link, new next hop |
| Source MAC | `a4:83:e7:12:34:56` | Router's own MAC | New link |
| TTL | `64` | `63` | Every router subtracts 1 |
| IP header checksum | `0x31f3` | `0xb89d` | Recalculated because the header changed |
| Source IP | `192.168.1.10` | `203.0.113.7` | **NAT**: home and office routers only |
| Source port | `51514` | often kept, may change | NAT may change it too (the TCP checksum then updates) |
| Destination IP | `198.51.100.80` | unchanged | Final destination |
| TLS + HTTP data | encrypted | unchanged | The router can't read it, and doesn't need to |

At the server, each layer removes its own wrapper in turn, from Layer 1 up to
Layer 7 (**de-encapsulation**). Each layer checks "is this for me?" before
passing the rest up. The reply then makes the whole journey back, with source
and destination swapped.

---

## 6. OSI vs TCP/IP

| OSI layer | TCP/IP layer | Protocols |
| --- | --- | --- |
| 7 Application, 6 Presentation, 5 Session | **Application** | HTTP, TLS, DNS, SSH |
| 4 Transport | **Transport** | TCP, UDP |
| 3 Network | **Internet** | IP, ICMP |
| 2 Data Link, 1 Physical | **Link** | Ethernet, Wi-Fi |

The real internet uses the four TCP/IP layers. Use OSI numbers when *talking*
about problems and kit ("a Layer 3 switch", "Layer 7 load balancing").

---

## 7. Troubleshooting from the bottom up

When something "doesn't work", climb the stack. Don't guess from the top.

| Layer | Question | How to check |
| --- | --- | --- |
| 1 | Is there a link at all? | Link lights, a different cable or port, Wi-Fi signal strength |
| 2 | Can I reach my local network? | `arp -a` (is the gateway's MAC listed?), switch port status, VLAN |
| 3 | Do I have an IP, and can I reach the gateway and beyond? | `ipconfig` / `ip addr`, `ping 192.168.1.1`, `ping 1.1.1.1`, `traceroute` |
| 4 | Is the port open? | `Test-NetConnection host -Port 443` (Windows), `nc -vz host 443` (Mac/Linux) |
| 7 | Does name lookup work, and does the app answer? | `nslookup example.com`, `curl -v https://example.com` |

A classic pattern: `ping 1.1.1.1` works but `ping example.com` fails. Layer 3 is
fine, so the problem is DNS (Layer 7).

---

## 8. Glossary

- **ACK**: acknowledgement; "I received your data up to here".
- **ARP**: Address Resolution Protocol; finds the MAC address that belongs to
  an IP on the local network.
- **Bandwidth**: how many bits per second a link can carry.
- **CIDR**: notation like `192.168.1.0/24` describing a block of IP addresses.
- **Checksum / CRC**: a small calculated value used to detect corruption.
- **De-encapsulation**: removing each layer's wrapper on the receiving side.
- **Default gateway**: the router a device sends traffic to when the destination
  isn't on its local network.
- **DNS**: Domain Name System; turns names into IP addresses.
- **DSCP**: a priority field in the IP header, used for quality of service (QoS).
- **Encapsulation**: wrapping data in a header (and sometimes a trailer) at each
  layer on the way down.
- **Frame**: the Layer 2 unit (Ethernet header + packet + FCS).
- **Header / trailer**: control information added before / after the data.
- **Hop**: one step from one router (or device) to the next.
- **IP address**: the Layer 3 address of a device; the *final* destination.
- **Latency**: how long a packet takes to arrive; jitter is how much that varies.
- **MAC address**: the Layer 2 hardware address of a network interface; the
  *next hop*.
- **MTU**: the maximum transmission unit, the largest packet a link will carry
  (usually 1,500 B on Ethernet).
- **NAT**: Network Address Translation; a router swaps private addresses for its
  public one.
- **Packet**: the Layer 3 unit (IP header + segment).
- **Payload**: the actual data being carried, as opposed to headers.
- **Port**: a number identifying a program or service on a device (443 = HTTPS,
  22 = SSH, 53 = DNS).
- **Protocol**: an agreed set of rules for communicating.
- **Router**: a Layer 3 device that forwards packets between networks.
- **Segment**: the Layer 4 unit (TCP header + data).
- **Switch**: a Layer 2 device that forwards frames within a network using MAC
  addresses.
- **TLS**: Transport Layer Security; encrypts traffic (the "S" in HTTPS).
- **TTL**: time to live; a hop counter that stops packets looping forever.
- **Tunnel**: wrapping a whole packet inside another packet so it can cross a
  network privately. The foundation of VPNs and SD-WAN.

---

## 9. Next steps toward SD-WAN

SD-WAN is **encapsulation again**. An SD-WAN edge device takes the whole
137-byte IP packet from our example, encrypts it, and wraps it in a *new* outer
header addressed to another site's SD-WAN device:

```text
[OUTER IP 20][UDP 8][ESP+IV 16][ original 137-byte packet, encrypted ][ESP TAIL 19]
```

That's roughly **63 bytes of extra overhead per packet** (IPsec with NAT
traversal and AES-GCM, a typical case). Because a standard Ethernet link
carries at most 1,500 bytes per packet, large packets may no longer fit. MTU
and fragmentation problems are an everyday SD-WAN issue.

The "software-defined" part is a central **controller** that sets policy, for
example: *video calls take the fibre link, backups take 4G, and if the fibre
starts dropping packets, move the calls*. To do that it reads information from
several layers: DSCP marks (L3), ports (L4), application signatures (L7) and
live link quality (latency, jitter, loss).

### Suggested learning path

1. **IP addressing**: subnets, CIDR, private vs public ranges, NAT
2. **Switching**: VLANs, ARP, MAC address tables, trunk ports
3. **Routing**: static routes, default routes, then OSPF and BGP basics
4. **VPNs and IPsec**: tunnels, encryption, MTU and MSS
5. **SD-WAN**: overlays vs underlays, policies, controllers, application-aware
   routing

### Further reading and practice

- **Wireshark** (free): capture your own traffic, load a web page, and click
  through the Ethernet, IP, TCP and TLS layers of a real packet. It's the best
  way to make this handout real.
- RFC 791: Internet Protocol (IPv4)
- RFC 9293: Transmission Control Protocol (TCP)
- RFC 8446: TLS 1.3
- IEEE 802.3: Ethernet
