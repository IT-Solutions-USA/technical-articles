# SS7 (Signaling System No. 7) — Technical Reference

**Author:** Ioannis Alexander Konstas — IT Solutions USA & World Telephony Services Corp

---

## What Is SS7

Signaling System No. 7 (SS7) is an architecture for performing **out-of-band signaling** in support of the call-establishment, billing, routing, and information-exchange functions of the Public Switched Telephone Network (PSTN). It identifies functions to be performed by a signaling-system network and a protocol to enable their performance.

SS7 carries **signaling only** — not voice. Voice is carried on separate bearer circuits (TDM or IP). SS7 tells the network what to do; the voice travels on a different path. Signaling links carry data at a rate of **56 or 64 kbps**.

### What Is Out-of-Band Signaling?

Out-of-band signaling is signaling that does not take place over the same path as the conversation. Traditional telephony used in-band signaling — dial tones, digits, and voice all traveled over the same wire. SS7 establishes a **separate digital channel** (the signaling link) exclusively for signaling messages. This provides several advantages:

- Allows transport of more data at higher speeds (56 kbps vs. MF outpulsing)
- Allows signaling at any time during the call, not only at the beginning
- Enables signaling to network elements with no direct trunk connection

### SS7 Message Examples

SS7 messages convey information such as:
- "I'm forwarding to you a call from 212-555-1234 to 718-555-5678. Look for it on trunk 067."
- "Someone just dialed 800-555-1212. Where do I route the call?"
- "The called subscriber for the call on trunk 11 is busy. Release the call and play a busy tone."
- "The route to XXX is congested. Don't send messages unless priority 2 or higher."

---

## Why SS7 Matters for WTS as a CLEC

As a CLEC, WTS must interconnect with the PSTN to:
- Originate calls (customer calls out to any phone number)
- Terminate calls (any phone number can call WTS customers)
- Handle number portability (port numbers to/from other carriers)
- Access CNAM (Caller ID name database)
- Interact with 911 (emergency services routing)
- Deliver SMS (A2P and P2P messaging)

All of these functions — at the carrier level — rely on SS7. The question is whether WTS interfaces directly with SS7 or through an intermediary.

---

## WTS SS7 Use Cases

| Use Case | SS7 Role | WTS Approach |
|---|---|---|
| Outbound PSTN calls | ISUP call setup via SS7 | Via SIP trunk provider (Phase 1) → Direct SS7 (Phase 3) |
| Inbound PSTN calls | ISUP routing to WTS gateway | Via SIP trunk provider (Phase 1) → Direct SS7 (Phase 3) |
| Number Portability (LNP) | TCAP query to NPAC/LNP database | Via SIP provider (Phase 1) → Direct (Phase 3) |
| CNAM lookup | TCAP query to CNAM database | Via TNS/Neustar API (all phases) |
| 911 routing | ISUP to PSAP via SS7 | Via Bandwidth E911 API (all phases) |
| SMS delivery | MAP protocol over SS7 | Via SMS gateway API (Phase 2+) |

---

## SS7 Network Architecture

```
┌──────────┐    SS7 Links    ┌──────────┐    SS7 Links    ┌──────────┐
│   SSP    │◀──────────────▶│   STP    │◀──────────────▶│   SCP    │
│ (Switch) │                 │(Transfer)│                 │(Database)│
└──────────┘                 └──────────┘                 └──────────┘
     │                            │
  Voice                      Signaling
  Circuits                   Messages
     │
┌──────────┐
│  PSTN    │
│  (ILEC)  │
└──────────┘
```

### Three Core Node Types

| Node | Full Name | Function |
|---|---|---|
| **SSP** | Service Switching Point | A telephone switch that originates, terminates, or tandem-switches calls. Generates SS7 messages when calls are set up or torn down. CLECs are SSPs. |
| **STP** | Signal Transfer Point | A packet router for SS7 messages. Does not originate or terminate calls — only routes SS7 signaling packets between SSPs and SCPs. Deployed in redundant pairs. |
| **SCP** | Service Control Point | A database server that responds to queries from SSPs. Used for 800-number translation, LNP lookups, calling card validation, and advanced services. |

