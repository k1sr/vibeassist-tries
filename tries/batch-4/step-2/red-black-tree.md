# Red-Black Trees: How They Work, With Worked Examples

A binary search tree (BST) is fast when it is bushy and slow when it is stringy. Insert the keys 1, 2, 3, ..., n into a plain BST and you get a linked list leaning to the right. Every search then takes O(n) time. A **red-black tree** is a BST that does a small, bounded amount of repair after each insert and delete. The repair keeps the height at O(log n), so search, insert and delete all run in O(log n) time in the worst case.

Red-black trees are everywhere. C++ `std::map` and `std::set` use them, as do Java's `TreeMap` and `TreeSet`. The Linux kernel uses them in its CFS scheduler and its memory-area tracking. They are popular because the repair work per update is small. An insert does at most two rotations and a delete at most three, so the cost is mostly a walk down the tree plus a few pointer changes.

This explainer covers:

1. The five invariants
2. Why they force logarithmic height (a proof sketch)
3. Rotations, the one structural tool
4. Insertion, with every case traced on a concrete key sequence
5. Deletion, with every case traced on the resulting tree
6. A complete, runnable Python implementation with a validator

---

## 1. The invariants

Every node is colored either **red** or **black**. We treat the empty child pointers as real leaf nodes called **NIL**, and every NIL is black. Many implementations use one shared sentinel object for all of them. The tree must satisfy:

1. **Color:** every node is red or black.
2. **Root:** the root is black.
3. **Leaves:** every NIL leaf is black.
4. **Red rule:** a red node has no red child. Equivalently, no path contains two reds in a row.
5. **Black rule:** for each node, every path from that node down to a NIL leaf contains the same number of black nodes.

The number in rule 5 is the node's **black-height**, written bh(x). We do not count x itself, but we do count the NIL at the end of the path. In this document's diagrams, NIL leaves are left out. A black node is written `12B` and a red node `5R`.

The intuition: the black nodes form a perfectly balanced "skeleton," because every root-to-leaf path holds the same number of them. Red nodes are allowed as extra padding, but rule 4 means you can't stack them. At most, a path alternates black, red, black, red, and so on. That makes it at most twice as long as an all-black path.

A useful mental model is that a red-black tree encodes a **2-3-4 tree** (a B-tree of order 4). Glue each black node to its red children and you get one "fat" node holding 1, 2 or 3 keys. All the leaves of that 2-3-4 tree sit at the same depth. The rotations and recolorings below are the binary-tree versions of splitting and merging fat nodes.

---

## 2. Why the height is O(log n)

**Claim.** A red-black tree with n internal (non-NIL) nodes has height h ≤ 2·log₂(n + 1).

**Lemma.** The subtree rooted at any node x contains at least 2^bh(x) − 1 internal nodes.

*Proof by induction on the height of x.*

- *Base case:* if x has height 0, then x is a NIL leaf. Its black-height is 0, and the subtree holds 2⁰ − 1 = 0 internal nodes. ✓
- *Inductive step:* take an internal node x with black-height b. Each child of x has black-height b if the child is red, or b − 1 if the child is black. Either way, each child's black-height is at least b − 1. Each child is lower than x, so the induction hypothesis applies: each child's subtree has at least 2^(b−1) − 1 internal nodes. Add x itself:

  n(x) ≥ 2·(2^(b−1) − 1) + 1 = 2^b − 1. ✓

**Height bound.** Let h be the height of the root. Rule 4 says that on any root-to-leaf path, at least half of the nodes are black, not counting the root. So bh(root) ≥ h/2. Applying the lemma to the root:

n ≥ 2^(h/2) − 1  ⇒  n + 1 ≥ 2^(h/2)  ⇒  h ≤ 2·log₂(n + 1).

So a search visits O(log n) nodes. Insert and delete each do one O(log n) descent plus repair work that climbs back up at most O(log n) levels. Every rule-1 to rule-5 tree with a million keys has height at most about 40.

---

## 3. Rotations

A rotation is a local restructuring that keeps the BST order intact. It changes three parent-child links and runs in O(1) time.

```
        Left-rotate(x)                   Right-rotate(y)
     x                 y               y                 x
    / \               / \             / \               / \
   a   y     ==>     x   c           x   c     ==>     a   y
      / \           / \             / \                   / \
     b   c         a   b           a   b                 b   c
```

