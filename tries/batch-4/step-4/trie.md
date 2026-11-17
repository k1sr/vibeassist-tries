# Tries and Radix Trees: A Worked Explainer

A **trie** (also called a *prefix tree*, and usually pronounced "try") is a tree that stores strings by spelling them out one character per edge. A **radix tree** (also called a *compressed trie* or *Patricia tree*) is the same idea, but with every chain of single-child nodes merged into one edge labelled with a whole substring.

Both structures answer one question well: *which stored keys start with this prefix?* A hash table can tell you whether `"cart"` is present, but it cannot list every key beginning with `"ca"` without scanning all of them. A balanced binary search tree can, but each comparison costs a full string comparison. A trie answers prefix questions by walking down the tree one character at a time, so the work depends on the length of the prefix, not on how many keys are stored.

This document builds both structures step by step with concrete words, then covers searching, deletion, compression, autocomplete, and the memory trade-offs that decide which variant to use in practice.

---

## 1. The core idea

Every node in a trie represents a **prefix**. The root is the empty prefix `""`. Each child edge adds one character, so the node reached by following `c → a → r` represents the prefix `"car"`.

Two facts follow:

1. **Shared prefixes share nodes.** `"car"` and `"cat"` share the nodes for `""`, `"c"`, and `"ca"`; they split only at the third character.
2. **A node is not automatically a word.** The node for `"ca"` exists because `"car"` passes through it, but `"ca"` was never inserted. Each node therefore carries a flag, here written `is_word`, that marks where a stored key ends. In diagrams below, a `*` marks such a node.

The flag matters as soon as one key is a prefix of another. If you store both `"car"` and `"cart"`, the node for `"car"` sits in the middle of the path to `"cart"`, and only the flag tells you that `"car"` is itself a key.

---

## 2. Inserting words, one at a time

We will insert five words in this order: `car`, `cat`, `cart`, `care`, `dog`.

### Insert `car`

The trie is empty except for the root. No child exists for `c`, so we create one; then `a`; then `r`. The final node gets the end-of-word flag.

```
(root)
 └── c
     └── a
         └── r*
```

Nodes created: 3.

### Insert `cat`

We walk from the root: `c` exists, so we follow it. `a` exists, so we follow it. `t` does not exist under `"ca"`, so we create it and mark it.

```
(root)
 └── c
     └── a
         ├── r*
         └── t*
```

Nodes created: 1. The first two characters cost nothing because the path was already there. This is where tries save space and time on data with common prefixes.

### Insert `cart`

Walk `c`, `a`, `r`: all exist. Create `t` under `"car"` and mark it. Note that `"car"` stays marked; it is still a word on its own.

```
(root)
 └── c
     └── a
         ├── r*
         │   └── t*
         └── t*
```

There are now two different nodes labelled `t`. That is fine: a node is identified by its *path* (`"cart"` versus `"cat"`), not by its character.

### Insert `care`

Walk `c`, `a`, `r`. Create `e` under `"car"`.

```
(root)
 └── c
     └── a
         ├── r*
         │   ├── t*
         │   └── e*
         └── t*
```

### Insert `dog`

Nothing under the root starts with `d`, so the whole word becomes a fresh branch.

```
(root)
 ├── c
 │   └── a
 │       ├── r*
 │       │   ├── t*
 │       │   └── e*
 │       └── t*
 └── d
     └── o
         └── g*
```

The finished trie stores 5 words using 10 nodes (including the root). Storing the same words naively would need 3 + 3 + 4 + 4 + 3 = 17 characters; the trie needs 9 non-root nodes because `car`, `cat`, `cart` and `care` share their opening letters.

**Cost of insertion:** O(L), where L is the length of the word. Each character is one dictionary lookup and, at most, one node creation. The number of words already stored does not appear in the bound.

---

## 3. Exact search and prefix search

Both searches use the same walk: start at the root and follow one edge per character. They differ only in what they check at the end.

### Exact search: is `cart` a word?

```
root --c--> "c" --a--> "ca" --r--> "car" --t--> "cart"*
```

Every edge exists, and the final node is marked. **Yes.**

### Exact search: is `ca` a word?

