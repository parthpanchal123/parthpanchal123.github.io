---
title: "From 'Definitely Not' to 'Maybe': Building a Bloom Filter in Java"
date: 2026-03-28
tags: ["Java", "Data Structures", "Algorithms", "System Design"]
categories: ["Engineering"]
author: "Parth Panchal"
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "A deep dive into implementing a space-efficient probabilistic data structure from scratch in Java — covering the math, the code, the gotchas, and when to use one in production."
canonicalURL: "https://parthpanchal123.github.io/blog/bloom-filter-from-scratch"
disableHLJS: false
disableShare: false
hideSummary: false
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowCodeCopyButtons: true
math: true
cover:
    image: "images/bloom_filter_cover.svg"
    alt: "Bloom Filter Visualization"
    caption: "The magic of probabilistic data structures."
    relative: true
---

Ever wondered how Google tells you *instantly* that a username is already taken? Or how Chrome knows a URL is malicious before you even click it?

Checking a database with billions of records for every single keystroke would be a latency nightmare. Instead, these systems use a compact probabilistic structure called a **Bloom Filter** — one that can definitively tell you something is *absent*, but can only say something *might* be present. Today I'm walking through how I built one from scratch in Java, from raw logic to a production-ready generic class — including the math, the non-obvious gotchas, and when you'd actually use this over Guava's built-in.

---

## 1. What is a Bloom Filter?

To understand why this structure exists, think about the analogy first — then we'll look at the mechanism.

### The Kirana Store Analogy

Imagine you're managing a massive **Kirana store godown** (warehouse) during the peak Diwali rush. You have thousands of items stacked in piles, and digging through them to check if you stock a specific brand is a total nightmare.

So you use a **"Jugaad" shortcut**: a small index card at the front counter. Every time you stock a new item, you don't write its name. Instead, you follow a code and put **three specific colored stamps** on the card. For "Amul Ghee," the code might be **Red**, **Green**, and **Blue**.

This gives you two possible outcomes when a customer asks for something:

1. **The "Pakka No" (Definite No):** A customer asks for a rare brand of tea. You check the card — the **Red** stamp is missing. You know for a fact it's not there, without even walking to the back. *"Nahi hai, zepto karlo."* You just saved 10 minutes of pointless searching.

2. **The "Shaayad" (Maybe):** Another customer asks for "Saffola Oil." All three stamps are present. It *might* be in the back. But you still have to go check — because the Red, Green, and Blue stamps might have been left behind by three *different* items you stocked earlier (say, Parle-G set Red, Papad set Green, and Dettol set Blue). That's a false positive — the card lied.

A Bloom Filter is exactly this card, implemented as a **bit array with multiple hash functions**.

---

## 2. How It Works: The Mechanism

The structure is a bit array of size $m$, initialized to all zeros. Adding an item means running it through $k$ hash functions, each producing an index, then flipping those $k$ bits to **1**. Checking membership means running through the same $k$ functions and verifying all those bits are still **1**.

### The asymmetry that matters

- **ABSENT = CERTAIN:** If even one of the $k$ bits is `0`, the item was never added. It is *physically impossible* for it to be present, because adding it would have set that bit.
- **PRESENT = MAYBE:** All $k$ bits are `1`, but those bits might have been flipped by a combination of *other* items. That's a false positive.

The diagram below shows both scenarios. Panel A shows a true add: "Amul Ghee" sets bits 2, 6, and 11 via three hash functions. Panel B shows the false positive trap: three different items each happen to set one of those same bits. A query for "Saffola Oil" checks positions 2, 6, and 11 — finds all three at `1` — and incorrectly returns "maybe present."

![Bloom Filter false positive diagram — Panel A shows bits set by adding "Amul Ghee"; Panel B shows the same three bits set by three different items causing a false positive for "Saffola Oil"](/images/bloom_filter_false_positives_svg.svg)

This asymmetry is the core property — and the reason Bloom Filters are used as a fast pre-filter before hitting a slower, accurate store (like a database or disk).

### False positive probability

The probability of a false positive after inserting $n$ items into a filter of size $m$ with $k$ hash functions is:

$$P \approx \left(1 - e^{-kn/m}\right)^k$$

This is what we're optimizing when we choose $m$ and $k$ — we want $P$ to stay below our acceptable error rate.

---

## 3. The Math of Optimization

We don't pick $m$ and $k$ by guessing. Given an expected item count $n$ and a target false positive rate $p$, the optimal values are:

**Optimal bit array size ($m$):**
$$m = -\frac{n \ln p}{(\ln 2)^2}$$

