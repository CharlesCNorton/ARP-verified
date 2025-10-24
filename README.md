# RFC826: Verified ARP with Partial Extensions

**Author:** Charles C Norton  
**Date:** October 24, 2025  

A comprehensive formal verification of the Address Resolution Protocol in Coq with security hardening beyond the original 1982 specification.

---

## Abstract

This work presents a complete formal verification of RFC 826 (Address Resolution Protocol) in the Coq proof assistant, extending the base specification with RFC 5227 (IPv4 Address Conflict Detection), RFC 903 (Reverse ARP), and modern security mechanisms. The formalization comprises 9,083 lines of Coq with over 200 theorems proven complete without axioms or admits. Unlike typical protocol formalizations that focus solely on correctness properties, this work proves security guarantees including resistance to cache poisoning, amplification attacks, and cross-subnet spoofing. The verified implementation extracts to executable OCaml code.

---

## Introduction

The Address Resolution Protocol, specified in RFC 826 (Plummer, 1982), remains fundamental to IPv4 networking despite its age. ARP maps network layer addresses (IPv4) to link layer addresses (MAC), enabling packet delivery on local networks. While conceptually simple, ARP's security model reflects the implicit trust assumptions of early Internet design. Modern deployments require defenses against attacks unknown when the protocol was designed: cache poisoning for traffic interception, gratuitous ARP storms for denial of service, and cross-subnet spoofing in multi-homed environments.

This formalization addresses the gap between RFC 826's informal specification and the security requirements of contemporary networks. We provide machine-checked proofs of both the core protocol's correctness and the security properties of our extensions. The work demonstrates that formal verification can simultaneously ensure standards compliance and prove security guarantees beyond those contemplated by the original specification.

---

## Scope and Methodology

### RFC Implementations

The formalization implements three RFCs with security extensions:

1. **RFC 826** (Plummer, 1982): Core protocol including:
   - Packet format (28-byte wire representation)
   - Request/reply semantics
   - Merge algorithm for bidirectional cache updates
   - Ethernet frame encapsulation with CRC-32
   - IEEE 802.1Q VLAN support

2. **RFC 5227** (Cheshire, 2008): IPv4 Address Conflict Detection (ACD):
   - Duplicate Address Detection (DAD)
   - Probe/announce state machine
   - Conflict detection and defense mechanisms

3. **RFC 903** (Finlayson et al., 1984): Reverse ARP:
   - Client/server validation
   - Boot-time address assignment support

### Security Extensions

Beyond these specifications, we implement security mechanisms absent from the RFCs:

- **Broadcast sender rejection**: Prevents amplification attacks by rejecting packets with broadcast source MAC addresses (FF:FF:FF:FF:FF:FF)

- **Multicast sender rejection**: Rejects packets with multicast source MAC addresses, preventing a class of spoofing attacks

- **Static entry protection**: Makes designated cache entries immutable, protecting critical infrastructure mappings (e.g., default gateways) from poisoning

- **Cross-subnet validation**: Rejects packets whose source IP addresses fall outside the receiving interface's subnet, preventing spoofing in multi-interface deployments

- **Flood prevention**: Per-target rate limiting (5 requests per 1000ms window) to mitigate request storms

- **Negative caching**: Records resolution failures with TTL (default 60s), reducing redundant queries for non-existent addresses

- **Cache size bounds**: Maximum 1024 entries to prevent memory exhaustion

### Multi-Interface Support

The implementation extends RFC 826's implicit single-interface model:

- Independent per-interface caches
- Subnet-specific validation
- Routing integration for interface selection
- Per-interface state machines for DAD
- Global flood table across interfaces

---

## Formal Properties

The formalization proves properties across four categories:

### 1. RFC Compliance Theorems

Establish that our implementation correctly realizes the specifications:

- **RFC 826 merge algorithm**: Complete implementation including bidirectional cache updates
- **Sender learning**: Cache updates preserve sender information per RFC 826 Section 2
- **Wire format correctness**: Serialization/parsing proven inverse (`serialize_parse_identity`)
- **Packet structure validation**: 28-byte format enforced (`parse_validates_structure`)
- **Deterministic execution**: Identical inputs produce identical outputs

### 2. Security Theorems

Prove attack resistance properties:

- **Broadcast attack prevention** (`broadcast_sender_rejected`): Packets with broadcast source MAC cannot update caches or generate replies
- **Reply safety** (`reply_never_broadcast`): ARP replies are never sent to broadcast addresses
- **Static entry immutability** (`static_entry_not_overwritten`): Designated entries cannot be modified by incoming traffic
- **Subnet isolation** (`subnet_validation_rejects_cross_subnet`): Cross-subnet packets rejected even when otherwise well-formed
- **DAD conflict detection** (`detect_probe_conflict`): Simultaneous address probing is detected and prevented
- **Flood table correctness**: Rate limiting enforced within specified windows

### 3. Resource Safety Theorems

Bound memory consumption:

- **Cache size limits** (`cache_bounded_invariant_strong`): Maximum 1024 entries, insertion operations respect bound
- **Flood table bounds**: Rate-limiting data structure bounded to 512 entries
- **Negative cache bounds**: Maximum 256 failed resolutions cached
- **Memory exhaustion prevention**: Total memory bounded across all data structures
- **Aging preserves bounds** (`age_cache_length_le`): Cache cleanup never increases size

### 4. Correctness Theorems

Ensure behavioral properties:

- **Determinism**: All processing functions produce unique outputs for given inputs
- **Serialization round-trip**: `parse(serialize(p)) = Some p` for all valid packets
- **Cache lookup soundness**: Presence in cache proven by lookup success
- **State machine transitions**: DAD states transition according to RFC 5227 timing
- **Interface isolation**: Multi-interface operations preserve per-interface invariants

---

## Implementation Structure

### Type System Foundation

- **MAC addresses**: 6-byte tuples `(byte * byte * byte * byte * byte * byte)`
- **IPv4 addresses**: 4-byte records with dotted-decimal semantics
- **ARP packets**: Records with hardware/protocol types, operation codes, and addresses
- **Numeric types**: Binary naturals (type `N`) with proven truncation and bounds

### Wire Format

- **Serialization**: Bitwise operations on binary naturals for efficiency
- **Parsing**: Inverse functions with round-trip identity proofs
- **Ethernet frames**: Header construction, padding, CRC-32 computation
- **VLAN support**: IEEE 802.1Q tagging with priority and ID fields

### Cache Operations

RFC 826 merge algorithm with bounds enforcement:

- **Lookup**: Returns `option MACAddress` with soundness proofs
- **Update**: Size preservation for existing entries, bounded growth for insertions
- **Static entries**: Never aged or replaced, protected from updates
- **Dynamic entries**: Subject to TTL expiration (default 300 seconds)
- **Aging**: Linear-time cleanup removing expired entries

### Advanced Features

- **Flood tables**: Per-IP timestamps and counters for rate limiting
- **Negative caches**: Failed resolution records with bounded size and expiration
- **Timer subsystem**: Monotonic timestamps with expiration predicates
- **State machines**: DAD with explicit states (Idle, Probe, Announce, Conflict, Defend)
- **Pending requests**: Queue with retry logic and maximum attempt limits

### Proof Techniques

- **Induction**: On packet sequences and cache states for protocol properties
- **Adversarial reasoning**: Security proofs assume attacker-controlled packets
- **Arithmetic automation**: `lia` tactic for automatic discharge of size constraints
- **Case analysis**: Exhaustive coverage of operation codes and validation paths
- **Constructive proofs**: End-to-end scenario verification (Alice-Bob transaction, lines 8206-8854)

---

## Comparison with RFC 826

The implementation diverges from RFC 826 in specified ways for security hardening:

| Extension | RFC 826 | This Implementation | Rationale |
|-----------|---------|---------------------|-----------|
| Cache size | Unbounded | Max 1024 entries | Prevent memory exhaustion |
| Broadcast source | Silent | Explicitly rejected | Prevent amplification attacks |
| Static entries | Not mentioned | Immutable entries | Protect infrastructure mappings |
| Subnet validation | Single interface | Per-interface subnets | Prevent cross-subnet spoofing |
| Negative caching | Success only | Failure recording | Reduce redundant queries |
| Rate limiting | None | 5 req/1000ms per target | Mitigate request floods |
| Multicast source | Not addressed | Rejected | Prevent spoofing via multicast |

These extensions represent security hardening informed by decades of deployment experience. **None violate RFC 826's requirements**; they impose additional constraints that strengthen security while maintaining interoperability with compliant implementations.

