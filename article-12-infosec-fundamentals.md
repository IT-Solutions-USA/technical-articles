# Information Security Fundamentals: CIA, Risk, Access Control, Cryptography & Compliance

**Date:** May 2, 2026  
**Author:** IT Solutions USA  
**Category:** Security / Education  
**Tags:** CISSP, CIA Triad, Risk Management, Bell-LaPadula, Biba, Clark-Wilson, Cryptography, AES, PKI, SOX, HIPAA, GDPR, Penetration Testing, Physical Security, Network Security, Firewalls

---

Sound information security practice rests on a small number of foundational concepts that appear repeatedly across every framework, standard, and regulation in the field. Understanding these concepts deeply — not just their definitions, but their relationships and practical implications — is what separates a security practitioner from someone who has memorised a glossary.

This article covers: the CIA triad, identity and access frameworks, risk quantification, security controls taxonomy, access control models, cryptographic fundamentals, key compliance obligations, and physical and network security principles.

---

## The CIA Triad

| Pillar | Definition |
|---|---|
| **Confidentiality** | Ensures that information is disclosed only to authorised individuals, programs, and processes. Controls include encryption, logical access controls, physical access restrictions, and need-to-know enforcement. The opposite threat is unauthorised Disclosure. |
| **Integrity** | Guarantees that data is not modified in an unauthorised or undetected manner. Data must be consistent and accurate throughout its lifecycle. The opposite threat is Alteration — whether accidental or malicious. |
| **Availability** | Ensures that systems and data are accessible and usable when needed by authorised parties. Achieved through fault tolerance, redundancy, backup, and recovery procedures. The opposite threat is Destruction or denial of service. |

---

## Identity, Authentication, Authorisation, and Accountability (IAAA)

| Term | Definition |
|---|---|
| Identification | The act of a subject claiming an identity — a username, employee ID, or certificate. Used as the basis for all subsequent access control decisions. |
| Authentication | Testing the evidence of identity. Proves the subject is who they claim to be through something they know (password), something they have (token), or something they are (biometric). |
| Authorisation | Defining and enforcing what rights and permissions an authenticated identity is granted — what resources they may access and what actions they may perform on them. |
| Accountability | The ability to trace specific actions to a specific individual. Requires that actions are logged and that those logs are protected from modification. Non-repudiation is the evidence dimension of accountability. |

---

## Risk Quantification

| Formula | Label | Description |
|---|---|---|
| `SLE = Asset Value × Exposure Factor` | Single Loss Expectancy | The dollar value lost when an asset is successfully attacked. The exposure factor (0–1) represents the percentage of the asset's value destroyed in a single incident. If a server worth $100,000 is 40% likely to be destroyed in a fire, SLE = $40,000. |
| `ALE = SLE × ARO` | Annual Loss Expectancy | The expected annual cost of a threat. ARO is the Annualised Rate of Occurrence — how frequently the event is expected in a year. A once-per-century event has ARO = 0.01. ALE drives control investment decisions: a control costing more than ALE is economically unjustifiable. |
| `Residual Risk = Total Risk − Controls Gap` | Residual Risk | The risk remaining after controls are implemented. The controls gap is the reduction in risk achieved by the safeguards in place. Organisations do not eliminate risk — they reduce it to an acceptable level. Legally, remaining residual risk is not held against an organisation when due care has been exercised. |

---

## Risk Responses

| Response | Definition |
|---|---|
| Avoidance | Discontinue the business activity that generates the risk. Complete elimination of exposure but also of the associated business benefit. |
| Transfer | Pass the financial consequences of the risk to another entity — typically through insurance or contractual indemnification. The risk event itself is not eliminated, only the financial exposure. |
| Mitigation | Implement controls to reduce the likelihood or impact of the risk to an acceptable level. The most common response. Cost of mitigation should not exceed ALE. |
| Acceptance | Acknowledge the risk and consciously decide to live with it. Appropriate when the cost of mitigation exceeds the potential loss, or when residual risk after mitigation is below the organisation's risk threshold. |

---

## Security Controls Taxonomy

### By Type

| Type | Examples | Preventive | Detective |
|---|---|---|---|
| Administrative / Managerial | Hiring policies, security awareness training, background checks, separation of duties, mandatory vacations, rotation of duties, acceptable use agreements | Policies, screening, awareness training | Job rotation, audit record review, background verification |
| Technical / Logical | Encryption, firewalls, IDS/IPS, access control lists, biometrics, smart cards, audit logs, PKI | Encryption, firewalls, authentication mechanisms | IDS, automated violation reports, audit trail analysis |
| Physical | Fences, locks, guards, doors, windows, CCTV, motion detectors, badge readers | Fences, locks, security guards, badge access | CCTV, motion sensors, thermal detectors |

### By Category

