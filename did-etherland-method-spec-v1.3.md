# DID Method Specification: `did:etherland`

**W3C DID Core v1.0 Conformant**

**Version 1.3 — Draft**
**March 2026**

Author: Etherland Technologies Lda — hub@etherland.tech
https://etherland.tech

*This document is intended for submission to the W3C DID Specification Registries.*

---

## 1. Abstract

This specification defines the Etherland DID Method (`did:etherland`), a W3C DID Core v1.0 conformant decentralized identifier method designed for enterprise-grade data infrastructure across multiple EVM-compatible blockchains.

The `did:etherland` method uses a hierarchical submethod architecture that enables domain-specific identity management. Each submethod declares its own EVM chain deployment, identifier types, governance model, and metadata extensions, while sharing a common smart contract interface, CRUD lifecycle, resolution protocol, and capability-based authorization framework.

This root specification defines the method syntax, the chain-agnostic Verifiable Data Registry (VDR) smart contract interface, the four normative DID operations (Create, Read, Update, Deactivate), and the Submethod Extension Framework through which new vertical domains are registered. Submethod-specific details — including identifier type definitions, DID Document extensions, and domain governance models — are published as companion submethod specifications.

Conformance with [W3C DID Core v1.0](https://www.w3.org/TR/did-core/) is a normative requirement throughout this specification. All terms and definitions are used consistently with the DID Core specification unless explicitly noted otherwise.

## 2. Status of This Document

This is a Draft specification prepared for submission to the [W3C DID Specification Registries](https://github.com/w3c/did-extensions). It has not yet undergone formal W3C review. Implementers are cautioned that this specification may change prior to registration.

The latest version of this document is maintained at: https://github.com/EtherlandCertificates/did-etherland/blob/main/did-etherland-method-spec-v1.3.md

Feedback: Please file issues at the Etherland DID Method repository or contact hub@etherland.tech.

**Changes from v1.2:** This version refactors the specification into a chain-agnostic root method specification with independently published submethod companion documents. The VDR smart contract interface is generalized for any EVM-compatible chain, with chain-specific deployments declared per submethod. The `mylegacy` submethod content (previously Section 5 and Section 10) and the `kya` submethod content are published as separate companion specifications.

## 3. Introduction

Etherland is a decentralized infrastructure company operating a hub-and-spoke architecture where a core technology platform (The Hub) powers multiple vertical applications across compliance, luxury asset management, cultural heritage preservation, payment agent verification, developer tooling, and enterprise consulting.

The need for a custom DID method arises from the specific requirements of Etherland's ecosystem, which demands identity management for heterogeneous subject types including: human users and contributors, organizations and institutions, physical cultural heritage items, digital assets and their provenance chains, AI agents and autonomous software actors, IoT devices and sensor networks, and AI-generated content attestations.

Existing general-purpose DID methods (`did:ethr`, `did:key`, `did:web`) lack the domain-specific semantics, hierarchical submethod architecture, and tight integration with Etherland's proprietary technology stack (DEFS storage, UFAC authorization, APFA permissions) required for enterprise deployment.

### 3.1 Design Goals

**Domain-Specific Semantics.** Submethods carry contextual meaning (e.g., `mylegacy` for cultural heritage, `kya` for payment agent verification), enabling resolvers and verifiers to understand the subject domain from the identifier itself.

**Chain-Agnostic EVM Deployment.** The VDR smart contract interface is EVM-native and chain-agnostic. Each submethod declares its specific EVM chain deployment via an EIP-155 chain ID, enabling the `did:etherland` ecosystem to span multiple blockchains without modifying the root method. The resolution protocol determines which chain to query based on the submethod.

**Enterprise Scalability.** Support for millions of identifiers per submethod with fast resolution via smart contract indexing on EVM-compatible chains.

**Privacy by Design.** Minimal on-chain footprint (CID pointers only), with sensitive metadata stored in encrypted DEFS off-chain storage.

**Interoperability.** Full W3C DID Core v1.0 conformance enabling cross-method resolution through the Universal Resolver.

**Extensibility.** The submethod architecture allows new verticals — whether Etherland's own or third-party builders' — to be added without modifying the root method specification. The Submethod Extension Framework (Section 9) formalizes this extensibility.

**Decentralization.** No single point of control; the VDR operates on public blockchains with permissionless read access.

### 3.2 Terminology

This specification uses terms as defined in W3C DID Core v1.0. Additional terms:

- **DEFS** (Decentralized Encrypted File System): Etherland's IPFS-based off-chain storage layer.
- **UFAC** (User-First Access Credentials): Etherland's capability-based authorization protocol, implementing UCAN (User-Controlled Authorization Network) semantics within W3C Verifiable Credential containers.
- **APFA** (Advanced Permission Framework Authority): Etherland's ecosystem-specific permission layer.
- **VDR** (Verifiable Data Registry): A smart contract deployed on an EVM-compatible chain that serves as the authoritative registry for `did:etherland` identifiers within a given submethod.
- **Chain ID**: An EIP-155 numeric identifier uniquely identifying an EVM-compatible blockchain network.
- **Submethod**: A registered namespace segment within the `did:etherland` method that defines a vertical domain with its own identifier types, chain deployment, and governance model.

## 4. Method Syntax

The `did:etherland` method defines a hierarchical identifier structure conformant with the DID Syntax ABNF defined in DID Core v1.0 Section 3.1.

### 4.1 Root Method Identifier

The general form of a `did:etherland` identifier is:

```
did:etherland:<submethod>:<method-specific-id>
```

Where:
- `did` is the DID URI scheme, as specified in DID Core.
- `etherland` is the DID method name. This value MUST be lowercase.
- `<submethod>` is a lowercase alphanumeric string identifying the vertical domain (e.g., `mylegacy`, `kya`, `mycompliance`, `myattendant`). This segment is REQUIRED.
- `<method-specific-id>` is a unique identifier within the submethod namespace. Encoding rules are defined per submethod.

### 4.2 ABNF Definition

The following ABNF defines the complete syntax:

```abnf
etherland-did      = "did:etherland:" submethod ":" method-specific-id
submethod           = 1*submethod-char
submethod-char      = ALPHA / DIGIT
method-specific-id  = 1*(ALPHA / DIGIT / "_" / "-" / ".")
```

### 4.3 DID Path, Query, and Fragment

`did:etherland` identifiers support the standard DID URL syntax as defined in DID Core v1.0 Section 3.2, including paths, queries, and fragments. For example:

```
did:etherland:mylegacy:item-abc123/contributions       (path)
did:etherland:kya:agent-0x7c3f.../telemetry            (path)
did:etherland:mylegacy:item-abc123?versionId=2         (query)
did:etherland:kya:agent-0x7c3f...#agent-key            (fragment)
```

## 5. Submethod Chain Resolution

Each registered submethod declares an EVM chain deployment, expressed as an EIP-155 chain ID and the contract address of its VDR instance. This information is maintained in the Etherland Submethod Registry (Section 9.3) and is available to resolvers.

### 5.1 Resolution Routing

When resolving a `did:etherland` identifier, the resolver performs the following steps:

1. **Parse the submethod segment** from the DID string.
2. **Look up the submethod** in the Submethod Registry to obtain the chain ID and VDR contract address.
3. **Connect to the appropriate EVM chain** using the chain ID.
4. **Execute the resolution protocol** (Section 7.2) against the declared VDR contract address on that chain.

If the submethod is not found in the registry, the resolver MUST return a `methodNotSupported` error in the DID Resolution Metadata.

### 5.2 Currently Registered Submethods

| Submethod | Chain | Chain ID | VDR Contract | Companion Specification |
|-----------|-------|----------|-------------|------------------------|
| `mylegacy` | Avalanche C-Chain | `43114` | *See submethod spec* | did:etherland:mylegacy Submethod Specification v1.0 |
| `kya` | BNB Smart Chain (BSC) | `56` | *See submethod spec* | did:etherland:kya Submethod Specification v1.0 |

Additional submethods (e.g., `mycompliance`, `myattendant`) will be registered as their companion specifications are published.

## 6. Verifiable Data Registry (VDR)

The Verifiable Data Registry for `did:etherland` is a Solidity smart contract deployable on any EVM-compatible blockchain. Each submethod deploys an instance of this contract on its declared chain. All instances share the same interface, ensuring consistent behavior across the ecosystem.

The contract stores only the minimum data required for on-chain resolution: the DID hash, the IPFS CID pointer, the controller address, the active flag, version counter, and timestamps. All substantive DID Document content — including domain-specific extensions, provenance records, and metadata — is stored off-chain in DEFS, keeping gas costs minimal and avoiding on-chain storage of any personally identifiable information.

### 6.1 Smart Contract Interface

The `EtherlandDIDRegistry` contract exposes the following public interface:

```solidity
interface IEtherlandDIDRegistry {

  struct DIDRecord {
    string  cid;
    address controller;
    uint256 created;
    uint256 lastUpdated;
    uint256 version;
    bool    active;
  }

  function registerDID(bytes32 didHash, string calldata cid, address controller) external;
  function resolveDID(bytes32 didHash) external view returns (DIDRecord memory);
  function updateDID(bytes32 didHash, string calldata newCid) external;
  function deactivateDID(bytes32 didHash) external;
  function transferController(bytes32 didHash, address newController) external;

  event DIDRegistered(
    bytes32 indexed didHash, string cid,
    address indexed controller, uint256 timestamp
  );
  event DIDUpdated(
    bytes32 indexed didHash, string newCid,
    uint256 version, uint256 timestamp
  );
  event DIDDeactivated(bytes32 indexed didHash, uint256 timestamp);
  event ControllerTransferred(
    bytes32 indexed didHash,
    address indexed previousController,
    address indexed newController, uint256 timestamp
  );
}
```

This interface is chain-agnostic. The contract relies solely on EVM primitives (`msg.sender`, `keccak256`, Solidity events) and contains no chain-specific dependencies. Deployment on a new EVM chain requires only specifying the chain's RPC endpoint and funding the deployer address with native gas tokens.

### 6.2 On-Chain Data Minimization

The VDR stores only:
- A 32-byte `didHash` (the keccak256 hash of the full DID string)
- An IPFS CID string (pointer to the off-chain DID Document in DEFS)
- A controller `address` (the Ethereum-style address authorized for write operations)
- A `bool active` flag
- A `uint256 version` counter
- `uint256` timestamps for creation and last update

No personally identifiable information, domain-specific metadata, or credential content is stored on-chain.

## 7. DID Operations (CRUD)

This section specifies the four normative operations for the `did:etherland` method, conformant with DID Core v1.0 Section 8. Submethod specifications MAY impose additional preconditions on these operations (e.g., requiring approval authority for certain identifier types), but MUST NOT weaken the base requirements defined here.

### 7.1 Create

Creating a new `did:etherland` identifier involves a two-phase commit: first to DEFS (off-chain), then to the VDR smart contract (on-chain).

**Preconditions:**
- The creator MUST hold an account on the submethod's declared EVM chain with sufficient native tokens for gas fees.
- The creator MUST have generated a valid DID Document conformant with DID Core v1.0 and this specification.
- Any additional preconditions defined by the submethod specification MUST be satisfied.

**Algorithm:**

1. **Generate Identifier.** Compute the `method-specific-id` using the deterministic generation rules defined by the relevant submethod specification.
2. **Construct DID Document.** Build a valid DID Document including at minimum one `verificationMethod`, the `authentication` relationship, and any required service endpoints. Include any submethod-required extensions.
3. **Store DID Document in DEFS.** Encrypt the DID Document using AES-256 (with public portions remaining accessible) and upload to DEFS. The system returns an IPFS CID as the content-addressed pointer.
4. **Anchor on VDR.** Call `registerDID(bytes32 didHash, string cid, address controller)` on the submethod's VDR contract instance. The `didHash` is `keccak256(abi.encodePacked(didString))` where `didString` is the full DID URI.
5. **Emit Event.** The smart contract MUST emit a `DIDRegistered(bytes32 indexed didHash, string cid, address indexed controller, uint256 timestamp)` event for off-chain indexing.

### 7.2 Read (Resolve)

Resolution is the process of obtaining a DID Document from a `did:etherland` identifier. The resolver operates in three stages: submethod routing, on-chain lookup, and off-chain retrieval.

**Algorithm:**

1. **Parse Identifier.** Validate the DID syntax against the ABNF in Section 4.2. Extract the `submethod` and `method-specific-id`.
2. **Route to Chain.** Look up the submethod in the Submethod Registry (Section 5.1) to obtain the chain ID and VDR contract address.
3. **Query VDR.** Call `resolveDID(bytes32 didHash)` on the VDR contract. The contract returns a `DIDRecord` struct containing: `cid` (string), `controller` (address), `created` (uint256), `lastUpdated` (uint256), `version` (uint256), and `active` (bool).
4. **Check Active Status.** If `active` is `false`, the resolver MUST return a DID Resolution Result with `didDocument` set to `null` and `didDocumentMetadata.deactivated` set to `true`.
5. **Fetch from DEFS.** Using the CID from the on-chain record, retrieve the DID Document from DEFS/IPFS. Validate the document's JSON-LD structure and verify that the `id` property matches the resolved DID.
6. **Construct Resolution Result.** Return a DID Resolution Result conformant with DID Core v1.0 Section 8.1, including `didResolutionMetadata` (contentType, retrieved, chainId), `didDocument`, and `didDocumentMetadata` (created, updated, versionId, deactivated).

The `didResolutionMetadata` SHOULD include a `chainId` property indicating the EVM chain on which the VDR was queried, enabling verifiers to confirm the authoritative registry.

### 7.3 Update

Updating a DID Document replaces the IPFS CID pointer on the VDR while preserving the complete version history in DEFS.

**Preconditions:**
- The transaction sender MUST be the current `controller` address as recorded in the VDR.
- The DID MUST be in an active (non-deactivated) state.
- Any additional preconditions defined by the submethod specification MUST be satisfied.

**Algorithm:**

1. Construct the updated DID Document. The `id` property MUST NOT change. Include any submethod-required update metadata (e.g., update history entries).
2. Upload the updated document to DEFS. DEFS automatically creates a new CID while maintaining a linked version chain to the previous document.
3. Call `updateDID(bytes32 didHash, string newCid)` on the VDR. The controller address is verified by `msg.sender`.
4. The smart contract MUST emit a `DIDUpdated(bytes32 indexed didHash, string newCid, uint256 version, uint256 timestamp)` event.

**Key Rotation:** When rotating verification keys, the updated DID Document SHOULD include both the new and the old key (marked for decommission) during a transition period to avoid service disruption.

### 7.4 Deactivate

Deactivation permanently disables a DID. This operation is irreversible.

**Algorithm:**

1. Call `deactivateDID(bytes32 didHash)` on the VDR. The controller address is verified by `msg.sender`.
2. The contract sets the `active` flag to `false` and records a `deactivationTimestamp`.
3. The smart contract MUST emit a `DIDDeactivated(bytes32 indexed didHash, uint256 timestamp)` event.

**Post-Deactivation Behavior:** Once deactivated, any attempt to resolve the DID MUST return a DID Resolution Result with `didDocument` set to `null` and `didDocumentMetadata.deactivated` set to `true`. The VDR MUST reject any update or reactivation attempts for deactivated DIDs.

## 8. Security Considerations

This section addresses the security considerations required by DID Core v1.0 Section 10.

### 8.1 Authentication of the DID Controller

Controller authentication is enforced at two levels. On-chain, the VDR smart contract verifies that the transaction sender (`msg.sender`) matches the registered controller address for all write operations (update, deactivate, transferController). Off-chain, Etherland's WalletAuth component verifies DID ownership by requesting an EIP-712 typed data signature.

### 8.2 Key Rotation and Recovery

Key rotation is supported through the Update operation (Section 7.3). Controllers can replace their verification keys by publishing an updated DID Document with new cryptographic material. The controller address itself can be changed via the `transferController` function, which enables account recovery through social recovery mechanisms or hardware wallet migration.

### 8.3 Replay Attack Prevention

Each Update and Deactivate transaction carries a monotonically increasing version counter enforced by the smart contract. The contract rejects any operation that does not increment the version, preventing replay of stale transactions.

### 8.4 Integrity of the Verifiable Data Registry

The VDR inherits the security properties of the underlying EVM chain's consensus mechanism. The specific security guarantees (finality time, fault tolerance model) depend on the submethod's declared chain. Submethod specifications SHOULD document the security properties of their chosen chain.

The registry contract is immutable once deployed (non-upgradeable), with no admin keys or proxy patterns, regardless of the deployment chain.

### 8.5 Cryptographic Suites

The `did:etherland` method supports the following cryptographic suites for verification methods:
- `EcdsaSecp256k1RecoveryMethod2020`
- `JsonWebKey2020` with the `secp256k1` curve
- `Ed25519VerificationKey2020`

AES-256 encryption is used for DID Document storage in DEFS, providing quantum-resistant data protection.

### 8.6 Denial of Service

On-chain registration requires gas payment on the submethod's declared chain, which provides natural rate limiting against mass DID creation attacks. Off-chain resolution via IPFS benefits from the distributed nature of the DEFS cluster.

### 8.7 Multi-Chain Integrity

Each submethod is authoritative on exactly one EVM chain. Resolvers MUST query only the chain declared in the Submethod Registry. If a DID is registered on a chain other than the submethod's declared chain, the resolver MUST NOT return it. This prevents cross-chain replay attacks where a DID registered on a test chain is presented as valid on a production chain.

## 9. Submethod Extension Framework

The `did:etherland` method is designed as an extensible platform where new vertical submethods can be registered and deployed without modifying the root method specification. This framework defines the requirements, process, and governance for submethod extensions, enabling third-party builders — including those leveraging Etherland's Technology Launchpad — to create domain-specific DID submethods that share the common VDR interface while introducing their own identifier types, service endpoints, governance models, and metadata extensions.

### 9.1 Rationale

The submethod extension framework addresses a specific need in the EVM ecosystem: providing W3C-conformant DID infrastructure that any builder can extend for their domain without deploying separate DID methods. Rather than positioning `did:etherland` as a generic identity method (which would conflict with existing general-purpose methods like `did:key` and `did:pkh`), this framework positions it as a governed infrastructure layer that builders extend through submethods. Each submethod carries its own domain semantics, chain deployment, governance model, and extension vocabulary, while inheriting the shared VDR interface, CRUD operations, resolution infrastructure, and UFAC capability framework.

### 9.2 Shared Infrastructure

All submethods — whether Etherland-operated or third-party — share the following infrastructure:

**Verifiable Data Registry Interface.** The `IEtherlandDIDRegistry` smart contract interface defined in Section 6.1. All submethods deploy an instance of this contract on their declared EVM chain, using the same `registerDID`, `resolveDID`, `updateDID`, `deactivateDID`, and `transferController` functions.

**Resolution Protocol.** The three-stage resolution process (submethod routing + on-chain CID lookup + off-chain DEFS retrieval) defined in Section 7.2.

**Identifier Syntax.** All submethods conform to the ABNF in Section 4.2. The submethod segment is registered and unique within the `did:etherland` namespace.

**UFAC Authorization Framework.** Submethods MAY use Etherland's UFAC capability delegation mechanism (implementing UCAN semantics) for their authorization needs. The UCAN command namespace is extensible: submethod builders define commands under their own namespace (e.g., `/etherland/<submethod>/...`) and policy predicates specific to their domain.

**DEFS Storage.** Off-chain DID Document storage, encryption, and content addressing through Etherland's DEFS infrastructure.

### 9.3 Submethod Registration Requirements

A conformant submethod specification MUST define the following:

1. **Submethod Name.** A lowercase alphanumeric string conformant with the ABNF in Section 4.2. MUST be unique within the `did:etherland` namespace.
2. **Chain Deployment.** The EIP-155 chain ID and the deployed VDR contract address on that chain.
3. **Identifier Types.** The set of DID subject types supported by the submethod, each with a type prefix convention and deterministic `method-specific-id` generation rules.
4. **DID Document Structure.** Normative examples of DID Documents for each identifier type, including required and optional properties, service types, and any custom JSON-LD context extensions.
5. **Governance Model.** The rules governing who may create, update, and deactivate DIDs within the submethod, including any approval workflows or role-based restrictions beyond the base controller authentication.
6. **UCAN Command Namespace.** The set of commands registered under `/etherland/<submethod>/...` for use with the UFAC authorization framework.
7. **Security and Privacy Considerations.** Domain-specific threat analysis and mitigation strategies, supplementing the considerations in Section 8 of this root specification.

A conformant submethod specification SHOULD additionally define:

8. **Extension Vocabulary.** Custom JSON-LD properties (e.g., `etherland:agentProfile`, `etherland:heritageMetadata`) with their semantics and validation rules.
9. **Conformance Profiles.** Application-level expectations for production deployments, distinguishing between protocol-level requirements and recommended structures.
10. **Companion Specifications.** References to any companion documents (e.g., Verifiable Credentials Profiles) that define credential schemas, issuance protocols, and revocation mechanisms for the submethod's domain.

### 9.4 Registration Process

**Phase 1 — Proposal.** The builder submits a submethod proposal to Etherland, including the submethod specification conformant with Section 9.3, a description of the use case and target domain, and contact information.

**Phase 2 — Review.** Etherland reviews the proposal for: ABNF conformance of the submethod name and identifier types, absence of conflicts with existing registered submethod names, DID Core v1.0 conformance of the proposed DID Document structure, security and privacy adequacy for the target domain, and validity of the declared chain deployment.

**Phase 3 — Registration.** Upon approval, the submethod name is registered in the `did:etherland` Submethod Registry. The builder receives: access to the shared VDR interface for their submethod's identifiers, UFAC command namespace allocation under `/etherland/<submethod>/...`, and DEFS storage allocation.

**Phase 4 — Publication.** The submethod specification is published as a companion document alongside the root `did:etherland` specification and referenced in the W3C DID Method Registration entry.

### 9.5 Governance

Submethod registration is currently managed by Etherland as the method operator. The registration process is invitation-based, initially available to builders participating in Etherland's Technology Launchpad vertical. As the ecosystem matures, Etherland anticipates formalizing a community governance process for submethod registration, potentially involving an advisory committee of submethod operators.

Etherland provides the shared VDR interface specification on an open-source basis. Builders requiring dedicated governance guarantees or independent operations MAY deploy their own VDR instance using the open smart contract specification defined in Section 6.1, while maintaining ABNF-conformant identifiers that remain resolvable by any `did:etherland` resolver.

### 9.6 Published Submethods

| Submethod | Domain | Chain (Chain ID) | Status | Specification |
|-----------|--------|-----------------|--------|---------------|
| `mylegacy` | Cultural heritage preservation | Avalanche C-Chain (`43114`) | Published | did:etherland:mylegacy Submethod Specification v1.0 |

### 9.7 Planned Submethods

| Submethod | Domain | Target Chain | Status |
|-----------|--------|-------------|--------|
| `kya` | Payment agent verification (Know Your Agent) | BNB Smart Chain (`56`) | Private Draft |
| `mycompliance` | ESG & real estate compliance | TBD | In development |
| `myattendant` | Luxury inventory management | TBD | In development |

## 10. Privacy Considerations

This section addresses the privacy considerations required by DID Core v1.0 Section 10.

### 10.1 On-Chain Data Minimization

The VDR stores only the minimum data necessary for resolution: a 32-byte DID hash, an IPFS CID string, a controller address, a boolean active flag, a version counter, and timestamps. No personally identifiable information (PII) or domain-specific content is stored on-chain, regardless of the submethod or chain deployment.

### 10.2 Off-Chain Data Protection

DID Documents stored in DEFS benefit from multiple layers of protection. Private DID Documents (e.g., user profiles with personal information) are encrypted using AES-256 before IPFS pinning. Public DID Documents (e.g., institutional registrations intended for open access) remain unencrypted but contain no PII beyond publicly available organizational information.

### 10.3 Pairwise and Pseudonymous Identifiers

The `did:etherland` method supports the creation of multiple DIDs per entity. Subjects may create separate DIDs for different contexts, maintaining pseudonymity across submethods and chains.

### 10.4 Selective Disclosure and Zero-Knowledge Proofs

Etherland's architecture includes Zero-Knowledge Proof capabilities via zkSNARKs, enabling DID subjects to prove specific attributes without revealing the underlying data. ZKP selective disclosure patterns are specified in companion Verifiable Credentials Profile documents for each submethod.

### 10.5 Right to Erasure Considerations

While the on-chain DID hash and CID pointer are immutable, the actual DID Document content in DEFS can be made inaccessible by the controller. Deactivation of the DID on-chain, combined with unpinning of the IPFS content from all DEFS cluster nodes, effectively removes access to the DID Document.

## 11. Conformance

### 11.1 Protocol-Level Requirements

A conformant `did:etherland` implementation MUST satisfy the following requirements:

- Implement all four DID operations (Create, Read, Update, Deactivate) as specified in Section 7.
- Produce DID Documents conformant with DID Core v1.0.
- Support the JSON-LD representation with the DID Core v1.0 context (`https://www.w3.org/ns/did/v1`).
- Deploy the VDR smart contract on the submethod's declared EVM chain.
- Validate all DID syntax against the ABNF defined in Section 4.2.
- Enforce controller authentication for all write operations as specified in Section 8.1.
- Return conformant DID Resolution Results as specified in Section 7.2.
- Route resolution to the correct chain based on the submethod segment as specified in Section 5.1.

### 11.2 Conformance Profiles

This specification distinguishes between protocol-level requirements (the rules above, which all implementations MUST follow) and application-level expectations (the recommended structures that production deployments SHOULD follow for consistency across the platform).

Protocol-level requirements define what constitutes a valid `did:etherland` DID Document. They cover the W3C-mandated elements: `@context`, `id`, at least one `verificationMethod`, and at least one verification relationship. All custom extensions — including submethod-specific service endpoints, metadata blocks, and profile structures — are defined by their respective submethod specifications and are optional at the root protocol level.

## 12. W3C DID Method Registration

The following JSON object represents the registration entry for the W3C DID Specification Registries:

```json
{
  "name": "etherland",
  "status": "registered",
  "specification": "https://github.com/EtherlandCertificates/did-etherland",
  "contactName": "Etherland / Alexis Brand",
  "contactEmail": "hub@etherland.tech",
  "contactWebsite": "https://etherland.tech",
  "verifiableDataRegistry": "EVM-compatible blockchains (chain-specific deployments per submethod)"
}
```

Registration is submitted via pull request to `github.com/w3c/did-extensions` in the `./methods` directory as `etherland.json`.

## 13. References

### 13.1 Normative References

- **[DID-CORE]** W3C. *Decentralized Identifiers (DIDs) v1.0.* W3C Recommendation, 2022. https://www.w3.org/TR/did-core/
- **[DID-SPEC-REGISTRIES]** W3C. *DID Specification Registries.* https://www.w3.org/TR/did-spec-registries/
- **[VC-DATA-MODEL]** W3C. *Verifiable Credentials Data Model v2.0.* https://www.w3.org/TR/vc-data-model-2.0/
- **[STATUS-LIST]** W3C. *StatusList2021.* https://www.w3.org/TR/vc-status-list/
- **[UCAN]** UCAN Working Group. *User-Controlled Authorization Network Specification v1.0.0-rc.1.* https://github.com/ucan-wg/spec
- **[RFC3986]** IETF. *Uniform Resource Identifier (URI): Generic Syntax.*
- **[RFC5234]** IETF. *Augmented BNF for Syntax Specifications: ABNF.*
- **[EIP-155]** Ethereum. *Simple replay attack protection.* https://eips.ethereum.org/EIPS/eip-155

### 13.2 Informative References

- **[DID-IMP-GUIDE]** W3C. *DID Implementation Guide v1.0.* https://w3c.github.io/did-imp-guide/
- **[EIP-712]** Ethereum. *Typed structured data hashing and signing.*
- **[IPFS]** Protocol Labs. *InterPlanetary File System.* https://ipfs.tech/
- **[ETHR-DID-REGISTRY]** Veramo. *ethr-did-registry.* https://github.com/uport-project/ethr-did-registry

### 13.3 Companion Specifications

- **[SUBMETHOD-MYLEGACY]** Etherland. *did:etherland:mylegacy Submethod Specification v1.0.* [https://github.com/EtherlandCertificates/did-etherland/blob/main/submethods/did-etherland-mylegacy-submethod-v1.0.md](https://github.com/EtherlandCertificates/did-etherland/blob/main/submethod/did-etherland-mylegacy-submethod-v1.0.md)
- **[VC-PROFILE-MYLEGACY]** Etherland. *Verifiable Credentials Profile for MyLegacy v1.1.* https://github.com/EtherlandCertificates/did-etherland/blob/main/profiles/vc-profile-mylegacy-v1.1.md

---

*End of Specification*