---

## Extraction and Deployment

### OCaml Extraction

The formalization extracts to OCaml through Coq's extraction mechanism:

```coq
Extraction "arp.ml"
  process_arp_packet
  make_arp_request
  make_arp_reply
  (* ... 100+ additional functions ... *)
```

**Type mappings:**
- `bool` → OCaml `bool`
- `list` → OCaml `list`
- `prod` → OCaml tuples `(*)`
- `option` → OCaml `option`
- `N` (binary naturals) → Efficient big integer representations

**Extracted interface:**
```ocaml
val process_arp_packet : 
  arp_context -> arp_ethernet_ipv4 -> 
  arp_context * arp_ethernet_ipv4 option

val make_arp_request : 
  mac_address -> ipv4_address -> ipv4_address -> 
  arp_ethernet_ipv4

val validate_arp_packet : 
  arp_ethernet_ipv4 -> mac_address -> bool
```

### Integration Requirements

Production deployment requires interfacing with OS network stacks:

1. **Packet I/O**: Raw socket access to Ethernet frames
2. **Timer management**: System clock for TTL tracking and DAD timing
3. **Interface state**: Network interface enumeration and status monitoring
4. **Memory management**: OCaml runtime integration

The extracted functions provide **pure functional interfaces**, separating verified protocol logic from unverified system interaction. This localizes the trusted computing base to the Coq-proven code.

### Potential Integration Targets

- **MirageOS**: OCaml unikernel framework
- **DPDK**: Data Plane Development Kit with OCaml bindings
- **Linux kernel module**: Via OCaml FFI to C wrapper
- **Embedded systems**: Cross-compilation to ARM/MIPS targets
- **Network appliances**: Dedicated ARP proxy or security devices

---

## Verification Statistics

| Metric | Value |
|--------|-------|
| Lines of Coq | 9,083 |
| Definitions | 500+ |
| Theorems/Lemmas | 200+ |
| Admitted proofs | 0 |
| Axioms used | 0 |
| Extracted functions | 100+ |
| RFCs formalized | 3 (826, 903, 5227) |
| Security properties | 10+ major theorems |
| Wire format theorems | 15+ |
| Cache theorems | 30+ |
| Compilation checkpoints | 12 |

---

## Building and Usage

### Requirements

- **Coq**: Version 8.17.1 or later
- **Standard library**: `List`, `NArith`, `Bool`, `Arith`, `Lia`, `Eqdep_dec`
- **Extraction**: Built-in Coq extraction to OCaml

### Compilation

```bash
coqc ARP-verified.v
```

Expected compilation time: 5-15 minutes on modern hardware (varies by proof complexity).

### Extraction

OCaml extraction is configured at the end of the source file:

```bash
coqc ARP-verified.v  # Produces arp.ml
ocamlc -c arp.ml
```

### Testing Integration

The extracted code can be tested with QuickCheck-style property testing:

```ocaml
(* Example: Round-trip property *)
let test_serialize_parse pkt =
  match parse_arp_packet (serialize_arp_packet pkt) with
  | Some pkt' -> pkt = pkt'
  | None -> false
```

---

## Limitations and Future Work

### Current Limitations

1. **IPv4 only**: No IPv6 Neighbor Discovery (ND) support
2. **Single-threaded model**: No concurrent request handling
3. **No hardware offload**: All operations in software
4. **Limited RARP verification**: Legacy protocol less thoroughly proven
5. **No driver integration**: Stops at protocol layer, requires OS bindings

### Future Directions