The in-order sequence `a x b y c` is the same before and after. Only the shape changes. Left-rotate lifts the right child. Right-rotate lifts the left child, and the two operations undo each other. A rotation doesn't change any colors by itself. The fixup procedures pair rotations with recolorings.

---

## 4. Insertion

### The procedure

1. Insert the new key z exactly as in an ordinary BST, as a leaf.
2. Color z **red**.
3. Run **insert-fixup** to repair any violation.

Why red? A new black node would add one to the black count on exactly one set of paths, which breaks rule 5. That is hard to fix. A red node leaves every black count unchanged. The only rules it can break are:

- rule 2, if z is the root (fix: paint it black), or
- rule 4, if z's parent is also red (the "double red").

### The fixup cases

Suppose z is red and its parent p is red. Then p cannot be the root, because the root is black. So z has a grandparent g, and g must be black, because the tree was valid before the insert. Let u be the **uncle**, meaning p's sibling. The cases below assume p is a *left* child of g. The mirrored cases swap left and right.

- **Case 1, uncle is red: recolor.** Paint p and u black and g red. Every path through g still has the same black count. Rule 4 might now fail between g and *g's* parent, so set z ← g and loop. This is the only case that moves upward. In 2-3-4 terms, it splits a full 4-node and pushes the middle key up.
- **Case 2, uncle is black and z is an "inner" grandchild (a triangle).** z is p's right child while p is g's left child. Left-rotate at p, then set z ← the old p. This turns the triangle into a straight line, which is case 3. Colors don't change.
- **Case 3, uncle is black and z is an "outer" grandchild (a line).** z is p's left child, and p is g's left child. Right-rotate at g, paint p black and g red. p now sits where g was, colored as g was, so the black count above is unchanged and the red-red pair is gone. **The loop ends here.**

At the end, always paint the root black. That handles both the "first insert" situation and a case 1 cascade that reaches the top.

Each case 1 step climbs two levels, and cases 2 and 3 end the loop. So the fixup does O(log n) recolorings and **at most 2 rotations**.

### Worked example: inserting 10, 20, 30, 15, 25, 12, 5, 1, 3

This sequence hits every case, including a case 1 cascade that ends in a case 3 rotation near the root.

**Insert 10.** The tree is empty, so 10 becomes the root. It starts red, and the final step paints it black.

```
10B
```

**Insert 20.** 20 goes to the right of 10, colored red. Its parent is black, so nothing is broken.

```
10B
   \
   20R
```

**Insert 30.** 30 goes right of 20. Now 20R → 30R is a double red. The uncle is 10's left child, a black NIL. 30 is the right child of a right child, so this is a line: **case 3 (mirrored)**. Left-rotate at 10, paint 20 black and 10 red.

```
Before:            After left-rotate(10) + recolor:
10B                     20B
   \                   /   \
   20R               10R   30R
      \
      30R
```

**Insert 15.** The path is 20 → 10 → right, so 15R becomes the right child of 10R: a double red. The uncle, 30, is **red**, so this is **case 1**. Paint 10 and 30 black and 20 red. Then z ← 20, which is the root, so the final step paints it black again. Net effect: the tree's black-height grows by one.

```
Before:                  After recolor:
      20B                      20B
     /   \                    /   \
   10R   30R                10B   30B
      \                        \
      15R                      15R
```

**Insert 25.** The path is 20 → 30 → left. 25R's parent, 30, is black, so no fix is needed.

```
       20B
      /   \
    10B   30B
       \  /
      15R 25R
```

**Insert 12.** The path is 20 → 10 → 15 → left, so 12R becomes the left child of 15R: a double red. The uncle is 10's left child, a black NIL. 15 is a *right* child and 12 is a *left* child, so this is a triangle: **case 2 (mirrored)**. Right-rotate at 15, then z ← 15.

```
    10B                 10B
       \                   \
       15R     ==>         12R
       /                      \
     12R                      15R
```

The pair is now a straight line, 10 → 12 → 15, all right children, so **case 3 (mirrored)** applies. Left-rotate at 10, paint 12 black and 10 red.

