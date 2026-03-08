# Trie Streaming: Verifiable Data Streaming for Ethereum Transaction Tries

## 1. Background

Ethereum's transaction trie is a Merkle Patricia Tree (MPT) where keys are `RLP(index)` for 0-based transaction indices. Since the number of transactions N fully determines the trie structure (all keys are known), we can exploit this to stream verifiable data with minimal overhead.

## 2. RLP Key Encoding and Nibble Paths

| Index Range | RLP Key | Nibble Path | First Nibble |
|---|---|---|---|
| 0 | `0x80` | `[8, 0]` | 8 |
| 1-15 | `0x01`-`0x0f` | `[0, 1]`-`[0, f]` | 0 |
| 16-31 | `0x10`-`0x1f` | `[1, 0]`-`[1, f]` | 1 |
| 32-47 | `0x20`-`0x2f` | `[2, 0]`-`[2, f]` | 2 |
| ... | ... | ... | ... |
| 112-127 | `0x70`-`0x7f` | `[7, 0]`-`[7, f]` | 7 |
| 128-255 | `0x8180`-`0x81ff` | `[8, 1, 8, 0]`-`[8, 1, f, f]` | 8 |
| 256-65535 | `0x820100`-`0x82ffff` | `[8, 2, 0, 1, 0, 0]`-`[8, 2, f, f, f, f]` | 8 |

**Key observation:** The trie is heavily unbalanced. Nibble `8` contains index 0 _plus_ all indices >= 128. Nibbles 0-7 each get ~16 entries. Nibbles 9-f are empty for typical blocks.

## 3. Streaming Overhead Analysis for the Standard MPT

### 3.1 Core Theorem

Both sender and receiver know the trie structure (given N). The receiver can reconstruct any subtree from its leaf values. The **only overhead** for incremental verification is **sibling hashes** -- hashes of subtrees the receiver hasn't yet computed.

At any branch node with k populated children, processing the first child's subtree requires sending **k - 1 sibling hashes**. All subsequent children verify for free (their expected hashes were already communicated).

**Theorem:** Total overhead for per-leaf verification = **N - 1 hashes** (each 32 bytes).

**Proof:** Let B = number of branch nodes, k_i = populated children of branch i. The sum of overhead across all branch nodes is:

```
Sum(k_i - 1) = Sum(k_i) - B
```

In the tree, total edges = total nodes - 1. Edges from branch nodes = Sum(k_i), edges from extension nodes = N_ext. So:

```
Sum(k_i) + N_ext = N_leaves + B + N_ext - 1
Sum(k_i) = N_leaves + B - 1
Sum(k_i - 1) = N_leaves + B - 1 - B = N_leaves - 1 = N - 1  QED
```

### 3.2 Overhead Spectrum

| Strategy | Overhead | First verification after |
|---|---|---|
| Verify at end only | **0 hashes** (0 bytes) | All N txs received |
| Per top-level subtree | **k_root - 1 hashes** (~8 x 32 = 256 bytes) | First subtree (~15 txs) |
| Per leaf (finest) | **N - 1 hashes** (~6.4 KB for N=200) | 1 tx |

### 3.3 Optimal Strategy for Standard MPT

Stream in **nibble-path (trie) order**, not index order, with **subtree-level batching**:

1. Send N (the tx count) -- a few bytes
2. Batch 1: txs for nibble-0 subtree (indices 1-15) + **8 sibling hashes** (256 bytes)
3. Batch 2-8: txs for nibble-1 through nibble-7 subtrees -- **0 overhead** each
4. Batch 9: txs for nibble-8 subtree (index 0 + indices 128+) -- **0 overhead**

Receiver verifies each batch by computing the subtree hash and checking against the hash received/computed earlier.

**Total overhead: ~256 bytes for a typical block.** Overhead ratio < 1% of transaction data.

---

## 4. Alternative Trie Structures

### 4.1 Fixed-Width Big-Endian Keys

**Encoding:** Use 2-byte big-endian index instead of RLP.