| Category | Definition |
|---|---|
| Directive | Specify rules of behaviour — acceptable use policies, codes of conduct, legal compliance requirements. |
| Deterrent | Discourage potential attackers by raising the perceived cost or risk of an attack — warning banners, visible cameras, legal notices. |
| Preventive | Stop an incident from occurring — access controls, encryption, firewalls, physical locks. |
| Compensating | Alternative controls that substitute for a primary control when it cannot be implemented — used when a direct control is cost-prohibitive or technically infeasible. |
| Detective | Signal that an incident has occurred or is occurring — IDS alerts, audit logs, CCTV recording. |
| Corrective | Mitigate damage and restore systems after an incident — patch deployment, incident response procedures. |
| Recovery | Restore full operational capability after an incident — backup restoration, disaster recovery failover. |

---

## Administrative Controls

| Control | Definition |
|---|---|
| Separation of duties | Assigns different parts of a sensitive task to different individuals so no single person has end-to-end control. Prevents both accidental errors and deliberate fraud. Effective against insider threats and collusion. |
| Least privilege | Users receive only the minimum rights necessary to perform their assigned function, and only for the duration required. Three access levels: read-only, read/write, and access/change. Reduces the blast radius of a compromised account. |
| Two-man / Dual control | Requires two individuals to complete a sensitive operation. Two-man control means both persons review each other's work. Dual control means both are required to complete the action at all. Used for key escrow recovery, critical configuration changes, and financial authorisations. |
| M of N Control | A generalisation of dual control: a minimum of M agents out of N total authorised agents must cooperate to perform a high-security operation. Example: 3 of 8 key escrow recovery agents must act together to recover a single key. |
| Rotation of duties | Limits how long a person holds a specific security-sensitive role. Reduces collusion risk and allows independent review of prior decisions. Mandatory vacations serve a similar purpose — fraudulent schemes often require continuous presence to maintain. |
| Need to know | A subject is given only the specific information required to complete their assigned task — even if their clearance level would permit broader access. Clearance is necessary but not sufficient; need to know is an additional gate. |

---

## Access Control Models

| Model | Focus | Origin | Notes |
|---|---|---|---|
| Bell-LaPadula | Confidentiality (MAC) | U.S. Department of Defense | Designed to prevent information from flowing from high to low classification. Addresses confidentiality only — not integrity. |
| Biba | Integrity (MAC) | Inverse of Bell-LaPadula | Designed to prevent information from flowing from low integrity to high integrity — protecting data from contamination by untrusted sources. Addresses integrity only, not confidentiality. |
| Clark-Wilson | Integrity (commercial) | Commercial environments | The first model designed for commercial use. Enforces well-formed transactions and separation of duty — more practical than Bell-LaPadula or Biba for enterprise environments. |
| Brewer-Nash (Chinese Wall) | Conflict of Interest | Financial / consulting environments | Prevents members of an organisation from accessing information that creates a conflict of interest with data they have already viewed. Relevant in financial services, legal, and consulting contexts. |

---

## Cryptography

### Symmetric Algorithms

| Algorithm | Detail |
|---|---|
| DES | 56-bit key, 64-bit blocks, 16 rounds of substitution/transposition. Replaced by AES. TripleDES (3DES) applies DES three times with up to three keys (168-bit key length; effective security is approximately 112 bits due to the meet-in-the-middle attack, 48 rounds). |
| AES (Rijndael) | 128/192/256-bit keys, 128-bit blocks. NIST standard since 2001. Used in BitLocker, Microsoft EFS, WPA2, and US government classified data — AES-128 is approved for Secret, AES-256 for Top Secret. The current symmetric encryption standard. |
| Blowfish | 32–448 bit variable key length. Used in bcrypt on Linux systems. Alternative to DES on resource-constrained systems. By Bruce Schneier. |
| RC4 | Stream cipher. Used in WEP and early WPA (TKIP). WEP is broken and should never be used. RC4 itself is considered weak. |

### Asymmetric Algorithms

| Algorithm | Detail |
|---|---|
| RSA | Based on the difficulty of factoring the product of two large prime numbers (trapdoor function). Supports encryption, key exchange, and digital signatures. Standard for asymmetric encryption. |
| Diffie-Hellman | Key exchange protocol — not for encryption directly. Enables two parties to establish a shared secret over an insecure channel without ever transmitting the secret itself. |
| ECC | Elliptic Curve Cryptography. Equivalent security to RSA at much smaller key sizes. Requires fewer computational resources — standard for mobile and embedded systems. |
| DSA | Digital Signature Algorithm. The US government's standard for digital signatures. Used in conjunction with SHA for signing — not for encryption. |

### Hash Algorithms

| Algorithm | Detail |
|---|---|
| MD5 | 128-bit digest. Produces output in four rounds over 512-bit blocks. Subject to collision attacks — two different certificates can produce the same MD5 hash. Not suitable for integrity verification of high-value data or password storage. |
| SHA-1 | 160-bit digest. Designed by NIST/NSA. No longer considered secure for new systems — collisions have been demonstrated. |
| SHA-2 / SHA-3 | Current standards. SHA-256 and SHA-512 are widely deployed. SHA-3 is the newest standard though SHA-2 remains the dominant production choice. |

---

## Key Laws and Regulations