```
root --c--> "c" --a--> "ca"
```

Every edge exists, but the `"ca"` node is not marked. **No.** The prefix is present only as part of longer words.

### Exact search: is `cow` a word?

```
root --c--> "c" --o--> (no such edge)
```

The walk falls off the tree at `o`. **No**, and we know that after two steps, without looking at any other key.

### Prefix search: does any word start with `ca`?

Walk to `"ca"`. The node exists, so **yes**. A prefix query does not care about the `is_word` flag: any node on the tree is the prefix of at least one stored word, because nodes only exist when some word was inserted through them (and, as Section 4 shows, deletion removes nodes that no longer lead anywhere).

The prefix walk is the foundation of autocomplete (Section 6): once you are standing at the `"ca"` node, every word in the subtree below it is a completion.

**Cost:** O(L) for a prefix of length L, regardless of the number of stored keys. For a dictionary of a million words, checking whether `"zyzzyv"` is a valid prefix touches at most six nodes.

---

## 4. Deletion

Deletion has two cases, and the tricky part is knowing when to stop removing nodes.

1. **Unmark** the node for the word. If the word is not present (the path breaks, or the final node is not marked), there is nothing to do.
2. **Prune** upward: if that node now has no children and is not a word, it serves no purpose, so remove it from its parent. Repeat with the parent. Stop at the first node that is still a word or still has other children.

Starting from the five-word trie, consider three deletions.

### Delete `cart` (a leaf)

Unmark `"cart"`. It has no children and is no longer a word, so remove it. Move up to `"car"`: it is still a word (and also still has the child `e`), so stop.

```
(root)
 ├── c
 │   └── a
 │       ├── r*
 │       │   └── e*
 │       └── t*
 └── d
     └── o
         └── g*
```

### Delete `car` (an interior word)

Unmark `"car"`. It still has the child `e` (leading to `"care"`), so nothing is pruned. The node stays, just without its flag.

```
(root)
 ├── c
 │   └── a
 │       ├── r
 │       │   └── e*
 │       └── t*
 └── d
     └── o
         └── g*
```

This is the most common mistake in hand-written tries: removing `"car"`'s node here would silently delete `"care"` too.

### Delete `dog` (a whole branch)

Unmark `"dog"`. Remove `g` (leaf, not a word). Move to `"do"`: no children left, not a word, remove it. Move to `"d"`: same, remove it. Move to the root: the root is never removed. The entire `d` branch is gone.

```
(root)
 └── c
     └── a
         ├── r
         │   └── e*
         └── t*
```

Some implementations skip pruning and only unmark ("lazy deletion"). That is simpler and keeps deletion strictly O(L), but dead branches accumulate, and prefix queries can then report a prefix as present when no word below it exists. If you use lazy deletion, prefix search must check that the subtree contains at least one marked node, which defeats the purpose. Pruning is usually worth it.

**Cost:** O(L). The walk down is L steps and the prune back up is at most L steps.

---

## 5. Compressing into a radix tree

Look again at the original five-word trie and notice the chains of nodes with exactly one child that are not words: `c → a`, and `d → o → g`. A traversal must step through each one even though no decision happens there. A **radix tree** collapses each such chain into a single edge labelled with a string.

The compression rule: **any node that is not a word and has exactly one child is merged with that child**, concatenating their labels. Apply it until no such node remains.

### From trie to radix tree

The original trie:

```
(root)
 ├── c
 │   └── a
 │       ├── r*
 │       │   ├── t*
 │       │   └── e*
 │       └── t*
 └── d
     └── o
         └── g*
```

- `c` has one child (`a`) and is not a word: merge into `ca`.
- `ca` has two children (`r` and `t`): stop.
- `r` is a word: stop (it must remain a distinct node so it can carry the flag).
- `d` → `o` → `g`: `d` and `o` are single-child non-words, so the chain becomes `dog`.

```
(root)
 ├── "ca"
 │    ├── "r"*
 │    │    ├── "t"*
 │    │    └── "e"*
 │    └── "t"*
 └── "dog"*
```

Ten nodes became seven. On this tiny example the savings look modest, but on real data with long unique suffixes (URLs, file paths, IP routing prefixes, long identifiers) most of a plain trie consists of single-child chains, and compression removes them all.