**Optimal number of hash functions ($k$):**
$$k = \frac{m}{n} \ln 2$$

### Worked example

Say you're building a "have we seen this URL before?" cache for 1,000 items with a 1% false positive rate:

- $m = -\frac{1000 \times \ln(0.01)}{(\ln 2)^2} \approx 9{,}586$ bits — roughly **1.2 KB**
- $k = \frac{9586}{1000} \times \ln 2 \approx 7$ hash functions

Compare that to storing 1,000 strings (averaging ~40 bytes each) = ~40 KB. The Bloom Filter uses **33× less memory**, with only a 1% chance of a false positive.

---

## 4. The Hashing Strategy

We don't actually implement $k$ separate, complex hash algorithms. Instead, we use a technique from [Kirsch & Mitzenmacher (2008)](#references) — often called **hash function simulation** — which generates $k$ independent-looking indices from just two base hashes:

- **Primary hash ($h_1$):** `value.hashCode()`
- **Secondary hash ($h_2$):** `Integer.rotateRight(h1, 16)` — scrambles the bits efficiently
- **Derived indices:** For each $i$ from $0$ to $k-1$:

$$H_i(x) = (h_1 + i \times h_2) \pmod{m}$$

This is distinct from *double hashing* in the open-addressing hash table sense. Here we're simulating independent hash functions without the cost of actually computing them. The paper proves this approach introduces negligible degradation in false positive rates.

---

## 5. Implementation: Step-by-Step

### Core data structures

- **`java.util.BitSet`** — our storage layer; far more memory-efficient than a `boolean[]` since it packs 64 bits per `long`.
- **`int m`** — total number of bits in the filter.
- **`int k`** — number of hash indices we compute per item.

### Step 1 — The non-generic version

Start simple: a filter that only handles `String` values. This isolates the hashing logic before we generalize.

```java
public class BloomFilter {
    private final BitSet bitSet;
    private final int m, k;

    public BloomFilter(int m, int k) {
        this.bitSet = new BitSet(m);
        this.m = m;
        this.k = k;
    }

    public void add(String value) {
        int h1 = value.hashCode();
        int h2 = Integer.rotateRight(h1, 16);

        for (int i = 0; i < k; i++) {
            int combinedHash = h1 + (i * h2);
            // The & 0x7fffffff mask strips the sign bit — see callout below
            int index = (combinedHash & 0x7fffffff) % m;
            bitSet.set(index);
        }
    }

    public boolean mightContain(String value) {
        int h1 = value.hashCode();
        int h2 = Integer.rotateRight(h1, 16);

        for (int i = 0; i < k; i++) {
            int index = ((h1 + (i * h2)) & 0x7fffffff) % m;
            if (!bitSet.get(index)) return false; // Definite NO
        }
        return true; // Maybe
    }
}
```

> **The sign-bit trap:** In Java, `Math.abs(Integer.MIN_VALUE)` returns `Integer.MIN_VALUE` — it stays negative because there's no positive counterpart in 32-bit signed integers. Using the raw value with `% m` would produce a negative index and throw an `ArrayIndexOutOfBoundsException`. The bitwise AND `& 0x7fffffff` clears only the sign bit, forcing the value positive without any branch or overflow risk.

### Tracing a concrete add

Let's trace `add("hello")` on a filter with `m=16`, `k=3`:

```text
h1 = "hello".hashCode()        = 99162322
h2 = Integer.rotateRight(h1,16) = 1513677412  (approximate)

i=0: index = (99162322 + 0) & 0x7fffffff % 16 = 2
i=1: index = (99162322 + 1513677412) & 0x7fffffff % 16 = 6
i=2: index = (99162322 + 3027354824) & 0x7fffffff % 16 = 11

Bits set: [2, 6, 11]
```

A subsequent `mightContain("hello")` recomputes the same three indices and confirms bits 2, 6, and 11 are all `1`. A word that hashes to indices [2, 5, 11] would return `false` immediately at bit 5 — a definite NO.

---

## 6. Refactoring for Generics

The only real change is the type parameter — the hashing logic is identical since we're still calling `.hashCode()`, which every Java object has via `Object`.