```
Index 0   -> 0x0000 -> nibbles [0, 0, 0, 0]
Index 1   -> 0x0001 -> nibbles [0, 0, 0, 1]
Index 15  -> 0x000f -> nibbles [0, 0, 0, f]
Index 16  -> 0x0010 -> nibbles [0, 0, 1, 0]
Index 255 -> 0x00ff -> nibbles [0, 0, f, f]
Index 256 -> 0x0100 -> nibbles [0, 1, 0, 0]
```

**Properties:**
- For N <= 256: All keys share prefix `[0, 0]`, so an extension node covers the first two nibbles. The remaining 2 nibbles give a balanced 16x16 tree.
- For N <= 4096: Keys spread across `[0, 0]` and `[0, 1]`-`[0, f]`, still reasonably balanced.
- Top-level branch only populates nibble 0 for N <= 65536, so this **degenerates to a deep chain** with extension nodes. Not good for streaming.

**Verdict:** Worse than RLP for streaming. The shared prefix creates long extension nodes and deep, unbalanced structure.

### 4.2 Reversed-Byte Keys

**Encoding:** Reverse the bytes of the big-endian representation.

```
Index 0   -> 0x00   -> nibbles [0, 0]
Index 1   -> 0x01   -> nibbles [0, 1]
Index 255 -> 0xff   -> nibbles [f, f]
Index 256 -> 0x0001 -> nibbles [0, 0, 0, 1]  (reverse of 0x0100)
Index 511 -> 0xff01 -> nibbles [f, f, 0, 1]
```

**Properties:**
- For N <= 256: All first nibbles 0-f are populated. Indices 0-15 go to nibble 0, 16-31 to nibble 1, ..., 240-255 to nibble f. Perfectly balanced (16 entries per top-level slot).
- For N = 257-512: Index 256 maps to `[0, 0, 0, 1]` which collides in prefix with index 0 at `[0, 0]`. This creates deeper structure only in the slots that need it.

**Streaming overhead (N=200):**
- 13 populated first nibbles (nibbles 0-c, since 12*16=192, and nibble c has indices 192-199)
- Top-level: 12 sibling hashes = 384 bytes
- Balanced subtrees = predictable batch sizes of 16

**Verdict:** Better balance than RLP, but the variable key length when crossing 256 boundaries creates complexity.

### 4.3 Nibble-Spread Keys (Optimal for MPT Streaming)

**Encoding:** Encode the index directly as a sequence of nibbles, most-significant nibble first, with a fixed width determined by N.

For N transactions, use `ceil(log16(N))` nibbles:
- N <= 16: 1 nibble (keys 0x0-0xf)
- N <= 256: 2 nibbles (keys 0x00-0xff)
- N <= 4096: 3 nibbles (keys 0x000-0xfff)

```
N = 200 (2 nibbles):
  Index 0   -> nibbles [0, 0]
  Index 1   -> nibbles [0, 1]
  Index 15  -> nibbles [0, f]
  Index 16  -> nibbles [1, 0]
  Index 199 -> nibbles [c, 7]
```

**Properties:**
- **Maximally balanced** for any N: top-level branch has ceil(N/16) or floor(N/16) entries per slot
- **Fixed depth** for all keys: exactly ceil(log16(N)) nibbles
- **Uniform subtree sizes**: each top-level subtree has either floor(N/16) or ceil(N/16) leaves

**Streaming overhead (N=200):**
- 13 populated top-level nibbles (0-c)
- Top-level batch verification: 12 sibling hashes = 384 bytes
- Each subtree has 15-16 leaves, all at depth 2 (leaf nodes directly under the branch)
- No further internal branching needed = **no recursive overhead**
- Total: **384 bytes**

**Streaming schedule:**
- Batch size: 16 txs (one full subtree) for nibbles 0-b, then 8 txs for nibble c
- 13 batches total, first batch carries 12 sibling hashes
- Every batch is independently verifiable after batch 1

**Verdict:** Optimal for MPT streaming. Minimal depth, balanced distribution, predictable batch sizes.

### 4.4 Bit-Reversal Permutation Keys

**Encoding:** For N transactions, use bit-reversal of the index within the range [0, next_power_of_2(N)).

```
N = 200, using 8 bits (next power of 2 = 256):
  Index 0   -> reverse(00000000) = 00000000 -> 0x00
  Index 1   -> reverse(00000001) = 10000000 -> 0x80
  Index 2   -> reverse(00000010) = 01000000 -> 0x40
  Index 3   -> reverse(00000011) = 11000000 -> 0xc0
  Index 4   -> reverse(00000100) = 00100000 -> 0x20
```