---

## SS7 Protocol Stack

SS7 is a layered protocol, analogous to the OSI model:

```
┌──────────────────────────────────┐
│  ISUP / MAP / TCAP / INAP       │  ← Application layer (call control)
├──────────────────────────────────┤
│  SCCP                            │  ← Network services (addressing)
├──────────────────────────────────┤
│  MTP Level 3                     │  ← Network layer (routing)
├──────────────────────────────────┤
│  MTP Level 2                     │  ← Data link layer (error correction)
├──────────────────────────────────┤
│  MTP Level 1                     │  ← Physical layer (56/64 kbps links)
└──────────────────────────────────┘
```

### Layer Details

| Layer | Name | Function |
|---|---|---|
| **MTP 1** | Message Transfer Part Level 1 | Physical signaling link — 56 or 64 kbps digital circuits (DS0) |
| **MTP 2** | Message Transfer Part Level 2 | Error detection and correction on each signaling link. Ensures reliable delivery between two directly connected nodes. |
| **MTP 3** | Message Transfer Part Level 3 | Routing of SS7 messages through the network. Equivalent to IP routing — routes messages based on Point Codes. |
| **SCCP** | Signaling Connection Control Part | Provides additional routing using Global Title Addressing (phone numbers, service codes). Required for TCAP transactions. |
| **TCAP** | Transaction Capabilities Application Part | Enables query/response transactions between SSPs and SCPs. Used for LNP lookups, 800-number routing, calling card validation. |
| **ISUP** | ISDN User Part | Controls circuit-switched voice call setup and teardown between switches. The most important SS7 protocol for a CLEC. |
| **MAP** | Mobile Application Part | Protocols used by mobile networks for HLR/VLR queries, SMS routing, roaming authentication. |

---

## ISUP — How a Call Is Set Up (Step by Step)

ISUP (ISDN User Part) is the SS7 protocol that sets up and tears down telephone calls between carrier switches. The following is the exact message sequence from the reference tutorial (IEC Web ProForum):

**Scenario:** Subscriber on Switch A calls subscriber on Switch B, routed through STPs W and X.

1. Switch A analyses dialed digits and selects an idle trunk to Switch B
2. Switch A formulates an **IAM** (Initial Address Message) containing: calling number, called number, trunk selected, originating switch ID
3. Switch A sends IAM over A link (AW) to STP W
4. STP W inspects routing label → forwards IAM to Switch B over link BW
5. Switch B determines it serves the called number → called number is idle
6. Switch B formulates **ACM** (Address Complete Message), sends ringing tone on trunk toward Switch A, rings called subscriber
7. Switch B sends ACM over A link (BX) to STP X
8. STP X routes ACM to Switch A over link AX
9. Switch A connects calling subscriber to trunk so caller hears ringing
10. Called subscriber answers → Switch B formulates **ANM** (Answer Message)
11. Switch B sends ANM over BX → STP X → AX → Switch A
12. Switch A connects both directions — **conversation begins**, billing starts
13. Calling subscriber hangs up → Switch A formulates **REL** (Release Message)
14. Switch A sends REL over AW → STP W → WB → Switch B
15. Switch B disconnects trunk, returns to idle, formulates **RLC** (Release Complete)
16. Switch B sends RLC over BX → STP X → AX → Switch A
17. Switch A idles the trunk → **call fully torn down**

