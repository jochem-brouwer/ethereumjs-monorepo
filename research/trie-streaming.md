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
| 65536-16777215 | `0x83010000`-`0x83ffffff` | `[8, 3, 0, 1, 0, 0, 0, 0]`-`[8, 3, f, f, f, f, f, f]` | 8 |

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

---

## 8. The 65536 Boundary Problem in RLP Encoding

### 8.1 The Depth Discontinuity

RLP encoding creates variable-length keys. At powers of 256, the key length jumps:

| Boundary | Last key before | Nibble depth | First key after | Nibble depth | Depth jump |
|---|---|---|---|---|---|
| 128 | `0x7f` → `[7, f]` | 2 | `0x8180` → `[8, 1, 8, 0]` | 4 | +2 |
| 256 | `0x81ff` → `[8, 1, f, f]` | 4 | `0x820100` → `[8, 2, 0, 1, 0, 0]` | 6 | +2 |
| 65536 | `0x82ffff` → `[8, 2, f, f, f, f]` | 6 | `0x83010000` → `[8, 3, 0, 1, 0, 0, 0, 0]` | 8 | +2 |

Each boundary adds 2 nibbles (1 byte) of depth. The keys all nest under nibble `[8]` at the root, creating a deeply unbalanced subtree.

### 8.2 Overhead Spike at the 65536 Boundary

Consider a trie with N = 100,000 transactions (indices 0-99999). Assume we stream in trie order with subtree batching. After streaming all indices in `[8, 2, ...]` (up to index 65535), we enter the `[8, 3, ...]` subtree.

**Trie path to index 65536:** `[8, 3, 0, 1, 0, 0, 0, 0]`

All indices 65536-99999 have RLP keys `0x83010000`-`0x8301869f`. In nibbles, they all share prefix `[8, 3, 0, 1]` (since the first byte of the 3-byte value is always `0x01` for this range). This creates an extension node from `[8, 3]` to `[8, 3, 0, 1]`.

The branch structure below `[8, 3, 0, 1]`:

```
[8, 3, 0, 1]  (branch, 9 children: nibbles 0-8)
  [8, 3, 0, 1, 0]  (branch, 16 children)
    [8, 3, 0, 1, 0, 0]  (branch, 16 children)
      [8, 3, 0, 1, 0, 0, 0]  (branch, 16 children)
        [8, 3, 0, 1, 0, 0, 0, 0]  = index 65536 (leaf)
```

**To verify the first batch in `[8, 3]`** (e.g., 16 leaves at `[8, 3, 0, 1, 0, 0, 0, _]`):

| Level | Node | Sibling hashes needed |
|---|---|---|
| `[8, 3, 0, 1]` | Branch with 9 children (nibbles 0-8) | 8 |
| `[8, 3, 0, 1, 0]` | Branch with 16 children | 15 |
| `[8, 3, 0, 1, 0, 0]` | Branch with 16 children | 15 |
| `[8, 3, 0, 1, 0, 0, 0]` | Branch with 16 children | 15 |

**Total: 53 sibling hashes = 1,696 bytes** just to verify 16 transactions.

This is the **overhead spike**: crossing the 65536 boundary forces sending hashes at 4 new branch levels simultaneously. Compare this to streaming within `[8, 2]` where the deepest batch requires hashes at only 2 levels below `[8, 2]`.

The root cause: RLP's variable-length encoding adds 2 nibbles of depth per power-of-256, and all these extra levels pile up under the single nibble `[8]`. The trie grows DEEP rather than WIDE.

---

## 9. Hex-Indexed Trie with BFS Streaming

### 9.1 Key Encoding

Replace RLP-encoded keys with **minimal hex representation**:

```
Index 0   → key: (empty)        → depth 0 (root value)
Index 1   → key: 0x1   = [1]    → depth 1
Index 15  → key: 0xf   = [f]    → depth 1
Index 16  → key: 0x10  = [1, 0] → depth 2
Index 255 → key: 0xff  = [f, f] → depth 2
Index 256 → key: 0x100 = [1, 0, 0] → depth 3
Index 4095  → key: 0xfff  = [f, f, f]    → depth 3
Index 4096  → key: 0x1000 = [1, 0, 0, 0] → depth 4
Index 65535 → key: 0xffff = [f, f, f, f]  → depth 4
Index 65536 → key: 0x10000 = [1, 0, 0, 0, 0] → depth 5
```

**Depth of index i:** `ceil(log16(i))` for i >= 1, and 0 for i = 0.