```
         20B
        /   \
      12B   30B
     /  \    /
   10R 15R  25R
```

These two rotations are the "double rotation" for this insert.

**Insert 5.** The path is 20 → 12 → 10 → left. 5R under 10R is a double red. The uncle, 15, is **red**, so this is **case 1**. Paint 10 and 15 black and 12 red. Then z ← 12, whose parent 20 is black, so the loop stops.

```
           20B
          /   \
        12R   30B
       /  \    /
     10B  15B 25R
     /
    5R
```

Check rule 5: 20→12→10→NIL passes the black nodes {20, 10}, and 20→12→15→NIL passes {20, 15}. Both counts are 2, and so is 20→30→NIL.

**Insert 1.** The path is 20 → 12 → 10 → 5 → left. 1R under 5R is a double red. The uncle is 10's right child, a black NIL. 1 is a left child of a left child, so this is a line: **case 3**. Right-rotate at 10, paint 5 black and 10 red.

```
            20B
           /   \
         12R   30B
        /  \    /
      5B   15B 25R
     /  \
   1R   10R
```

**Insert 3.** This is the interesting one. The path is 20 → 12 → 5 → 1 → right, so 3R is the right child of 1R: a double red.

*Step 1, case 1.* The uncle, 10, is red. Paint 1 and 10 black and 5 red, then set z ← 5.

```
            20B
           /   \
         12R   30B
        /  \    /
      5R   15B 25R
     /  \
   1B   10B
     \
     3R
```

*Step 2.* Now 5R under 12R is a double red, one level higher. The new grandparent is 20 and the uncle is 30, which is **black**. 5 is a left child and 12 is a left child, so this is a line: **case 3**. Right-rotate at 20, paint 12 black and 20 red. Note that 12's old right subtree, 15, becomes 20's new left subtree.

```
               12B
             /     \
          5R         20R
         /  \       /   \
       1B   10B   15B   30B
         \              /
         3R           25R
```

**Verify the final tree.** The root is black. No red node has a red child: 5R's children are black, 20R's children are black, and 3R and 25R are leaves. Every root-to-NIL path passes exactly 2 black nodes:

| Path | Black nodes |
|---|---|
| 12 → 5 → 1 → NIL | 12, 1 |
| 12 → 5 → 1 → 3 → NIL | 12, 1 |
| 12 → 5 → 10 → NIL | 12, 10 |
| 12 → 20 → 15 → NIL | 12, 15 |
| 12 → 20 → 30 → 25 → NIL | 12, 30 |

Nine keys inserted in an awkward order left us with a tree of height 4. The bound 2·log₂(10) ≈ 6.6 is comfortably met.

---

## 5. Deletion

Deletion is the harder half. The idea is still simple. Do a normal BST delete. If the node that physically left the tree was black, every path through that spot is now one black node short. Repair it.

### The BST part

Let z be the node to delete.

- **z has at most one non-NIL child.** Splice z out and move its child x into its place. The "removed color" is z's color.
- **z has two children.** Find its in-order successor y, the leftmost node of z's right subtree. y has no left child. Move y into z's position and give y **z's color**. y's right child x moves into y's old spot. The "removed color" is y's original color, because that is the color that disappeared from the tree.

If the removed color was **red**, nothing is broken. No black counts changed, and no new red-red pair can appear. If it was **black**, the paths through x have lost one black node. Think of x as carrying an "extra black": x is "doubly black" if it was already black, or "red-and-black" if it was red. Fixup pushes that extra black up the tree until it can be absorbed.

### The fixup cases

The loop runs while x is not the root and x is black. If x is red-and-black, simply paint it black and stop. The cases assume x is a *left* child. The mirrored cases swap left and right. Let w be x's sibling. Because x carries an extra black, w cannot be NIL.

- **Case 1, w is red.** Paint w black and the parent red, then left-rotate at the parent. x now has a new sibling, which is black, so continue with case 2, 3 or 4. In 2-3-4 terms, this just picks a different binary encoding of the same fat node.
- **Case 2, w is black and both of w's children are black.** Paint w red. This removes one black from w's side to match x's side. The extra black moves up: x ← parent. If the new x is red, the loop ends and x is painted black. This is the only case that climbs, and it corresponds to merging two 2-3-4 nodes.
- **Case 3, w is black, w's left child is red, and w's right child is black.** Paint w's left child black and w red, then right-rotate at w. x's new sibling is black and has a red right child, which is case 4.
- **Case 4, w is black and w's right child is red.** Give w the parent's color, paint the parent black, and paint w's right child black. Left-rotate at the parent. The extra black is absorbed, so set x ← root to end the loop. This is a 2-3-4 "borrow from sibling."

