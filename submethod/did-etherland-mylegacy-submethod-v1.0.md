# Submethod Specification: `did:etherland:mylegacy`

**Companion to the did:etherland Method Specification v1.3**

**Cultural Heritage Preservation · Artifact Provenance · Contributor Identity · Institutional Collaboration**

**Version 1.0 — Draft**
**March 2026**

Author: Etherland Technologies Lda — hub@etherland.tech
https://github.com/EtherlandCertificates/did-etherland/blob/main/submethods/did-etherland-mylegacy-submethod-v1.0.md

*This document is a companion to the did:etherland Method Specification v1.3 and should be read alongside it.*

---

## 1. Abstract

This specification defines the `mylegacy` submethod of the `did:etherland` DID method. The submethod is purpose-built for cultural heritage preservation, addressing the identity requirements of tangible and intangible cultural assets, the contributors who document them, and the institutions that steward them.

The `mylegacy` submethod is deployed on Avalanche C-Chain (EIP-155 chain ID: `43114`) and defines three identifier types: heritage items (`item-`), contributors (`user-`), and institutions (`inst-`). Heritage item DIDs follow a governed lifecycle model where authorized approval authorities verify content quality before on-chain registration, ensuring data integrity while maintaining full provenance transparency.

This specification conforms to the root `did:etherland` Method Specification v1.3 and inherits its EVM-agnostic VDR smart contract interface, CRUD operations, resolution protocol, and security/privacy considerations.

## 2. Status of This Document

This is a Draft specification published as a companion to the `did:etherland` Method Specification v1.3. It has not undergone formal W3C review. The root method specification is the document submitted to the W3C DID Specification Registries; this companion document is referenced by and published alongside it.

The latest version is maintained at: https://github.com/EtherlandCertificates/did-etherland/blob/main/submethods/did-etherland-mylegacy-submethod-v1.0.md

Feedback: hub@etherland.tech

## 3. Chain Deployment

| Property | Value |
|----------|-------|
| **Submethod name** | `mylegacy` |
| **EVM chain** | Avalanche C-Chain |
| **EIP-155 chain ID** | `43114` |
| **Consensus mechanism** | Snowman (Avalanche consensus protocol) |
| **Finality** | Sub-second |
| **VDR contract** | `EtherlandDIDRegistry` (address published at deployment) |

The Avalanche C-Chain is selected for its sub-second finality, low transaction costs, and EVM compatibility. All `did:etherland:mylegacy` identifiers are resolved by querying this chain's VDR instance, following the resolution routing protocol defined in the root specification (Section 5.1).

## 4. Identifier Types

The `mylegacy` submethod supports three categories of DID subjects, each distinguished by a type prefix in the `method-specific-id`:

| Prefix | Subject Type | Controller Model |
|--------|-------------|-----------------|
| `item-` | Heritage items (monuments, artifacts, documents, sites) | Governed: controlled by approval authority (institution or Etherland) |
| `user-` | Contributors (individuals documenting heritage) | Self-sovereign: controlled by the contributor |
| `inst-` | Institutions (museums, universities, heritage bodies) | Self-sovereign: controlled by the institution |

## 5. Method-Specific-ID Generation

The `method-specific-id` for each subject type is generated deterministically:

**Heritage Items (`item-`).** The suffix is derived from `keccak256(abi.encodePacked(submitterAddress, externalReferenceId, registrationTimestamp))`, truncated to 20 bytes and hex-encoded. The `externalReferenceId` may be a UNESCO World Heritage ID, a national inventory number, a cadastral reference, or a platform-assigned sequential identifier.

**Contributors (`user-`).** The suffix is the Avalanche C-Chain address of the contributor's primary wallet, linked via WalletAuth. This creates a 1:1 binding between the contributor's Web3 identity and their MyLegacy DID.

**Institutions (`inst-`).** The suffix is derived from `keccak256(abi.encodePacked(adminAddress, institutionName, registrationTimestamp))`, ensuring globally unique institutional identifiers.