**Critical property:** Nibble `0` is never a leading nibble (no leading zeros in hex). So the root branch has at most **15 populated children** (nibbles 1-f). At depth 2+, all 16 nibbles can be populated (e.g., index 16 = `[1, 0]` uses nibble 0 as second nibble).

### 9.2 Node Structure

Every node in the trie is a uniform **branch-with-value node**:

```
Node = [value, branch_0, branch_1, ..., branch_f]
     = 17 × 32 bytes = 544 bytes
```

- **value**: The hash of the data at this key (32 bytes). Zero if no data at this key.
- **branch_0 .. branch_f**: Hashes of the 16 child nodes (32 bytes each). Zero if child doesn't exist.
- **Node hash**: `keccak256(Node)` -- the hash of the full 544-byte serialization.

The trie root hash = hash of the root node.

### 9.3 Trie Layout Example (N = 300)

```
Root (depth 0):
  value = hash(tx[0])
  branches 1-f → depth-1 nodes (branch 0 always empty at root)

Node [1] (depth 1):
  value = hash(tx[1])
  branches 0-f → nodes [1,0]..[1,f] (indices 16-31)

Node [1, 0] (depth 2):
  value = hash(tx[16])
  branches 0-f → nodes [1,0,0]..[1,0,f] (indices 256-271)

Node [1, 0, 0] (depth 3):
  value = hash(tx[256])
  branches: all zero (no indices 4096+ under this path for N=300)
```

Level population for N = 300:

| Level | Indices | Count | Nodes |
|---|---|---|---|
| 0 | 0 | 1 | Root |
| 1 | 1-15 | 15 | Nodes [1]..[f] |
| 2 | 16-255 | 240 | Nodes [1,0]..[f,f] |
| 3 | 256-299 | 44 | Nodes [1,0,0]..[2,b,3] |

### 9.4 BFS Streaming Protocol

Stream one level at a time. At each level, the branch hashes received from the **previous** level serve as the expected hashes for verification.

**Level 0 — Root node:**

```
Send: tx[0] data + 15 branch hashes (branches 1-f; branch 0 is always empty)
Receiver: computes value = hash(tx[0]), reconstructs root node, verifies hash == known root
Result: receiver now holds expected hashes for all 15 depth-1 nodes
Overhead: 15 hashes (480 bytes)
```

**Level 1 — Nodes at depth 1 (indices 1-15):**

```
For each nibble k in 1..f:
  Send: tx[k] data + 16 branch hashes of node [k]
  Receiver: computes value = hash(tx[k]), reconstructs node [k], verifies hash == parent's branch[k]
  Result: receiver now holds expected hashes for 16 depth-2 children of [k]
Overhead: 15 nodes × 16 branches = 240 hashes (7,680 bytes)
  (Many branches may be zero for small N — only non-zero ones carry information)
```

**Level 2 — Nodes at depth 2 (indices 16-255):**

```
For each populated depth-2 position [k, j]:
  Send: tx[k*16 + j] data + 16 branch hashes
  Receiver: verifies node hash against parent's branch[j] from level 1
Overhead: up to 240 nodes × 16 branches = 3,840 hashes
  (Again, many zeros for small N)
```

**General level d:**

```
For each populated node at depth d:
  Send: tx data + 16 branch hashes (or fewer if node is a leaf with all-zero branches)
  Receiver: verifies against expected hash from parent at depth d-1
```

### 9.5 Key Insight: Branch Hashes Serve Double Duty

The branch hashes sent at level d are **simultaneously**:

1. **Verification data** for the level-d node (receiver needs them to reconstruct the node hash and verify against the parent's expected hash)
2. **Expected hashes** for verifying level d+1 nodes

There is no separate "proof overhead". The trie's structural data IS the streaming proof. Every hash sent is both necessary for verification AND useful for the next level.

### 9.6 Per-Level Overhead (Non-Leaf Branches Only)

For leaf nodes (no children), all 16 branches are zero. The receiver knows this (it knows N, so it knows which nodes are leaves). The overhead for leaf nodes is **zero** -- the receiver reconstructs the node from the tx data alone and verifies it.

The only overhead is the **non-zero branch hashes in internal (non-leaf) nodes**:

| Level | Internal nodes | Max branches per node | Level overhead |
|---|---|---|---|
| 0 | 1 (root) | 15 (nibble 0 always empty) | 15 hashes |
| 1 | Up to 15 | 16 | Up to 240 hashes |
| 2 | Up to 240 | 16 | Up to 3,840 hashes |
| d | Up to 15 × 16^(d-1) | 16 | Up to 15 × 16^d hashes |

But the actual overhead is much less because only nodes with children contribute. The total across all levels equals the total number of parent-child edges in the trie = **N - 1 hashes** (same fundamental limit as Section 3.1).

### 9.7 Comparison at the 65536 Boundary

**RLP encoding (N = 100,000):**

Streaming in trie order, upon entering the `[8, 3]` subtree, the first verifiable batch of 16 transactions requires **53 sibling hashes (1,696 bytes)** sent all at once — a spike caused by 4 new branch levels appearing simultaneously.

**Hex-indexed BFS (N = 100,000):**

Index 65536 = key `[1, 0, 0, 0, 0]` at depth 5. In BFS, its ancestor nodes were sent at earlier levels:

| BFS level | Node sent | Verification |
|---|---|---|
| 0 | Root: includes branch[1] hash | Verified against root hash |
| 1 | Node [1]: includes branch[0] hash | Verified against root's branch[1] |
| 2 | Node [1,0]: includes branch[0] hash | Verified against [1]'s branch[0] |
| 3 | Node [1,0,0]: includes branch[0] hash | Verified against [1,0]'s branch[0] |
| 4 | Node [1,0,0,0]: includes branch[0] hash | Verified against [1,0,0]'s branch[0] |
| 5 | Node [1,0,0,0,0]: leaf, tx[65536] | Verified against [1,0,0,0]'s branch[0] |

**Zero overhead spike.** The verification chain was built incrementally across 5 BFS levels. Each level added exactly 1 hash to the chain (the branch[0] slot in each ancestor). These hashes were amortized within the 15-16 branch hashes already sent for each ancestor node.

**The boundary between depth 4 (index 65535 = `[f, f, f, f]`) and depth 5 (index 65536 = `[1, 0, 0, 0, 0]`) is invisible in BFS streaming.** Level 4 simply processes all depth-4 nodes, and level 5 processes the new depth-5 nodes. No spike.

### 9.8 Concrete Overhead Comparison (N = 100,000)

**Hex-indexed BFS streaming:**

Level-by-level overhead (branch hashes of internal nodes only):

| Level | Indices at this level | Internal nodes (have children) | Branch hashes sent | Cumulative txs verified |
|---|---|---|---|---|
| 0 | 0 | 1 (root) | 15 | 1 |
| 1 | 1-15 | 15 | 15 × 16 = 240 | 16 |
| 2 | 16-255 | 240 | 240 × 16 = 3,840 | 256 |
| 3 | 256-4095 | 3,840 | 3,840 × 16 = 61,440 | 4,096 |
| 4 | 4096-65535 | ~6,090* | ~6,090 × 16 = 97,440* | 65,536 |
| 5 | 65536-99999 | 0 (all leaves) | 0 | 100,000 |

*Level 4 internal node count: only nodes that have children at level 5. Indices 65536-99999 = 34,464 level-5 leaves. These sit under ceil(34464/16) = 2,154 level-4 nodes that need non-zero branches. The remaining ~4,382 level-4 nodes are leaves (their index is in range but they have no deeper children). So actual branch hashes at level 4: ~2,154 × (avg populated branches) ≈ 34,464 hashes.

**Corrected level 4:** Only the branch hashes pointing to actual level-5 nodes count as overhead. That's exactly 34,464 hashes (one per level-5 leaf), distributed across 2,154 internal level-4 nodes.

**Total overhead: 15 + 240 + 3,840 + 34,464 ≈ 38,559 hashes** for non-leaf branches. But most of these are **zero** (empty branch slots). The actual non-zero overhead = N - 1 = 99,999 hashes = 3.2 MB.

Wait -- this reveals something important. The N-1 overhead is the same regardless of trie structure. The advantage of BFS isn't in total overhead reduction, it's in **distribution**: no spike, smooth per-level cost, and each level is independently verifiable.

### 9.9 The Real Advantage: Verifiable After Each Level

The total overhead (N-1 hashes) is a **fundamental lower bound** that no trie structure can beat. But the hex-indexed BFS trie has unique advantages in HOW that overhead is distributed:

1. **Smooth distribution.** Each level adds overhead proportional to the number of nodes at that level. No spikes at boundaries.

2. **Progressive verification.** After receiving level d, ALL indices at depths 0..d are verified. The receiver has a firm verification frontier that advances uniformly.

3. **Natural batching.** Each level IS a batch. Level sizes grow geometrically (1, 15, 240, 3840, ...) providing a natural ramp-up.

4. **Branch hashes are not wasted.** Every branch hash sent at level d is either:
   - Zero (known to receiver, no overhead) -- for empty branches
   - The expected hash of a level d+1 node (reused for next-level verification)

   No hash is sent purely as a "sibling proof" that serves no other purpose.

### 9.10 Relaxing the BFS Order

**What if we don't stream in strict BFS order?**

For BFS verification, we need parent nodes before children. But we have flexibility:

**Within a level, order doesn't matter.** All nodes at depth d can be verified independently against their parent's branch hash (received at depth d-1). So within each level, transactions can arrive in any order.

**DFS is also viable.** Send the root first. Then for any subtree, send nodes depth-first. Each node is verified immediately upon receipt because its parent was already processed. DFS lets you "complete" subtrees early rather than leaving all verification open until the last level.

**Skeleton-first approach for full flexibility:**

1. Send all internal nodes top-down (BFS or DFS) — this is the "skeleton" of the trie
2. Then send leaf data in ANY order — each leaf is independently verifiable

Skeleton size for N = 100,000:
- Internal nodes ≈ N/15 ≈ 6,667 nodes × 544 bytes ≈ 3.5 MB
- After skeleton: any of the ~100,000 leaves can be verified independently in any order

**Partial skeleton for earlier leaf verification:**

Send only the top K levels of the skeleton, creating independently verifiable subtrees:

| Skeleton depth | Skeleton size | Number of independent subtrees | Leaves per subtree |
|---|---|---|---|
| 1 | 544 bytes | 15 | ~6,667 |
| 2 | 544 + 15×544 = 8.7 KB | 240 | ~417 |
| 3 | ~131 KB | 3,840 | ~26 |
| 4 | ~2 MB | ~61,000 | ~1-2 |

A 2-level skeleton (8.7 KB) gives 240 subtrees that can be streamed and verified independently, in any order, with no further proof overhead within each subtree. This may be the best practical tradeoff for unordered streaming.

### 9.11 Optimal Combined Strategy for Hex-Indexed Trie

**For ordered BFS streaming (most efficient):**

```
Stream level 0: root (1 tx + 15 hashes = ~480 + tx_size bytes)
  → All indices at depth 0 verified
Stream level 1: 15 txs + up to 240 hashes
  → All indices at depths 0-1 verified (16 total)
Stream level 2: 240 txs + up to 3840 hashes
  → All indices at depths 0-2 verified (256 total)
...continue until all levels sent...
```

Verification frontier advances one full depth level per batch. No boundary overhead. Smooth throughput.

**For unordered streaming (maximum flexibility):**

```
Phase 1: Send 2-level skeleton (~8.7 KB)
  → 240 independently verifiable subtrees established
Phase 2: Stream subtrees in any order, leaves in any order within subtrees
  → Each leaf verified immediately on receipt
```

Total overhead: skeleton + leaf verification within subtrees. Skeleton is a one-time cost. Within each subtree, leaves at the same depth need no proof (the skeleton already provided expected hashes down to depth 2, and within-subtree nodes can be sent in any parent-before-child order).

---

## 10. Summary of Key Results (Updated)

1. **MPT overhead theorem:** Streaming N transactions with per-leaf verification costs exactly N-1 sibling hashes. This is a fundamental lower bound independent of trie structure.

2. **RLP boundary problem:** At powers of 256, RLP encoding causes depth jumps of +2 nibbles. At 65536, this creates an overhead spike of ~53 hashes (1,696 bytes) for the first verification in the new depth range. All deep keys pile up under nibble `[8]`.

3. **Hex-indexed trie with BFS eliminates boundary spikes.** Keys use minimal hex representation (index 0 = empty, 16 = `[1,0]`, 65536 = `[1,0,0,0,0]`). BFS streaming sends one level at a time. Branch hashes serve double duty as node verification AND next-level expected hashes. The 65536 boundary is invisible — just another BFS level.

4. **Hash-linked list** is simplest for pure ordered streaming with same per-item overhead.

5. **Batching is the key optimization** for all structures. The hex-indexed trie's natural batch boundaries (BFS levels) grow geometrically: 1, 15, 240, 3840, ... providing a smooth ramp-up.

6. **For unordered streaming**, a 2-level skeleton (~8.7 KB for N=100K) establishes 240 independently verifiable subtrees, enabling fully flexible leaf delivery order.

7. **The optimal structure depends on requirements:**

| Requirement | Best Structure | Overhead |
|---|---|---|
| Ordered streaming, no spikes | Hex-indexed BFS | N-1 hashes, smooth |
| Ordered streaming, simplest code | Hash-linked list | N-1 hashes, smooth |
| Unordered streaming + random access | Hex-indexed + skeleton | Skeleton + N-1 hashes |
| Ethereum compatibility | RLP MPT, trie-ordered | ~256 bytes (small N), spikes at boundaries |
