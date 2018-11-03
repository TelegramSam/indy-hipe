- Name: Connection Protocol
- Authors: Ryan West ryan.west@sovrin.org, Daniel Bluhm daniel.bluhm@sovrin.org, Matthew Hailstone, Stephen Curran, Sam Curren <sam@sovrin.org>
- Start Date: 2018-6-29
- PR:
- Jira Issue: https://jira.hyperledger.org/browse/IA-18

# Summary
[summary]: #summary

This HIPE describes the protocol to establish connections between agents with the assumption that message transportation is [solved](https://github.com/hyperledger/indy-hipe/pull/21). We assume that the DIDs exchanged are recorded on a public ledger. We also describe how we will accommodate Peer DIDs and how we will adapt this connection protocol when that work is complete.

# Motivation
[motivation]: #motivation

Indy agent developers want to create agents that are able to establish connections with each other and exchange secure information over those connections. For this to happen there must be a clear connection protocol.

# Tutorial
[tutorial]: #tutorial

We present the scenario in which Alice and Bob wish to communicate. The following interactions and messages must be sent between them to establish a secure, persistent channel for communication:

0. [Invitation to connect](#0-invitation) (optional; dependent on scenario as explained later)
1. [Connection Request](#1-connection-request)
2. [Connection Response](#2-connection-response)

Each of these steps is explored in detail below.

## 0. Invitation to Connect
[0-invitation]: #1-invitation

An invitation to connect may be transferred using any method that can reliably transmit text. The result  must be the essential data necessary to initiate a [Connection Request](#2.-connection-request) message. An connection invitation is a valid agent message type, but is an **out-of-band communication** and therefore not communicated using wire level encoding or encryption. The necessary data that an invitation to connect must result in is:
* endpoint did
* suggested label

#### Standard Invitation Encoding

Using a standard invitation encoding allows for easier interoperability between multiple projects and software platforms.

The standard invitation format is a URL with a Base64URLEncoded json object as a query parameter. Using a URL allows mobile apps to register as handlers for the URL for users who already have a Wallet App installed, and new users can be provided with getting started instructions.

The URL format is as follows, with some elements described below:

```text
https://<domain>/<path>?c_i=<invitationstring>
```

`<domain>` and `<path>` should be kept as short as possible, and the full URL should return human readable instructions when loaded in a browser. This is intended to aid new users. Additional path elements or query parameters are allowed, and can be leveraged to provide coupons or other promise of payment for new users. 

The `<invitationstring>` is an agent message (not a wire level message), with the following fields:

```javascript
b64urlencode({
	'@type': 'did:sov:BzCbsNYhMrjHiqZDTUASHg;spec/connections/1.0/invitation',
    'did': 'A.did@B:A',
	'label': 'Alice'
})
```

Example, using a sample DID of real length:

```javascript
b64urlencode({
	'@type': 'did:sov:BzCbsNYhMrjHiqZDTUASHg;spec/connections/1.0/invitation',
    'did': 'did:sov:QmWbsNYhMrjHiqZDTUTEJs',
	'label': 'Alice'
})
```
During encoding, whitespace should be eliminated to keep the resulting invitation string as short as possible.
```text
eydAdHlwZSc6J2RpZDpzb3Y6QnpDYnNOWWhNcmpIaXFaRFRVQVNIZztzcGVjL2Nvbm5lY3Rpb25zLzEuMC9pbnZpdGF0aW9uJywnZGlkJzonZGlkOnNvdjpRbVdic05ZaE1yakhpcVpEVFVURUpzJywnbGFiZWwnOidBbGljZSd9
```
Example URL:
```text
http://example.com/ssi?c_i=eydAdHlwZSc6J2RpZDpzb3Y6QnpDYnNOWWhNcmpIaXFaRFRVQVNIZztzcGVjL2Nvbm5lY3Rpb25zLzEuMC9pbnZpdGF0aW9uJywnZGlkJzonZGlkOnNvdjpRbVdic05ZaE1yakhpcVpEVFVURUpzJywnbGFiZWwnOidBbGljZSd9
```
Example URL encoded as a QR Code:

![exampleqr](exampleqr.png)

#### Example
Alice first creates an invitation to connect. She'll create a new DID with associated key, endpoint, and routing information, and writes it to the ledger. She wraps this into an invitation, which gives Bob the necessary information to initiate a connect request with her. This can be done in person with a phone to phone QR code scan, or remotely by sharing the URL. The correlatability of the resulting connection depends on the security of the invitation transfer. 

After receiving Alice's invitation to connect, Bob extracts the invitation string from the `c_i` query parameter in the URL, B64UrlDecodes the text, and JSON parses it. If Bob wishes to connect, he will generate the DID and keys that will be used in the Alice to Bob (`A:B`) relationship and create a connection request message.

#### Invitation via published DID

When it is possible for Bob to discover Alice's DID, no invitation is necessary. This is the likely flow when Alice is an institution or other public service such as a Bank or a government department. 


## 1. Connection Request
[1-connection-request]: #2-connection-request

The connection request message is used to communicate the DID and Verification key generated for a pairwise relationship from one of the connecting parties to the other. This message, and all others in this exchange, uses a fully encrypted agent message at the wire level.

#### Example
When Bob receives Alice's invitation to connect, he initiates establishing the connection by sending a connection request message. Bob sends Alice a message containing the following:

```
{
  "@type": "did:sov:BzCbsNYhMrjHiqZDTUASHg;spec/connections/1.0/request",
  "DID": "B.did@B:A",
  "label": "Bob"
}
```
#### Attributes
* The `@type` attribute is a required string value that denotes that the received message is a connection request.
* The `DID` indicates the DID of the user requesting the connection.
* The `label` attribute provides a suggested label for the connection. This allows the user to tell multiple connection offers apart. This is not a trusted attribute.

#### Alice Receives the Request
After receiving the connection request, Alice resolves the DID provided by Bob. She then compares the sender key used in the wire level encryption to the key found in Bob's DID Document. The keys must match for the request to be authentic. 

Alice can accept, reject, or ignore the request. If she chooses to accept it, she then stores the DID that Bob provided, along with a cached version of the verification key, and endpoint information in her wallet. 

If Alice chooses to reject the request, she'll discard the associated information.

## 2. Connection Response
[2-connection-response]: #3-connection-response

The connection response message is used to confirm the connection. This message is required in the flow, as it will be needed in the future for Peer DID relationship initialization.

#### Example
Alice sends her connection decision to Bob in the connection response.

```
{
  "@type": "did:sov:BzCbsNYhMrjHiqZDTUASHg;spec/connections/1.0/response",
  "result: "accepted || rejected"
}
```

#### Attributes

* The `@type` attribute is a required string value that denotes that the received message is a connection request.
* The `result` attribute is a required string value and denotes success or failure of the connection request. 

#### Connection Established
The connection between Alice and Bob is now established. This connection has no trust associated with it. The next step should be the exchange of proofs to built trust sufficient for the purpose of the relationship.

# Peer DID Connections

Ongoing work with Peer DIDs will allow for pairwise connections without any record on a public ledger. A Peer DID is a method who's resolution is not based on a public ledger lookup. Instead, agents on both sides of the relationship maintain a record of a Peer DID Document. This allows for key rotations and endpoint updates between peers. Peer DIDs and Peer DID Documents allow for the same functionality available from ledger based DID methods, but without any publicly readable record.

When this work is finished, this connection protocol will need to be expanded to allow the bootstrapping of a peer connection. We anticipate expanding this spec to allow for DIDs in the transaction to be replaced by a key, endpoint, and routing information. The flow of of an optional invitation, a request, and a response will remain the same.

# Reference
[reference]: #reference

* https://docs.google.com/document/d/1mRLPOK4VmU9YYdxHJSxgqBp19gNh3fT7Qk4Q069VPY8/edit#heading=h.7sxkr7hbou5i
* [Agent to Agent Communication Video](https://drive.google.com/file/d/1PHAy8dMefZG9JNg87Zi33SfKkZvUvXvx/view)
* [Agent to Agent Communication Presentation](https://docs.google.com/presentation/d/1H7KKccqYB-2l8iknnSlGt7T_sBPLb9rfTkL-waSCux0/edit#slide=id.p)

# Drawbacks
[drawbacks]: #drawbacks

* DIDs be placed on the public ledger. This will be improved with Peer DID work.
* Public invitations (say, a slide at the end of a presentation) all use the same DID. This is not a problem for public institutions, and only provides a minor increase in correlation over sharing an endpoint, key, and routing information in a way that is observable by multiple parties.

# Prior art
[prior-art]: #prior-art

- This process is similar to other key exchange protocols.

# Unresolved questions
[unresolved]: #unresolved-questions

- This HIPE makes some assumptions about the underlying secure transport protocol in the absence of an official HIPE detailing the specifics of that protocol. In general, this HIPE assumes that message transportation has been solved.