A useful guarantee: in a radix tree with *n* stored keys, every internal node either marks a word or has at least two children. So the tree has at most about 2*n* nodes, **independent of key length**. A plain trie can have as many nodes as the total number of characters across all keys.

### Inserting into a radix tree: splitting edges

Insertion is where radix trees get interesting, because a new key can diverge from an existing edge *partway through its label*. The edge must then be split. Let us build a radix tree from scratch with the words `test`, `team`, `toast`, `te`.

**Insert `test`.** The root has no child starting with `t`, so add a single edge carrying the whole word.

```
(root)
 └── "test"*
```

**Insert `team`.** The root has a child starting with `t`: the edge `"test"`. Compute the longest common prefix of `"team"` and `"test"`: it is `"te"` (length 2), shorter than the edge label. So split the edge:

- Create a new intermediate node with label `"te"`.
- The old node keeps the remainder of its label, `"st"`, and becomes a child of `"te"`.
- The rest of the new word, `"am"`, becomes a new leaf under `"te"`.

```
(root)
 └── "te"
      ├── "st"*
      └── "am"*
```

**Insert `toast`.** The root's `t` child is `"te"`. The common prefix of `"toast"` and `"te"` is just `"t"`, so split again: `"te"` becomes `"t"` with child `"e"`, and `"oast"` becomes a sibling of `"e"`.

```
(root)
 └── "t"
      ├── "e"
      │    ├── "st"*
      │    └── "am"*
      └── "oast"*
```

**Insert `te`.** Walk `"t"` (fully matched, `"e"` remains), then `"e"` (fully matched, nothing remains). We end exactly at an existing node, so we only set its flag. No new nodes.

```
(root)
 └── "t"
      ├── "e"*
      │    ├── "st"*
      │    └── "am"*
      └── "oast"*
```

Every insertion falls into one of four cases:

| Situation at the current node | Action |
|---|---|
| No child shares the first character | Add a new leaf with the entire remaining key |
| Child label is a full prefix of the remaining key | Descend into the child, consume its label, repeat |
| Remaining key and child label diverge partway | Split the child's edge at the divergence point, then continue |
| Remaining key is empty | Mark the current node as a word |

### Deleting from a radix tree: re-merging

Deletion follows the trie rules (unmark, then prune empty leaves) with one extra step: after removing a node, its parent may be left with a single child and no word flag, which violates the compression rule. Merge it back.

Take the compressed five-word tree and delete `cat`:

```
Before:                         After removing "t":         After merging:
(root)                          (root)                      (root)
 ├── "ca"                        ├── "ca"                    ├── "car"*
 │    ├── "r"*                   │    └── "r"*               │    ├── "t"*
 │    │    ├── "t"*              │         ├── "t"*          │    └── "e"*
 │    │    └── "e"*              │         └── "e"*          └── "dog"*
 │    └── "t"*                   └── "dog"*
 └── "dog"*
```

`"ca"` was left as a non-word with one child, so it was fused with `"r"` into `"car"`. The tree is again in compressed form.

Note the symmetric case: deleting `car` from the compressed tree just unmarks the `"r"` node. It keeps two children (`t` and `e`), so no merge is needed.

---

## 6. Autocomplete

Autocomplete is the canonical trie application. Given a typed prefix, return the stored words that begin with it.

The algorithm has two phases:

1. **Locate.** Walk the prefix from the root. If the walk falls off the tree, there are no completions.
2. **Collect.** Do a depth-first traversal of the subtree under the located node, building up the string as you go, and emit every node that is marked as a word.

Using the five-word trie and the prefix `"car"`:

```
Locate:  root -> c -> a -> r      (at node "car")
Collect: "car"*   emit "car"
         └ "care"*  emit "care"
         └ "cart"*  emit "cart"
```

If children are visited in sorted order, the results come out in lexicographic order for free: `car, care, cart`. That is another advantage over a hash table, which has no notion of order.

With the prefix `"ca"`, the collection also reaches the `t` branch, yielding `car, care, cart, cat`. With `"d"`, it yields `dog`. With `"cow"`, the locate phase fails at `o` and the result is empty.

