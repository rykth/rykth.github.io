---
title: "Demystifying (custom) certificates"
date: 2025-09-08T18:07:58+05:30
description: "Custom certificates are like ID cards issued by your own company instead of a public authority. They're mainly used inside private systems, such as microservices, where services only need to trust each other. A central service acts as the certificate authority (CA), creates these certificates, and gives them to each service. When combined with TLS, they let services prove their identity and communicate securely, without relying on external certificate providers."
tags: [security, tls, certificates]
---

Imagine you have a neighborhood full of small shops (these are like your
microservices). Each shop has a different role - one sells bread, another sells
fruits, another sells clothes. To work together, they need to trust each other
when they exchange goods.

Now, if a fake shop shows up pretending to be "the bread shop", the others could
get tricked. To prevent this, every real shop carries a certificate - think of
it as an official ID card issued by a trusted authority. This certificate
proves:
- __Who they are__ (the shop's name and details).
- __That they were verified__ by someone trusted (like a government office or a certificate authority).
- __That their ID hasn't expired__.

In the digital world, certificates do the exact same thing. They are digital ID
cards for computers and services. When one service talks to another, it shows
its certificate to prove it's genuine.

---

Now, imagine these shops not only need ID cards, but they also want to make sure
nobody eavesdrops when they're negotiating prices or sharing secret recipes. For
this, they use a __secure tunnel__ - like whispering inside a soundproof booth. In
technology, this is called TLS (Transport Layer Security).

TLS makes sure that:
1. __No one can listen__ on the messages (they're encrypted).
2. __The messages aren't tampered__ with while traveling.
3. __The shop is really who it says it is__ (thanks to certificates).

In microservices, TLS and certificates are used together. Whenever one service
calls another (say, the "orders service" talks to the "payment service"), TLS
ensures the communication is private and safe. Certificates make sure both sides
can trust each other and aren't talking to an impostor.

---

In the earlier example, each shop in the neighborhood (microservice) got its
official ID card (certificate) from a trusted government office (a public
certificate authority, like Let's Encrypt). That's great for the outside world,
but sometimes the shops are only dealing within their own neighborhood - no
outsiders involved.

In this case, instead of going to the government for IDs, the neighborhood
decides to make their own ID system. One shop becomes the "chief" and acts as
the certificate authority (CA). This CA creates and hands out custom
certificates (ID cards) to all the other shops.

---

Why would they do this?
Because it's faster, cheaper, and entirely under their control. For example:
- The orders service only trusts certificates issued by its own company's CA.
- The payments service also has a certificate from the same CA.
- When they talk, they check each other's IDs, and since both are signed by
their CA, they immediately trust each other.

This is called using custom (or private) certificates. It's like a private club
where only members with a special badge can enter.

---

Now, combine this with TLS (the secure tunnel). Not only are the services
proving who they are with custom certificates, but all their conversations are
encrypted and protected from outsiders. Even if someone tries to spy on the
messages, they'll only see nonsense because of encryption.

## CURVE25519

Think of an elliptic curve as a special playground with "points". There's a
rule that lets you add points together. If you add the same point to itself
again and again, that's called scalar multiplication-like turning a crank N
times.
- Your secret key = how many crank turns you do (a big random number).
- Your public key = where you land on the playground after those turns (a point).
- The hard problem: Given the start point and the landing point, figuring out
how many turns you used is believed to be infeasible. That's what gives security.

---

A merry-go-round (sometimes called a carousel) is a ride you often see at
playgrounds or amusement parks. It's a big circular platform with seats (or bars
to hold onto). Kids can sit or stand on it, and then someone pushes it so it
spins around in a circle.

---

At its core, Curve25519 is a way to turn a secret number into a public point in
such a way that going forward is easy, but going backward is practically
impossible. To imagine this, picture a huge merry-go-round with billions upon
billions of seats arranged in a circle. Everyone agrees on a starting seat, and
that seat is the same for everyone.

Now, suppose you secretly choose a number - let's say 13. Starting from the
agreed-upon seat, you spin the merry-go-round forward 13 steps.  Where you land
is your public result. You don't reveal the number 13 itself; you only tell
people which seat you ended up on. In this picture, your private key is the
secret number of steps you took, and your public key is the final seat number
you share.

The beauty of this system is that it's very easy to move forward - just spin the
merry-go-round the required number of steps. But if someone only sees your
ending seat, it's almost impossible for them to work backward and figure out
exactly how many spins you used. This is because the merry-go-round isn't small;
it's unimaginably large (based on the huge prime number 2²⁵⁵ - 19 ), so
brute-forcing the number of spins is completely infeasible.

When two people, say Alice and Bob, want to agree on a shared secret, they each
choose their own private number of spins. Alice spins her number of times and
tells Bob which seat she landed on. Bob does the same and tells Alice his seat.
Then comes the magic: Alice takes Bob's seat and spins her secret number of
times again. Bob takes Alice's seat and spins his secret number again. Despite
using different secrets, they both land on the same final seat. That final seat
is their shared secret, which they can use for encryption. An eavesdropper only
sees the seats Alice and Bob announced publicly, but without knowing their
secret spin counts, it's impossible to compute the shared final seat.

This is the basic idea behind Curve25519: a giant mathematical merry-go-round
that lets people exchange information safely. The design makes it fast,
resistant to common mistakes, and extremely hard to attack, which is why it's
become one of the most widely used cryptographic tools today.

### What "25519" means?

It's the prime number used for the math: p = 2²⁵⁵ − 19.    
Working "mod p" (wrap-around arithmetic) makes the math fast and tidy for
computers, and the shape of the curve is tuned for speed and safety.

## 	How key exchange (X25519) works-step by step
	
Goal: Alice and Bob create a shared secret over the internet where anyone can
listen, but no one else can compute that secret.
1. Agree on a public starting point (the "base point", fixed for everyone).
2. Alice picks a long random secret number (her private key), "turns the crank",
and publishes the result (her public key).
3. Bob does the same and publishes his public key.
4. To get the shared secret:
	- Alice takes Bob's public key and turns the crank her secret number of times.
	- Bob takes Alice's public key and turns the crank his secret number of times.
5. Both land on the same point (that's math magic), which becomes their shared
secret for encryption.

## Certificate authorities

When you visit a website, you want to be sure you are really connecting to the
correct place and not to a fake site created by hackers. The problem is that, on
the internet, anyone can set up a website and claim to be your bank, an online
shop, or even your email provider. Without a way to prove identity, it would be
impossible to know which sites are safe to trust.

This is where Certificate Authorities, or CAs, come in. A CA is like a trusted
notary or passport office for the internet. Just as a government can issue
official documents to prove who you are, a CA can issue a digital certificate to
a website. This certificate works as proof that the website is genuine and
really belongs to the organization it claims to represent.

The process works like this: the owner of a website first requests a certificate
from a CA. The CA then checks whether the requester truly controls the website's
domain name, and in some cases, also verifies the organization behind it. Once
the CA is satisfied, it issues a digital certificate containing the website's
public key and a special signature from the CA itself.

When you visit a website, your browser looks at its certificate. If the
certificate is signed by a CA that the browser already trusts, the browser
accepts it as genuine. This is why web browsers come with a built-in list of
trusted CAs, similar to how you might trust certain government agencies to issue
passports. If the certificate is valid, your browser will display the familiar
padlock symbol 🔒 in the address bar, signaling that the site's identity has
been verified and the connection is secure.

Certificate Authorities are essential because they make secure connections,
known as HTTPS, possible. Without them, there would be no reliable way to tell
safe websites apart from malicious imposters. In short, CAs act as trusted
organizations that vouch for the identity of websites, allowing people to
browse, shop, and bank online with confidence.

## ASN.1

When you connect to a secure website, your browser checks a digital certificate
to make sure the site is genuine. That certificate contains details such as the
website's name, who issued the certificate, when it expires, and the site's
public key. All of this information needs to be stored and transmitted in a very
precise way so that any computer in the world can read it without confusion.
This is where ASN.1 and DER encoding come into play.

ASN.1, which stands for Abstract Syntax Notation One, is a standard way of
describing and packaging data so that different computer systems can understand
each other. At its core, it is simply a rulebook that defines how information
should be structured and encoded when it is shared between devices, programs, or
organizations.

The main problem ASN.1 solves is that computers often represent information
differently. One system might store a number in a certain way, while another
might represent text differently. Without a common standard, two systems
exchanging data could easily misinterpret what the other is sending. ASN.1
provides that shared language, making sure that data like names, numbers, and
keys are described in a consistent and universally understood format.

ASN.1 works by separating two ideas: the description of the data, and the way it
is encoded into bytes. The description, or abstract syntax, defines what kind of
information is being represented - for example, whether it is a number, a date,
or a piece of text. The encoding rules then specify exactly how that information
should be written down in digital form. There are several encoding rules, such
as BER (Basic Encoding Rules) and DER (Distinguished Encoding Rules), which are
simply different ways of turning the same abstract data into bytes.
	
ASN.1 is the rulebook that defines how the pieces of information inside a
certificate are described. For example, it specifies that there should be a
field for the certificate's version, a field for the serial number, a field for
the validity dates, and so on. Think of it as the blueprint or outline that says
what data must exist and in what order.

## DER
	
Once the blueprint is defined, the information still has to be written down in a
concrete digital form. This is done using DER encoding, which stands for
Distinguished Encoding Rules. DER is a strict and unambiguous way of turning the
ASN.1 blueprint into raw bytes. It makes sure that no matter which computer or
software generates the certificate, the final binary format will always look the
same. That consistency is critical, especially for security, because even a tiny
difference in encoding could make a certificate invalid.
	
In practice, DER encoding means that every field inside the certificate is
tagged with its type (such as number, string, or date), its length, and its
value. For example, the "Issuer" field might say who issued the certificate, and
DER encoding ensures that this is written in a standard way so that your browser
can reliably read it. The same applies to the subject name, the public key, and
other technical details.
	
In short, ASN.1 provides the structure for certificates, and DER provides the
precise digital handwriting that guarantees every certificate looks the same to
all computers. Without this system, browsers and servers would struggle to agree
on how to read certificate details, and secure connections across the internet
would not be possible.

DER encoding is binary, so when you open a certificate file (like cert.der),
you'll see gibberish. For humans, we often show it in hexadecimal.

Let's generate one to see an example of the DER encoding:

1. First create an elliptic private key
```bash
openssl genpkey -algorithm ED25519 > ed25519.key
```

2. Next create an elliptic public key
```bash
openssl pkey -in ed25519.key -pubout -out ed25519.pub
```

3. Next create a configuration file for the certificate [cert.config](https://github.com/rickKoch/certs/blob/main/example_certs/cert.config)

4. Now generate the certificate
```bash
openssl req -config cert.config -new -x509 -days 3652 -key ed25519.key -out ed25519.crt
```
	
5. Print the binary representation of the certificate. [cert file](https://github.com/rickKoch/certs/blob/main/example_certs/ed25519.crt)
```bash
echo MIIB9TCCAaegAwIBAgIUbpT... | base64 -D | xxd
```
	
Looks scary, right? Let's break it down. 
	
DER uses a simple rule: Tag – Length – Value (TLV).
	
- Tag: what type of data this is (e.g., sequence, integer, string).
- Length: how many bytes of data follow.
- Value: the actual data.
	
Take the first few bytes
```
30 82 01 f5
```
- 30 means SEQUENCE (a group of items)
- 82 means the length is stored in the next 2 bytes
- 01 f5 = 0x01f5 = 501 bytes long
- So this is the entire certificate structure.
	
```
30 82 01 a7
```
- another SEQUENCE
- This usually contains the "tbsCertificate" section (the main body of the certificate).
	
```
A0 03 02 01 02
```
- A0 = [0] EXPLICIT (a context-specific tag).
- Inside it: 02 01 02.
- That means INTEGER, length 1, value 02.
- This is the version field → Version = 2 (X.509 v3 certificate).
	
```
02 14 6E 94 ...
```
- 02 = INTEGER.
- 14 = 20 bytes.
- 14 6E = the serial number (unique ID for the certificate).

What looks like random hex is actually a very strict recipe:
- 30 ... → "Here starts the whole certificate".
	
All of this is wrapped in DER encoding so that any browser or server can parse
it without ambiguity.

## Simple Implementation of Custom CA

Let's start with a simple implementation in golang:
```go
pub, rawPriv, err := ed25519.GenerateKey(rand.Reader)
handleError(err, "failed to generate Ed25519 key pair")
```
[func GenerateKey(rand io.Reader) (PublicKey, PrivateKey, error)](https://pkg.go.dev/crypto/ed25519#GenerateKey) creates a random private key and its
corresponding public key. Ed25519 is a modern, high-performance elliptic curve
signature scheme based on Curve25519. The private key is used for digital
signatures; the public key is embedded in the certificate.   
[rand.Reader](https://pkg.go.dev/crypto/rand#Reader) provides cryptographically secure randomness for key generation.

```go
// The certInfo struct is a custom data structure designed specifically
// for our custom certificates. It organizes all certificate metadata (name,
// CA flag, issuer, etc) in a way that is efficient for microservices. 
//
// Why use certInfo?
// It provides a clear, type-safe way to encode and decode certificate
// fields, supports forward compatibility (easy to add new fields), and
// enables deterministic ASN.1 DER encoding for cryptographic signatures.
type certInfo struct {
	name      string
	notBefore time.Time
	notAfter  time.Time
	isCA      bool
	publicKey []byte
	issuer    string
}

// Marshal serializes the certInfo struct into ASN.1 DER encoding for
// certificate details.
// Each field is encoded with a context-specific tag for forward compatibility
// and type safety.
func (ci *certInfo) Marshal() ([]byte, error) {
	// cryptobyte is a Go package for safe, efficient ASN.1 and TLS binary
	// encoding/decoding.
	// cryptobyte.Builder is a helper type that incrementally builds binary
	// data (like ASN.1 DER)
	// by appending fields, tags, and values in a structured way. It ensures
	// correct encoding,
	// proper tag/length/value formatting, and prevents buffer overflows or
	// malformed output.
	// https://pkg.go.dev/golang.org/x/crypto/cryptobyte#Builder
	var b cryptobyte.Builder
	var err error

	// Begin encoding the details structure with the context-specific
	// constructed tag (TagCertInfo).
	// TagCertInfo is a context-specific constructed ASN.1 tag ([0], 0xA0)
	// used to wrap all certificate details.
	// This ensures the info is encoded as a single structured field, making
	// it easy to identify, parse, and verify during unmarshaling and signature
	// validation. Context-specific tags provide forward compatibility, allowing
	// new fields to be added without breaking older parsers. The constructed
	// flag indicates that this field contains nested elements (name, isCA,
	// notBefore, etc.), matching ASN.1 best practices for extensible structures.
	b.AddASN1(TagCertInfo, func(b *cryptobyte.Builder) {
		// Encode the certificate name as context-specific string field.
		b.AddASN1(TagInfoName, func(b *cryptobyte.Builder) {
			b.AddBytes([]byte(ci.name))
		})

		// Encode the IsCA flag only if true (saves space for non-CA certs).
		if ci.isCA {
			b.AddASN1(TagInfoIsCA, func(b *cryptobyte.Builder) {
				b.AddUint8(0xff)
			})
		}
		// Encode the notBefore timestamp as a context-specific integer field.
		b.AddASN1Int64WithTag(ci.notBefore.Unix(), TagInfoNotBefore)
		// Encode the notAfter timestamp as a context-specific integer field.
		b.AddASN1Int64WithTag(ci.notAfter.Unix(), TagInfoNotAfter)

		// Encode the issuer fingerprint if present (hex string to bytes).
		if ci.issuer != "" {
			issuerBytes, innerErr := hex.DecodeString(ci.issuer)
			if innerErr != nil {
				err = fmt.Errorf("failed to decode issuer: %w", innerErr)
				return
			}
			b.AddASN1(TagInfoIssuer, func(b *cryptobyte.Builder) {
				b.AddBytes(issuerBytes)
			})
		}
	})

	if err != nil {
		return nil, err
	}

	return b.Bytes()
}

// Certificate represents a custom certificate with its associated data
type Certificate struct {
	info      *certInfo
	rawInfo   []byte
	publicKey []byte
	signature []byte
}

func (c *Certificate) Sign(signer *Certificate, pk ed25519.PrivateKey) error {
	if signer != nil {
		if c.info.isCA {
			return ErrCannotSignWithCA
		}

		issuer, err := signer.Fingerprint()
		if err != nil {
			return ErrCannotGetIssuer
		}

		c.info.issuer = issuer
	}

	var err error
	if c.rawInfo, err = c.info.Marshal(); err != nil {
		return fmt.Errorf("failed to marshal certificate info: %w", err)
	}

	// Create signature over rawInfo and publicKey
	h := sha256.New()
	h.Write(c.rawInfo)
	h.Write(c.publicKey)
	c.signature = ed25519.Sign(pk, h.Sum(nil))

	return nil
}
```

Full example with generating CAs and signing certificates can be found in the
example [github repository](https://github.com/rickKoch/certs)