```
Calling Party          SSP (CLEC/WTS)         SSP (ILEC/AT&T)        Called Party
      │                      │                       │                      │
      │──── Dial digit ──────▶│                       │                      │
      │                      │──── IAM ──────────────▶│                      │
      │                      │  (Initial Address Msg) │                      │
      │                      │                       │──── ring ────────────▶│
      │                      │◀─── ACM ──────────────│                      │
      │                      │ (Address Complete Msg) │                      │
      │◀─── ringback ────────│                       │                      │
      │                      │                       │◀─── ANM ─────────────│
      │                      │◀─── ANM ──────────────│  (Answer Message)    │
      │                      │  (Answer Message)      │                      │
      │◀═══ voice path ══════╪═══════════════════════╪═════════════════════▶│
      │    (call connected)  │                       │                      │
      │                      │                       │                      │
      │──── hang up ─────────▶│                       │                      │
      │                      │──── REL ──────────────▶│                      │
      │                      │  (Release Message)     │                      │
      │                      │◀─── RLC ──────────────│                      │
      │                      │  (Release Complete)    │                      │
```

### Key ISUP Messages

| Message | Abbreviation | Direction | Purpose |
|---|---|---|---|
| Initial Address Message | IAM | Originating → Terminating | Starts call setup. Contains called number, calling number, circuit ID. |
| Address Complete Message | ACM | Terminating → Originating | Called party is being alerted (ringing). |
| Answer Message | ANM | Terminating → Originating | Called party answered. Start billing. |
| Release | REL | Either direction | One party hung up. Request to release the circuit. |
| Release Complete | RLC | Response to REL | Circuit released. Call fully torn down. |
| Continuity | COT | Originating | Tests the voice circuit before completing the call. |

---

## TCAP — Database Queries (SCP Lookups)

TCAP enables an SSP to query a remote SCP database mid-call or pre-call. The following 800-number example illustrates the exact TCAP query flow (from IEC Web ProForum reference):

**Scenario:** Subscriber on Switch A dials an 800 number.

1. Subscriber dials 800 number on Switch A
2. Switch A recognises it is an 800 call requiring database assistance — **suspends call setup**
3. Switch A formulates an **800 query message** (TCAP) with calling number + dialed 800 number, sends to STP X over A link
4. STP X determines this is an 800 query → selects appropriate SCP (e.g., SCP M)
5. STP X forwards query to SCP M over A link
6. SCP M looks up the 800 number → determines the real routing number (or carrier) based on calling number, time of day, day of week
7. SCP M formulates a **TCAP response** with the routing destination, sends back via STP W → Switch A
8. Switch A receives the response → uses routing info to select trunk → generates **IAM** → proceeds with normal ISUP call setup

This same TCAP mechanism is used for WTS:
- **LNP queries** — is the dialed number ported to another carrier?
- **CNAM lookups** — what is the caller ID name for this number?
- **Calling card validation** — is this PIN valid and does the account have credit?

---

## TCAP — Database Queries (WTS Use Cases)

### LNP Query (Local Number Portability)
Before routing a call, WTS must check if the dialed number has been ported to another carrier. This is a TCAP query to the NPAC (Number Portability Administration Center) SCP.

```
WTS SSP ──── TCAP Query (dialed number) ────▶ NPAC SCP
WTS SSP ◀─── TCAP Response (routing number) ─ NPAC SCP
       └─── Route call to correct carrier ───▶
```

### 800-Number Routing
Toll-free numbers are translated via TCAP query to an SCP that returns the actual routing number.

---

## SS7 Point Codes

Every node in an SS7 network has a unique **Point Code** — an address analogous to an IP address. WTS as a CLEC requires its own Point Code assigned by the NANC.

| Network | Point Code Format | Example |
|---|---|---|
| North America (ANSI) | 3-8-8 bit format | 001-001-001 |
| International (ITU) | 14-bit format | varies |

---

## SS7 Link Types

All SS7 signaling links are 56 or 64 kbps bidirectional data links. What differs is their **use** within the signaling network.

| Link | Name | Function |
|---|---|---|
| **A** | Access | Connects SSP or SCP to its home STP pair. All signaling to/from an SSP travels on A links. WTS uses A links to connect to the carrier STP. |
| **B** | Bridge | Interconnects peer mated STP pairs at the same hierarchical level (quad of 4 links). |
| **C** | Cross | Interconnects the two STPs of a mated pair. Used when one STP cannot reach its destination — reroutes via its mate. |
| **D** | Diagonal | Interconnects mated STP pairs at different hierarchical levels. |
| **E** | Extended | Optional backup A links from an SSP to a second STP pair, for enhanced reliability. |
| **F** | Fully Associated | Direct SSP-to-SSP link, bypasses STP. Not typically used between networks. |

