# Bamboo 2

**status: draft without many explanations - far from superseding the original bamboo spec, although that is the eventual goal**

## Preliminary Definitions (slightly sloppy at the moment)

A *authorship scheme* consists of some digital signature schemes together with a way of uniquely encoding/decoding author is and signatures. If `A` is a authorship scheme and `author` is a author id, we write `A::encode_author(author)` to refer to the unique sequence of bytes that encodes the author id. If `sig` is a signature, we write `A::encode_sig(sig)` to refer to the unique sequence of bytes that encodes the signature. TODO define signing function from byte-strings to signature, define verify function from signature, bytes and author id to yes/no.

Examples of different authorship schemes:

- [ed25519](https://ed25519.cr.yp.to/), author id is the public key, public keys and signatures encoded as fixed-width integers
- ed25519, author is a pair of a public key and a 64-bit integer, encoding them as the actual key followed by the integer as a VarU64 (this provides exactly the author + log id functionality of bamboo 1.0)
- some sort of multi-format

A *hashing scheme* consists of some hash functions together with a way of uniquely encoding/decoding hashes. If `H` is a hashing scheme and `h` is a hash, we write `H::encode_hash(h)` to refer to the unique sequence of bytes that encodes the hash.

- blake3, hashes encoded as fixed-width integers
- some sort of non-cryptographically secure hash function
- a multi-format (yamf-hash provides exactly the same functionality as bamboo 1.0)

## Bamboo<A, H, SignatureDensity, Aggregates>

Bamboo is a spec parameterized over four different arguments, any set of such four arguments defines a specification of a log format. The first argument is a authorship scheme, the second a hashing scheme, the third is either `Dense` or `Sparse`, and the last argument is a natural number (0, 1, 2, ...).

Given a choice of arguments `A`, `H`, `SignatureDensity` and `Aggregates`, a log entry conceptually consists of the following values:

- the *author* of the entry, a author id of `A`
- the *sequence number* of the entry
- the *predecessor_link*, the hash of an entry with a sequence number that is one smaller than this entry's sequence number
- the *skip_link*, the hash of an entry with a sequence number that is the [obal](Todo) of this entry's sequence number
- the *payload_hash*
- `Aggregates` many pairs of 64-bit integers, such that for each pair the second entry is equal to the sum of all first entries (of that pair, not of any other pairs) of the entries from the skip_link up to and including the entry itself

The aggregates allow storing integral metadata that accumulates as a log grows, replication protocols can use this to set limits on the number of entries to replicate based on target values for the aggregates. Some example values:

- the size of the payload of the entry (allows to request data up until a total size limit has been reached)
- the time in milliseconds that has passed since creating the previous entry (allows to request data from within a timeframe)
- 1 if the entry is supposed to be the last entry of the log, 0 otherwise (analysis to request data up until the first message that terminates the log)
- 1 if the entry is an "about" message, 0 otherwise (allow to check that the correct number of "about" messages has been transferred by the peer)

We need to specify how to compute the hash of an entry. That is done by encoding the entry as a byte string in the following way and then computing a hash over that byte string:

Begin with `A::encode_author(author)`, then append the sequence number encoded as a canonical VarU64, then append `H::encode_hash(skip_link)`, then append `H::encode_hash(predecessor_link)`, then append the aggregates (each pair is encoded as two canonical VarU64s), then append the payload_hash. Finally, if the SignatureDensity is `Dense`, append `A::encode_sig(signature)`, where `signature` is the signature over all that previous data. If the SignatureDensity is `Sparse`, then the signature is only appended if the sequence number is equal to `(3^k) - 1` for some natural number `k`.

TODO: Omit the skip link and the second entries of the aggregates for entries for which predecessor and skip link target coincide. Special-case entries of sequence number 1 analogically to bamboo 1.0.

## Magma<S, H, SignatureDensity, Aggregates, f>

Magma<S, H, SignatureDensity, Aggregates> is almost identical to bamboo<S, H, SignatureDensity, Aggregates>, but there is also a function `f` that maps two payloads to a new payload (actual payloads, not hashes). Each entry with payload `n` and whose target of the skip link has payload `s` also contains a *skip_payload_hash*, which is the hash of `f(f(...f(s, s + 1), ...), n)` (where `s + 1` would be the payload of the next entry, and so on). In the encoding the skip_payload_hash is appended after the payload_hash. 
