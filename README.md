# signcryption-token-spec
A binary, encrypted, authenticated token format called a *Sign-Cryption Token* aka SCT.


While adding `libsodium-signcryption`, `libsodium-xchacha20-siv` and support for helpers like `sodium_bin2base64` to pgsodium it occurred to me that signcryption could be an excellent basis for a libsodium token format, so I typed up my thoughts for discussion.  If this isn't the appropriate forum for discussion lmk and I'll move it.

# SCT: Libsodium Sign-Cryption Token

A binary, encrypted, authenticated token format called a
*Sign-Cryption Token* aka SCT.

## SCT version 0x0000

 - SCT requires `libsodium`, including the additional function
   libraries in `libsodium-xchacha20-siv` and
   `libsodium-signcryption`.

 - SCTs have an encrypted payload and unencrypted authenticated
   additional data payload. Either payload, but not both may be empty.

 - SCT does NOT support shared-secret encryption. All SCT token
   *producers* MUST have a keypair consisting of public and secret
   keys.

 - Token *consumers* without possession of the recipient secret key
   CANNOT decrypt the *encrypted* payload.

 - Token consumers with the public key of the token producer can
   verify the authenticity of the *unencrypted* portion of a token.

 - SCT gives you no choice of cryptographic algorithm. SCTs MUST use
   `XChaCha20-SIV` combined mode deterministic authenticated
   encryption with additional data using
   `crypto_aead_det_xchacha20_*`.

 - The `shared_key` used for encryption MUST be generated by the
   `crypto_signcrypt_tbsbr_sign_*` API using the sender's secret key
   and the recipient's public key.  The recipient can then recover
   this key using `crypto_signcrypt_tbsbr_verify_*`.

 - SCT does NOT care about encodings such as JSON or any kind of human
   readability. The input payloads, sender/recipient ids, and key
   pairs are binary byte arrays and the output is a base64 character
   array. Encoding payloads to and from formats like JSON to a byte
   representation is a trivial operation and entirely up to the user.
   
 - The output of SCT is a byte string of four encoded base64 sections:
 
   `<version>.<sender_id>.<ciphertext>.<context>.<signature>`
   
   The function `sodium_bin2base64` can be used to encode using the
   `sodium_base64_VARIANT_URLSAFE_NO_PADDING` variant.

## Example function with pgsodium

This is a PostgreSQL function that generates a token:

```sql
CREATE OR REPLACE FUNCTION crypto_signcrypt_token(
    sender       bytea,
    recipient    bytea,
    sender_sk    bytea,
    recipient_pk bytea,
    message      bytea,
    additional   bytea)
RETURNS text AS $$
WITH
    sign_before AS (
        SELECT state, shared_key
        FROM crypto_signcrypt_sign_before(
            sender,
            recipient,
            sender_sk,
            recipient_pk,
            additional)
    ),
    ciphertext AS (
        SELECT crypto_aead_det_encrypt(
            message,
            additional,
            b.shared_key
        ) AS ciphertext
        FROM sign_before b
    ),
    signature AS (
        SELECT crypto_signcrypt_sign_after(
            b.state,
            sender_sk,
            c.ciphertext
        ) AS signature
        FROM
            sign_before b,
            ciphertext c
    )
    SELECT format(
        '0000.%s.%s.%s.%s',
        sodium_bin2base64(sender),
        sodium_bin2base64(c.ciphertext),
        sodium_bin2base64(additional),
        sodium_bin2base64(s.signature))
    FROM
        ciphertext c,
        signature s;
$$ LANGUAGE SQL STRICT;
```

Here's an example of a token generated from keypairs.  First create
keypairs for `bob` and `alice`.  Typically this would be done in
separate edge processes but are shown together here for brevity:

```sql
postgres=# select public as pk, secret as sk from crypto_signcrypt_new_keypair () \gset bob_
postgres=# select public as pk, secret as sk from crypto_signcrypt_new_keypair () \gset alice_
```

Now generate a token from bob to alice.  In this exasmple the `sender`
and `recipient` ids are corresponding public keys, but this is not
necessary, the id can be any unique identifier for a user or group:

```sql
postgres=# select crypto_signcrypt_token(:'bob_pk', :'alice_pk', :'bob_sk', :'alice_pk', 'this is encrypted s3kret message', 'this is unencrpyted additional data');

0000.YvWFDwWg1sBVhwnccoMwDbvw7PzE0SeHvj3g7fDhqBQ.XnPzbou0Rr-NahE3nEGW6EC5QAFvT11iQzAFHu9NjOksdzV61fuftjDfLgU_vZp7IMAfryeoUAGlQCP7h4RM5g.dGhpcyBpcyB1bmVuY3JweXRlZCBhZGRpdGlvbmFsIGRhdGE.qS1slA8qW4J_uKO079VlzKC5BUazG1W67TVuYCqRKgY8CHybwfgho5U_LNGQTZ60nkDxfU4Q9U3o2w2BAwAAAA
```

## Rational and Comparisons

"Perfection is achieved, not when there is nothing more to add, but
when there is nothing left to take away." - Antoine de Saint Exupery

JWT is bad.  There are endless blog posts on the badness of JWT.  One
of the big baddnesses is that the token specifies the encryption
algorithm in its header, and a baddie can influence the decrypting of
the token by choosing the algorithm.  This is called an algorithm
confusion attack.

PASETO is better in many ways, for example by specifying specific
versions, but it also introduced the same flaw that it was meant to
fix with JWT, an attacker can *choose* either `local` or `public`
token "purposes" in an attempt to fool the server with the same kind
of algorithm confusion attacks.

So, PASETO version 3 and 4 put a new section in their spec about
"Algorithm Lucidity" and how every language MUST use whatever type
enforcement features exist to safeguard against using the wrong key
for the wrong purpose.

SCT does not allow algorithm confusion attacks as there is one and
only one algorithm that can cover either or both purposes that `local`
and `public` serve in PASETO.  SCT does NOT support shared secret
encryption, and so there is no possibility of Algorithm Lucidity
attacks being an issue.  All SCT producers MUST have a valid key pair.

PASETO's current two "purposes", `local` and `public` are quite
different, `local` uses shared secret encryption and its payload is
encrypted and authenticated.  `public` does NO ENCRYPTION and just
authenticates its payload data.  These two very different purposes do
not overlap.

SCT has combines both purposes using an algorithm called
*signcryption*.  A token can have encrypted payload, in which case the
the decryptor must have the recipient secret key.  But a token can
also be authenticated by anyone with the sender's public key.  They
can't decrypt the encrypted payload, but the can verify the token is
authentic including the unencrypted additional data.

So now, instead of one token spec with two distinct purposes, and
rules about algorithm lucidity and type checking, etc, SCT has one and
only one token format.  The token can contain an authenticated
encrypted payload, and/or it can authenticated unencrypted additional
data.  If you have the recipient's secret key, you can decrypt it, but
anyone with the senders public key can authenticate it.

One last thing I personally don't like about JWT and PASETO is their
focus on JSON payloads.  SCT payloads are *bytes*.  If you want to
encode your JSON into bytes, go for it, every language in the world
can trivially do that. But JSON is *not required*.