Delete-fixup does O(log n) recolorings and **at most 3 rotations**: case 1, then case 3, then case 4. Case 1 never leads into case 2 with a black parent, so the climb stops right after.

### Worked example: deleting 3, 12, 1, 5, 10

We start from the tree built above:

```
               12B
             /     \
          5R         20R
         /  \       /   \
       1B   10B   15B   30B
         \              /
         3R           25R
```

**Delete 3.** 3 is a red leaf. Splicing it out removes a red node, so no fixup is needed.

```
               12B
             /     \
          5R         20R
         /  \       /   \
       1B   10B   15B   30B
                        /
                      25R
```

**Delete 12 (the root, which has two children).** The successor y is 15, the leftmost node in 12's right subtree. Its original color is **black**, so we will need a fixup. y's right child x is NIL. x takes y's old place as 20's left child. 15 moves to the root and takes 12's color, black.

```
               15B
             /     \
          5R         20R
         /  \       /   \
       1B   10B   [x]   30B
                        /
                      25R
```

x is a doubly-black NIL, its parent is 20, and x is a left child. The sibling w is 30B, whose left child 25 is red and whose right child is NIL (black). That is **case 3**. Paint 25 black and 30 red, then right-rotate at 30:

```
          20R
         /   \
       [x]   25B
                \
                30R
```

Now w = 25B and its right child is red: **case 4**. Give 25 its parent's color (red), paint 20 black and 30 black, then left-rotate at 20. Done.

```
               15B
             /     \
          5R         25R
         /  \       /   \
       1B   10B   20B   30B
```

Every path now passes 2 black nodes: for example 15→25→20→NIL has {15, 20}, and 15→5→1→NIL has {15, 1}.

**Delete 1.** 1 is a black leaf, so x = NIL as 5's left child, doubly black. The sibling w = 10B has two NIL children, so this is **case 2**. Paint 10 red and move the extra black up: x ← 5. Since 5 is **red**, the loop exits and 5 is painted black. The red parent absorbed the missing black.

```
            15B
          /     \
        5B       25R
          \     /   \
          10R  20B  30B
```

**Delete 5 (which has one child).** 5's left child is NIL, so its right child x = 10 moves up into its place. The removed color is black. But x is **red**, so the fixup loop is skipped and x is painted black.

```
            15B
          /     \
       10B       25R
                /   \
              20B   30B
```

**Delete 10.** 10 is a black leaf, so x = NIL as 15's left child. The sibling w = 25 is **red**: **case 1**. Paint 25 black and 15 red, then left-rotate at 15:

```
            25B
           /   \
         15R   30B
        /   \
      [x]   20B
```

x's new sibling is 20B with two NIL children: **case 2**. Paint 20 red and set x ← 15. Since 15 is red, the loop exits and 15 is painted black.

```
            25B
           /   \
         15B   30B
            \
            20R
```

Check: 25→15→NIL has {25, 15}, 25→15→20→NIL has {25, 15}, and 25→30→NIL has {25, 30}. All counts are 2. ✓

Across these five deletes we used every fixup case: case 1 (delete 10), case 2 (deletes 1 and 10), case 3 and case 4 (delete 12). We also used the two short-cuts: removing a red node (delete 3) and absorbing the extra black into a red child (delete 5).

---

## 6. Implementation

This implementation follows the classic CLRS structure. It uses one shared black `NIL` sentinel, so the code never checks for `None` when it reads a child's color. Assigning to the sentinel's `parent` field is intentional: delete-fixup relies on `x.parent` even when x is NIL.