| Law | Area | Detail |
|---|---|---|
| SOX (Sarbanes-Oxley, 2002) | Financial / Corporate | Section 302: CEOs and CFOs personally certify financial statements — criminal liability for knowingly signing incorrect information. Section 404: Internal controls assessment over financial reporting. COSO is the supporting framework for SOX 404 compliance. |
| HIPAA (1996) + HITECH (2009) | Healthcare | HIPAA establishes privacy and security requirements for Protected Health Information (PHI). HITECH extended HIPAA obligations directly to Business Associates (BAs) and introduced mandatory breach notification requirements. Any covered entity–BA relationship must be governed by a Business Associate Agreement (BAA). |
| GLBA (Gramm-Leach-Bliley Act) | Financial / PII | Requires financial institutions to protect the privacy of customer non-public personal information. Mandates a written information security program, privacy notices, and third-party service provider oversight. |
| FERPA | Education | Protects the privacy of student educational records. Requires parental or student consent before records are disclosed to third parties. |
| CFAA (Computer Fraud and Abuse Act, 1986/1996) | Computer Crime | Federal computer crime law. Criminalises unauthorised access to computers, trafficking in computer passwords, and attacks that cause losses of $1,000 or more or could impair medical treatment. |
| ECPA (Electronic Communications Privacy Act, 1986) | Communications Privacy | Prohibits eavesdropping or interception of electronic communications. CALEA (1994) amended ECPA to require communications carriers to support lawful wiretap capability regardless of technology. |
| FISMA (2002) | Federal Government | Requires federal agencies to implement an information security programme covering categorisation (Phase 1), selection of minimum controls, and assessment. NIST SP 800 series provides the implementation framework. |
| GDPR (General Data Protection Regulation, 2018) | Privacy (EU) | Seven principles (Article 5): Lawfulness/Fairness/Transparency, Purpose Limitation, Data Minimisation, Accuracy, Storage Limitation, Integrity/Confidentiality, and Accountability. Individuals have the right to access, correct, and erase their data. Data must be collected for a specified, explicit, and legitimate purpose and not further processed in an incompatible manner. Requires explicit consent or a lawful basis for processing. |

---

## Physical Security: Fire Classification

| Class | Fuel | Suppression Agent | Note |
|---|---|---|---|
| A | Common combustibles (wood, paper) | Water, Soda Acid | Water suppresses temperature |
| B | Liquid and gas fuels | CO2, Dry Chemical | CO2 removes oxygen; dry chemical interrupts the combustion chain. Never use water or soda acid on Class B fires. |
| C | Electrical equipment | CO2, non-conductive gas | Never use water on electrical fires |
| D | Combustible metals | Dry powder | Specialised agent only |

---

## Physical Security: Power Events

| Event | Type | Duration | Response |
|---|---|---|---|
| Spike | Excess | Short | Surge protector |
| Surge | Excess | Prolonged | Surge protector / UPS |
| Fault | Loss | Momentary | UPS |
| Blackout | Loss | Extended | Generator + UPS |
| Sag / Dip | Reduction | Momentary | UPS / CVT |
| Brownout | Reduction | Prolonged | Constant Voltage Transformer |

---

## Network Security: Firewall Generations

| Generation | Name | Layer | Detail |
|---|---|---|---|
| 1st Gen | Static Packet Filtering | Layer 3–4 | Examines source/destination IP, protocol, and port. Uses ACLs to permit or deny. Fast but cannot inspect session state or payload content. Easily evaded by fragmentation attacks. |
| 2nd Gen | Stateful Packet Inspection | Layer 4–7 | Maintains a state table of established connections. Makes decisions based on conversation context, not just individual packets. Detects anomalies that stateless filters miss — such as SYN floods and unexpected packet sequences. |
| 3rd Gen | Application-Level Proxy | Layer 7 | Operates as a relay — client connects to proxy, proxy connects to server. Inspects full payload, can authenticate, re-encrypt, and log application-layer content. Slowest but most thorough inspection. |
| Circuit | Circuit-Level Proxy | Layer 5 | Monitors TCP/UDP handshaking to verify session legitimacy. Once the circuit is established, data tunnels between parties without further inspection. Less granular than application proxies but covers a wider protocol range. |

---

## Intellectual Property

| Type | Duration | Detail |
|---|---|---|
| Patent | 20 years from application | Grants the owner exclusive rights to an invention. After 20 years the invention enters the public domain. Requires public disclosure (registration) — trade-off: protection in exchange for disclosure. |
| Copyright | 70 years after author's death | Protects the expression of ideas, not the ideas themselves. Applies automatically upon creation — no registration required. Covers software, documentation, creative works. |
| Trademark | Renewable every 10 years | Protects words, names, symbols, or combinations used to identify a product or service. Distinguishes the source of goods from competitors. |
| Trade Secret | Indefinite (while kept secret) | Protects proprietary business information critical to commercial advantage — formulas, processes, customer lists. No registration. Protection is lost if the secret is disclosed or independently discovered. |
