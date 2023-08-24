# Tree

In computer science, a tree is a widely used abstract data type that **represents a hierarchical tree structure with a set of connected nodes. Each node in the tree can be connected to many children (depending on the type of tree), but must be connected to exactly one parent,[1] except for the root node, which has no parent** (i.e., the root node as the top-most node in the tree hierarchy). These constraints mean **there are no cycles or "loops" (no node can be its own ancestor), and also that each child can be treated like the root node of its own subtree**, making recursion a useful technique for tree traversal. In contrast to linear data structures, many trees cannot be represented by relationships between neighboring nodes (parent and children nodes of a node under consideration if they exists) in a single straight line (called edge or link between two adjacent nodes).

Binary trees are a commonly used type, which constrain the number of children for each parent to at most two. When the order of the children is specified, this data structure corresponds to an ordered tree in graph theory. A value or pointer to other data may be associated with every node in the tree, or sometimes only with the leaf nodes, which have no children nodes.

A node is a structure which may contain data and connections to other nodes, sometimes called edges or links. Each node in a tree has zero or more child nodes, which are below it in the tree (by convention, trees are drawn with descendants going downwards). A node that has a child is called the child's parent node (or superior). All nodes have exactly one parent, except the topmost root node, which has none. A node might have many ancestor nodes, such as the parent's parent. Child nodes with the same parent are sibling nodes. Typically siblings have an order, with the first one conventionally drawn on the left. Some definitions allow a tree to have no nodes at all, in which case it is called empty.

An internal node (also known as an inner node, inode for short, or branch node) is any node of a tree that has child nodes. Similarly, an external node (also known as an outer node, leaf node, or terminal node) is any node that does not have child nodes.

The height of a node is the length of the longest downward path to a leaf from that node. The height of the root is the height of the tree. The depth of a node is the length of the path to its root (i.e., its root path). Thus the root node has depth zero, leaf nodes have height zero, and a tree with only a single node (hence both a root and leaf) has depth and height zero. Conventionally, an empty tree (tree with no nodes, if such are allowed) has height −1.

Each non-root node can be treated as the root node of its own subtree, which includes that node and all its descendants.[a][2]

## binary trees

Tree terminology is not well-standardized and so varies in literatures.

A rooted binary tree has a root node and every node has **at most** two children.

- **A full binary tree** (sometimes referred to as a proper[15] or plane or strict binary tree)[16][17] is a tree in which every node has **either 0 or 2** children. Another way of defining a full binary tree is a recursive definition. A full binary tree is either:[11]
  - A single vertex (a single node as the root node).
  - A tree whose root node has two subtrees, both of which are full binary trees.

     A
   /   \
  B     C
 / \   
D   E 

- **A perfect binary** tree is a binary tree in which **all interior nodes have two children and all leaves have the same depth or same level** (the level of a node defined as the number of edges or links from the root node to a node).[18] An example of a perfect binary tree is the ancestry chart of a person to a given depth less than that at which an ancestor would appear more than once in the chart (at which point the chart is no longer a tree with unique nodes; note that the same ancestor can appear at different depths in the chart), as each person has exactly two biological parents (one mother and one father). Provided the ancestry chart always displays the mother and the father on the same side for a given node, their sex can be seen as an analogy of left and right children, children being understood here as an algorithmic term. **A perfect binary tree is a full binary tree, but the converse of this is not necessarily true**.
- **A complete binary** tree is a binary tree in which **every level, except possibly the last, is completely filled**, and **all nodes in the last level are as far left as possible**. It can have between 1 and 2h nodes at the last level h.[19] A perfect tree is therefore always complete but a complete tree is not necessarily perfect. Some authors use the term complete to refer instead to a perfect binary tree as defined above, in which case they call this type of tree (with a possibly not filled last level) an almost complete binary tree or nearly complete binary tree.[20][21] A complete binary tree can be efficiently represented using an array.[19]

     A
   /   \
  B     C
 / \   /
D   E F

- **A balanced binary tree** is a binary tree structure in which the **left and right subtrees of every node differ in height (the number of edges from the top-most node to the farthest node in a subtree) by no more than 1**.[22] One may also consider binary trees where **no leaf is much farther away from the root than any other leaf**. (Different balancing schemes allow different definitions of "much farther".[23])
- **A degenerate (or pathological) tree** is where each parent node has only one associated child node.[24] This means that the tree will behave like a linked list data structure. In this case, an advantage of using a binary tree is significantly reduced because it is essentially a linked list which time complexity is O(n) (n as the number of nodes) and it has more data space than the linked list due to two pointers per node, while the complexity of O(log2n) for data search in a balanced binary tree is normally expected.

## Properties of Binary Tree

## Binary search tree

In computer science, a binary search tree (BST), also called an ordered or sorted binary tree, is a rooted binary tree data structure with the key of each internal node being greater than all the keys in the respective node's left subtree and less than the ones in its right subtree. The time complexity of operations on the binary search tree is directly proportional to the height of the tree.