```python
RED, BLACK = 0, 1


class Node:
    __slots__ = ("key", "color", "left", "right", "parent")

    def __init__(self, key, color=RED, nil=None):
        self.key = key
        self.color = color
        self.left = nil
        self.right = nil
        self.parent = nil


class RedBlackTree:
    def __init__(self):
        self.NIL = Node(None, BLACK)
        self.NIL.left = self.NIL.right = self.NIL.parent = self.NIL
        self.root = self.NIL

    # ---------- search ----------
    def search(self, key):
        x = self.root
        while x is not self.NIL and key != x.key:
            x = x.left if key < x.key else x.right
        return x if x is not self.NIL else None

    def _minimum(self, x):
        while x.left is not self.NIL:
            x = x.left
        return x

    # ---------- rotations ----------
    def _left_rotate(self, x):
        y = x.right
        x.right = y.left
        if y.left is not self.NIL:
            y.left.parent = x
        y.parent = x.parent
        if x.parent is self.NIL:
            self.root = y
        elif x is x.parent.left:
            x.parent.left = y
        else:
            x.parent.right = y
        y.left = x
        x.parent = y

    def _right_rotate(self, y):
        x = y.left
        y.left = x.right
        if x.right is not self.NIL:
            x.right.parent = y
        x.parent = y.parent
        if y.parent is self.NIL:
            self.root = x
        elif y is y.parent.right:
            y.parent.right = x
        else:
            y.parent.left = x
        x.right = y
        y.parent = x

    # ---------- insertion ----------
    def insert(self, key):
        z = Node(key, RED, self.NIL)
        parent, x = self.NIL, self.root
        while x is not self.NIL:
            parent = x
            x = x.left if z.key < x.key else x.right
        z.parent = parent
        if parent is self.NIL:
            self.root = z
        elif z.key < parent.key:
            parent.left = z
        else:
            parent.right = z
        self._insert_fixup(z)

    def _insert_fixup(self, z):
        while z.parent.color == RED:
            g = z.parent.parent
            if z.parent is g.left:
                u = g.right
                if u.color == RED:                      # case 1
                    z.parent.color = BLACK
                    u.color = BLACK
                    g.color = RED
                    z = g
                else:
                    if z is z.parent.right:             # case 2
                        z = z.parent
                        self._left_rotate(z)
                    z.parent.color = BLACK              # case 3
                    g.color = RED
                    self._right_rotate(g)
            else:                                       # mirror image
                u = g.left
                if u.color == RED:
                    z.parent.color = BLACK
                    u.color = BLACK
                    g.color = RED
                    z = g
                else:
                    if z is z.parent.left:
                        z = z.parent
                        self._right_rotate(z)
                    z.parent.color = BLACK
                    g.color = RED
                    self._left_rotate(g)
        self.root.color = BLACK

    # ---------- deletion ----------
    def _transplant(self, u, v):
        if u.parent is self.NIL:
            self.root = v
        elif u is u.parent.left:
            u.parent.left = v
        else:
            u.parent.right = v
        v.parent = u.parent          # may write to the sentinel; intended

    def delete(self, key):
        z = self.search(key)
        if z is None:
            raise KeyError(key)
        y = z
        y_original_color = y.color
        if z.left is self.NIL:
            x = z.right
            self._transplant(z, z.right)
        elif z.right is self.NIL:
            x = z.left
            self._transplant(z, z.left)
        else:
            y = self._minimum(z.right)
            y_original_color = y.color
            x = y.right
            if y.parent is z:
                x.parent = y
            else:
                self._transplant(y, y.right)
                y.right = z.right
                y.right.parent = y
            self._transplant(z, y)
            y.left = z.left
            y.left.parent = y
            y.color = z.color
        if y_original_color == BLACK:
            self._delete_fixup(x)

    def _delete_fixup(self, x):
        while x is not self.root and x.color == BLACK:
            if x is x.parent.left:
                w = x.parent.right
                if w.color == RED:                                  # case 1
                    w.color = BLACK
                    x.parent.color = RED
                    self._left_rotate(x.parent)
                    w = x.parent.right
                if w.left.color == BLACK and w.right.color == BLACK:  # case 2
                    w.color = RED
                    x = x.parent
                else:
                    if w.right.color == BLACK:                      # case 3
                        w.left.color = BLACK
                        w.color = RED
                        self._right_rotate(w)
                        w = x.parent.right
                    w.color = x.parent.color                        # case 4
                    x.parent.color = BLACK
                    w.right.color = BLACK
                    self._left_rotate(x.parent)
                    x = self.root
            else:                                                   # mirror image
                w = x.parent.left
                if w.color == RED:
                    w.color = BLACK
                    x.parent.color = RED
                    self._right_rotate(x.parent)
                    w = x.parent.left
                if w.right.color == BLACK and w.left.color == BLACK:
                    w.color = RED
                    x = x.parent
                else:
                    if w.left.color == BLACK:
                        w.right.color = BLACK
                        w.color = RED
                        self._left_rotate(w)
                        w = x.parent.left
                    w.color = x.parent.color
                    x.parent.color = BLACK
                    w.left.color = BLACK
                    self._right_rotate(x.parent)
                    x = self.root
        x.color = BLACK

    # ---------- validation ----------
    def check(self):
        """Return the black-height; raise AssertionError on any violation."""
        assert self.root.color == BLACK, "root must be black"

        def walk(node, lo, hi):
            if node is self.NIL:
                return 1
            assert (lo is None or node.key > lo) and (hi is None or node.key < hi), "BST order"
            if node.color == RED:
                assert node.left.color == BLACK and node.right.color == BLACK, "red-red"
            lh = walk(node.left, lo, node.key)
            rh = walk(node.right, node.key, hi)
            assert lh == rh, "unequal black-heights"
            return lh + (node.color == BLACK)

        return walk(self.root, None, None)


if __name__ == "__main__":
    import random

    t = RedBlackTree()
    for k in [10, 20, 30, 15, 25, 12, 5, 1, 3]:
        t.insert(k)
        t.check()
    assert t.root.key == 12
    for k in [3, 12, 1, 5, 10]:
        t.delete(k)
        t.check()
    assert t.root.key == 25

    keys = random.sample(range(100_000), 5_000)
    for k in keys:
        t.insert(k) if t.search(k) is None else None
    random.shuffle(keys)
    for k in keys[:2_500]:
        if t.search(k):
            t.delete(k)
        t.check()
    print("all invariants hold")
```

