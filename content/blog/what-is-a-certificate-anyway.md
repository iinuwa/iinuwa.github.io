+++
title = "What is a Certificate, Anyway?"
date = "2019-07-02"
tags = [["cryptography", "pki"]]
+++


When using public-key cryptography (or _asymmetric cryptography_), you'll often come across certificates. This page explains what certificates are and their purpose in secure communication.

# Introduction to Certificates

Generally speaking, certificates are documents that are used to verify
information with a certain level of trust. For example, if you can produce a
birth certificate dated before 2003 issued by the state of California, the DMV
can trust that you are old enough to operate a motor vehicle. If you can
produce a bachelor's degree issued by an accredited university, a potential
employer can trust that you have studied a certain subject to satisfaction (at
least sat through classes for a certain number of hours). If you can produce a
valid passport issued by the United States, then airport officials can trust
that you are an American citizen. When you get married, you are issued a
marriage certificate by a state to verify your record of marriage. Or, if you
are planning to get married in Louisiana and are buying flowers, if your
florist can produce a horticultural professional license issued by the state of
Louisiana, then you can trust that they can... arrange flowers? (And that the
Louisiana Horticulture Commission will not [shut down their shop][flower-shop] before the
wedding.)

[flower-shop]: https://www.usatoday.com/story/opinion/2018/03/28/louisiana-only-state-requires-occupational-licenses-florists-its-absurd-column/459619002/

Anyway, each of these documents can be trusted because they:
- are issued by a trusted third-party,
- are hard to fake (maintaining integrity) and/or
- can be easily validated by the issuing party (validation).

In other words, a certificate is a means for a third-party (verifier) to
validate the identity of the certificate holder (subject) based on the third
party's trust of the certificate's issuer (authority) within a delegated trust
system.

## PKI and the X.509 Standard

A digital certificate is just like these other documents: a digital certificate
is a file that can be used by third parties to validate the identify of its
subject based on the trust relationship between that third party and the
signator of the certificate. Digital certificates are used in many contexts.
The most common one is in the HTTPS protocol in which clients examine the
digital certificates presented by the issuer, verifying that the subject
matches the hostname that they intended to navigate to, and that the issuer of
the certificate is a reputable certificate authority.

Other uses of digital certificates include smart card for authentication, code
signing certificates for verifying the developer and integrity of an
application, email sender verification, and SSH.

## X.509 Certificates