### 5.1 ABNF Extension

```abnf
mylegacy-did        = "did:etherland:mylegacy:" mylegacy-id
mylegacy-id         = mylegacy-type "-" hex-suffix
mylegacy-type       = "item" / "user" / "inst"
hex-suffix          = 40HEXDIG
```

## 6. DID Documents

### 6.1 Heritage Item DID Document

A heritage item DID Document contains the cryptographic material, service endpoints, and provenance tracking records needed to verify the item's origin, manage contributions, and maintain a complete audit trail of all lifecycle events. Heritage items are not self-sovereign: their controller is Etherland or a designated institution with verification authority, and their DID Documents record the identity of both the requestor and the approval authority for every creation and update operation.

The following is a normative example:

```json
{
  "@context": [
    "https://www.w3.org/ns/did/v1",
    "https://w3id.org/security/suites/jws-2020/v1",
    "https://schema.etherland.tech/did/v1"
  ],
  "id": "did:etherland:mylegacy:item-0x4a3f82e1b9d0c5f6a7e3d2c1b0a9f8e7d6c5b4a3",
  "controller": "did:etherland:mylegacy:inst-0xf1c3a2b4d5e6f7089a1b2c3d4e5f60718293a5e7",
  "verificationMethod": [{
    "id": "...#key-1",
    "type": "JsonWebKey2020",
    "controller": "did:etherland:mylegacy:inst-0xf1c3...a5e7",
    "publicKeyJwk": { "kty": "EC", "crv": "secp256k1", "x": "...", "y": "..." }
  }],
  "authentication": ["...#key-1"],
  "assertionMethod": ["...#key-1"],
  "service": [
    {
      "id": "...#defs-storage",
      "type": "DEFSStorageEndpoint",
      "serviceEndpoint": "ipfs://Qm...",
      "description": "Primary DEFS vault for heritage documentation"
    },
    {
      "id": "...#heritage-metadata",
      "type": "HeritageMetadataService",
      "serviceEndpoint": "https://api.mylegacy.etherland.tech/v1/items/...",
      "description": "REST API for item metadata, 3D scans, and contribution history"
    },
    {
      "id": "...#contribution-feed",
      "type": "ContributionFeed",
      "serviceEndpoint": "wss://feed.mylegacy.etherland.tech/v1/items/...",
      "description": "Real-time WebSocket feed for new contributions"
    }
  ],
  "etherland:heritageMetadata": {
    "itemType": "monument",
    "externalIds": {
      "unescoWhId": "1153",
      "nationalInventoryId": "PA00085796"
    },
    "registeredAt": "2026-03-15T10:30:00Z",
    "geoLocation": { "latitude": 48.8606, "longitude": 2.3376 }
  },
  "etherland:provenance": {
    "requestedBy": "did:etherland:mylegacy:user-0xb8e2d3c4a5f607182930...",
    "approvedBy": "did:etherland:mylegacy:inst-0xf1c3a2b4d5e6f7089a1b...",
    "approvalTimestamp": "2026-03-15T10:28:00Z",
    "justificationCid": "ipfs://QmR9f3a...creation-justification.pdf"
  },
  "etherland:updateHistory": [{
    "versionId": 2,
    "requestedBy": "did:etherland:mylegacy:user-0xc7d4e5f6a8b9012345...",
    "approvedBy": "did:etherland:mylegacy:inst-0xf1c3a2b4d5e6f7089a1b...",
    "approvalTimestamp": "2026-06-20T14:15:00Z",
    "justificationCid": "ipfs://QmT7e2b...update-review-report.pdf",
    "changeDescription": "Added 3D photogrammetry scan and condition update"
  }]
}
```

### 6.2 Contributor DID Document

Contributors are individuals who participate in cultural heritage documentation through the contribute-to-earn model. Their DID Document links their cryptographic identity to their contribution history and earned reputation within the platform:

```json
{
  "@context": [
    "https://www.w3.org/ns/did/v1",
    "https://w3id.org/security/suites/jws-2020/v1",
    "https://schema.etherland.tech/did/v1"
  ],
  "id": "did:etherland:mylegacy:user-0xb8e2d3c4a5f6071829304a5b6c7d8e9f0a1b2c3d",
  "controller": "did:etherland:mylegacy:user-0xb8e2d3c4a5f6071829304a5b6c7d8e9f0a1b2c3d",
  "verificationMethod": [{
    "id": "...#wallet-key",
    "type": "EcdsaSecp256k1RecoveryMethod2020",
    "controller": "...user-0xb8e2...2c3d",
    "blockchainAccountId": "eip155:43114:0xb8e2d3c4..."
  }],
  "authentication": ["...#wallet-key"],
  "assertionMethod": ["...#wallet-key"],
  "service": [{
    "id": "...#contribution-portfolio",
    "type": "ContributorPortfolio",
    "serviceEndpoint": "https://api.mylegacy.etherland.tech/v1/contributors/..."
  }],
  "etherland:contributorProfile": {
    "tier": "expert",
    "totalPoints": 15200,
    "verifiedContributions": 347,
    "specializations": ["photography", "3d-scanning", "architectural-history"]
  }
}
```

Note the `blockchainAccountId` uses the CAIP-10 format with chain ID `43114` (Avalanche C-Chain).

### 6.3 Institution DID Document

Institutions serve as stewards, quality verifiers, and approval authorities. Their DID Document reflects their organizational authority, verification capabilities, and their role as potential controllers for heritage item DIDs:

```json
{
  "@context": [
    "https://www.w3.org/ns/did/v1",
    "https://w3id.org/security/suites/jws-2020/v1",
    "https://schema.etherland.tech/did/v1"
  ],
  "id": "did:etherland:mylegacy:inst-0xf1c3a2b4d5e6f7089a1b2c3d4e5f60718293a5e7",
  "controller": "did:etherland:mylegacy:inst-0xf1c3a2b4d5e6f7089a1b2c3d4e5f60718293a5e7",
  "verificationMethod": [{
    "id": "...#admin-key",
    "type": "JsonWebKey2020",
    "controller": "...inst-0xf1c3...a5e7",
    "publicKeyJwk": { "kty": "EC", "crv": "secp256k1", "x": "...", "y": "..." }
  }],
  "authentication": ["...#admin-key"],
  "assertionMethod": ["...#admin-key"],
  "capabilityDelegation": ["...#admin-key"],
  "service": [{
    "id": "...#institutional-api",
    "type": "InstitutionalAPI",
    "serviceEndpoint": "https://api.mylegacy.etherland.tech/v1/institutions/..."
  }],
  "etherland:institutionProfile": {
    "institutionType": "museum",
    "verificationAuthority": true,
    "jurisdictions": ["FR", "EU"],
    "accreditationLevel": "academic"
  }
}
```

## 7. Heritage Item Governance Model

Heritage item DIDs follow a governed lifecycle model that separates the roles of requestor, approval authority, and controller. This ensures that all content published to the permanent cultural heritage record has been quality-verified before on-chain registration, while maintaining full transparency about who initiated and who authorized every change.

### 7.1 Roles

**Requestor.** Any entity holding a valid `did:etherland:mylegacy` DID (`user-` or `inst-` prefix) may submit a creation or update request. The requestor's DID is permanently recorded in the item's provenance and update history.

**Approval Authority.** An entity authorized to approve requests and execute the on-chain operation. Two classes of approval authority exist:
- *Etherland* (the platform operator, acting as root authority for the `mylegacy` submethod)
- *Designated Institutions* (institutions whose DID Documents contain `verificationAuthority: true` in their `institutionProfile` extension, as granted by Etherland via UFAC capability delegation)