**Key deployment rule:** Each SSP connects to **both STPs of a mated pair** via two A links. This ensures that if one STP fails, the other handles all signaling without interruption. STPs and SCPs are always deployed in **mated pairs** for redundancy.

WTS Phase 3 deployment connects to the carrier STP network via **A links**.

---

## SS7 vs SIP — Comparison

| Feature | SS7 | SIP |
|---|---|---|
| Origin | 1975 (ITU-T) | 1996 (IETF) |
| Network | PSTN (TDM) | Internet (IP) |
| Transport | Dedicated 56/64 kbps signaling links | UDP / TCP / TLS |
| Voice path | Separate TDM circuits | RTP over IP |
| Call setup | ISUP IAM → ACM → ANM | INVITE → 180 Ringing → 200 OK |
| Used by | ILECs, mobile carriers, PSTN core | VoIP providers, CLECs, enterprises |
| Security | Physical circuit security | TLS, SRTP, STIR/SHAKEN |
| Cost | Very high (hardware + co-location) | Low (software + internet) |

---

## WTS SS7 Deployment Roadmap

### Phase 1 — SIP Trunking (Launch, Months 1–9)
SIP trunk provider (Bandwidth.com) handles all PSTN interconnection. They do SS7. WTS does SIP.

```
WTS Asterisk ──SIP──▶ Bandwidth.com ──SS7──▶ PSTN/ILECs
```
- No SS7 hardware required
- Cost: per-minute termination rates
- Suitable for: 0–5 million minutes/month

### Phase 2 — SS7 via Gateway (Year 2–3)
At higher volumes, WTS deploys a **Signaling Gateway** that converts between SIP and SS7. This gives direct PSTN interconnection while keeping IP infrastructure.

```
WTS Asterisk ──SIP──▶ Signaling Gateway ──SS7──▶ PSTN/ILECs
                      (Sangoma/Dialogic)
```
- Hardware: Sangoma SS7 cards or Dialogic gateway appliance
- Cost: $15,000–$40,000 hardware + co-location
- Suitable for: 5–50 million minutes/month

### Phase 3 — Direct SS7 Interconnection (Year 3+)
WTS obtains its own SS7 Point Code and connects directly to carrier STPs via A Links. Full carrier-grade interconnection.

```
WTS SS7 Switch ──A Link──▶ Carrier STP ──▶ PSTN
```
- Requires: NANC Point Code, physical co-location, STP access agreements
- Cost: $100,000+ setup + monthly co-location
- Suitable for: 50+ million minutes/month

---

## SS7 Security Considerations

SS7 has well-documented security vulnerabilities due to its age and lack of authentication:

| Vulnerability | Risk | Mitigation |
|---|---|---|
| SS7 location tracking | Caller location exposed via MAP queries | Physical circuit security, monitoring |
| Call interception | ISUP manipulation can reroute calls | Carrier-level monitoring, anomaly detection |
| SMS interception | MAP protocol exposes SMS routing | SMPP over TLS for A2P SMS |
| Denial of service | Flood of ISUP messages can overwhelm a switch | STP-level filtering, rate limiting |

WTS mitigates these by using a managed STP provider (Phase 1–2) and implementing SS7 firewall/monitoring (Phase 3).

---

## Open Source SS7 Tools

| Tool | Purpose |
|---|---|
| **OpenSS7** | Open source SS7 stack for Linux (openss7.org) |
| **Sangoma SS7** | Commercial SS7 cards + open source drivers for Asterisk/Dahdi |
| **osmocom** | Open source SS7/mobile stack (used in research and testing) |
| **SigPloit** | SS7 vulnerability testing framework (security research) |

---

*World Telephony Services Corp — Confidential*
