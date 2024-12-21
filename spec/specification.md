# DID Extension Specification: /whois

## Introduction

The `<did>/whois` path introduces a convention for all DID Methods to enable a
resolver to learn more about the public identity of a [[ref: DID Controller]] through the
discovery of Verifiable Credentials about the DID. Inspired by the WHOIS [[spec:rfc3912]]
protocol from the early ARPANET days, this feature allows resolving
`<did>/whois` to retrieve a Verifiable Presentation (VP) containing Verifiable
Credentials containing public information about the [[ref: DID Controller]]. This capability provides:

- A decentralized mechanism for querying verifiable information about DIDs.
- A trust model based on cryptographic verification of attestations from
  (possibly domain specific) authorities rather than centralized registries.

When `<did>/whois` is resolved using a DID service compliant with the [[spec:linked-vp]], the response is a Verifiable Presentation (VP)
signed by the DID. The VP may contain:

- Information about the DID and the legal entity of the [[ref: DID Controller]] that the
  DID Controller wants to be made public.
- Verifiable Credentials (VCs) with the DID as the `credentialSubject` that may
  bind other identifiers (legal name, legal entity registration identifiers,
  etc.) to the DID.
- VCs with other identifiers bound (via other VCs in the VP) to the DID as the
  `credentialSubject`.

For example, a VC in the VP may use the DID as the `credentialSubject`,
published by an authority, indicating the DID is associated with a legal entity
with a set of identifiers—such as the legal name of the entity and a
registration identifier (e.g., an LEI as published by [GLEIF]). A second VC in
the VP might use the entity identifier from that first VC as the
`credentialSubject` to include attestations such as ISO certifications. The
resolver decides whether to trust the issuers of the VCs, potentially using
`/whois` with those DIDs to gather information about their identity.

[GLEIF]: https://www.gleif.org/

### /whois Use Case

#### Example Scenario: Importing a Product Shipment

Consider the [[ref: DID Controller]] of a mining company exporting a "product"
-- a shipment of the material produced by their mine. The company publishes a
"Digital Product Passport" (DPP) in the form of a Verifiable Credential about
the shipment. An importer resolves the product identifier and retrieves and
verifies the DPP. It wants to know who it was that issued the (self-attested)
DPP verifiable credential. It finds in the DPP the DID of the issuer and then
resolves `<did>/whois`. The mining company has published a VP about its DID,
putting the following verifiable credentials containing public information into
the VP and signing it with its DID:

1. **A Verifiable Credential attesting to the Legal Entity's Registration:**

   - Details such as the company's legal name, registration date, and contact information might be included.
   - The `credentialSubject` of the VC is the same DID that signed the DPP, and the `/whois` VP.
   - This enables the importer to identify and verify the legitimacy of the company.
   - If necessary, the importer can resolve the `<did>/whois` of the issuer of the VC to learn more about them.

2. **A Verifiable Credential for a Mining Permit:**

   - Issued by the mining authority (likely a government) in the company's jurisdiction.
   - This second VC might use the company’s legal name as the
     `credentialSubject` rather than the entity’s DID. The data in the first VC
     provides the binding from the legal name to the DID.
   - Allows further resolution of the mining permit VC's DID and its `/whois`
     path for additional validation.

3. **A Verifiable Credential for Auditing Practices:**

   - Details about the company's compliance with standards audited by a trusted entity.
   - Further resolution of the auditor's DID `/whois` path may verify its accreditation.

The mining company determines what VCs it wants to publish in its `/whois` VP,
and the importer resolving the DPP can resolve "up the hierarchy" of the various
VCs to find the authorities issuing the collection of VCs. In the end, the
importer software may recognize and trust the authorities issuing the VCs and
make a trust decision on the mining company. If not, the software can take the
collected information (VCs and issuers) and pass that along to a person to make
the trust decision.

### Decentralized Trust Registry

The `/whois` path enables:

- Verification of entities across multiple domains using cryptographic proofs.
- Automated or manual trust decisions based on retrieved information.
- Efficient and decentralized trust establishment comparable to a Certificate Authority hierarchy but self-managed by [[ref: DID Controllers]] and issuers.