**Controller.** The on-chain controller of the heritage item DID, which is always the approval authority that executed the creation. The controller may be Etherland's own institutional DID or a designated institution's DID. The controller is the only entity that can execute subsequent updates or deactivation on the VDR.

### 7.2 Request-Approve Workflow

The creation and update of heritage item DIDs follows this workflow:

**Step 1 — Submission.** The requestor submits a creation or update request through the MyLegacy platform, including the proposed content (documentation, metadata, media) and a justification document explaining the basis for the submission.

**Step 2 — Quality Verification.** The approval authority reviews the submission through the multi-layered verification process: automated AI screening for format and completeness, followed by human expert review for accuracy and scholarly merit.

**Step 3 — Justification Document.** Upon approval, the approval authority creates a justification document summarizing the review outcome and stores it in DEFS. The resulting CID becomes the `justificationCid` recorded in the item's provenance or update history.

**Step 4 — DID Document Construction.** The approval authority constructs (or updates) the heritage item DID Document, populating the `etherland:provenance` block (for creation) or appending to the `etherland:updateHistory` array (for updates) with the requestor's DID, its own DID as approver, the timestamp, and the justification CID.

**Step 5 — On-Chain Registration.** The approval authority, acting as the controller, executes the `registerDID` or `updateDID` operation on the VDR smart contract on Avalanche C-Chain.

### 7.3 Approval Authority Designation

Etherland designates institutions as approval authorities through the UFAC capability delegation mechanism. UFAC implements UCAN capability semantics within W3C Verifiable Credential containers. An institution receives approval authority when Etherland issues a UFAC credential with the `/etherland/heritage/approve` command to the institution's DID. This credential specifies the scope of the delegation (which geographic jurisdictions or heritage categories the institution may approve) via UCAN policy predicates, and its time restriction via `validUntil`.

UFAC credentials support hierarchical delegation: an institution with the `delegate` capability may sub-delegate to curators or partner institutions, subject to the attenuation rule (sub-delegated scope MUST be equal or narrower). Etherland or any issuer in the delegation chain may revoke this delegation at any time through the UFAC revocation mechanism (conformant with both UCAN Revocation and W3C StatusList2021), at which point the institution retains controller status for items it has already approved but cannot approve new ones.

The complete UFAC credential schemas, UCAN command namespace, delegation and invocation protocols, revocation registry, and ZKP selective disclosure circuits are specified in the companion *Verifiable Credentials Profile for MyLegacy v1.1*.

## 8. Registration Flow

Contributor onboarding follows a progressive five-phase lifecycle:

**Phase 1 — Email Onboarding (Off-Chain).** The user signs up with email. A `did:key` (secp256k1) is generated client-side, with the private key stored in device secure storage. A minimal DID Document is persisted to DEFS.

**Phase 2 — Platform Participation (`did:key`).** The platform issues a UFAC credential granting `/etherland/contribution/submit` to the user's `did:key`. Contributions (photos, 3D scans, metadata) are submitted and attributed to this ephemeral identity. VCs are issued to the `did:key` as contributions accumulate.

**Phase 3 — Wallet Linking (User-Initiated).** The user connects an Avalanche C-Chain wallet via WalletAuth. An EIP-712 typed data signature challenge confirms ownership.

**Phase 4 — On-Chain DID Creation.** The `method-specific-id` is generated (`user-0xb8e2d3c4...`). A full DID Document with `etherland:contributorProfile` is constructed, stored in DEFS, and anchored on the VDR via `registerDID`. The VDR emits a `DIDRegistered` event.

**Phase 5 — Credential Migration.** Contribution VCs are re-issued from `did:key` to `did:etherland:mylegacy:user-0xb8e2...`. New platform delegation is issued to the on-chain DID. The `did:key` becomes a device delegate under the on-chain DID, enabling seamless continued operation on the same device.

## 9. MyLegacy-Specific Extensions

### 9.1 Heritage Service Types

The `mylegacy` submethod defines the following service types for use in DID Documents:

| Service Type | Description | Used By |
|-------------|-------------|---------|
| `DEFSStorageEndpoint` | Primary DEFS vault for heritage documentation | Heritage items |
| `HeritageMetadataService` | REST API for item metadata, 3D scans, contribution history | Heritage items |
| `ContributionFeed` | Real-time WebSocket feed for new contributions | Heritage items |
| `ContributorPortfolio` | Contributor's contribution history and reputation | Contributors |
| `InstitutionalAPI` | Institution management and verification endpoints | Institutions |
| `UFACCredentialEndpoint` | Endpoint for UFAC credential exchange | All types |

### 9.2 Provenance and Audit Trail Extensions

The `mylegacy` submethod defines two structured extensions for heritage item lifecycle tracking:

**`etherland:provenance`** — A single object recorded at item creation, containing: `requestedBy` (the DID of the user or institution that submitted the creation request), `approvedBy` (the DID of the approval authority that verified and executed the creation), `approvalTimestamp` (ISO 8601 timestamp of the approval decision), and `justificationCid` (IPFS CID pointing to the justification document stored in DEFS).

**`etherland:updateHistory`** — An append-only array where each entry records a single update event, containing: `versionId` (integer version number corresponding to the VDR version counter), `requestedBy`, `approvedBy`, `approvalTimestamp`, `justificationCid`, and `changeDescription` (a human-readable summary of what was changed and why).

### 9.3 Verification Relationships

The `mylegacy` submethod uses DID Core verification relationships to encode authorization semantics specific to heritage preservation:

- **`authentication`**: Used to prove control of the DID during platform interactions (login, contribution submission, institutional actions).
- **`assertionMethod`**: Used to sign Verifiable Credentials attesting to heritage item provenance, condition assessments, scholarly reviews, and contribution quality verification.
- **`capabilityDelegation`**: Used by Etherland to delegate approval authority to institutions, and by institutions to delegate verification roles to curators and researchers. Delegation is performed via UFAC credentials implementing UCAN capability semantics.
- **`capabilityInvocation`**: Used to authorize operations on behalf of the DID subject, such as automated IoT sensor updates to a heritage item's condition data.

### 9.4 Contribute-to-Earn Integration

MyLegacy's contribute-to-earn economic model relies on DID-based identity for contribution attribution, quality verification, and reward distribution. Each contribution is cryptographically signed by the contributor's DID, creating an immutable attribution chain. The `requestedBy` field in heritage item provenance permanently links the contributor's identity to the items they help create and enrich.

Reward distribution flows to controller addresses associated with contributor DIDs, ensuring that ELAND token earnings are linked to verifiable identities without exposing personal information. Platform-generated rewards are funded by subscription revenue — contributors do not pay each other directly.

### 9.5 UCAN Command Namespace

The `mylegacy` submethod registers the following commands under `/etherland/`:

| Command | Description |
|---------|-------------|
| `/etherland/heritage` | Root command for all heritage operations |
| `/etherland/heritage/approve` | Approve heritage item creation or update |
| `/etherland/heritage/update` | Submit an update request for a heritage item |
| `/etherland/heritage/certify` | Issue a Heritage Authenticity Certificate |
| `/etherland/contribution/submit` | Submit a contribution to a heritage item |
| `/etherland/contribution/verify` | Verify and attest to contribution quality |
| `/etherland/institution/accredit` | Issue institutional accreditation |
| `/etherland/tier/attest` | Issue or update contributor tier attestation |

### 9.6 Verifiable Credentials

The `did:etherland:mylegacy` method is designed to interoperate with the W3C Verifiable Credentials Data Model 2.0. Credential types include: Heritage Authenticity Certificate, Contribution Verification Credential, Institutional Accreditation Credential, Contributor Tier Credential, and Approval Authority Designation Credential.