The `__main__` block replays the two traces from this document and checks that the roots come out as 12 and then 25. It then stress-tests random inserts and deletes, validating every invariant after each delete.

### Implementation notes

- **The sentinel matters.** In delete, x is often NIL, and the fixup reads `x.parent`. `_transplant` writes `v.parent = u.parent` even when v is the sentinel. That temporary write is what lets the fixup find the right parent. Implementations that use `None` for leaves have to track the parent of x separately.
- **Duplicates.** This code sends equal keys to the right on insert, but `search` and `delete` find the first match. For a map, check for an existing key and update its value instead of inserting a second node.
- **Augmentation.** Because every structural change is a rotation, it is easy to keep subtree data such as size, sum or max endpoint. Recompute it for the two nodes involved in each rotation, and along the path to the root on insert and delete. This is how order-statistic trees and interval trees are built.

---

## 7. Comparison and summary

| Operation | Time | Rotations (worst case) |
|---|---|---|
| Search | O(log n) | 0 |
| Insert | O(log n) | 2 |
| Delete | O(log n) | 3 |
| Height | ≤ 2·log₂(n+1) | — |

Compared with **AVL trees**, red-black trees are less strictly balanced. AVL height is about 1.44·log₂ n, against up to 2·log₂ n here. So lookups in an AVL tree can be slightly faster. But an AVL delete can trigger rotations at every level, while red-black updates use a constant number of rotations. That makes red-black trees a better fit for update-heavy workloads and for structures where rotations are costly, for example when nodes carry augmented data. **Left-leaning red-black trees** (Sedgewick) restrict red links to lean left, which roughly halves the number of cases at some cost in constant factors. **B-trees** apply the same 2-3-4 idea with much wider nodes, which suits disks and caches.

To summarize:

- **Five rules.** The rule that no red node has a red child, plus the equal-black-count rule, together guarantee logarithmic height.
- **Insertion.** New nodes are red. Fix a double red by recoloring when the uncle is red (and moving up), or by one or two rotations when the uncle is black (and stopping).
- **Deletion.** Removing a black node leaves an "extra black." Push it up by recoloring the sibling, or absorb it with at most three rotations.

Every case is a local, constant-time change. All of the cleverness lies in proving that those local changes are always enough.
