# Verifiable Credentials Profile for MyLegacy

**Companion to the did:etherland Method Specification**

**UFAC Capability Schemas · UCAN Command Namespace · Heritage Credential Types · Issuance Protocols · Invocation Layer · Revocation · ZKP Selective Disclosure**

**Version 1.1 — Draft**
**March 2026**

Author: Etherland Technologies Lda — hub@etherland.tech
https://github.com/EtherlandCertificates/did-etherland/blob/main/profile/vc-profile-mylegacy-v1.1.md

*This document is a companion to the [did:etherland Method Specification v1.3](https://github.com/EtherlandCertificates/did-etherland/blob/main/did-etherland-method-spec-v1.3.md) and the [did:etherland:mylegacy Submethod Specification v1.0](https://github.com/EtherlandCertificates/did-etherland/blob/main/submethod/did-etherland-mylegacy-submethod-v1.0.md), and should be read alongside them.*

---

## 1. Abstract

This document defines the Verifiable Credentials Profile for the `did:etherland:mylegacy` submethod. It specifies the credential schemas, UFAC-based capability delegation mechanisms, issuance and verification protocols, capability invocation layer, revocation registry design, and Zero-Knowledge Proof selective disclosure patterns used within the MyLegacy cultural heritage preservation platform.

This profile is conformant with the [W3C Verifiable Credentials Data Model 2.0](https://www.w3.org/TR/vc-data-model-2.0/) and is designed to interoperate with `did:etherland` DID Documents as specified in the [did:etherland Method Specification v1.3](https://github.com/EtherlandCertificates/did-etherland/blob/main/did-etherland-method-spec-v1.3.md).

UFAC credentials implement UCAN capability semantics within W3C VC containers to maximize interoperability with enterprise identity ecosystems. The semantic model — capability delegation, attenuation via command namespace hierarchy, time bounds, and least authority — derives from the [User-Controlled Authorization Network (UCAN) specification v1.0.0-rc.1](https://github.com/ucan-wg/spec). The serialization format and verification infrastructure conform to W3C VC Data Model 2.0. This hybrid approach preserves the partition-tolerant, delegable, cryptographically verifiable authorization properties of UCAN while leveraging the enterprise-grade tooling and broad institutional adoption of W3C Verifiable Credentials.

The UFAC (User-First Access Credentials) protocol, Etherland's capability-based authorization system, serves as the foundation for all credential issuance, delegation, invocation, and revocation within this profile. UFAC credentials are cryptographically signed certificates that define and transfer permissions between DIDs, adhering to the principles of Time Restriction and Least Authority.

## 2. Scope and Relationship to the DID Method Specification

The [did:etherland Method Specification](https://github.com/EtherlandCertificates/did-etherland/blob/main/did-etherland-method-spec-v1.3.md) (the root document) defines how identifiers are created, resolved, updated, and deactivated. The [did:etherland:mylegacy Submethod Specification](https://github.com/EtherlandCertificates/did-etherland/blob/main/submethod/did-etherland-mylegacy-submethod-v1.0.md) defines the identifier types, DID Document structures, and governance model for the cultural heritage domain. This companion document defines what claims can be made about those identifiers, how those claims are issued and verified, how authority is delegated between entities, and how delegated authority is exercised (invoked) and revoked.

This document MAY be revised independently of the DID method specification and submethod specification. Credential schemas can be added, modified, or deprecated without requiring changes to the core DID method or submethod.

## 3. UFAC Credential Foundation

All verifiable credentials and capability delegations in the MyLegacy ecosystem are built on Etherland's UFAC (User-First Access Credentials) protocol. UFAC provides the cryptographic primitive layer upon which W3C-conformant Verifiable Credentials are constructed.

UFAC is Etherland's domain-specific adaptation of the UCAN (User-Controlled Authorization Network) specification. The UCAN specification defines a trustless, secure, local-first, user-originated, distributed authorization scheme based on certificate capabilities. UFAC inherits UCAN's core lifecycle (Delegation, Invocation, Revocation) and semantic model (Subject, Command, Policy), while expressing credentials in W3C VC JSON-LD format rather than UCAN's native DAG-CBOR envelope. This design decision maximizes interoperability with enterprise identity ecosystems and existing VC verification tooling, while retaining the delegation chain verifiability and capability attenuation properties that distinguish capability-based authorization from traditional ACL models.

UFAC credentials use JSON-LD as their primary encoding when expressed as W3C Verifiable Credentials. For internal system operations, storage indexing, and peer-to-peer credential exchange, UFAC credentials MAY alternatively be encoded in DAG-CBOR format conformant with UCAN's canonical encoding requirements (CIDv1, base58btc multibase, SHA-256 multihash, DAG-CBOR multicodec).

### 3.1 UFAC Core Structure

Every UFAC credential contains the following mandatory elements:

**Subject (`sub`).** The DID of the principal whose authority is being delegated. The Subject identifies the root of the capability chain and MUST remain constant throughout the entire delegation chain. In the MyLegacy context, the Subject is typically the Etherland root authority DID (for platform-delegated capabilities) or a heritage item DID (for item-scoped operations). This field derives from UCAN's `sub` field and ensures that delegation chains are self-certifying: every link in the chain can be traced back to the original authority.

**Issuer (`iss`).** The DID of the entity creating and signing the credential. The issuer determines what permissions are granted and binds them cryptographically to their identity using asymmetric key pairs (ECC secp256k1). The issuer MUST hold sufficient authority over the Subject to grant the specified capabilities.

**Audience (`aud`).** The DID of the entity receiving the credential. The audience is authorized to exercise the granted capabilities within the specified constraints.

**Command (`cmd`).** A slash-delimited path string identifying the action being delegated. Commands follow the UCAN command namespace pattern, where shorter paths prove longer paths (e.g., `/etherland/heritage` proves `/etherland/heritage/approve`). The top command (`/`) delegates all capabilities and SHOULD be used with extreme care. The `/ucan` namespace is reserved.

**Policy (`pol`).** An array of predicate logic statements that constrain the invocation arguments. Policies follow UCAN's syntactically-driven policy language, supporting equality, inequality, glob matching, connectives (`and`, `or`, `not`), and quantifiers (`all`, `any`) with jq-inspired selectors operating on the eventual invocation's `args` field.

**Nonce.** A random value ensuring that every UFAC credential hashes to a unique content identifier. This field is REQUIRED and prevents replay attacks. A randomly generated 12-byte value is RECOMMENDED. For idempotent operations, the nonce SHOULD be deterministic.

**Meta.** An OPTIONAL map of arbitrary metadata, facts, and proofs of knowledge. The enclosed data MUST be self-evident and externally verifiable. It MAY include information such as GPS coordinates at time of attestation, device identifiers, environmental conditions, hash preimages, or server challenges. The data contained in this map MUST NOT be semantically meaningful to delegation chains: it is signed but not delegated authority.

### 3.2 UFAC Principles

**Time Restriction.** Every UFAC credential MUST include an expiration timestamp (`validUntil` / `exp`). Access is only valid for the defined period, reducing the risk of prolonged exposure to sensitive resources. Credentials MAY also include a `validFrom` / `nbf` timestamp for delayed activation. When omitted, the token is treated as valid from the Unix epoch. Keeping the validity window as short as practical is RECOMMENDED. A clock-skew buffer of ±60 seconds is RECOMMENDED for validation.

**Least Authority.** Capabilities MUST be limited to the minimum permissions necessary for the delegated task. A credential granting `/etherland/heritage/approve` for a specific geographic jurisdiction MUST NOT implicitly grant broader permissions such as `/etherland/admin` or `/etherland/finance`.

**Delegation Chains.** UFAC supports hierarchical delegation where a credential holder can issue sub-credentials to other entities, provided the original credential includes the `delegate` capability. Each delegation MUST reduce or maintain the scope of available capabilities (attenuation), ensuring that sub-delegates never exceed the authority of their delegator. The complete chain of delegations is verifiable through cryptographic proof chains. In delegation, the `aud` field of every proof MUST match the `iss` field of the next credential in the chain (Principal Alignment).

### 3.3 UFAC-to-VC Mapping

UFAC credentials are expressed as W3C Verifiable Credentials when they need to interoperate with external systems. The mapping is:

| UFAC Field | W3C VC Field | Notes |
|-----------|-------------|-------|
| `iss` | `issuer` | DID of the credential issuer |
| `aud` | `credentialSubject.id` | DID of the credential recipient |
| `sub` | `credentialSubject.delegationRoot` | Root of the capability chain |
| `cmd` | `credentialSubject.command` | UCAN command path |
| `pol` | `credentialSubject.policy` | UCAN policy predicates |
| `exp` | `validUntil` | Expiration timestamp |
| `nbf` | `validFrom` | Not-before timestamp |
| `nonce` | `credentialSubject.nonce` | Replay prevention |
| `meta` | `credentialSubject.meta` | Non-delegated metadata |

### 3.4 UCAN Command Namespace

The MyLegacy ecosystem defines the following command namespace hierarchy. Shorter command paths delegate authority over all sub-paths. For example, delegating `/etherland/heritage` implicitly includes `/etherland/heritage/approve`, `/etherland/heritage/update`, and all other sub-commands.

| Command | Description |
|---------|-------------|
| `/etherland/heritage` | Root for all heritage operations |
| `/etherland/heritage/approve` | Approve heritage item creation or update |
| `/etherland/heritage/update` | Submit an update to a heritage item |
| `/etherland/heritage/certify` | Issue a Heritage Authenticity Certificate |
| `/etherland/contribution/submit` | Submit a contribution to a heritage item |
| `/etherland/contribution/verify` | Verify and attest to contribution quality |
| `/etherland/institution/accredit` | Issue institutional accreditation |
| `/etherland/tier/attest` | Issue or update contributor tier attestation |
| `/ucan/revoke` | Revoke a delegation (UCAN reserved namespace) |

Implementations MUST NOT define commands under the `/ucan` namespace other than those specified in the UCAN specification. Custom commands MUST be lowercase and slash-delimited per UCAN command syntax rules.

## 4. Credential Type Definitions

This section defines the credential schemas used within the MyLegacy ecosystem. Each credential type is a W3C Verifiable Credential with UFAC-derived capability semantics. All credentials include the REQUIRED `nonce`, `sub` (`delegationRoot`), `cmd`, and `pol` fields per Section 3.1.

### 4.1 Approval Authority Designation Credential

Issued by Etherland to institutions, granting them the authority to approve heritage item creation and updates within a defined scope.

```json
{
  "@context": [
    "https://www.w3.org/ns/credentials/v2",
    "https://schema.etherland.tech/did/v1"
  ],
  "type": ["VerifiableCredential", "ApprovalAuthorityDesignation"],
  "issuer": "did:etherland:mylegacy:inst-0x0000...etherland-root",
  "validFrom": "2026-03-01T00:00:00Z",
  "validUntil": "2027-03-01T00:00:00Z",
  "credentialSubject": {
    "id": "did:etherland:mylegacy:inst-0xf1c3a2b4d5e6f7089a1b...",
    "delegationRoot": "did:etherland:mylegacy:inst-0x0000...etherland-root",
    "command": "/etherland/heritage/approve",
    "policy": [
      ["any", ".jurisdictions", ["or", [["==", ".", "FR"], ["==", ".", "EU"]]]],
      ["any", ".categories", ["or", [["==", ".", "monument"], ["==", ".", "artifact"], ["==", ".", "document"]]]],
      ["<=", ".itemCount", 100]
    ],
    "capability": [{
      "name": "heritageApproval",
      "scope": {
        "jurisdictions": ["FR", "EU"],
        "categories": ["monument", "artifact", "document"],
        "maxItemsPerMonth": 100
      },
      "delegate": true
    }],
    "nonce": "0x7a3f...random12bytes",
    "meta": {}
  },
  "credentialStatus": {
    "id": "https://registry.etherland.tech/revocation/v1/status/1#42",
    "type": "StatusList2021Entry",
    "statusPurpose": "revocation",
    "statusListIndex": "42",
    "statusListCredential": "https://registry.etherland.tech/revocation/v1/status/1"
  },
  "proof": { "type": "DataIntegrityProof", "..." : "..." }
}
```

**Scope Parameters:**
- `jurisdictions`: ISO 3166 country/region codes where the institution may approve items.
- `categories`: Heritage item types the institution may approve (`monument`, `artifact`, `document`, `site`, `intangible`).
- `maxItemsPerMonth`: Optional rate limit on approvals to prevent abuse; omission means no rate limit.
- `delegate`: Controls whether the institution may sub-delegate approval authority to partner institutions or curators. When `true`, sub-delegated credentials MUST have equal or narrower scope (attenuation).

**Policy Mapping:** The `policy` array provides the UCAN-conformant predicate equivalent of the `scope` parameters. When both `capability.scope` and `policy` are present, the `policy` is normative and the `scope` is informative. Verifiers MUST evaluate the policy predicates against invocation arguments.

### 4.2 Heritage Authenticity Certificate

Issued by an approval authority (Etherland or designated institution) to attest to the authenticity and provenance of a heritage item. This credential is created alongside the heritage item DID and references the provenance data in the DID Document.

```json
{
  "@context": [
    "https://www.w3.org/ns/credentials/v2",
    "https://schema.etherland.tech/did/v1"
  ],
  "type": ["VerifiableCredential", "HeritageAuthenticityCertificate"],
  "issuer": "did:etherland:mylegacy:inst-0xf1c3a2b4...",
  "validFrom": "2026-03-15T10:30:00Z",
  "credentialSubject": {
    "id": "did:etherland:mylegacy:item-0x4a3f82e1...",
    "delegationRoot": "did:etherland:mylegacy:inst-0x0000...etherland-root",
    "command": "/etherland/heritage/certify",
    "authenticityStatus": "verified",
    "verificationMethod": "expert-review",
    "verificationDate": "2026-03-15T10:28:00Z",
    "justificationCid": "ipfs://QmR9f3a...creation-justification.pdf",
    "heritageClassification": "UNESCO World Heritage",
    "externalReferenceId": "1153",
    "nonce": "0x8b2e...random12bytes",
    "meta": {}
  },
  "proof": { "type": "DataIntegrityProof", "..." : "..." }
}
```

### 4.3 Contribution Verification Credential

Issued to a contributor's DID after their submission has passed the multi-layered quality verification process (AI screening, peer review, and/or expert verification). This credential serves as proof of contribution quality and is used by the contribute-to-earn reward distribution system.

```json
{
  "@context": [
    "https://www.w3.org/ns/credentials/v2",
    "https://schema.etherland.tech/did/v1"
  ],
  "type": ["VerifiableCredential", "ContributionVerification"],
  "issuer": "did:etherland:mylegacy:inst-0xf1c3a2b4...",
  "validFrom": "2026-06-20T14:15:00Z",
  "credentialSubject": {
    "id": "did:etherland:mylegacy:user-0xb8e2d3c4...",
    "delegationRoot": "did:etherland:mylegacy:inst-0x0000...etherland-root",
    "command": "/etherland/contribution/verify",
    "contributionType": "3d-photogrammetry",
    "targetItem": "did:etherland:mylegacy:item-0x4a3f82e1...",
    "qualityScore": 92,
    "verificationLevel": "expert",
    "pointsAwarded": 450,
    "reviewCid": "ipfs://QmT7e2b...review-report.pdf",
    "nonce": "0x9c3f...random12bytes",
    "meta": {}
  },
  "proof": { "type": "DataIntegrityProof", "..." : "..." }
}
```

### 4.4 Institutional Accreditation Credential

Issued by Etherland to certify an institution's identity, domain expertise, and platform roles. This credential is distinct from the Approval Authority Designation: accreditation certifies *who* the institution is, while approval authority defines *what* the institution can do.

```json
{
  "@context": [
    "https://www.w3.org/ns/credentials/v2",
    "https://schema.etherland.tech/did/v1"
  ],
  "type": ["VerifiableCredential", "InstitutionalAccreditation"],
  "issuer": "did:etherland:mylegacy:inst-0x0000...etherland-root",
  "validFrom": "2026-01-01T00:00:00Z",
  "validUntil": "2028-01-01T00:00:00Z",
  "credentialSubject": {
    "id": "did:etherland:mylegacy:inst-0xf1c3a2b4...",
    "command": "/etherland/institution/accredit",
    "institutionName": "Musée National d'Histoire",
    "institutionType": "museum",
    "accreditationLevel": "academic",
    "jurisdictions": ["FR"],
    "domains": ["medieval-architecture", "decorative-arts"],
    "nonce": "0xa4d1...random12bytes",
    "meta": {}
  },
  "proof": { "type": "DataIntegrityProof", "..." : "..." }
}
```

### 4.5 Contributor Tier Credential

Issued by the MyLegacy platform (automated or manually) to attest to a contributor's current tier status within the contribute-to-earn framework. This credential enables ZKP-based selective disclosure where contributors can prove their tier without revealing their full contribution history.

```json
{
  "@context": [
    "https://www.w3.org/ns/credentials/v2",
    "https://schema.etherland.tech/did/v1"
  ],
  "type": ["VerifiableCredential", "ContributorTierAttestation"],
  "issuer": "did:etherland:mylegacy:inst-0x0000...etherland-root",
  "validFrom": "2026-04-01T00:00:00Z",
  "validUntil": "2026-07-01T00:00:00Z",
  "credentialSubject": {
    "id": "did:etherland:mylegacy:user-0xb8e2d3c4...",
    "command": "/etherland/tier/attest",
    "tier": "expert",
    "totalPoints": 15200,
    "qualifiedSince": "2026-02-15T00:00:00Z",
    "bonusMultiplier": 1.5,
    "nonce": "0xb5e2...random12bytes",
    "meta": {}
  },
  "proof": { "type": "DataIntegrityProof", "..." : "..." }
}
```

**Tier Credential Lifecycle:** Contributor Tier Credentials are short-lived (typically 90 days) and automatically reissued when the contributor's tier is recalculated based on their cumulative points. If a contributor's points drop below the threshold for their current tier, the existing credential is revoked and a new credential with the updated tier is issued.

## 5. Issuance Protocols

This section defines who may issue each credential type and under what conditions.

### 5.1 Issuance Workflow

The general issuance workflow for all credential types follows these steps:

**Step 1 — Precondition Check.** The issuer verifies that the credential subject's DID exists and is active on the VDR, and that the issuer holds the necessary authority (e.g., an Approval Authority Designation credential for heritage-related issuance).

**Step 2 — Credential Construction.** The issuer constructs the credential payload conformant with the schemas defined in Section 4, including all required fields for the credential type. The `nonce` field MUST be populated with a unique random value. The `command` and `policy` fields MUST be set according to the UCAN command namespace (Section 3.4).

**Step 3 — Cryptographic Signing.** The issuer signs the credential using their DID's `assertionMethod` key. The proof is generated using `DataIntegrityProof` with the `ecdsa-secp256k1-2019` cryptosuite, ensuring compatibility with Avalanche's native key infrastructure.

**Step 4 — Storage.** The signed credential is stored in DEFS and optionally delivered directly to the subject's DID-linked endpoint. The credential's CID is recorded in any relevant provenance fields (e.g., `justificationCid` in heritage item DID Documents).

**Step 5 — Status Registration.** The credential's status entry is registered in the StatusList2021 revocation registry (Section 7), enabling future revocation checks by verifiers.

### 5.2 Delegation Issuance

When an institution with the `delegate` capability in its Approval Authority Designation credential wishes to sub-delegate authority, the delegation issuance follows the same workflow with additional constraints:

- The sub-delegated credential's `command` MUST be equal to or a sub-path of the delegator's command (command attenuation).
- The sub-delegated credential's `policy` MUST be equal to or more restrictive than the delegator's policy.
- The sub-delegated credential's `validUntil` MUST NOT exceed the delegator's credential's `validUntil`.
- The sub-delegated credential MUST reference the delegator's credential in its `termsOfUse` property, enabling verifiers to walk the full delegation chain.
- The `sub` field (`delegationRoot`) MUST remain unchanged from the parent credential.
- If the delegator's credential is revoked, all sub-delegated credentials are implicitly invalidated (cascading revocation).
- The `aud` of the parent credential MUST match the `iss` of the sub-delegated credential (Principal Alignment).

## 6. Capability Invocation Layer

UFAC implements the UCAN Invocation specification to formalize the act of exercising delegated authority. While Sections 3–5 describe *who can do what* (delegation), this section describes *the act of doing it* (invocation). The invocation layer is critical for audit trail generation, provenance integrity, and verifiable evidence that authority was properly exercised at a specific moment.

### 6.1 Invocation Structure

An UFAC Invocation is a signed message requesting that an Executor perform a specific action using delegated authority. It contains the following fields:

| Field | Description |
|-------|-------------|
| `iss` | DID of the invoker (the entity exercising the delegated authority) |
| `sub` | DID of the Subject (root of the delegation chain; MUST match the delegation) |
| `aud` | DID of the Executor (the entity that will perform the action) |
| `cmd` | The command being invoked (MUST be within the delegated scope) |
| `args` | The arguments to the command (subject to policy evaluation) |
| `prf` | Array of delegation credential CIDs forming the proof chain |
| `exp` | Expiration timestamp for the invocation itself |
| `nonce` | Unique value for replay prevention |

### 6.2 Invocation Validation

The Executor MUST validate an invocation before executing the requested action. Validation includes the following checks:

**Proof Chain Integrity.** The `prf` array MUST form a direct, unbroken delegation chain from the Subject (`sub`) to the Invoker (`iss`). The `aud` of each delegation MUST match the `iss` of the next delegation in the chain (Principal Alignment). The `sub` field MUST remain constant throughout the chain.

**Command Validation.** The invocation's `cmd` MUST be equal to or a sub-path of the `cmd` in every delegation in the proof chain. For example, an invocation of `/etherland/heritage/approve` is valid against a delegation of `/etherland/heritage` but not against `/etherland/contribution`.

**Policy Evaluation.** The invocation's `args` MUST pass validation against the `policy` predicates of every delegation in the proof chain. Policies are evaluated by substituting `args` values into selectors and evaluating the resulting predicates. If any policy returns `false`, the invocation MUST be rejected.

**Temporal Validity.** All delegations in the proof chain MUST be temporally valid at invocation time: the current time MUST be after every `nbf` and before every `exp` in the chain. The invocation's own `exp` MUST not have passed.

**Revocation Check.** Every delegation CID in the proof chain MUST be checked against the StatusList2021 revocation registry (Section 7). If any delegation is revoked, the invocation MUST be rejected.

**Nonce Uniqueness.** The invocation's CID MUST be checked against a local store of previously seen invocation CIDs to prevent replay attacks.

### 6.3 Receipts

Upon execution of an invocation, the Executor produces a Receipt recording the outcome. Receipts provide the verifiable audit trail that is central to MyLegacy's heritage provenance model.

A Receipt contains: the CID of the invocation it responds to, the outcome (success or failure with error details), any output values produced by the execution, an optional array of new Tasks (follow-up invocations) enqueued by the execution, and the Executor's signature. Receipts are stored in DEFS and their CIDs MAY be referenced in heritage item DID Documents' `provenance` and `updateHistory` fields, creating a verifiable link between the authorization event and the resulting DID Document change.

### 6.4 MyLegacy Invocation Examples

**Heritage Item Approval.** When an institution exercises its `heritageApproval` capability, it issues an invocation with `cmd: /etherland/heritage/approve`, `args` containing the heritage item data, jurisdiction, and category, and `prf` referencing its Approval Authority Designation credential. The Executor (the VDR smart contract or a platform gateway) validates the proof chain, evaluates the policy (checking jurisdiction and category are within scope), and if valid, executes the `registerDID` operation.

**Contribution Verification.** When an institution verifies a contributor's submission, it issues an invocation with `cmd: /etherland/contribution/verify`, `args` containing the contribution type, quality score, and target item, and `prf` referencing its relevant delegation. The resulting Receipt triggers issuance of a Contribution Verification Credential.

## 7. Revocation Registry

The revocation mechanism enables issuers and other authorized parties to invalidate credentials before their natural expiration. This is critical for scenarios such as: institutional misconduct requiring immediate revocation of approval authority, contributor behavior violations, institutional departure from the platform, or key compromise anywhere in a delegation chain.

### 7.1 Registry Architecture

The revocation registry uses the W3C StatusList2021 standard. Each credential type has a dedicated status list credential hosted at a stable URI (e.g., `https://registry.etherland.tech/revocation/v1/status/1`). Each credential receives a unique index position in the status list. The status list is stored as a compact bitstring, where each bit position corresponds to a credential index. A bit value of `0` indicates an active credential; a value of `1` indicates revocation.

The status list credential is stored in DEFS and served over HTTPS at a stable, publicly accessible URL. Because the status list credential is itself a signed Verifiable Credential, its integrity is guaranteed by the issuer's signature regardless of the transport layer. Storing it in DEFS provides content-addressed immutability for each version, while the HTTPS endpoint ensures that standard VC verifier libraries can dereference the `statusListCredential` URL without requiring blockchain-specific tooling. This approach aligns with the W3C StatusList2021 specification's assumption that status lists are HTTPS-accessible resources.

### 7.2 Who May Revoke

Consistent with UCAN revocation semantics, any issuer in the delegation chain MAY revoke delegations they have issued, as well as any delegation that contains their delegation in its proof chain. This is broader than the W3C VC default (where only the original issuer revokes) and is essential for enterprise governance: it ensures that if a mid-chain delegator discovers misuse by a downstream delegate, they can act immediately without waiting for the root authority.

Specifically:
- The root authority (Etherland) MAY revoke any credential in any chain.
- An institution that sub-delegated authority to a curator MAY revoke the curator's credential.
- A curator MAY revoke credentials they issued to downstream parties.

Revocation of a particular delegation does not guarantee that the affected agent loses all access: if they hold an alternative, valid delegation chain that does not include the revoked credential, they retain the capabilities proven by that alternative chain.

### 7.3 Delegating Revocation Authority

The authority to revoke a specific delegation MAY itself be delegated to a principal not in the original delegation chain. This is achieved by issuing an UFAC credential with `cmd: /ucan/revoke` and `args` referencing the CID of the revocable delegation. This pattern is relevant for enterprise compliance scenarios where, for example, a compliance officer or an expert consortium needs revocation authority over institutional credentials without being part of the original delegation chain.

### 7.4 Revocation Process

**Step 1.** The revoker constructs a revocation invocation with `cmd: /ucan/revoke`, `args` containing the CID of the delegation to be revoked and an optional path witness (an ordered array of delegation CIDs connecting the revoker to the revoked delegation in the authority graph).

**Step 2.** The Executor (the Etherland revocation service) validates that the revoker is authorized: either their DID appears as `iss` in the delegation chain containing the revoked credential, or they hold a valid `/ucan/revoke` delegation for the target credential.

**Step 3.** The status list is updated: the bit at the credential's index is flipped from `0` to `1`. A new version of the status list credential is signed by the issuer, published to DEFS (producing a new CID), and the HTTPS endpoint is updated to serve the new version.

**Step 4.** The previous version of the status list credential remains available in DEFS at its original CID for audit purposes. Verifiers fetching the HTTPS endpoint always receive the latest version.

### 7.5 Immutability and Monotonicity

Revocations are immutable and irreversible. Once a credential's bit is flipped to `1`, it MUST NOT be flipped back to `0`. If a revocation was issued in error, the correct remedy is to issue a new delegation credential (with a new nonce, producing a new CID and a new status list index) rather than reversing the revocation. This monotonicity property ensures that revocation stores are append-only and highly amenable to caching, replication, and gossip propagation across distributed networks.

### 7.6 Path Witness

To prevent denial-of-service attacks via spurious revocation requests, Executors MAY require that the revoker include a path witness: an ordered array of delegation CIDs that proves the revoker's relationship to the revoked credential within the delegation graph. The path witness enables the Executor to verify the revoker's authority without traversing the entire delegation graph, and rejects revocation attempts from principals with no relationship to the target credential.

### 7.7 Cascading Revocation

When a parent credential in a delegation chain is revoked, all credentials issued under that delegation are implicitly invalid. Verifiers MUST walk the full delegation chain (via `termsOfUse` references) and check the revocation status of every credential in the chain against the StatusList2021 registry. If any ancestor credential is revoked, the leaf credential MUST be treated as invalid regardless of its own revocation status.

### 7.8 Revocation Store and Eviction

Agents controlling resources within the MyLegacy ecosystem MUST maintain a local cache of revocation status for delegations relevant to their resources. Revocation entries MAY be evicted once the underlying credential has expired (`validUntil` plus a clock-skew buffer), since expired credentials are invalid regardless of revocation status.

## 8. Verification Protocol

Verifiers checking a credential issued under this profile MUST perform the following checks:

1. **Structural Validity.** The credential conforms to W3C VC Data Model 2.0 and matches one of the schemas defined in Section 4.
2. **Signature Verification.** The `proof` is valid and was created by a key listed in the issuer's DID Document under the `assertionMethod` verification relationship.
3. **Issuer Authority.** The issuer is authorized to issue this credential type (per the issuance authority table in Section 5).
4. **Temporal Validity.** The current time is between `validFrom` and `validUntil`.
5. **Revocation Status.** The credential has not been revoked (checked against the StatusList2021 registry per Section 7).
6. **Delegation Chain (if applicable).** All ancestor credentials in the delegation chain are valid, unrevoked, and temporally active. Principal Alignment holds throughout the chain. The `sub` (`delegationRoot`) is constant throughout.
7. **Subject Liveness.** The credential subject's DID is active (not deactivated) on the VDR.
8. **Command and Policy (for invocations).** The invocation's `cmd` is within the scope of the delegation's `cmd`, and the invocation's `args` pass all policy predicates in the chain.

## 9. Zero-Knowledge Proof Selective Disclosure

Etherland's ZKP infrastructure, built on zkSNARKs via PrivadoID integration, enables credential holders to prove specific attributes from their credentials without revealing the full credential content.

### 9.1 Supported Disclosure Patterns

| Pattern | Description | Example |
|---------|-------------|---------|
| Tier threshold | Prove tier meets a minimum level | "I am at least Expert tier" without revealing exact points |
| Contribution count | Prove contributions exceed a threshold | "I have 100+ verified contributions" without revealing total |
| Jurisdiction membership | Prove institutional jurisdiction | "I am accredited in the EU" without revealing specific countries |
| Temporal attestation | Prove credential was valid at a point in time | "I was an active contributor on March 1" without revealing tier details |

### 9.2 ZKP Circuit Architecture

Each selective disclosure pattern is implemented as a zkSNARK circuit that takes the credential as a private input and produces a proof that a specific predicate is satisfied. The circuit architecture follows the PrivadoID pattern:

- **Private Inputs**: The full credential (known only to the holder).
- **Public Inputs**: The predicate to evaluate (e.g., `tier >= expert`), the issuer's DID (for verification), and the credential type.
- **Output**: A boolean result and a proof that the evaluation was performed correctly on a valid credential without revealing the credential content.

Verifiers can validate the ZKP against the issuer's public key without accessing the underlying credential data, preserving holder privacy while maintaining trust in the attestation.

## 10. Security Considerations

### 10.1 Credential Forgery

All credentials are signed using the issuer's `assertionMethod` key as listed in their DID Document. Forging a credential requires compromising the issuer's private key. The secp256k1 elliptic curve provides 128-bit security, and keys are managed through Etherland's WalletAuth component with optional hardware security module integration.

### 10.2 Replay Attacks

Each credential contains a unique `nonce` and a unique `id` URI. For invocations, a unique CID derived from the content (including nonce) is checked against a local store of previously seen CIDs. Verifiers MUST check temporal validity (`validFrom` / `validUntil`) to prevent acceptance of expired credentials. The StatusList2021 revocation registry provides an additional defense against credentials revoked before their natural expiration.

### 10.3 Delegation Abuse

The command hierarchy attenuation rule ensures that sub-delegated credentials can never exceed the command scope of their parent. Policy attenuation ensures that constraints become equal or tighter at each delegation step. Cascading revocation (Section 7.7) ensures that compromised delegation chains can be invalidated from any point by any issuer in the chain. The `delegate` flag in Approval Authority Designation credentials provides explicit control over whether sub-delegation is permitted at all.

### 10.4 Person-in-the-Middle Attacks

UFAC does not have special protection against person-in-the-middle (PITM) attacks on delegation delivery. If a PITM attack is successfully performed, the proof chain would contain the attacker's DID(s). This is detectable by inspecting the chain, and the relevant delegation can be revoked. It is RECOMMENDED to delegate only to agents that are both trusted and authenticated, and to use secure transport channels.

### 10.5 Privacy of ZKP Proofs

ZKP selective disclosure proofs are zero-knowledge: they reveal nothing about the credential beyond the boolean result of the predicate evaluation. However, repeated proofs for the same predicate from the same holder may enable linkability. Implementations SHOULD use randomized proof generation to minimize cross-session correlation.

### 10.6 Revocation Denial of Service

Spurious revocation requests from principals with no relationship to the target delegation are mitigated by the path witness requirement (Section 7.6). Executors SHOULD require a valid path witness for revocation invocations from non-root authorities.

## 11. Credential Lifecycle Summary

| Phase | Action | Actors | Output |
|-------|--------|--------|--------|
| Delegation | Authority is granted via UFAC credential | Etherland → Institution | Approval Authority Designation |
| Sub-delegation | Authority is narrowed and forwarded | Institution → Curator | Attenuated delegation credential |
| Invocation | Delegated authority is exercised | Institution/Curator → Executor | Receipt (stored in DEFS) |
| Issuance | Credential is created and signed | Issuer → Subject | Signed VC (stored in DEFS) |
| Verification | Credential is validated | Verifier | Boolean result |
| Revocation | Credential is invalidated | Any chain issuer | StatusList2021 update |

## 12. Future Considerations

### 12.1 Formal Attenuation Rules for Scope Parameters

The current version of this profile relies on UCAN's command hierarchy for syntactic attenuation (shorter paths prove longer paths) and requires that sub-delegated policies be "equal or more restrictive." A future revision will define formal, programmatic attenuation rules for each scope parameter: set subset validation for jurisdictions and categories (a sub-delegate's set MUST be a subset of the delegator's set), less-than-or-equal validation for numeric limits (`maxItemsPerMonth`, `bonusMultiplier`), and string containment for domain specializations. These rules will enable automated attenuation checking by verifiers without requiring human judgment.

### 12.2 Powerline (Multi-Device Credential Sharing)

UCAN defines a "Powerline" pattern (`sub: null`) that enables automatic delegation of all future delegations to another agent. This is designed for multi-device scenarios where a user's phone, tablet, and laptop should all act as the same identity. Given that MyLegacy contributors may capture photogrammetry data in the field on mobile devices, this pattern is directly relevant. A future revision of this profile will define the conditions under which Powerline delegations are permitted within the MyLegacy ecosystem, limited to the explicit use case of sharing an account's credentials across a user's own peripherals at their explicit request.

### 12.3 Promise Pipelining

The UCAN specification includes a Promise sub-specification that enables distributed promise pipelines: one invocation can await the result of another before completing. This pattern is relevant for complex heritage workflows (e.g., an approval that depends on the outcome of multiple peer reviews). A future revision may incorporate promise semantics for multi-step verification workflows.

## 13. References

### 13.1 Normative References

- **[VC-DATA-MODEL]** W3C. *Verifiable Credentials Data Model v2.0.* https://www.w3.org/TR/vc-data-model-2.0/
- **[DID-CORE]** W3C. *Decentralized Identifiers (DIDs) v1.0.* https://www.w3.org/TR/did-core/
- **[DID-ETHERLAND]** Etherland. *did:etherland Method Specification v1.3.* https://github.com/EtherlandCertificates/did-etherland/blob/main/did-etherland-method-spec-v1.3.md
- **[SUBMETHOD-MYLEGACY]** Etherland. *did:etherland:mylegacy Submethod Specification v1.0.* https://github.com/EtherlandCertificates/did-etherland/blob/main/submethod/did-etherland-mylegacy-submethod-v1.0.md
- **[STATUS-LIST]** W3C. *StatusList2021.* https://www.w3.org/TR/vc-status-list/
- **[UCAN]** UCAN Working Group. *User-Controlled Authorization Network Specification v1.0.0-rc.1.* https://github.com/ucan-wg/spec
- **[UCAN-DELEGATION]** UCAN Working Group. *UCAN Delegation Specification v1.0.0-rc.1.* https://github.com/ucan-wg/delegation
- **[UCAN-INVOCATION]** UCAN Working Group. *UCAN Invocation Specification v1.0.0-rc.1.* https://github.com/ucan-wg/invocation
- **[UCAN-REVOCATION]** UCAN Working Group. *UCAN Revocation Specification v1.0.0-rc.1.* https://github.com/ucan-wg/revocation

### 13.2 Informative References

- **[PRIVADOID]** PrivadoID. *Zero-Knowledge Identity Infrastructure.* https://www.privado.id/
- **[ZCAP-LD]** W3C CCG. *Authorization Capabilities for Linked Data.* https://w3c-ccg.github.io/zcap-spec/
- **[DATA-INTEGRITY]** W3C. *Data Integrity.* https://www.w3.org/TR/vc-data-integrity/
- **[DAG-CBOR]** IPLD. *DAG-CBOR Codec.* https://ipld.io/specs/codecs/dag-cbor/spec/

---

*End of Specification*
