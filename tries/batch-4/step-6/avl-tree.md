# AVL Trees

An AVL tree is a binary search tree that keeps itself balanced. Georgy Adelson-Velsky and Evgenii Landis published it in 1962, which makes it the first self-balancing binary search tree. The idea fits in one sentence: **at every node, the heights of the left and right subtrees differ by at most one.** That rule alone keeps the tree's height logarithmic in the number of keys, so search, insertion and deletion all run in O(log n) time in the worst case.

This explainer covers the invariant, the four rotation cases, a worked insertion sequence that triggers each case, a deletion where rebalancing cascades up the tree, a proof of the height bound using Fibonacci trees, a comparison with red-black trees, and a complete implementation.

---

## 1. Why balance matters

A plain binary search tree (BST) is fast only when it is bushy. Insert the keys 1, 2, 3, ..., n in sorted order and every node gets only a right child: the "tree" is a linked list of height n, and searching for n takes n comparisons. Sorted or nearly sorted input is common in practice (timestamps, auto-increment IDs, log sequence numbers), so this worst case is a real problem.

Self-balancing trees fix this by doing a small amount of repair work after each update so that the height stays O(log n) no matter what order keys arrive in. AVL trees use the strictest common balance rule, which gives them the shortest trees and so the fastest lookups among the standard balanced BSTs.

## 2. Definitions

Some conventions, used throughout:

- The **height** of an empty tree is 0, and the height of a single node is 1. In general, `height(node) = 1 + max(height(left), height(right))`.
- The **balance factor** of a node is `bf(node) = height(left) - height(right)`.
- A BST is an **AVL tree** if every node has a balance factor of -1, 0 or +1.

A positive balance factor means "left-heavy", a negative one means "right-heavy". After an insertion or deletion, some node on the path from the root to the changed spot may reach +2 or -2. That node is **unbalanced**, and a rotation fixes it.

Most implementations store either the height (a small integer) or the balance factor (two bits are enough) in each node. Storing the height is simpler to get right, and it is what the code in this article does.

## 3. Rotations

A rotation rearranges three nodes and their subtrees locally. It changes the shape of the tree but keeps the in-order sequence of keys the same, so the BST property is preserved. It takes O(1) time, since only a constant number of pointers change.

### Right rotation (around y)

```
        y                x
       / \              / \
      x   C    ==>     A   y
     / \                  / \
    A   B                B   C
```

`x` moves up, `y` moves down to become x's right child, and x's old right subtree `B` becomes y's left subtree. The in-order sequence is `A x B y C` both before and after.

### Left rotation (around x)

The mirror image:

```
      x                    y
     / \                  / \
    A   y      ==>       x   C
       / \              / \
      B   C            A   B
```

### The four cases

Say node `z` is the lowest node on the path that has become unbalanced. Which rotation to apply depends on where the excess height sits:

| Case | bf(z) | bf(child) | Fix |
|------|-------|-----------|-----|
| **LL** (left child's left subtree too tall) | +2 | +1 (or 0 on delete) | Right rotation at z |
| **RR** (right child's right subtree too tall) | -2 | -1 (or 0 on delete) | Left rotation at z |
| **LR** (left child's right subtree too tall) | +2 | -1 | Left rotation at z.left, then right rotation at z |
| **RL** (right child's left subtree too tall) | -2 | +1 | Right rotation at z.right, then left rotation at z |

LL and RR are "outside" cases: the heavy path runs in a straight line, and one rotation pulls the middle node up. LR and RL are "inside" cases: the heavy path zigzags, and a single rotation would just move the zigzag to the other side. The first rotation of the pair straightens the zigzag into an outside case, and the second one finishes the job. That is why they are called double rotations.

## 4. Worked insertion example

Insertion works like a normal BST insert, followed by a walk back up the path to the root. At each node on the way up, recompute the height and the balance factor. If some node reaches ±2, apply the matching rotation.

One useful fact: **during insertion, at most one rotation (single or double) is ever needed.** The rotation brings the subtree back to the height it had *before* the insert, so no ancestor above it sees any change.

We will insert this sequence:

```
30, 20, 10, 40, 50, 35, 15
```

In the diagrams below, each node is shown as `key[bf]`.

### Insert 30

```
30[0]
```

### Insert 20

```
    30[+1]
   /
20[0]
```

Still balanced.

### Insert 10: LL case

Before rebalancing:

```
        30[+2]   <- unbalanced
       /
    20[+1]
   /
10[0]
```

Node 30 has a left subtree of height 2 and an empty right subtree, so `bf = +2`. Its left child 20 is left-heavy (`+1`). The heavy path goes left then left: this is **LL**. Rotate right at 30:

```
    20[0]
   /     \
10[0]   30[0]
```

### Insert 40

```
    20[-1]
   /     \
10[0]   30[-1]
           \
          40[0]
```

All balance factors are within range.

### Insert 50: RR case

Before rebalancing:

```
    20[-2]?
   /     \
10[0]   30[-2]   <- lowest unbalanced node
           \
          40[-1]
             \
            50[0]
```

We walk up from 50. Node 40 has `bf = -1`, which is fine. Node 30 has a right subtree of height 2 and nothing on the left, so `bf = -2`. It is the lowest unbalanced node, and we fix it first. (We never need to look at 20's provisional value, because fixing 30 restores the subtree's height.) The path goes right then right: **RR**. Rotate left at 30:

```
    20[-1]
   /     \
10[0]   40[0]
       /     \
    30[0]   50[0]
```

The subtree rooted at 40 has height 2, exactly what the subtree rooted at 30 had before 50 arrived, so 20 ends up with `bf = -1`.

### Insert 35: RL case

35 is greater than 20, less than 40 and greater than 30, so it becomes 30's right child:

```
    20[-2]   <- unbalanced
   /     \
10[0]   40[+1]
       /     \
    30[-1]  50[0]
       \
      35[0]
```

Walking up: 30 is at -1 (fine), 40 is at +1 (fine), and 20 now has a left height of 1 and a right height of 3, so `bf = -2`. Its right child 40 is **left**-heavy. The path goes right, then left: **RL**.

Step 1: rotate right at 40, which straightens the zigzag:

```
    20
   /  \
 10    30
         \
          40
         /  \
       35    50
```

(30's right subtree was 35. When 40 drops below 30, 35 becomes 40's left child.)

Step 2: rotate left at 20:

```
          30[0]
         /     \
     20[+1]    40[0]
     /        /    \
  10[0]    35[0]  50[0]
```

30 was the "middle" key of the three nodes involved (20 < 30 < 40), and it ends up on top. That is the general shape of a double rotation: **the grandchild becomes the root of the subtree.**

### Insert 15: LR case

15 goes left of 30, left of 20, right of 10:

```
             30[+1]
            /      \
       20[+2]      40[0]
       /           /   \
   10[-1]       35[0]  50[0]
       \
      15[0]
```

Node 20 has `bf = +2`, and its left child 10 is **right**-heavy (`-1`). The path goes left, then right: **LR**.

Step 1: rotate left at 10:

```
       20
       /
     15
     /
   10
```

Step 2: rotate right at 20:

```
     15[0]
    /     \
 10[0]   20[0]
```

The final tree:

```
             30[0]
            /      \
        15[0]      40[0]
       /    \      /    \
    10[0] 20[0] 35[0]  50[0]
```

Seven keys in a perfect tree of height 3, the best a BST can do. A plain BST fed the same sequence would have height 4. The gap grows for longer, more adversarial sequences: sorted input gives height n in a plain BST and roughly log₂ n in an AVL tree.

## 5. Deletion, with cascading rebalances

Deletion starts out like a normal BST delete:

1. If the node is a leaf, remove it.
2. If it has one child, splice the child into its place.
3. If it has two children, copy in the key of its in-order successor (the leftmost node of its right subtree), then delete that successor from the right subtree. The successor has no left child, so this falls into case 1 or 2.

Then, as with insertion, walk back up toward the root, updating heights and rotating wherever a node reaches ±2.

The important difference: **a rotation after a deletion can make the subtree shorter.** After an insert, the rotation restores the subtree's previous height, so the repair stops there. After a delete, the subtree was already one level shorter, and the rotation may leave it that way. The parent then sees a shorter child, may go out of balance itself, and the repair continues upward. In the worst case, a single deletion causes a rotation at every level, which is O(log n) rotations.

A second subtlety: during deletion, the child on the heavy side can have a balance factor of **0**, which never happens during insertion. In that case, use the **single** rotation. A double rotation would be wrong there. (The single rotation leaves the subtree's height unchanged, so the cascade stops.)

### The starting tree

Here is a valid AVL tree on the keys 1 to 12. It is a *Fibonacci tree*, a minimal AVL tree that we will meet again in the height proof, and that is exactly why it shows cascading so well: every internal node is already leaning as far as it is allowed to.

```
                         8[+1]
                    /            \
               5[+1]              10[-1]
             /      \            /      \
         3[+1]     6[-1]      9[0]     11[-1]
         /   \        \                    \
      2[+1]  4[0]    7[0]                 12[0]
      /
    1[0]
```

Heights: the subtree at 3 has height 3, the one at 6 has height 2, so 5 has height 4. The subtree at 10 has height 3. The root has height 5.

### Delete 9

9 is a leaf, so we remove it directly. Then we walk upward.

**At node 10.** Its left subtree is now empty (height 0), and its right subtree, 11 → 12, has height 2. So `bf(10) = -2`. The right child 11 has `bf = -1`, so this is **RR**. Rotate left at 10:

```
      11[0]
     /     \
  10[0]   12[0]
```

This subtree now has height 2. Before the deletion it had height 3. **It got shorter**, so we must keep going.

**At node 8 (the root).** The left subtree (rooted at 5) has height 4. The right subtree (now rooted at 11) has height 2. So `bf(8) = +2`. The left child 5 has `bf = +1`, so this is **LL**. Rotate right at 8:

- 5 becomes the new root.
- 8 becomes 5's right child.
- 5's old right subtree (6 → 7) becomes 8's left subtree.

```
                   5[0]
             /             \
         3[+1]              8[0]
         /   \            /      \
      2[+1]  4[0]     6[-1]      11[0]
      /                  \       /    \
    1[0]                7[0]  10[0]  12[0]
```

Check: 8 has a left subtree of height 2 (6 → 7) and a right subtree of height 2 (11), so `bf(8) = 0` and its height is 3. The root 5 has height 3 on both sides, so `bf(5) = 0`. The tree is balanced again, with height 4.

One deletion, two rotations, at two different levels. In a larger Fibonacci tree, deleting the corresponding leaf would trigger a rotation at every level up to the root.

## 6. The height bound: Fibonacci trees

Why does the "differ by at most one" rule guarantee logarithmic height? The cleanest proof turns the question around. Instead of asking "how tall can a tree with n nodes be?", ask: **what is the fewest nodes an AVL tree of height h can have?** Call that number N(h).

### The recurrence

- N(0) = 0 (the empty tree).
- N(1) = 1 (a single node).
- For h ≥ 2, a minimal AVL tree of height h has a root and two subtrees. One subtree must have height h - 1, otherwise the whole tree would not reach height h. To use as few nodes as possible, the other subtree should be as short as the balance rule allows, which is height h - 2. Both subtrees should themselves be minimal. So:

```
N(h) = N(h-1) + N(h-2) + 1
```

The trees that achieve this minimum are called **Fibonacci trees**. The tree in the deletion example above is the Fibonacci tree of height 5: its left subtree (rooted at 5) is the Fibonacci tree of height 4, and its right subtree (rooted at 10) is the one of height 3.

### Connection to the Fibonacci numbers

Let F(k) be the Fibonacci numbers, with F(1) = F(2) = 1, F(3) = 2, and so on. The first few values of N(h) are:

| h | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
|---|---|---|---|---|---|---|---|---|
| N(h) | 0 | 1 | 2 | 4 | 7 | 12 | 20 | 33 |
| F(h+2) | 1 | 2 | 3 | 5 | 8 | 13 | 21 | 34 |

The pattern is **N(h) = F(h+2) - 1**. To prove it, add 1 to both sides of the recurrence:

```
N(h) + 1 = (N(h-1) + 1) + (N(h-2) + 1)
```

So M(h) = N(h) + 1 obeys exactly the Fibonacci recurrence, with M(0) = 1 = F(2) and M(1) = 2 = F(3). By induction, M(h) = F(h+2).

### From Fibonacci to logarithms

Let φ = (1 + √5) / 2 ≈ 1.618, the golden ratio. A standard fact is that **F(k) ≥ φ^(k-2) for every k ≥ 1**. Proof by induction: F(1) = 1 ≥ φ^(-1) and F(2) = 1 ≥ φ^0. For k ≥ 3,

```
F(k) = F(k-1) + F(k-2) ≥ φ^(k-3) + φ^(k-4) = φ^(k-4) (φ + 1) = φ^(k-4) · φ² = φ^(k-2)
```

using the identity φ² = φ + 1.

Now take any AVL tree with n nodes and height h. Since N(h) is the minimum number of nodes for that height,

```
n ≥ N(h) = F(h+2) - 1 ≥ φ^h - 1
```

so φ^h ≤ n + 1, which gives

```
h ≤ log_φ(n + 1) = log₂(n + 1) / log₂ φ ≈ 1.4404 · log₂(n + 1)
```

A slightly sharper analysis, using the exact closed form for F(k), gives the textbook bound h < 1.4405 · log₂(n + 2) - 0.3277.

**An AVL tree is never more than about 44% taller than a perfectly balanced tree with the same number of nodes.** With a million keys, a perfect tree has height 20, and an AVL tree has height at most 28. In practice, with random input, AVL trees are much closer to the ideal, averaging around log₂ n + 0.25.

Because the height is O(log n), and search, insert and delete each do O(1) work per level, all three operations take O(log n) time in the worst case.

## 7. AVL trees vs. red-black trees

Red-black trees are the other common self-balancing BST, and they are what most standard libraries use (C++ `std::map`, Java's `TreeMap`, the Linux kernel's `rbtree`). They use a looser rule: color every node red or black, never allow two reds in a row, and require every root-to-leaf path to contain the same number of black nodes. That rule allows the longest path to be up to twice the shortest.

| | AVL tree | Red-black tree |
|---|---|---|
| Balance rule | Subtree heights differ by ≤ 1 | No red-red; equal black-height on all paths |
| Worst-case height | ≈ 1.44 log₂ n | ≈ 2 log₂ n |
| Search | Faster (shorter trees) | Slightly slower |
| Rotations per insert | At most 1 (single or double) | At most 2 |
| Rotations per delete | Up to O(log n) | At most 3 |
| Extra bookkeeping per node | Height or 2-bit balance factor | 1 color bit |
| Height/color updates per insert or delete | O(log n) worst case, often stops early | O(1) amortized recolorings |
| Typical use | Read-heavy indexes, lookup tables | General-purpose maps, kernels, write-heavy workloads |

A few things this table leaves out:

- **Every AVL tree can be colored as a valid red-black tree**, but not the other way around. The AVL tree family is strictly a subset of the red-black family. You can think of AVL as red-black with a tighter leash.
- **Lookups dominate many workloads.** If a structure is built once and queried many times, the AVL tree's shorter height saves comparisons and cache misses on every query.
- **Writes favor red-black trees.** Their bounded number of rotations per deletion, and O(1) amortized restructuring, make them cheaper to update. This also matters for persistent (immutable) versions of the trees, where every node touched must be copied, and for concurrent versions, where every node touched must be locked.
- **In benchmarks the gap is usually small.** Both are O(log n), and constant factors such as memory layout, allocation and branch prediction often matter more than the theoretical difference. Pick red-black if you are unsure and writes are frequent. Pick AVL if lookups clearly dominate.
- **Simplicity.** Many people find AVL trees easier to reason about: there are four rotation cases, and you can verify the invariant just by computing heights. Red-black deletion, by contrast, has a well-known reputation for fiddly case analysis.

## 8. Implementation

The Python implementation below stores each node's height and uses recursion, so rebalancing happens naturally on the way back up. Every function that modifies a subtree returns that subtree's (possibly new) root, and the caller stores it back in the parent. This avoids keeping parent pointers.

```python
class Node:
    __slots__ = ("key", "left", "right", "height")

    def __init__(self, key):
        self.key = key
        self.left = None
        self.right = None
        self.height = 1


def height(node):
    return node.height if node else 0


def update(node):
    node.height = 1 + max(height(node.left), height(node.right))


def balance(node):
    return height(node.left) - height(node.right)


def rotate_right(y):
    """Right rotation around y; returns the new subtree root."""
    x = y.left
    y.left = x.right
    x.right = y
    update(y)          # y is now below x, so update it first
    update(x)
    return x


def rotate_left(x):
    """Left rotation around x; returns the new subtree root."""
    y = x.right
    x.right = y.left
    y.left = x
    update(x)
    update(y)
    return y


def rebalance(node):
    """Restore the AVL property at node, assuming its children are valid."""
    update(node)
    b = balance(node)
    if b > 1:                                   # left-heavy
        if balance(node.left) < 0:              # LR: straighten first
            node.left = rotate_left(node.left)
        return rotate_right(node)               # LL (or finished LR)
    if b < -1:                                  # right-heavy
        if balance(node.right) > 0:             # RL: straighten first
            node.right = rotate_right(node.right)
        return rotate_left(node)                # RR (or finished RL)
    return node


def insert(node, key):
    if node is None:
        return Node(key)
    if key < node.key:
        node.left = insert(node.left, key)
    elif key > node.key:
        node.right = insert(node.right, key)
    else:
        return node                             # duplicate: ignore
    return rebalance(node)


def delete(node, key):
    if node is None:
        return None                             # key not present
    if key < node.key:
        node.left = delete(node.left, key)
    elif key > node.key:
        node.right = delete(node.right, key)
    else:
        if node.left is None:
            return node.right
        if node.right is None:
            return node.left
        succ = node.right                       # two children:
        while succ.left:                        # find in-order successor
            succ = succ.left
        node.key = succ.key
        node.right = delete(node.right, succ.key)
    return rebalance(node)


def search(node, key):
    while node is not None and node.key != key:
        node = node.left if key < node.key else node.right
    return node
```

Notes on the code:

- **The strict comparisons in `rebalance` matter.** The double rotation fires only when the child leans the *opposite* way (`< 0` on the left side, `> 0` on the right). When the child's balance is exactly 0, which can only happen after a deletion, the single rotation is correct. Writing `<= 0` here is a classic bug: it produces a tree that is still unbalanced.
- **Update order in rotations.** After a rotation, the node that moved down must have its height recomputed before the node that moved up, because the new parent's height depends on it.
- **`rebalance` is called at every level**, even when nothing changed. It is cheap, and it keeps the code simple. An optimized version can stop walking up once a node's height comes out the same as before, which is how real implementations avoid touching all O(log n) ancestors on every update.
- **Recursion depth** is bounded by the tree's height, at most about 1.44 log₂ n, so even a billion keys need fewer than 45 stack frames.

### A wrapper and a validator

A small class makes the API friendlier, and a validator lets you check the invariants after every operation while testing:

```python
class AVLTree:
    def __init__(self):
        self.root = None

    def insert(self, key):
        self.root = insert(self.root, key)

    def delete(self, key):
        self.root = delete(self.root, key)

    def __contains__(self, key):
        return search(self.root, key) is not None

    def inorder(self):
        out, stack, node = [], [], self.root
        while stack or node:
            while node:
                stack.append(node)
                node = node.left
            node = stack.pop()
            out.append(node.key)
            node = node.right
        return out


def check(node, lo=float("-inf"), hi=float("inf")):
    """Return the subtree height; raise AssertionError if any invariant fails."""
    if node is None:
        return 0
    assert lo < node.key < hi, "BST order violated"
    hl = check(node.left, lo, node.key)
    hr = check(node.right, node.key, hi)
    assert abs(hl - hr) <= 1, f"unbalanced at {node.key}"
    assert node.height == 1 + max(hl, hr), f"stale height at {node.key}"
    return node.height
```

And a randomized test that exercises every rotation case many times:

```python
import random

t = AVLTree()
keys = list(range(10_000))
random.shuffle(keys)
for k in keys:
    t.insert(k)
    check(t.root)
random.shuffle(keys)
for k in keys[:5_000]:
    t.delete(k)
    check(t.root)
assert t.inorder() == sorted(keys[5_000:])
```

Replaying the worked examples is a good sanity check, too. Inserting `30, 20, 10, 40, 50, 35, 15` should leave `30` at the root with children `15` and `40`. Building the twelve-node Fibonacci tree and deleting `9` should leave `5` at the root with children `3` and `8`.

## 9. Summary

- An AVL tree is a BST in which every node's subtree heights differ by at most one.
- Imbalances after an update come in four shapes: LL, RR, LR and RL. Each is fixed by a single or double rotation in O(1) time.
- Insertion needs at most one (single or double) rotation. Deletion may need a rotation at every level, because a rotation after a deletion can shorten the subtree and pass the imbalance upward.
- The sparsest AVL trees are Fibonacci trees, with N(h) = F(h+2) - 1 nodes. This gives a height of at most about 1.44 log₂ n, so every operation runs in O(log n) time in the worst case.
- Compared to red-black trees, AVL trees are shorter and faster to search, but they do more restructuring on updates. That makes AVL a good fit for read-heavy workloads, and red-black the usual default for general-purpose maps.