```java
public class BloomFilter<T> {
    private final BitSet bitSet;
    private final int m, k;

    /**
     * @param n  Expected number of items to insert
     * @param p  Desired false positive probability (e.g. 0.01 for 1%)
     */
    public BloomFilter(int n, double p) {
        if (n <= 0) throw new IllegalArgumentException("n must be positive");
        if (p <= 0 || p >= 1) throw new IllegalArgumentException("p must be in (0, 1)");

        this.m = (int) Math.ceil(-n * Math.log(p) / Math.pow(Math.log(2), 2));
        this.k = (int) Math.ceil((m / (double) n) * Math.log(2));
        this.bitSet = new BitSet(m);
    }

    public void add(T value) {
        if (value == null) throw new IllegalArgumentException("Null values are not supported");

        int h1 = value.hashCode();
        int h2 = Integer.rotateRight(h1, 16);

        for (int i = 0; i < k; i++) {
            int index = ((h1 + (i * h2)) & 0x7fffffff) % m;
            bitSet.set(index);
        }
    }

    public boolean mightContain(T value) {
        if (value == null) return false;

        int h1 = value.hashCode();
        int h2 = Integer.rotateRight(h1, 16);

        for (int i = 0; i < k; i++) {
            int index = ((h1 + (i * h2)) & 0x7fffffff) % m;
            if (!bitSet.get(index)) return false;
        }
        return true;
    }

    /** Approximate current false positive rate based on items inserted so far. */
    public double currentFalsePositiveRate(int itemsInserted) {
        return Math.pow(1 - Math.exp(-(double) k * itemsInserted / m), k);
    }

    public int getBitArraySize() { return m; }
    public int getHashCount()    { return k; }
}
```

**Key change from the non-generic version:** The constructor now takes `n` and `p` — the user declares their intent, and the filter derives its own optimal `m` and `k`. You no longer have to manually calculate these and pass them in correctly.

**Usage:**
```java
BloomFilter<String> urlCache = new BloomFilter<>(1_000, 0.01);
urlCache.add("https://malicious-site.com");

if (urlCache.mightContain("https://malicious-site.com")) {
    // Hit the real DB/blocklist to confirm
}
```

---

## 7. Limitations to Know Before You Ship

A Bloom Filter is powerful, but it has hard constraints that make it unsuitable in some situations:

| Constraint | Detail |
| :--- | :--- |
| **No deletion** | You can't remove an item. Clearing a bit might unset a bit shared by another item, corrupting the filter. (Counting Bloom Filters solve this at the cost of more memory.) |
| **No value retrieval** | The filter only stores membership signals, not the items themselves. You can't ask "what items are in this filter?" |
| **False positive rate grows** | If you insert more items than the `n` you planned for, $P$ climbs above your target. The filter silently degrades — use `currentFalsePositiveRate()` to monitor this in production. |
| **Hash quality matters** | Java's default `hashCode()` is not a cryptographic or uniformly distributed hash. For high-sensitivity use cases, consider wrapping objects with a stronger hash (e.g. MurmurHash via a library). |

---

## 8. Roll Your Own vs. Use Guava

Google's Guava library ships a production-hardened `com.google.common.hash.BloomFilter`. It uses a better hash function (Murmur3), handles serialization, and is well-tested at scale.

**Use Guava when:**
- You're building something production-critical and Guava is already a dependency.
- You need serialization / persistence of the filter state.
- You want to avoid rolling your own hash quality decisions.

**Roll your own when:**
- You want to understand the structure deeply (exactly what you're doing here).
- You have no JVM dependencies available (embedded systems, Android with strict APK size limits).
- You need a custom eviction or counting variant that Guava doesn't support.

For most production Java services, Guava's `BloomFilter` is the right call. The custom implementation here is the foundation for understanding — and extending — it.

---

## 9. Performance and Complexity

| Operation | Time Complexity | Space |
| :--- | :--- | :--- |
| `add(value)` | $O(k)$ | $O(1)$ additional — bits are set in the existing array |
| `mightContain(value)` | $O(k)$ | $O(1)$ additional — no allocation |
| Construction | $O(m)$ | $O(m)$ bits — the bit array itself |

$k$ is typically a small constant (6–10 for common configurations), so both `add` and `mightContain` are effectively $O(1)$ in practice. The memory footprint is $m$ bits — for the 1,000-item / 1% error example above, that's under 1.2 KB, regardless of how large the inserted strings are.

---

## References

1. **Bloom, B. H. (1970).** *Space/Time Trade-offs in Hash Coding with Allowable Errors.* Communications of the ACM, 13(7), 422–426. — The original paper introducing the structure.

2. **Kirsch, A., & Mitzenmacher, M. (2008).** *Less Hashing, Same Performance: Building a Better Bloom Filter.* Random Structures & Algorithms, 33(2), 187–218. — The source for the two-hash simulation technique used here.

3. **Google Guava `BloomFilter` source** — [github.com/google/guava](https://github.com/google/guava/blob/master/guava/src/com/google/common/hash/BloomFilter.java) — worth reading after you've built your own.

---
