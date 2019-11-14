# Cash:web - BIP70 Payment Token Protocol

## Abstract

We introduce a general payment protocol which allows services to restrict access to resources until an appropriate payment is provided.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt). 

## Motivation

Monetization of goods over the internet often involves the restriction of specific resources via some form of paywall. As a result, much of web design involves architecting complex authorization schemes involving third party payment processors. 

Bitcoin allows payments across the internet without the need for trusted intermediaries and [BIP 70](https://github.com/bitcoin/bips/blob/master/bip-0070.mediawiki) provides an extra layer of customer assurance during payments.

A standardized protocol for paid authorization to specific resources would provide merchants with a pluggable way to monetize their services.

## Specification

The protocol sits ontop of HTTP and makes use of REST API semantics. It extends the [BIP 70 protocol](https://github.com/bitcoin/bips/blob/master/bip-0070.mediawiki) and, as such, prior knowledge of it is recommended.

It is to be noted that [BIP 70](https://github.com/bitcoin/bips/blob/master/bip-0070.mediawiki) must be suplemented with [BIP71](https://github.com/bitcoin/bips/blob/master/bip-0071.mediawiki) and [BIP72](https://github.com/bitcoin/bips/blob/master/bip-0072.mediawiki), and that both take on a [slightly altered](https://lists.linuxfoundation.org/pipermail/bitcoin-ml/2017-August/000177.html) form for Bitcoin Cash. We refer to this this collection of BIPs as [BIP 70](https://github.com/bitcoin/bips/blob/master/bip-0070.mediawiki) from hereon after.

The protocol's messages are encoded via [Google's Protocol Buffers](https://developers.google.com/protocol-buffers), authenticated using [X.509 certificates](https://tools.ietf.org/html/rfc5280), and communicated over HTTP/HTTPS.

At the most general level the protocol can be decomposed into three distinct round-trips:

1. An initial request for a guarded resource, responded to by a [BIP 70](https://github.com/bitcoin/bips/blob/master/bip-0070.mediawiki) payment request.
2. A POST containing payment, responded to with a [BIP 70](https://github.com/bitcoin/bips/blob/master/bip-0070.mediawiki) payment acknowledgement and a token.
3. A retry of the initial request, this time including the token, responded with the guarded resource.

### 1 - Initial Request

The client requests access to some guarded HTTP resource. The query string of the request MUST NOT include `code` and the `Authorization` header MUST NOT be included. If either of these are present the server MUST assume that the client is attempting authorize itself.

There are no additional restriction placed on the HTTP request.

A successful request MUST be responded to with status code `402` ("Payment Required") and the `PaymentRequest` message defined in [BIP 70](https://github.com/bitcoin/bips/blob/master/bip-0070.mediawiki) and details can be found in the "PaymentDetails/PaymentRequest" section.

```protobuf
message PaymentDetails {
        optional string network = 1 [default = "main"]; // "main" or "test"
        repeated Output outputs = 2;        // Where payment should be sent
        required uint64 time = 3;           // Timestamp; when payment request created
        optional uint64 expires = 4;        // Timestamp; when this request should be considered invalid
        optional string memo = 5;           // Human-readable description of request for the customer
        optional string payment_url = 6;    // URL to send Payment and get PaymentACK
        optional bytes merchant_data = 7;   // Arbitrary data to include in the Payment message
}

message PaymentRequest {
        optional uint32 payment_details_version = 1 [default = 1];
        optional string pki_type = 2 [default = "none"];  // none / x509+sha256 / x509+sha1
        optional bytes pki_data = 3;                      // depends on pki_type
        required bytes serialized_payment_details = 4;    // PaymentDetails
        optional bytes signature = 5;                     // pki-dependent signature
}
```

The server MUST populate `merchant_data` field of the `PaymentDetails` message, the exact details of this data is left for future specification and are inconsequential to the main flow of the protocol. Post-payment, this data will be signed by the server providing a "proof-of-payment" token to the client in order to authenticate metadata uploads. This is highlighted in more detail in subsequent sections.


### 2 - Payment and Token Issuance

The client sends the payment in the body of a POST request to the `payment_url` specified in the `PaymentDetails` and the server responds with a payment acknowledgement message. This is performed in accordance with [BIP 70](https://github.com/bitcoin/bips/blob/master/bip-0070.mediawiki). 

Note that the `merchant_data` in the clients `Payment` message MUST be equal to the `merchant_data` in the servers `PaymentDetails` and that malicious clients may modify the `merchant_data`, so should be authenticated in some way (for example, signed with a merchant-only key). 

```protobuf
message Payment {
        optional bytes merchant_data = 1;  // From PaymentDetails.merchant_data
        repeated bytes transactions = 2;   // Signed transactions that satisfy PaymentDetails.outputs
        repeated Output refund_to = 3;     // Where to send refunds, if a refund is necessary
        optional string memo = 4;          // Human-readable message for the merchant
}
```

In addition to the [BIP 70](https://github.com/bitcoin/bips/blob/master/bip-0070.mediawiki) procedure, the keyserver generates a token string using the following process:

1. Extract the `merchant_data` from the `Payment` message.
2. Perform HMAC SHA256 with a private secret to yield the raw token.
3. Encode the raw token using URL safe base64.

This response MUST have status code `202` and include both:

* A `Location` header `{resource path}?code={token string}`.
* A `Authorization` header `POP {token string}`.

It is RECOMMENDED that the `merchant_data` is chosen such that the `resource path` may be derived from it - avoiding session state.

### 3 - Authorized Request

The client reconstructs the initial request and MUST include either the code provided in the... TODO

It is RECOMMENDED that, if the client aims to PUT or POST a payload to the server, the body SHOULD be included in this final request rather than the initial request to avoid repeated transmission or excess session state.

## Rationale

### BIP 70

The [BIP 70](https://github.com/bitcoin/bips/blob/master/bip-0070.mediawiki) payment protocol is a widely adopted standard. Extending it in this way allows wallet developers to easily integrate the protocol with minimal work. 

### Statelessness

The protocol has been designed with statelessness in mind. By using tokens it requires no extra state in addition to that needed from [BIP 70](https://github.com/bitcoin/bips/blob/master/bip-0070.mediawiki).