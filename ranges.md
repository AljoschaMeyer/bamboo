# Bamboo Ranges

Bamboo supports partial replication of logs, it is possible to request only a subset of a log while still being able to verify the data. This raises the question on how peers can efficiently communicate which subset(s) they are interested in. In general, indicating interest in arbitrary subsets takes space linear in the size of the subset in the worst case. There are however some special cases that can be represented more efficiently. This text proposes one particular set of such mechanisms, called *bamboo ranges*. A particularly important use-case for ranges is that peers express what kind of data they can offer or are interested in to guide replication of bamboo logs.

There are many subsets of a log that can *not* be represented as a single range. This is by design, the desired efficiency could not be achieved otherwise. There are three main options for application designers to deal with this:

- restrict themselves to the efficient cases
- specify multiple ranges for the same log (e.g. an arbitrary subset can be emulated by requesting all items individually)
- define a more complex mechanism, then implement it efficiently (e.g. as additional database indexes and rpc parameters)

## Definition of Ranges

A range is defined by a *start* sequence number and either an *end* sequence number (a *closed* range) or it may be *open* (no fixed end point). It consists of the entries (metadata and payload) of all entries between the *start* and the *end* (inclusive), or all the entriess from the start (inclusive) and onwards in case of an open range. Additional parameters allow for limited filtering within that range:

- a range may be *sparse*: Instead of containing all entries between start and end, a sparse range only contains the entries on the shortest path from end to start. For open ranges, the shortest path from the newest available entry to the start is used.
- a range may be a *no-metadata* range: A no-metadata range excludes the metadata of its entries, leaving only their payloads.
- a range may specify a *minimum size* (a natural number between 0 and 2^64-1 inclusive): Payloads with size equal to or less than the minimum size are excluded from the range.
- a range may specify a *maximum size* (a natural number between 0 and 2^64-1 inclusive): Payloads with size equal to or greater than the minimum size are excluded from the range.
- a range carries a (possibly empty) set of *forbidden* sequence numbers: The payloads of entries in the forbidden set are excluded from the set.

These features impact the design of log storage and verification. An implementation should be able to efficiently retrieve, store and verify all data addressed by a range (necessitating some sort of sparse array data structure, and probably separate storage for metadata and payloads). Receiving multiple pieces of data from a range can also be an opportunity to use bulk signature verifications.

## Relation to Certificate Pools

Ranges can be used to guide log replication, and log replication should always involve replicating the certificate pools of all replicated entries (unless the range is a no-metadata range). The certificate pools in a range just so happen to combine in a simple manner: Take the shortest path from the start seqnum to 1 and from the next largest seqnum on the "spine" to the end seqnum, and together with the metadata in range itself, you catch all necessary data.

When replicating data based on a range, we don't want to send certificate pool entries that are already available at the other endpoint. Specifying the full set of certificate pool entries that need or don't need to be transmitted would be inefficient. Instead, the following scheme can be used: With each range, specify a `head-max` sequence number, and with closed ranges also specify a `tail-min` sequence number. These numbers indicate which parts of the requested entries' certificate pools are *not* being requested. We define the *head* of the certificate pool for an range of start seqnum `k` to be the shortest path from `k` to `1`, and the *tail* of the certificate pool for a range of end seqnum `l` to be the shortest path from the "spine" to `l`.

If `head-max` is part of the head of the certificate pool, then only the path from the start senqum up to (but excluding) `head-max` is requested. Otherwise, the full head of the certificate pool is requested (preferably `head-max` should be zero to indicate this). `tail-min` works analogously: Only the path from `tail-min` to the end seqnum is requested, or the full tail if `tail-min` is not part of the tail (again, this should be signaled with a zero).

## Encoding

TODO

- ideally, trimming, extending and merging ranges can be done without decoding (define operations on the binary representations such that there's an isomorphism with the logical structure and operations)
  - of these, an isomorphim for merging is the most important (e.g. a router would want to do merges efficiently, without a decoding step)
- canonicity? probably not needed, and sparse and non-sparse ranges of one entry are equivalent anyways
- the set of forbidden seqnums is supposed to stay small, favor simplicity over clever compression
- compact special cases: individual entry, individual payload, individual metadata?

## Appendix A: Omitted Features

Here are a few notable exclusions from this feature set and the rationales for leaving them out:

Arbitrary subsets instead of ranges could not be expressed concisely. While there are some fun ways of compressing them (e.g. [finite state automata](https://en.wikipedia.org/wiki/Deterministic_acyclic_finite_state_automaton) or [roaring bitmaps](http://www.roaringbitmap.org/)), the overall space requirement stays in O(n), whereas ranges have size O(1) (except for the set of forbidden sequence numbers, but these are expected to be used rarely).

There is no way to specify forbidden hashes, only forbidden sequence numbers. The idea is that in general only a very small portion of a log would be explictly excluded from replication. If you disagreed with the majority of a log, why would you replicate it in the first place?  
Over a large number of different logs, you might still aquire a large set of forbidden payload hashes. When defining a range over an unknown log, attaching the hashes of all of these would not be feasable. And if you did already know that the log contained a forbidden payload, then you could also address it efficiently via its sequence number.

The ability to express ranges such as "the newest 100 entries" is another deliberate omission. Such a range would lose some helpful properties: it would refer to different, not monotonically growing sets of entries at different points in time (i.e. at different lengths of the log). When this kind of best-effort data fetching is required, there a couple of alternatives: A sparse, open range starting at 1 can be used to obtain the length of the log, and then a proper starting point for a second range can be computed from that length. Or the length of the log replica at the other endpoint could be exchanged a priori. Finally, it might be worth designing the application such that a sparse range actually satisfies the request, e.g. by bundling important data in the sparse entries.

## Appendix B: Restricted Feature Sets

In some settings, it might make sense to remove open requests. For one, this makes the addressed data fully invariant over time. More importantly, such ranges would be suitable for implementing backpressure. Instead of fetching an arbitrary amount of data from the (potential) future, you would iteratively request a fixed amount of data from the future, thus controling the rate at which future data could flow to you.

A different feature whose removal might make sense in some settings is that of sparse ranges. Without them, all access to payloads within a single range is sequential.