**Properties:**
- Creates a perfectly balanced binary-like trie
- Indices that are close in number are maximally spread in the trie
- The first N entries of any power-of-2 sized bit-reversal sequence are optimally distributed

**Streaming:** Balanced but the key mapping is non-intuitive and offers no advantage over 4.3.

**Verdict:** Theoretically elegant but unnecessarily complex compared to nibble-spread keys.

### 4.5 Comparison Table

| Structure | Top-level overhead (N=200) | Total overhead | Depth uniformity | Batch predictability |
|---|---|---|---|---|
| RLP (current) | 256 bytes (8 hashes) | ~256-320 bytes | Poor (2-6 nibbles) | Poor (15 vs 73 per subtree) |
| Fixed-width BE | N/A (degenerates) | High | Poor | Poor |
| Reversed-byte | 384 bytes (12 hashes) | ~384 bytes | Moderate | Good (16 per subtree) |
| **Nibble-spread** | **384 bytes (12 hashes)** | **~384 bytes** | **Perfect** | **Excellent (15-16 per subtree)** |
| Bit-reversal | ~384 bytes | ~384 bytes | Good | Good |

---

## 5. Hash-Linked List Structure

### 5.1 Definition

Instead of a Merkle trie, use a **recursive hash chain** computed in reverse order:

```
H(N-1) = hash(tx[N-1])
H(i)   = hash(tx[i] ++ H(i+1))    for i = N-2, N-3, ..., 0

Root   = H(0)
```

Where `++` denotes concatenation. The root is H(0), the hash of the first transaction concatenated with the recursive hash of all subsequent transactions.

### 5.2 Streaming Protocol

Stream transactions in index order (0, 1, 2, ...):

```
Send: tx[0]
  Receiver knows: tx[0], needs H(1) to verify -> sender sends H(1)
  Receiver checks: hash(tx[0] ++ H(1)) == Root  ✓

Send: tx[1]
  Receiver knows: tx[1], has H(1) from step above
  Receiver needs: H(2) to verify -> sender sends H(2)
  Receiver checks: hash(tx[1] ++ H(2)) == H(1)  ✓

Send: tx[2]
  ...same pattern...

Send: tx[N-1]
  Receiver knows: tx[N-1], has H(N-1) from previous step
  Receiver checks: hash(tx[N-1]) == H(N-1)  ✓
```

### 5.3 Overhead Analysis

**Per-transaction verification overhead: exactly 1 hash (32 bytes) per transaction**, except the last one (0 bytes).

**Total overhead: (N-1) x 32 bytes.**

For N=200: 199 x 32 = **6,368 bytes**.

This matches the worst-case (per-leaf) MPT overhead, which is also N-1 hashes.

### 5.4 Batched Streaming

We can batch K transactions together and send one hash per batch:

```
Send: tx[0], tx[1], ..., tx[K-1], H(K)
  Receiver computes H(0) from these and verifies against Root

Send: tx[K], tx[K+1], ..., tx[2K-1], H(2K)
  Receiver computes H(K) from these and verifies against received H(K)

...
```

**Overhead: ceil(N/K) - 1 hashes.** For K=16, N=200: 12 hashes = 384 bytes. Comparable to the nibble-spread MPT approach.

### 5.5 Advantages

1. **Trivially simple implementation.** No trie logic needed -- just hashing.
2. **Natural streaming order.** Transactions stream in index order (0, 1, 2, ...), which is the order they appear in the block and the most intuitive order.
3. **Constant overhead per item.** Exactly 1 hash per verification point, regardless of trie depth or key encoding.
4. **No knowledge of N required upfront.** The receiver doesn't need to know the total count to verify each item. Each tx[i] + H(i+1) is self-contained proof against H(i). (Though knowing N helps for pre-allocation.)
5. **No structural overhead.** No branch nodes, extension nodes, or leaf encoding. The structure IS the data plus the hash chain.
6. **Perfect streaming granularity.** Can verify every single transaction with exactly 32 bytes overhead each, or batch arbitrarily. The tradeoff curve is smooth and linear.
7. **Minimal computation.** Each verification step is a single hash operation. No trie traversal, no nibble path matching.

