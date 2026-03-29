---
layout: default
title: "Bloom Filters: The Data Structure That Lies to Save Time"
date: 2026-03-30
---

## Bloom Filters: The Data Structure That Lies to Save Time

Most data structures give you exact answers. You ask "is this element in the set?" and you get yes or no, guaranteed correct. A Bloom filter gives you a different deal: it will never miss something that exists, but it will occasionally say "yes" when it should say "no." In exchange, it uses dramatically less memory than any exact alternative.

That tradeoff turns out to be useful in a surprising number of places.

### The problem

Imagine you have a list of 100 million usernames and you need to check whether a new signup already exists. You could store all 100 million strings in a hash set. That works, but it costs memory — roughly a few gigabytes, depending on username length. If you are running this check on every API request, that memory adds up.

Now imagine you do not need a perfect answer. You just need a fast "definitely not in the set" check to avoid hitting the database on every request. If the filter says "not here," you trust it completely. If it says "maybe here," you go check the database. Most lookups will be for usernames that do not exist, so the filter handles the common case and the database handles the rare one.

That is exactly what a Bloom filter does.

### How it works

A Bloom filter is a bit array — a long sequence of 0s and 1s — plus a set of hash functions. Start with every bit set to 0.

**To add an element**, run it through each hash function. Each hash function produces a position in the bit array. Set those bits to 1.

```text
Add "alice":
  hash1("alice") → position 3   → set bit 3 to 1
  hash2("alice") → position 7   → set bit 7 to 1
  hash3("alice") → position 12  → set bit 12 to 1

Bit array: [0 0 0 1 0 0 0 1 0 0 0 0 1 0 0 0]
```

**To check if an element exists**, run it through the same hash functions and look at the same positions. If all bits are 1, the element is "probably in the set." If any bit is 0, the element is "definitely not in the set."

```text
Check "bob":
  hash1("bob") → position 3   → bit is 1
  hash2("bob") → position 9   → bit is 0  ← stop here
  Result: definitely not in the set.

Check "eve":
  hash1("eve") → position 7   → bit is 1
  hash2("eve") → position 3   → bit is 1
  hash3("eve") → position 12  → bit is 1
  Result: maybe in the set (could be a false positive).
```

"eve" got a "maybe" because her hash positions happen to overlap with bits that "alice" already set. Nobody named "eve" was ever added. This is the false positive — the lie.

### Why false positives happen (but false negatives do not)

Once a bit is set to 1, it stays at 1 forever. Multiple elements can set the same bit. So when you check for an element, you might find all its bits already set by other elements. That is a false positive.

But if an element was actually added, its bits were definitely set to 1. No other operation can turn a bit back to 0. So if any bit is 0, the element was never added. That is why false negatives are impossible.

This asymmetry — never wrong about "no," sometimes wrong about "yes" — is the core property.

### The math

The false positive rate depends on three things:

- **m**: the number of bits in the array
- **n**: the number of elements you plan to insert
- **k**: the number of hash functions

The approximate false positive rate after inserting n elements is:

```text
p ≈ (1 - e^(-kn/m))^k
```

For a given n and desired false positive rate p, the optimal number of bits is:

```text
m = -(n · ln(p)) / (ln(2))²
```

And the optimal number of hash functions is:

```text
k = (m/n) · ln(2)
```

In practice, the numbers are generous. To store 100 million elements with a 1% false positive rate, you need about 114 MB. For a 0.1% rate, about 172 MB. Compare that to storing the actual data, which could easily be several gigabytes.

### What you cannot do

Bloom filters have hard constraints:

**No deletion.** You cannot remove an element, because setting a bit back to 0 might affect other elements that share that bit. (Counting Bloom filters solve this by replacing each bit with a counter, at the cost of more memory.)

**No enumeration.** You cannot list what is in the filter. The bit array does not store the elements themselves — only their hash fingerprints, smeared together.

**No counting.** You cannot ask "how many times was this element added?" A standard Bloom filter only answers membership questions.

**No resizing.** Once you pick m (the number of bits), that is it. If you insert more elements than planned, the false positive rate goes up. You have to rebuild the filter with a larger array.

### Where they get used

Bloom filters show up in systems where the cost of a false positive is low but the cost of a database lookup on every request is high:

**Databases.** LSM-tree storage engines (LevelDB, RocksDB, Cassandra) use Bloom filters to avoid reading disk for keys that do not exist. Before scanning an SSTable file, the engine checks the Bloom filter. If the key is not there, it skips the file entirely. This turns what would be multiple disk reads into a single bit check.

**Web caching.** CDNs use Bloom filters to decide whether a URL has been requested before. On the first request, the object is served but not cached (most URLs are only ever requested once). On the second request, the Bloom filter says "seen this before" and the object gets cached. This avoids filling the cache with one-hit wonders.

**Network routing.** Distributed systems use Bloom filters to track which nodes have which data. Instead of sending the full list of keys, a node sends its Bloom filter. Other nodes can check membership without a network round trip.

**Spell checkers.** Early spell checkers used Bloom filters to store dictionaries. The filter could answer "is this word in the dictionary?" with very little memory. A false positive meant an occasional misspelled word was accepted — a minor annoyance, not a disaster.

### Variants worth knowing

The basic Bloom filter has inspired several specialized versions:

**Counting Bloom filters** replace each bit with a small counter (usually 4 bits), which enables deletion. The tradeoff is 4x memory.

**Cuckoo filters** support deletion natively and often have better lookup performance for low false positive rates. They use a different approach: instead of hash-to-bit mapping, they store short fingerprints in a hash table with cuckoo hashing for collision resolution.

**Scalable Bloom filters** solve the resizing problem by chaining multiple filters together, each with a tighter false positive rate. When the current filter is full, a new one is added. The combined false positive rate stays bounded.

### The intuition

A Bloom filter is a lossy compression of a set. It throws away the actual elements and keeps only enough information to answer "definitely not" or "probably yes." The compression is extreme — from gigabytes of raw data to megabytes of bits — and the cost is a small, tunable probability of being wrong in one direction.

Most engineering decisions are about tradeoffs. A Bloom filter makes the tradeoff explicit and quantifiable: you choose your false positive rate, compute the memory cost, and decide whether the deal is worth it. For membership checks against large sets where a "maybe" is acceptable and a definitive "no" is valuable, it almost always is.