## Implementation Details

### DID URL Syntax

The `/whois` path conforms to the following syntax:

```
<did>/whois
```

### Resolution Process

1. Resolve the DID and retrieve and verify the DIDDoc.
2. Locate the `<did>#whois` service in the resolved DID Document. This service **MUST** be of type `LinkedVerifiablePresentation`.
   1. A DID Method **MAY** define an implicit `#whois` service that useful for that DID Method. For example, the [[spec:didwebvh]] DID Method has a DID-to-HTTPS-URL transformation used when resolving the DIDDoc, and implicitly defines a `#$whois` service that uses the same DID-to-HTTPS transformation to define the location of the `whois.vp` resource. A [[ref: DID Controller]] **MAY** explicitly include a `#whois` service in their DID Document, overriding the implicit `#whois` service.
3. Resolve the `serviceEndpoint` to obtain the Verifiable Presentation.
4. Verify the VP and the VCs in the VP.
5. Parse the VP to extract Verifiable Credentials and associated metadata.
6. Optionally resolve the `/whois` path of issuers of the VCs for further trust validation as needed and available.

### Example DID Document Service Item

The resolved DID Document must include a service item for `/whois` as follows:

```json
"service": [
  {
    "id": "did:example:123#whois",
    "type": "LinkedVerifiablePresentation",
    "serviceEndpoint": "https://bar.example.com/whois.vp"
  }
]
```

### Verifiable Presentation Requirements

- The returned `whois.vp` **MUST** contain a [[ref: W3C Verifiable Presentation]] signed by the DID.
- At least one of the Verifiable Credentials within the VP **MUST** use the DID as the `credentialSubject`.
  - Such a VC **MAY** attest to the binding of other identifiers to the DID. For example, the VC may be from an authority that attests to the legal name and registration identifier of the entity controlling the DID.
- The `credentialSubject` of other VCs may be identifiers cryptographically bound in other VCs within the VP to the DID being resolved.
  - For example, the legal name or legal entity registration identifier could be the `credentialSubject` if another VC in the VP binds that identifier to the DID. 
- The [[ref: DID Controller]] determines which Verifiable Credentials to include based on their usefulness for establishing the identity and trustworthiness of the entity controlling the DID.
- The resolver determines which Verifiable Credentials are useful in its determination of the identity and trustworthiness of the entity controlling the DID.

What Verifiable Credentials and who should issue those credentials are outside
of the scope of this specification. A future version of the specification may
define at least a credential that binds the DID to the other identifiers used
by/assigned to the legal entity controlling the DID. For example, the
[United Nations Transparency Protocol]'s [Digital Identity Anchor] might be a good
candidate for a standard VC to bind the DID to the legal entity of the
controller.

[United Nations Transparency Protocol]: https://uncefact.github.io/spec-untp
[Digital Identity Anchor]: https://uncefact.github.io/spec-untp/docs/specification/DigitalIdentityAnchor

### Error Handling

Resolvers **MUST** handle errors as follows:

- If the `serviceEndpoint` cannot be resolved, return `Error 404: Not Found`.
- If the file at the defined `serviceEndpoint` is not found, return `Error 404: Not Found`.
- If the VP fails verification, return `Error 500: Server Error`.

## Security and Privacy Considerations

- **Data Minimization:**
  - Ensure only necessary public credentials are included in the VP.
- **Cryptographic Integrity:**
  - All returned VPs must be signed by the DID Controller.
- **Access Control:**
  - There is not an intention to limit access to the `/whois` service endpoint. It is the responsibility of the DID Controller to ensure that only Verifiable Presentations containing public information are published at the service endpoint.
  - Personally identifiable information (PII) **MUST NOT** be published to the `<did>/whois` service endpoint.
- **Exposing Sensitive Information:**
  - Careful scoping of `/whois` data is necessary to avoid unintentional disclosure.

## Conclusion

The `/whois` path standardizes a decentralized mechanism for retrieving verifiable information about DIDs. It empowers trust decisions through cryptographic verification and promotes interoperability across decentralized systems.
