# Bamboo 🎍

A cryptographically secure, distributed, single-writer append-only log that supports transitive partial replication and local deletion of data.

Powered by [optimal anti-monotone binary graphs](https://pdfs.semanticscholar.org/76cc/ae87b47d7f11a4c2ae76510dde205a635cd0.pdf), this log format can serve as a more efficient alternative to [secure-scuttlebutt](https://www.scuttlebutt.nz/) while providing stronger guarantees regarding causal ordering of entries than [dat's hypercore](https://github.com/mafintosh/hypercore).

## Concepts and Properties

Each append-only *log* is identified by a public key of a cryptographic signature scheme. Conceptually, an *entry* in the log is a tuple of:

- the *sequence number* of the entry (i.e. the offset in the log)
- the *backlink*, a cryptographically secure hash of the previous entry in the log
- the *lipmaalink*, a cryptographically secure hash of some older entry in the log, chosen such that there are short paths between any pair of entries
- the hash of the actual *payload* of the entry
- the *size* of the payload in bytes
- a digital signature of all the previous data, issued with the log's public key
- a boolean that indicates whether this is a regular entry or an *end-of-log* marker

Since all entries are signed, only the holder of the log's private key (called the *author*) can create new entries. Thus logs can be replicated via potentially untrusted peers - any attempt to alter data or create new entries can be detected. The author however could *fork* a log by giving different entries of the same sequence number to different peers, resulting in a directed tree or even a directed acyclic graph rather than a log (aka a path in graph-theory parlance). This is where the backlinks come in: By iteratively traversing backlinks, any reader of the log can verify that no fork occured. Forked logs are considered invalid from the point of the earliest fork, so there is no incentive for the author to deliberately fork their log. Additionally, backlinks establish a causal order on the existence of entries: Each entry guarantees that all previous entries have already existed prior in time, else their hash could not have been (transitively) included.

As a result of these properties, replication of a full log between two peers becomes very simple: The peers compare the newest sequence numbers they are aware of, and the peer with newer entries sends them and the corresponding payloads to the other peer. The receiver verifies the integrity by checking signature, sequence numbers, payload sizes, backlink and lipmaalink, and then stores the entries in their local copy of the log. In this mode of replication, all peers are equally capable, it does not matter where the data originated. The worst a malicious peer could do is to deliberately withhold a suffix of the log.

This simple replication of full logs is inspired by [secure scuttlebutt](https://www.scuttlebutt.nz/) (ssb), and it only requires each entry to include a backlink to the previous entry. The core distinction between ssb and bamboo lies in the ability to partially replicate logs. By including the additional lipmaalink, bamboo gains two crucial properties: First, the shortest path between two entries has length logarithmic in the absolute difference of their sequence numbers. So to verify the created-before relation between two entries, only a small number of additional entries is needed.

While this property alone allows a peer to efficiently validate parts of a log, it does not automatically lead to transitive replication. Suppose a peer only has two entries `x` and `y`, as well as all entries needed to show that there is a path of links from `y` to `x` (we call those entries a *certificate for `x` and `y`*). Now if another peer has entry `z` and wants to get entry `y`, the first peer may not have the entries necessary to show that there's a path of backlinks from `z` to `y`, and thus they might not be able to replicate, even though entry `y` itself is available. Bamboo solves this problem by assigning a (logarithmically sized) set of entries `cert_pool(x)` (the *certificate pool of `x`*) to each entry `x`, such that for all entries `x` and `y` the union of `cert_pool(x)` and `cert_pool(y)` contains a path of links from `y` to `x` (or the other way around). This allows fully transitive partial replication where peers only need to store logarithmically more entries than they are directly interested in.

## Encoding

Since signatures and hashes are computed over concrete bytes rather than abstract descriptions, bamboo defines a precise binary encoding for the log entries. It uses multiformats so that new cryptographic primitives can be introduced without having to change the specification. The encoding is defined as the concatenation of the following byte strings:

- `tag`, either a zero byte (`0x00`) to indicate a regular log entry, or a one byte (`0x01`) to indicate an end-of-log marker
- `payload_hash`, the hash of the payload, encoded as a canonical, binary [yamf-hash](https://github.com/AljoschaMeyer/yamf-hash).
- `size`, the size of the payload, encoded as a canonical [VarU64](https://github.com/AljoschaMeyer/varu64).
- `seqnum`, the sequence number of the entry, encoded as a canonical [VarU64](https://github.com/AljoschaMeyer/varu64). Note that this limits the maximum size of logs to 2^64 - 1.
- `backlink`, the hash of the previous log entry, encoded as a canonical, binary [yamf-hash](https://github.com/AljoschaMeyer/yamf-hash). This is omitted if the `seqnum` is one.
- `lipmaalink`, the hash of an older log entry, encoded as a canonical, binary [yamf-hashe](https://github.com/AljoschaMeyer/yamf-hash). For details on which entry the lipmaalink has to point to, see the next section. This is omitted if the `seqnum` is one.
- `sig`, the signature obtained from signing the previous data with the author's private key, encoded as a canonical [VarU64](https://github.com/AljoschaMeyer/varu64) followed by that many bytes (the raw signature).

Note that peers don't necessarily have to adhere to this encoding when persisting or exchanging data. In some cases, tag, seqnum, backlinks and payload_hash can be reconstructed without them having to be transmitted. This encoding is only binding for signature verification and hash computation, nothing more.

The authenticity of an entry can be verified by checking whether the signature is correct. Further validity checks on sequence numbers and backlinks are necessary to guarantee absence of forks and verify the claimed entry creation order, as described in the next section.

## Links and Entry Verification

The lipmaalinks are chosen such that for any pair of entries there is a path from the newer to the older one of a length logarithmic in their distance. The lipmaalink target of the entry of sequence number `n` is computed through the function `f`, defined below:

```yay,math!
f(n) := if (n == (((3^k) - 1) / 2) for some natural number k) then {
  return n - (3^k);
} else {
  return n - (((3^g(n)) - 1) / 2);
}

g(n) := if (n == (((3^k) - 1) / 2) for some natural number k) then {
  return k;
} else {
  return g(n - (((3^(k - 1)) - 1) / 2));
}
```

Sorry for the math, but on the plus side, it works! This computes the edges according to the scheme presented in [Buldas, A., & Laud, P. (1998, December). New linking schemes for digital time-stamping](https://pdfs.semanticscholar.org/76cc/ae87b47d7f11a4c2ae76510dde205a635cd0.pdf). For a (slightly) more enjoyable overview of the theory behind this, I'd recommend [Helger Lipmaa's Thesis](https://kodu.ut.ee/~lipmaa/papers/thesis/thesis.pdf).

Whether the created-before relation claimed by the sequence numbers `x` and `y` of two entries is indeed valid can be verified by finding a path along the links from the (claimed) newer entry to the older one. By verifying the created-before relation over all known entries of a log, a peer can check that the entries form indeed a single log (rather than disparate logs, trees or dags).

An entry is considered *verified* if and only if:

- authenticity has been checked via the signature
- the entry has sequence number one OR there is a link path from the entry to the first entry, where all but the newest entry (which is the one to be verified) are already verified
- the entry has sequence number one OR the backlink and the lipmaalink point to the entry of the expected sequence number

Additionally, if the payload of an entry is available to a peer, the peer must check wether the size of the payload in bytes matches the claimed (and signed) size metadata. If it doesn't, the feed must be considered invalid from that entry onwards (there's zero tolerance for authors lying about payload sizes).

## Partial Replication and Log Verification

When partially replicating a log, a peer is only interested in a subset of the log's entries. Verifying all elements of a subset of a log independently does not yet verify the subset as a whole. For a subset to be considered verified, there must also be a backlink path from the newest entry to the first entry that contains all other entries of the subset. Otherwise, the created-before relation claimed by the entry's sequence numbers isn't verified.

To traverse that path, the peer may have to store other entries it isn't really interested in. It doesn't need to fully verify those, but it should still check authenticity and follow backlinks if their target is available, to check for forks and invalid sequence numbers. In particular, there is no need to request the payload of these entries.

For some entry `x` the peer is interested in, the *certificate pool of `x`* is the (logarithmically sized) set of further entries the peer needs to store. It is defined as the union of the shortest link paths from:

- `x` to `w`, where `w` is the largest natural number strictly less than `x` such that there exists a natural number `k` with `w == (((3^k) - 1) / 2)`
- `w` to `0`, where `y` is defined as above
- `z` to `x`, where `z` is the smallest natural number greater than or equal to `x` such that there exists a natural number `l` with `z == (((3^l) - 1) / 2)`

Note that this is a superset of the entries needed to verify `x` agains the first entry. The path from `z` to `x` is needed so that the union of two certificate pools for two entries `x` and `y` (`x` < `y`) always contains a path from `y` to `0` via `x`. By requesting full certificate pools from its peers, a peer can then always verify the full log subset it is interested in.

Note also that the entries of the path from `z` to `x` do not necessarily exist yet. The log subset can still be fully verified, since the non-existent entries are only needed to allow later extensions of the subset (at which point the new entries do exist). Finally note that peers can efficiently request full certificate pools by just specifying the single sequence number they are interested in. They can also tell their peers about subsets of the certificate pool they already have in the same way, resulting in very low communication overhead.

## Related Work

[Secure scuttlebutt](https://www.scuttlebutt.nz/) inspired bamboo. Bamboo can be seen as a generalization of ssb's signed linked lists to a binary anti-monotone graph to allow partial replication. To mitigate the lack of partial replication, ssb supports retrieval of log entries via hash, but forfeits the security guarantees of regularly replicated entries. As of writing, ssb also signs the payloads directly rather than their hash, making it impossible to locally delete log payloads without losing the ability to replicate the log to other peers.

[Hypercore](https://github.com/mafintosh/hypercore) is a distributed, sparsely-replicatable append-only log like bamboo. It uses merkle trees whereas bamboo only uses backlinks. Bamboo's focus on transitive sparse replication via certificate pools can also be implemented on top of hypercore. The largest difference is that hypercore does not provide guarantees about the created-before order of log entries.

[Leyline-core](https://github.com/AljoschaMeyer/leyline-core) is the author's clumsy first attempt at defining a log structure that supports partial replication. It ends up badly reinventing [authenticated append-only skip lists](https://arxiv.org/pdf/cs/0302010.pdf), which are inferior to the [anti-monotone scheme](https://kodu.ut.ee/~lipmaa/papers/thesis/thesis.pdf) that bamboo uses.
