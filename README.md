# `did:etherland` — W3C DID Method Specification

A W3C DID Core v1.0 conformant decentralized identifier method for enterprise-grade data infrastructure across EVM-compatible blockchains. Built by [Etherland Technologies Lda](https://etherland.tech).

---

## Repository Structure

```
.
├── README.md
├── did-etherland-method-spec-v1.3.md        # Root method specification (W3C submission)
├── submethods/
│   ├── did-etherland-mylegacy-submethod-v1.0.md   # Cultural heritage preservation
│   └── did-etherland-kya-submethod-v1.0.md        # Know Your Agent (payment verification)
└── profiles/
    └── vc-profile-mylegacy-v1.1.md          # Verifiable Credentials Profile for MyLegacy
```

## Specification Architecture

The `did:etherland` method uses a **three-tier document architecture** designed for independent versioning, clean W3C review, and multi-chain extensibility.

### Tier 1 — Root Method Specification

**[`did-etherland-method-spec-v1.3.md`](did-etherland-method-spec-v1.3.md)**

The document submitted to the [W3C DID Specification Registries](https://github.com/w3c/did-extensions). It defines:

- Method syntax and ABNF
- Chain-agnostic EVM Verifiable Data Registry (VDR) smart contract interface
- Four normative DID operations (Create, Read, Update, Deactivate)
- Submethod chain resolution protocol
- Submethod Extension Framework for registering new vertical domains
- Security and privacy considerations

The root specification contains **no chain-specific references**. The VDR is pure Solidity/EVM, deployable on any compatible chain. Each submethod declares its own chain deployment.

### Tier 2 — Submethod Specifications

Companion documents published alongside the root spec. Each submethod declares its EVM chain, identifier types, DID Document structures, governance model, and UCAN command namespace.

| Submethod | Domain | Chain | Chain ID | Specification |
|-----------|--------|-------|----------|---------------|
| `mylegacy` | Cultural heritage preservation | Avalanche C-Chain | `43114` | [`did-etherland-mylegacy-submethod-v1.0.md`](submethods/did-etherland-mylegacy-submethod-v1.0.md) |
| `kya` | Know Your Agent (payment agent verification) | BNB Smart Chain | `56` | [`did-etherland-kya-submethod-v1.0.md`](submethods/did-etherland-kya-submethod-v1.0.md) |

### Tier 3 — Verifiable Credentials Profiles

Companion documents defining credential schemas, UFAC capability delegation, issuance protocols, revocation registries, and ZKP selective disclosure for each submethod.

| Profile | Submethod | Specification |
|---------|-----------|---------------|
| MyLegacy VC Profile | `mylegacy` | [`vc-profile-mylegacy-v1.1.md`](profiles/vc-profile-mylegacy-v1.1.md) |
| KYA VC Profile | `kya` | *Planned* |

## Key Design Decisions

**Chain-agnostic root, chain-specific submethods.** The W3C registration presents `did:etherland` as an EVM-native method. Resolvers route to the correct chain by looking up the submethod's declared EIP-155 chain ID. This enables multi-chain deployment without modifying the root specification.

**Independent versioning.** Each document has its own version number and revision cycle. A change to the KYA submethod does not force a version bump on the MyLegacy submethod or the root spec.

**UFAC capability framework.** All submethods share Etherland's UFAC authorization protocol (implementing [UCAN](https://github.com/ucan-wg/spec) semantics within W3C VC containers), with submethod-specific command namespaces registered under `/etherland/<submethod>/...`.

## Status

| Document | Version | Status |
|----------|---------|--------|
| Root method spec | v1.3 | Draft — preparing W3C submission |
| MyLegacy submethod | v1.0 | Draft |
| KYA submethod | v1.0 | Draft |
| MyLegacy VC Profile | v1.1 | Draft |
| KYA VC Profile | — | Planned |

## W3C Registration

The registration entry for the W3C DID Specification Registries:

```json
{
  "name": "etherland",
  "status": "registered",
  "specification": "https://docs.etherland.tech/did-method/etherland",
  "contactName": "Etherland / Alexis Brand",
  "contactEmail": "hub@etherland.tech",
  "contactWebsite": "https://etherland.tech",
  "verifiableDataRegistry": "EVM-compatible blockchains (chain-specific deployments per submethod)"
}
```

Submission via pull request to [`github.com/w3c/did-extensions`](https://github.com/w3c/did-extensions) in the `./methods` directory as `etherland.json`.

## Contact

- **Author:** Etherland Technologies Lda
- **Email:** hub@etherland.tech
- **Website:** [etherland.tech](https://etherland.tech)
- **Documentation:** [docs.etherland.tech](https://docs.etherland.tech)

## License

   Copyright 2026 Etherland Technologies Lda.

   Licensed under the Apache License, Version 2.0 (the "License");
   you may not use this file except in compliance with the License.
   You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.