The credential schemas, UFAC capability definitions, issuance and verification protocols, capability invocation layer, revocation mechanisms, and Zero-Knowledge Proof circuits for selective disclosure are specified in the companion document: *Verifiable Credentials Profile for MyLegacy v1.1*.

## 10. Conformance Profiles

This section defines application-level expectations for production MyLegacy deployments, supplementing the protocol-level requirements in the root specification (Section 11).

**Heritage Item Profile (`item-`).** Production heritage item DID Documents SHOULD include: the `etherland:provenance` block (REQUIRED by governance model), the `etherland:updateHistory` array (appended on each update), at least one service endpoint of type `DEFSStorageEndpoint` pointing to the item's DEFS vault, the `etherland:heritageMetadata` block with at minimum `itemType` and `registeredAt`, and a `controller` field referencing the approval authority's institutional DID.

**Contributor Profile (`user-`).** Production contributor DID Documents SHOULD include: a `verificationMethod` linked to the contributor's Avalanche wallet via `blockchainAccountId` (CAIP-10 format with chain ID `43114`), the `etherland:contributorProfile` block with at minimum `tier` and `totalPoints`, and a service endpoint of type `ContributorPortfolio`.

**Institution Profile (`inst-`).** Production institution DID Documents SHOULD include: the `etherland:institutionProfile` block with at minimum `institutionType` and `verificationAuthority`, the `capabilityDelegation` verification relationship (for institutions acting as approval authorities), and a service endpoint of type `InstitutionalAPI`.

## 11. Security Considerations

This section supplements the security considerations in the root `did:etherland` specification (Section 8).

### 11.1 VDR Chain Security

The Avalanche C-Chain VDR inherits the security properties of the Snowman consensus protocol, which provides finality in under two seconds with Byzantine fault tolerance. The registry contract is immutable once deployed (non-upgradeable), with no admin keys or proxy patterns.

### 11.2 Provenance Integrity

The `etherland:provenance` and `etherland:updateHistory` records in heritage item DID Documents are protected by the same integrity guarantees as the DID Document itself: the IPFS CID is content-addressed (any modification changes the CID), and the CID is anchored on-chain by the controller. Tampering with provenance records would require both compromising the controller's private key and re-pinning modified content to DEFS, which the IPFS content-addressing model makes detectable.

### 11.3 Requestor Privacy in Provenance Records

The `requestedBy` field in heritage item DID Documents contains a `did:etherland:mylegacy` DID, which is pseudonymous by default. The requestor's real-world identity is not exposed unless they have voluntarily linked their DID to a verifiable credential containing personal information. Requestors should be informed during the submission process that their DID will be permanently and publicly recorded in the item's provenance metadata.

## 12. References

### 12.1 Normative References

- **[DID-ETHERLAND]** Etherland. *did:etherland Method Specification v1.3.* https://github.com/EtherlandCertificates/did-etherland/blob/main/did-etherland-method-spec-v1.3.md
- **[DID-CORE]** W3C. *Decentralized Identifiers (DIDs) v1.0.* https://www.w3.org/TR/did-core/
- **[VC-DATA-MODEL]** W3C. *Verifiable Credentials Data Model v2.0.* https://www.w3.org/TR/vc-data-model-2.0/
- **[STATUS-LIST]** W3C. *StatusList2021.* https://www.w3.org/TR/vc-status-list/
- **[UCAN]** UCAN Working Group. *User-Controlled Authorization Network Specification v1.0.0-rc.1.* https://github.com/ucan-wg/spec
- **[AVALANCHE]** Avalanche Platform Documentation. https://build.avax.network/

### 12.2 Informative References

- **[VC-PROFILE-MYLEGACY]** Etherland. *Verifiable Credentials Profile for MyLegacy v1.1.* https://github.com/EtherlandCertificates/did-etherland/blob/main/profiles/vc-profile-mylegacy-v1.1.md
- **[IPFS]** Protocol Labs. *InterPlanetary File System.* https://ipfs.tech/

---

*End of Specification*