### Making it fast enough for a real search box

The cost of collection is proportional to the size of the subtree, which can be huge for short prefixes: typing `"s"` into an English dictionary matches tens of thousands of words. Real systems use some combination of:

- **A result limit.** Stop the traversal after the first *k* matches. With sorted traversal this gives the *k* alphabetically first completions.
- **Ranking by weight.** Store a frequency or score at each word node. To return the top *k* by score, run a best-first search with a priority queue, or precompute.
- **Top-k caching per node.** Store, at every node, the *k* highest-scoring words in its subtree. A query is then O(L) to locate plus O(*k*) to read the cached list. Updates cost more (every ancestor's cache may change), which is acceptable when queries vastly outnumber writes, as they usually do in search suggestions.
- **Radix compression.** Fewer nodes means fewer pointer hops during collection. Locating a prefix in a radix tree needs one extra case: the prefix may end *in the middle of an edge* (typing `"tea"` stops partway through `"am"` in the `test/team` tree). In that case every word under that edge's child is a match.

---

## 7. Code

The following Python is complete and runnable. It favours clarity over raw speed; Section 8 discusses faster layouts.

### A plain trie

```python
class TrieNode:
    __slots__ = ("children", "is_word")

    def __init__(self):
        self.children = {}      # char -> TrieNode
        self.is_word = False


class Trie:
    def __init__(self):
        self.root = TrieNode()

    def insert(self, word):
        node = self.root
        for ch in word:
            node = node.children.setdefault(ch, TrieNode())
        node.is_word = True

    def _find(self, prefix):
        """Return the node for `prefix`, or None if the path breaks."""
        node = self.root
        for ch in prefix:
            node = node.children.get(ch)
            if node is None:
                return None
        return node

    def contains(self, word):
        node = self._find(word)
        return node is not None and node.is_word

    def has_prefix(self, prefix):
        return self._find(prefix) is not None

    def delete(self, word):
        """Remove `word`; prune nodes that no longer lead to any word."""
        path = [self.root]
        node = self.root
        for ch in word:
            node = node.children.get(ch)
            if node is None:
                return False
            path.append(node)
        if not node.is_word:
            return False
        node.is_word = False
        # Walk back up, removing childless non-word nodes.
        for i in range(len(word) - 1, -1, -1):
            child = path[i + 1]
            if child.is_word or child.children:
                break
            del path[i].children[word[i]]
        return True

    def complete(self, prefix, limit=10):
        """Return up to `limit` words starting with `prefix`, in sorted order."""
        start = self._find(prefix)
        if start is None:
            return []
        out = []

        def dfs(node, buf):
            if node.is_word:
                out.append("".join(buf))
            for ch in sorted(node.children):
                if len(out) >= limit:
                    return
                buf.append(ch)
                dfs(node.children[ch], buf)
                buf.pop()

        dfs(start, list(prefix))
        return out[:limit]
```

Exercising it with the running example:

```python
t = Trie()
for w in ["car", "cat", "cart", "care", "dog"]:
    t.insert(w)

t.contains("cart")     # True
t.contains("ca")       # False
t.has_prefix("ca")     # True
t.complete("ca")       # ['car', 'care', 'cart', 'cat']
t.delete("car")        # True
t.complete("car")      # ['care', 'cart']   ("car" unmarked, branch kept)
t.delete("dog")
t.has_prefix("d")      # False              (whole branch pruned)
```

### A radix tree

Each node stores the label of the edge leading into it. Children are keyed by the first character of their label; since sibling labels never share a first character (otherwise they would have been merged into a common parent), this lookup is unambiguous.

```python
def common_prefix_len(a, b):
    n = min(len(a), len(b))
    i = 0
    while i < n and a[i] == b[i]:
        i += 1
    return i


class RadixNode:
    __slots__ = ("label", "children", "is_word")

    def __init__(self, label=""):
        self.label = label
        self.children = {}      # first char of child label -> RadixNode
        self.is_word = False


class RadixTree:
    def __init__(self):
        self.root = RadixNode()

    def insert(self, word):
        node = self.root
        while True:
            if not word:
                node.is_word = True
                return
            child = node.children.get(word[0])
            if child is None:
                leaf = RadixNode(word)
                leaf.is_word = True
                node.children[word[0]] = leaf
                return
            k = common_prefix_len(word, child.label)
            if k < len(child.label):
                # Split the edge: node -> mid("label[:k]") -> child("label[k:]")
                mid = RadixNode(child.label[:k])
                child.label = child.label[k:]
                mid.children[child.label[0]] = child
                node.children[word[0]] = mid
                child = mid
            node = child
            word = word[k:]

    def contains(self, word):
        node = self.root
        while word:
            child = node.children.get(word[0])
            if child is None or not word.startswith(child.label):
                return False
            word = word[len(child.label):]
            node = child
        return node.is_word

    def delete(self, word):
        path = []                       # (parent, child) pairs along the walk
        node = self.root
        while word:
            child = node.children.get(word[0])
            if child is None or not word.startswith(child.label):
                return False
            path.append((node, child))
            word = word[len(child.label):]
            node = child
        if not node.is_word:
            return False
        node.is_word = False
        if not path:                    # deleted the empty string at the root
            return True
        parent = path[-1][0]
        if not node.children:
            del parent.children[node.label[0]]
            if len(path) >= 2:          # parent may now need re-merging
                self._merge(path[-2][0], parent)
        else:
            self._merge(parent, node)
        return True

    def _merge(self, parent, node):
        """Fuse `node` with its only child if `node` is a non-word."""
        if node.is_word or len(node.children) != 1:
            return
        (only,) = node.children.values()
        only.label = node.label + only.label
        parent.children[only.label[0]] = only

    def complete(self, prefix, limit=10):
        node, acc = self.root, ""
        while prefix:
            child = node.children.get(prefix[0])
            if child is None:
                return []
            if prefix.startswith(child.label):      # consume whole edge
                prefix = prefix[len(child.label):]
            elif child.label.startswith(prefix):    # prefix ends mid-edge
                prefix = ""
            else:                                   # diverges mid-edge
                return []
            acc += child.label
            node = child
        out = []

        def dfs(n, s):
            if n.is_word:
                out.append(s)
            for key in sorted(n.children):
                if len(out) >= limit:
                    return
                c = n.children[key]
                dfs(c, s + c.label)

        dfs(node, acc)
        return out[:limit]
```

Replaying the examples from Section 5:

```python
r = RadixTree()
for w in ["test", "team", "toast", "te"]:
    r.insert(w)
r.complete("te")       # ['te', 'team', 'test']
r.complete("tea")      # ['team']            (prefix ends mid-edge "am")
r.contains("tes")      # False

r = RadixTree()
for w in ["car", "cat", "cart", "care", "dog"]:
    r.insert(w)
r.delete("cat")
r.root.children["c"].label   # 'car'   ("ca" + "r" re-merged)
```

---

## 8. Complexity and memory trade-offs

### Time

| Operation | Trie | Radix tree | Hash set | Balanced BST |
|---|---|---|---|---|
| Insert / lookup / delete | O(L) | O(L) | O(L) expected (hashing) | O(L log n) worst case |
| Prefix exists? | O(L) | O(L) | O(n·L) scan | O(L log n) |
| List *k* completions | O(L + subtree) | O(L + subtree) | O(n·L) scan | O(L log n + k·L) |
| Ordered iteration | Yes | Yes | No | Yes |

L is key length, n is the number of keys. The trie bounds hold no matter how many keys are stored, which is why tries shine for large dictionaries with prefix queries. For exact-match-only workloads, a hash table is usually faster in practice because it touches one or two cache lines rather than L scattered nodes.

### Space: where the memory goes

A trie's memory is dominated by **per-node overhead**, not by the characters themselves. Every node needs a child container and a flag, and in a language like Python each node is a separate heap object with a dictionary. Roughly:

- **Plain trie, dict children (Python):** easily 100 to 200+ bytes per node. A 100,000-word English dictionary can produce on the order of 250,000 nodes, which means tens of megabytes to hold what is a ~1 MB word list.
- **Plain trie, fixed array of 26 child pointers (C, lowercase ASCII):** 26 × 8 = 208 bytes of pointers per node, almost all of them null near the leaves. Lookup is a single array index, which is very fast, but the waste is large, and it does not extend to Unicode.
- **Radix tree:** at most about 2n nodes for n keys, regardless of key length, plus the label strings. For long keys with unique tails (URLs, paths, UUID-like IDs) this is often an order of magnitude fewer nodes than a plain trie.

The general tension is between **fast child lookup** (big arrays, wasteful) and **compact children** (lists, maps, or sorted arrays, slower to search). Several well-known designs attack it:

- **Adaptive Radix Tree (ART):** a radix tree whose nodes change representation with their fan-out, using node types sized for 4, 16, 48, or 256 children. Sparse nodes stay small, dense nodes get direct indexing. ART is used as an in-memory index in database systems.
- **Ternary search tree:** each node holds one character and three pointers (less than, equal, greater than). Far less memory than array tries, at the cost of extra comparisons per character.
- **Double-array trie:** packs the whole trie into two integer arrays, giving array-speed transitions with compact storage. Common in morphological analysers and input methods, especially for CJK text. Updates are awkward, so it suits mostly-static dictionaries.
- **DAWG / minimal acyclic automaton and FSTs:** a trie shares *prefixes*; a DAWG also shares *suffixes* (`-ing`, `-tion`) by merging identical subtrees into a graph. Finite-state transducers extend this to store a value per key. Lucene uses FSTs for its term dictionary. These are excellent for static sets and hard to update.
- **Burst tries / HAT-tries:** use trie nodes near the top, where branching is high, and switch to small cache-friendly arrays or hash buckets below a threshold, combining the trie's prefix navigation with a hash table's cache efficiency.

### Choosing between them

- Use a **plain trie** for teaching, prototypes, small alphabets, or when simplicity matters more than memory. It is the easiest to get right.
- Use a **radix tree** when keys are long or share long substrings, when memory matters, or when keys are bytes rather than characters. It is the standard choice for HTTP routers (matching `/users/:id/posts`), IP routing tables (longest-prefix match on address bits), and in-memory key indexes.
- Use a **hash table** when you only need exact lookups and never prefixes or ordering.
- Use a **DAWG, FST, or double-array trie** for large, read-mostly dictionaries such as spell checkers, search-engine term lists, and on-device keyboards, where you build once and query millions of times.

---

## 9. Pitfalls and practical notes

- **Forgetting the end-of-word flag.** Without it, inserting `"cart"` makes `"car"` look present. Every implementation needs to distinguish "this prefix exists" from "this word exists."
- **Over-pruning on delete.** Only remove nodes that are neither words nor parents. Deleting `"car"` must not take `"care"` with it.
- **Character encoding.** Decide what an edge represents: a byte, a UTF-16 code unit, or a Unicode code point. Mixing them corrupts prefix matches for non-ASCII text. Normalize input (case folding, Unicode NFC) before inserting if the application treats `"Café"` and `"café"` as the same word.
- **Recursion depth.** Recursive traversal on very long keys can overflow the stack in some languages. The iterative versions of insert, search, and delete above avoid this; collection can be made iterative with an explicit stack.
- **Radix split correctness.** When splitting an edge, the existing child keeps its `is_word` flag and its children; only the new intermediate node starts unmarked (unless the inserted key ends exactly at the split point, in which case the next loop iteration marks it). Off-by-one errors in the split index are the most common radix-tree bug, so test with keys that are prefixes of each other in both insertion orders (`te` then `test`, and `test` then `te`).

---

## 10. Summary

A trie stores strings as paths, one character per edge, so shared prefixes share storage and every prefix query costs time proportional to the prefix length alone. Insertion walks and creates nodes; search walks and checks a flag; deletion unmarks and prunes back up to the first node still in use; autocomplete locates the prefix node and collects its subtree, in sorted order if children are visited in order.

A radix tree applies one rule on top of that: collapse every chain of single-child, non-word nodes into one labelled edge. That bounds the node count by the number of keys rather than their total length, at the cost of edge splitting on insert and edge merging on delete. Between them, and the specialised variants built on them, they cover most problems where you need to ask what a set of strings starts with.