The vast majority of digital certificates are based on the X.509 standard, and
more specifically the X.509 v3 standard (ratified in [RFC
5280](https://tools.ietf.org/html/rfc5280)). (Some other certificate standards
exist, such as the OpenSSH certificates generated by
ssh-keygen, but this article focuses on X.509 v3 certificates.) The X.509 v3
standard defines a format for certificates to be used in a public key
infrastructure (AKA PKI; defined below) model of trust. Each of these
certificates contain, among other things, the subject name (e.g. me@example.com
or yourdomain.example.org), which identifies the holder of the request and the
subject public key (which will be necessary for use in an asymmetric encryption
cipher).

## PKI

Now you might ask, "Can any subject name be put into a certificate? How can I
know that `https://www.mybank.com` is actually being run by my bank?" Good
question! The subject name of a certificate can be any string, so there needs
to be a way to verify that the certificate presented by a client or server
actually belongs to that subject and not an impersonator. This is where public
key infrastructure, or PKI comes in. PKI is a hierarchical system that
establishes a chain of trust through certificate authorities (CA). A CA is an
organization that validates the identities of certificate holders and issues
valid certificates by signing them with a cryptographic signing algorithm. The
process for gaining a valid certificate is:

1. An individual/organization generates a private key and a certificate signing
   request or CSR, which is basically an unsigned certificate that includes the
   corresponding public key and the subject information (e.g. subject name,
   public key, etc.) intended to be signed.
1. The individual/organization sends the CSR (which includes the public key, but
  not the private key) to a CA to be signed.
1. The CA runs through a validation process to vet the individual/organization
  making the request. This can include a phone call to the certificate owner, a
  domain registration verification, proof of organization, etc.
1. After the subject is validated, the CA takes the subject information, as well
  as a few other details (validity period for the certificate, signature
  algorithm, CA public key, serial number, etc) and cryptographically signs the
  CSR with its private key, creating a valid certificate.
1. After creating the certificate, the CA sends the valid certificate to the
  individual/organization.

At this point, the individual or organization now has a valid certificate
signed by a CA. What good does that do? Just like [faking a death
certificate][faking-death] can allow criminals to receive life insurance
payments that he should not able receive, faking a CAs signature on a
certificate can cause a criminal to gain access to information she should not
be able to receive. However, since the CA signed the certificate with a
cryptographic hashing algorithm, anyone presented with this certificate will be
able to use the CA's public key to verify that the CA signed the certificate
with its private key. If the person trusts that certificate authority, then
they will also trust the certificate.
[faking-death]: https://www.cbsnews.com/news/playing-a-risky-game-people-who-fake-death-for-big-money/

For example, say that Alice has a certificate that has been signed by Charlie,
who operates a certificate authority. Bob trusts certificates signed by
Charlie. If Alice presents Bob with this certificate, and Bob trusts
certificates issued by Charlie, then he will trust that Alice's certificate is
valid.

Sidenote: There is an alternative to the PKI trust model, called [web of
trust](https://en.wikipedia.org/wiki/Web_of_trust), which is used in PGP
systems. It is a decentralized model that relies on public keys being signed by
multiple individuals. As the number of signators increases for a public key,
the greater the level of trust that key  provides. It's a cool peer-to-peer
network that we won't discuss here. 🤓

# Other Information

## Certificate chains

In order for PKI to work, there must be a trust anchor, an entity that is
trusted without reference to any other entity. These certificate authorities
that are trust anchors are called root certificate authorities. Each
application, operating system and device has its own default list of root
certificate authorities included with it (sometimes called a trust store or CA
bundle), and there about 100-200 root CAs that are commonly trusted in most
applications. (Cf. [this StackExchange][question] question for more information.)
[question]: https://security.stackexchange.com/questions/49006/list-of-certificate-authorities-in-browsers-and-mobile-platforms

Root CAs can delegate their trust to intermediate certificate authorities,
which are any CAs that are in between the root CA and the leaf certificate, the
final certificate being validated. This delegation of authority is called a
certificate chain. Certificate chains are helpful because it allows greater
security for root CAs. If a root CA only signs certificates for a few
intermediate authorities, then there is less exposure for that root CA key.
Also, if any of the intermediate certificate authorities are compromised, the
root certificate authority can revoke or invalidate the intermediate CAs
certificates. This will require all certificates signed by that intermediate to
need to be re-signed, but it will not require redistribution of the root CA
public key to client devices.

## Certificate Validation

There are several attributes in a certificate that are used together to
validate a certificate, including:

- Validity period: Generally a CA will only issue certificates that are valid
  for just over 3 years maximum. This date range is stored in the certificate
  and should be checked by any clients presented with the certificate.
- Key Usage: When signing the certificate, CA's can specify what the
  certificate can be used for: e.g., email signing, server certificates, code
  signing, certificate signing (for intermediate CAs), etc.
- Subject Validation: Clients validate the subject of certificates. For
  example, web clients validate that the hostname that they requested is
  included in the Subject or Subject Alternative Name (SAN) fields of the
  certificate; email clients check that the subject field matches the From
  header of the received message.
- CRL/OCSP: CRL (client revocation list) and OCSP (Online Certificate Status
  Protocol) are two ways that CAs can revoke certificates. If the private key
  of certificate signed by a CA is reported to be compromised, or the owner of
  the certificate violates the CA's policies, the CA can update the CRL to
  include this certificate's serial number (blacklisting it from use), or it
  can run an OCSP responder server that responds with a failed OCSP response
  for that certificate's serial number. Whenever possible, clients should use
  OCSP or CRLs to verify whether the presented certificate is still valid.

## Certificate Storage formats

X.509 Certificates can be stored in a variety of formats.

### PEM

The PEM format is a Base64 encoded string of the certificate information. The
certificate data is surrounded by a header (e.g. `-----BEGIN CERTIFICATE-----`
) and a footer (`-----END CERTIFICATE-----`) that denotes the type of
information inside the file. (Note, the 5 hyphens surrounding the header and
footer text necessary to successfully parse a PEM file.) It is the most common
format that certificates are delivered in. Other information can be stored in
PEM files, like private keys (`BEGIN`/`END PRIVATE KEY`, `BEGIN`/`END RSA PRIVATE KEY`)
and CSRs (`BEGIN`/`END CERTFICATE REQUEST`).

A common file extension for this file type is .pem, but some applications use .crt, or .cer.

PEM files can be concatenated to form a certificate chain. To do that, add a
new line to the PEM file and place the text of the next certificate in the
chain below it. The order should go from leaf, to intermediate CA certificates
(if any), to the root CA certificate (if any). For example:

```
-----BEGIN CERTIFICATE-----
<leaf certificate text>
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
<intermediate certificate text>
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
<root certificate text>
-----END CERTIFICATE-----
```

Besides certificate chains, multiple PEM certificates concatenated into a
single file can also represent a CA bundle.

### DER

DER files store the same information as PEM files, but in a binary format
rather than ASCII. The most common file extension for these types of files is
.der, but .crt and .cer are often used as well. DER files cannot be
concatenated like PEM files can, so if you need to use multiple certificates
together, you should convert the certificates to a PEM or place them in a
container (like PKCS12).

### PKCS12

A PKC12 file is a certificate store that can hold multiple public and private
keys. It is most commonly use to store a public certificate and private key
pair along with its CA certicates to provide a full chain of trust. The common
file extensions for this format are .p12 and .pfx.


# More Information

- [What's in a Digital Certificate (video)](https://www.youtube.com/watch?v=XmIlynkR8J8)
- [X.509 (Wikipedia article)](https://en.wikipedia.org/wiki/X.509)
- [What is PKI? (video)](https://www.youtube.com/watch?v=i-rtxrEz_E8)
- ["How to be a Certificate Authority with Ryan Sleevi" episode of _Security, Cryptography, Whatever_ podcast](https://securitycryptographywhatever.com/2021/09/06/how-to-be-a-certificate-authority-with-ryan-sleevi/)