### 5.6 Disadvantages

1. **No random access proofs.** To prove tx[i] is correct, you need H(i+1), which requires knowing all of tx[i+1], ..., tx[N-1] (or at least their recursive hash). In a Merkle trie, you can prove any single element with O(log N) hashes. Here, proving tx[0] alone requires ALL subsequent data.
2. **Sequential verification dependency.** Each verification step depends on the previous one. You MUST process transactions in order (0, 1, 2, ...). With the MPT approach, you can verify subtrees independently and in parallel.
3. **No subset proofs.** Cannot prove "tx[i] is in this set" without revealing the entire tail of the list. This makes the structure unsuitable for light client proofs or sparse queries.
4. **Recomputation cost.** To build the root hash, you must process ALL transactions in reverse order. If one transaction changes, you must recompute from that index back to 0. In a Merkle trie, updates are O(log N).
5. **Incompatible with existing Ethereum infrastructure.** The current protocol specifies MPT for transaction tries. Changing this requires a hard fork and ecosystem-wide updates.
6. **No parallel construction.** The hash chain must be built sequentially (in reverse). The MPT can be built with parallel subtree construction.

### 5.7 When to Use Which

| Requirement | Best Structure |
|---|---|
| Stream all txs in order, verify incrementally | **Hash-linked list** (simplest, same overhead) |
| Random access proofs (prove single tx) | **MPT** (O(log N) proof) |
| Parallel verification of independent subsets | **MPT** (independent subtrees) |
| Light client sparse queries | **MPT** (Merkle proofs) |
| Minimal implementation complexity | **Hash-linked list** |
| Existing Ethereum compatibility | **MPT with RLP keys** |
| Optimal new design for ordered streaming | **Hash-linked list with batching** or **Nibble-spread MPT** |

---

## 6. Optimal Combined Strategy

### 6.1 For Pure Streaming (new protocol, no random access needed)

**Use the hash-linked list with adaptive batching:**

- Batch size K = tunable parameter (e.g., 16 or 32)
- Overhead: ceil(N/K) - 1 hashes
- For N=200, K=16: 12 hashes = 384 bytes overhead
- For N=200, K=32: 6 hashes = 192 bytes overhead
- First verification after K transactions
- Implementation: ~20 lines of code

### 6.2 For Streaming with Random Access (hybrid needs)

**Use nibble-spread MPT with subtree batching:**

- Keys = nibble-encoded indices with fixed width ceil(log16(N))
- Stream in trie order, one subtree per batch
- Overhead: k_root - 1 hashes at top level (~384 bytes for N=200)
- Retains O(log N) random access proofs
- Batch size naturally = floor(N/16) or ceil(N/16) for top-level

### 6.3 For Ethereum Compatibility

**Use current RLP keys with trie-ordered streaming:**

- Stream transactions in nibble-path order, not index order
- Batch by top-level subtree
- Overhead: 8 hashes = 256 bytes for typical blocks
- No protocol changes required, only changes to streaming logic
- Streaming order for N=200: indices [1-15, 16-31, ..., 112-127, 0, 128-199]

---

## 7. Summary of Key Results

1. **MPT overhead theorem:** Streaming N transactions through an MPT with per-leaf verification costs exactly N-1 sibling hashes. Subtree batching reduces this to k_root - 1 hashes.

2. **Hash-linked list equivalence:** For ordered streaming, the hash-linked list achieves the same per-item overhead (1 hash) as per-leaf MPT verification, but with dramatically simpler implementation.

3. **Batching is the key optimization** for both structures. With batch size K:
   - MPT: ~(N/K - 1) hashes if batches align with subtrees
   - Hash list: exactly ceil(N/K) - 1 hashes, always

4. **The optimal structure depends on access patterns:**
   - Streaming only -> hash-linked list (simplest, optimal overhead)
   - Streaming + random access -> nibble-spread MPT
   - Ethereum compatibility -> RLP MPT with trie-ordered streaming

5. **For Ethereum's current design**, the best achievable without protocol changes is trie-ordered streaming with subtree batching: **~256 bytes overhead** for a typical block of 200 transactions.
