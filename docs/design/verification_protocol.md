---
layout: default
---
# Public Health Authority Diagnosis Verification Protocol

This design covers the exposure notification key server's ability to verify
diagnosis certifications from public health authorities.

The actual process of issuing these certificates is not covered in this
document, but will be at a later date. A simple example of certificate signing
is available in [tools/example-verification-signing](https://github.com/google/exposure-notifications-server/tree/main/tools/example-verification-signing)

## Motivation

The current exposure notifications publication protocol only has the ability
to authenticate that a request came from a mobile device via the vendor
specific device attestation protocols. This doesn't provide a way
to certify that the diagnosis came from a trusted (by the server) public
health authority in the jurisdiction.

## Definitions

* PHA : Public Health Authority
* JWT : JSON Web Token
* ECDSA : Elliptic Curve Digital Signature Algorithm
* PIN : personal identification number. Could be a different authenticator.
* HMAC : hash-based message authentication code
* TEK : Temporary Exposure Key

## High Level Flow

1. App user is diagnosed with Covid-19. PHA issues a PIN code to the user.
2. The user enters the PIN code on their App
3. The App exchange the PIN for a claim check token and walks the user through
   some health related questions (i.e. symptom onset date).
4. The app creates a HMAC value that combines the TEKs from the device along
   with calculated transmission risk value, rolling period start and rolling
	 period count values (the key data + metadata).
	 These values are sent to the PHA server along with the token from step 3.
5. If the token is valid, the PHA issues a JWT that is signed using ECDSA over
   the P-256 elliptic curve with SHA-256 as a hash function. The JWT includes
   additional claims about the data (see below).
6. The app on the user's device sends this signed JWT to the exposure
   notifications server (this project).
7. If the JWT is valid and signed by a trusted PHA (verified because the public
	 key has been previously shared with the server), then the keys are imported
	 into the server for distribution to other devices in the geography. Because
   we are using pre-shared public keys, this verification can be done with
   offline verification.

## JWT Verification + Accepted Claims

The exposure notification key server will accept this JWT in the current
`verificationPayload` field in the exposure reporting API. When this is present,
device attestations should be disabled. In addition, the exposure publishing
API will also accept a new field `hmackey` to be used as the key for
re-calculating the HMAC that was signed by the PHA verification server. The
`hmackey` field must be base64 encoded.

The JWT itself will be unpacked and the following validations will be performed.
First, using the standard claims.

* `iss` : The issuer will be used to determine which public key(s) are valid for
verification. This is to allow for key rotation.
* `aud` : The audience must be as configured for this installation of the
exposure notification key server. The operator of the exposure notification key server
is the one to define this value and should be shared to all participating health
authorities.
* `iat` : The unix timestamp at which the token was issued.
* `exp` : The unix timestamp at which the token will expire.
* `nbf` : If present, the "not before" timestamp will be honored.

We also prescribe a set of private claims to transmit data from the PHA
verification server to the exposure notification key server.

* `tekmac` : _REQUIRED_ The HMAC of the TEKs that was presented to the PHA verification
server. This must be calculated in a specific way (see below). This is REQUIRED.
Base64 encoded string property.
* `reportType` : _REQUIRED_ One of 'confirmed', 'likely', or 'negative' to indicate the
  diagnosis report type that the verification server is attesting to. At the key
  server, if no transmission risk values are set, the report type is used to
  assign transmission risk for compatibility with older apps/clients.
  This is the __only__ way set reportType on TEKs is through a verification certificate.
* `symptomOnsetInterval` : _OPTIONAL_ uses the same 10 minute interval timing as TEKs use. If an interval is provided that is not the start of a UTC day, then it will be rounded down to the beginning of that UTC day. And from there the days +/- symptom onset will be calculated. Int property.

Standard JWT headers must be provided.

* `alg` : _REQUIRED_ and must be set to `ES256`
* `kid` : _REQUIRED_ and indicate a specific key ID to use for verification
* `typ` : _REQUIRED_ and must be set to `JWT`

### HMAC Calculation

In order to calculate the HMAC, the application needs to combine all the
relevant data from the publish request.

* The key should be at least 128 bits of random data generated on the device.
* The hash function used must be SHA 256

To calculate the HMAC, the device must create the following cleartext string:
`key[,key]`

Each `key` segment contains the following data, period separated: `base64(TEK)`,
`rolling period start`, `rolling period count`, `transmission risk`.

The `key` segments are sorted lexicographically based on the base64 encoding.

As an example:

```
key1.rp1.rpc1.tr1,key2.rp2.rpc2.tr2
```

For newer clients that are not assigning transmission risk values, the 4th
segment of the per-TEK segment can be omitted. If this is done, then a value
of 0 (zero) must be passed on the publish request at the key server, or the
transmission risk must be omitted entirely. That would make the clear text
portion look like:

```
key1.rp1.rpc1,key2.rp2.rpc2
```