1. **IPv6 Neighbor Discovery**: Formalize RFC 4861/4862 (ND is ARP's successor)
2. **Compositional verification**: Prove interaction with IP and Ethernet layers
3. **Performance optimization**: Verify fast-path implementations
4. **Hardware integration**: Formal models of NIC behavior
5. **Distributed properties**: Multi-host cache consistency
6. **Dynamic reconfiguration**: Hot interface addition/removal with proofs
7. **Advanced routing**: ECMP, policy-based routing with verification

---

## Related Work

### Protocol Verification

- **CompCert** (Leroy, 2006-2016): Pioneering work in large-scale systems verification, demonstrated feasibility of formally verified compilers in Coq. Established patterns for extraction and correctness preservation.

- **Project Everest** (Bhargavan et al., 2017): Verified TLS implementation in F* achieving both correctness and performance. Demonstrated that verified cryptographic protocols can match unverified implementations in speed.

- **HOL4 TCP** (Bishop et al., 2006): Formalization of TCP in HOL4 with extensive testing against real network traces. Established methodology for validating protocol models against implementations.

- **seL4** (Klein et al., 2009): Verified microkernel proving security and isolation properties. Influenced this work's approach to resource bounds and security properties.

- **Verdi** (Wilcox et al., 2015): Framework for verifying distributed systems in Coq. Provided inspiration for state machine verification approaches.

### Network Security Verification

- **IronFleet** (Hawblitzel et al., 2015): Verified distributed systems with liveness properties in Dafny. Demonstrated composition of safety and liveness in verified systems.

- **Vale** (Bond et al., 2017): Verified cryptographic implementations with performance competitive to hand-optimized assembly. Showed formal verification compatible with efficiency.

### ARP-Specific Work

Prior work on ARP has focused on model checking finite-state abstractions or runtime verification rather than complete implementation proofs. To our knowledge, this is the first formally verified, executable ARP implementation with comprehensive security properties.

---

## Conclusion

This formalization demonstrates that complete verification of production network protocols with security extensions is achievable in proof assistants. The 9,083-line development with zero admits provides mathematical certainty about both RFC compliance and security properties. The extractable implementation offers a path toward more secure network infrastructure.

The methodology developed here—combining standards compliance, security hardening, and complete proofs—provides a template for verifying the broader Internet protocol suite. Future work on composition with higher-layer protocols could enable end-to-end verification from link layer to application layer.

---

## References

1. Plummer, D. C. (1982). **RFC 826: An Ethernet Address Resolution Protocol.** Internet Engineering Task Force.

2. Cheshire, S. (2008). **RFC 5227: IPv4 Address Conflict Detection.** Internet Engineering Task Force.

3. Finlayson, R., Mann, T., Mogul, J. C., & Theimer, M. (1984). **RFC 903: A Reverse Address Resolution Protocol.** Internet Engineering Task Force.

4. Leroy, X. (2009). **Formal verification of a realistic compiler.** *Communications of the ACM*, 52(7), 107-115.

5. Bhargavan, K., Bond, B., Delignat-Lavaud, A., Fournet, C., Hawblitzel, C., Hriţcu, C., ... & Zinzindohoue, J. K. (2017). **Everest: Towards a verified, drop-in replacement of HTTPS.** *SNAPL 2017*.

6. Bishop, S., Fairbairn, M., Norrish, M., Sewell, P., Smith, M., & Wansbrough, K. (2006). **Engineering with logic: HOL specification and symbolic-evaluation testing for TCP implementations.** *ACM SIGPLAN Notices*, 41(1), 55-66.

7. Klein, G., Elphinstone, K., Heiser, G., Andronick, J., Cock, D., Derrin, P., ... & Winwood, S. (2009). **seL4: Formal verification of an OS kernel.** *SOSP '09*, 207-220.

8. Wilcox, J. R., Woos, D., Panchekha, P., Tatlock, Z., Wang, X., Ernst, M. D., & Anderson, T. (2015). **Verdi: A framework for implementing and formally verifying distributed systems.** *PLDI 2015*, 357-368.

9. Hawblitzel, C., Howell, J., Kapritsos, M., Lorch, J. R., Parno, B., Roberts, M. L., ... & Zill, B. (2015). **IronFleet: Proving practical distributed systems correct.** *SOSP '15*, 1-17.

10. Bond, B., Hawblitzel, C., Kapritsos, M., Leino, K. R. M., Lorch, J. R., Parno, B., ... & Zill, B. (2017). **Vale: Verifying high-performance cryptographic assembly code.** *USENIX Security '17*.

---

## Acknowledgments

This work builds upon decades of research in formal verification, protocol design, and network security. The author thanks the Coq development team for providing a robust proof assistant, and the network protocols community for maintaining clear specifications.

---

## Contact

For questions, bug reports, or collaboration inquiries, please contact the author or file issues in the project repository.

**Author:** Charles C Norton  
**Project:** RFC826 Formal Verification  
**Repository:** [To be specified]  
**Version:** 1.0 (August 29, 2025)
