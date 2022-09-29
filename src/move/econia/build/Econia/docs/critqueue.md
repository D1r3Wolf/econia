
<a name="0xc0deb00c_critqueue"></a>

# Module `0xc0deb00c::critqueue`

Crit-queue: A hybrid between a crit-bit tree and a queue.

A crit-queue contains an inner crit-bit tree with a sub-queue at
each leaf node, which can store multiple instances of the same
insertion key. Each such instance is stored in its own
sub-queue node at a crit-bit tree leaf corresponding to the given
insertion key, with multiple instances of the same insertion key
sorted by order of insertion within a sub-queue. Different
insertion keys, corresponding to different leaves in the crit-bit
tree, are sorted in lexicographical order, with the effect that
individual insertion key instances can be dequeued from the
crit-queue in:

1. Either ascending or descending order of insertion key (with
polarity set upon initialization), then by
2. Ascending order of insertion within a sub-queue.

Like a crit-bit tree, a crit-queue allows for insertions and
removals from anywhere inside the data structure, and can similarly
be used as an associative array that maps from keys to values, as in
the present implementation.

The present implementation, based on hash tables, offers:

* Insertions that are $O(1)$ in the best case, $O(log_2(n))$ in the
intermediate case, and parallelizable in the general case.
* Removals that are always $O(1)$, and parallelizable in the general
case.
* Iterable dequeues that are always $O(1)$.


<a name="@General_overview_sections_0"></a>

## General overview sections


[Bit conventions](#bit-conventions)

* [Number](#number)
* [Status](#status)
* [Masking](#masking)

[Crit-bit trees](#crit-bit-trees)

* [General](#general)
* [Structure](#structure)
* [Insertions](#insertions)
* [Removals](#removals)
* [As a map](#as-a-map)
* [References](#references)

[Crit-queues](#crit-queues)

* [Key storage multiplicity](#key-storage-multiplicity)
* [Sorting order](#sorting-order)
* [Leaves](#leaves)
* [Sub-queue nodes](#sub-queue-nodes)
* [Inner keys](#inner-keys)
* [Insertion counts](#insertion-counts)
* [Dequeue order preservation](#dequeue-order-preservation)
* [Sub-queue removal updates](#sub-queue-removal-updates)
* [Free leaves](#free-leaves)
* [Dequeues](#dequeues)

[Implementation analysis](#implementation-analysis)

* [Core functionality](#core-functionality)
* [Inserting](#inserting)
* [Removing](#removing)
* [Dequeuing](#dequeuing)

[Functions](#functions)

* [Public function index](#public-function-index)
* [Dependency charts](#dependency-charts)

[Complete docgen index](#complete-docgen-index)


<a name="@Bit_conventions_1"></a>

## Bit conventions



<a name="@Number_2"></a>

### Number


Bit numbers are 0-indexed from the least-significant bit (LSB):

>     11101...1010010101
>       bit 5 = 0 ^    ^ bit 0 = 1


<a name="@Status_3"></a>

### Status


<code>0</code> is considered an "unset" bit, and <code>1</code> is considered a "set" bit.
Hence <code>11101</code> is set at bit 0 and unset at bit 1.


<a name="@Masking_4"></a>

### Masking


In the present implementation, a bitmask refers to a bitstring that
is only set at the indicated bit. For example, a bitmask with bit 0
set corresponds to <code>000...001</code>, and a bitmask with bit 3 set
corresponds to <code>000...01000</code>.


<a name="@Crit-bit_trees_5"></a>

## Crit-bit trees



<a name="@General_6"></a>

### General


A critical bit (crit-bit) tree is a compact binary prefix tree
that stores a prefix-free set of bitstrings, like n-bit integers or
variable-length 0-terminated byte strings. For a given set of keys
there exists a unique crit-bit tree representing the set, such that
crit-bit trees do not require complex rebalancing algorithms like
those of AVL or red-black binary search trees. Crit-bit trees
support the following operations:

* Membership testing
* Insertion
* Deletion
* Inorder predecessor iteration
* Inorder successor iteration


<a name="@Structure_7"></a>

### Structure


Crit-bit trees have two types of nodes: inner nodes, and leaf nodes.
Inner nodes have two leaf children each, and leaf nodes do not
have children. Inner nodes store a bitmask set at the node's
critical bit (crit-bit), which indicates the most-significant bit of
divergence between keys from the node's two subtrees: keys in an
inner node's left subtree are unset at the critical bit, while
keys in an inner node's right subtree are set at the critical bit.

Inner nodes are arranged hierarchically, with the most-significant
critical bits at the top of the tree. For example, the binary keys
<code>001</code>, <code>101</code>, <code>110</code>, and <code>111</code> produce the following crit-bit tree:

>        2nd
>       /   \
>     001   1st
>          /   \
>        101   0th
>             /   \
>           110   111

Here, the inner node marked <code>2nd</code> stores a bitmask set at bit 2, the
inner node marked <code>1st</code> stores a bitmask set at bit 1, and the inner
node marked <code>0th</code> stores a bitmask set at bit 0. Hence, the sole key
in the left subtree of <code>2nd</code> is unset at bit 2, while all the keys
in the right subtree of <code>2nd</code> are set at bit 2. And similarly for
<code>0th</code>, the key of its left child is unset at bit 0, while the key of
its right child is set at bit 0.


<a name="@Insertions_8"></a>

### Insertions


Crit-bit trees are automatically sorted upon insertion, such that
inserting <code>111</code> to

>        2nd
>       /   \
>     001   1st
>          /   \
>        101    110

produces:

>                    2nd
>                   /   \
>                 001   1st <- has new right child
>                      /   \
>                    101   0th <- new inner node
>                         /   \
>     has new parent -> 110   111 <- new leaf

Here, <code>111</code> may not be re-inserted unless it is first removed from
the tree.


<a name="@Removals_9"></a>

### Removals


Continuing the above example, crit-bit trees are automatically
compacted and sorted upon removal, such that removing <code>111</code> again
results in:

>        2nd
>       /   \
>     001   1st <- has new right child
>          /   \
>        101    110 <- has new parent


<a name="@As_a_map_10"></a>

### As a map


Crit-bit trees can be used as an associative array that maps from
keys to values, simply by storing values in the leaves of the tree.
For example, the insertion sequence

1. $\langle \texttt{0b001}, v_0 \rangle$
2. $\langle \texttt{0b111}, v_1 \rangle$
3. $\langle \texttt{0b110}, v_2 \rangle$
4. $\langle \texttt{0b101}, v_3 \rangle$

produces the following tree:

>                2nd
>               /   \
>     <001, v_0>    1st
>                  /   \
>        <101, v_3>    0th
>                     /   \
>           <110, v_2>     <111, v_1>


<a name="@References_11"></a>

### References


* [Bernstein 2004] (Earliest identified author)
* [Langley 2008] (Primary reference for this implementation)
* [Langley 2012]
* [Tcler's Wiki 2021]

[Bernstein 2004]:
https://cr.yp.to/critbit.html
[Langley 2008]:
https://www.imperialviolet.org/2008/09/29/critbit-trees.html
[Langley 2012]:
https://github.com/agl/critbit
[Tcler's Wiki 2021]:
https://wiki.tcl-lang.org/page/critbit


<a name="@Crit-queues_12"></a>

## Crit-queues



<a name="@Key_storage_multiplicity_13"></a>

### Key storage multiplicity


Unlike a crit-bit tree, which can only store one instance of a given
key, crit-queues can store multiple instances. For example, the
following insertion sequence, without intermediate removals, is
invalid in a crit-bit tree but valid in a crit-queue:

1. $p_{3, 0} = \langle 3, 5 \rangle$
2. $p_{3, 1} = \langle 3, 8 \rangle$
3. $p_{3, 2} = \langle 3, 2 \rangle$
4. $p_{3, 3} = \langle 3, 5 \rangle$

Here, the "key-value insertion pair"
$p_{i, j} = \langle i, v_j \rangle$ has:

* "Insertion key" $i$: the inserted key.
* "Insertion count" $j$: the number of key-value insertion pairs,
having the same insertion key, that were previously inserted.
* "Insertion value" $v_j$: the value from the key-value
insertion pair having insertion count $j$.


<a name="@Sorting_order_14"></a>

### Sorting order


Key-value insertion pairs in a crit-queue are sorted by:

1. Either ascending or descending order of insertion key, then by
2. Ascending order of insertion count.

For example, consider the following binary insertion key sequence,
where $k_{i, j}$ denotes insertion key $i$ with insertion count $j$:

1. $k_{0, 0} = \texttt{0b00}$
2. $k_{1, 0} = \texttt{0b01}$
3. $k_{1, 1} = \texttt{0b01}$
4. $k_{0, 1} = \texttt{0b00}$
5. $k_{3, 0} = \texttt{0b11}$

In an ascending crit-queue, the dequeue sequence would be:

1. $k_{0, 0} = \texttt{0b00}$
2. $k_{0, 1} = \texttt{0b00}$
3. $k_{1, 0} = \texttt{0b01}$
4. $k_{1, 1} = \texttt{0b01}$
5. $k_{3, 0} = \texttt{0b11}$

In a descending crit-queue, the dequeue sequence would instead be:

1. $k_{3, 0} = \texttt{0b11}$
2. $k_{1, 0} = \texttt{0b01}$
3. $k_{1, 1} = \texttt{0b01}$
4. $k_{0, 0} = \texttt{0b00}$
5. $k_{0, 1} = \texttt{0b00}$


<a name="@Leaves_15"></a>

### Leaves


The present crit-queue implementation involves a crit-bit tree with
a leaf node for each insertion key, where each "leaf key" has the
following bit structure:

| Bit(s) | Value         |
|--------|---------------|
| 64-127 | Insertion key |
| 0-63   | 0             |

Continuing the above example:

| Insertion key   | Leaf key bits 64-127 | Leaf key bits 0-63 |
|-----------------|----------------------|--------------------|
| <code>0</code> = <code>0b00</code>    | <code>000...000</code>          | <code>000...000</code>        |
| <code>1</code> = <code>0b01</code>    | <code>000...001</code>          | <code>000...000</code>        |
| <code>3</code> = <code>0b11</code>    | <code>000...011</code>          | <code>000...000</code>        |

Each leaf contains a nested sub-queue of key-value insertion pairs
all sharing the corresponding insertion key, with lower insertion
counts at the front of the queue. Continuing the above example,
this yields the following:

>                                   65th
>                                  /    \
>                              64th      000...011000...000
>                             /    \     [k_{3, 0}]
>                            /      \
>          000...000000...000        000...001000...000
>      [k_{0, 0} -> k_{0, 1}]        [k_{1, 0} -> k_{1, 1}]
>       ^ sub-queue head              ^ sub-queue head

Leaf keys are guaranteed to be unique, and all leaf nodes are stored
in a single hash table.


<a name="@Sub-queue_nodes_16"></a>

### Sub-queue nodes


All sub-queue nodes are similarly stored in single hash table, and
assigned a unique "access key" with the following bit structure
(<code>NOT</code> denotes bitwise complement):

| Bit(s) | Ascending crit-queue | Descending crit-queue |
|--------|----------------------|-----------------------|
| 64-127 | Insertion key        | Insertion key         |
| 63     | 0                    | 0                     |
| 62     | 0                    | 1                     |
| 0-61   | Insertion count      | <code>NOT</code> insertion count |

For an ascending crit-queue, access keys are thus dequeued in
ascending lexicographical order:

| Insertion key | Access key bits 64-127 | Access key bits 0-63 |
|---------------|------------------------|----------------------|
| $k_{0, 0}$    | <code>000...000</code>            | <code>000...000</code>          |
| $k_{0, 1}$    | <code>000...000</code>            | <code>000...001</code>          |
| $k_{1, 0}$    | <code>000...001</code>            | <code>000...000</code>          |
| $k_{1, 1}$    | <code>000...001</code>            | <code>000...001</code>          |
| $k_{3, 0}$    | <code>000...011</code>            | <code>000...000</code>          |

Conversely, for a descending crit-queue, access keys are thus
dequeued in descending lexicographical order:

| Insertion key | Access key bits 64-127 | Access key bits 0-63 |
|---------------|----------------------|------------------------|
| $k_{3, 0}$    | <code>000...011</code>          | <code>011...111</code>            |
| $k_{1, 0}$    | <code>000...001</code>          | <code>011...111</code>            |
| $k_{1, 1}$    | <code>000...001</code>          | <code>011...110</code>            |
| $k_{0, 0}$    | <code>000...000</code>          | <code>011...111</code>            |
| $k_{0, 1}$    | <code>000...000</code>          | <code>011...110</code>            |


<a name="@Inner_keys_17"></a>

### Inner keys


After access key assignment, if the insertion of a key-value
insertion pair requires the creation of a new inner node, the inner
node is assigned a unique "inner key" that is identical to the new
access key, except with bit 63 set. This schema allows for
discrimination between inner keys and leaf keys based solely on bit
63, and guarantees that no two inner nodes share the same inner key.

All inner nodes are stored in a single hash table.


<a name="@Insertion_counts_18"></a>

### Insertion counts


Insertion counts are tracked in leaf nodes, such that before the
insertion of the first instance of a given insertion key,
$k_{i, 0}$, the leaf table does not have an entry corresponding
to insertion key $i$.

When $k_{i, 0}$ is inserted, a new leaf node is initialized with
an insertion counter set to 0, then added to the leaves hash table.
The new leaf node is inserted to the crit-bit tree, and a
corresponding sub-queue node is placed at the head of the new leaf's
sub-queue. For each subsequent insertion of the same insertion key,
$k_{i, n}$, the leaf insertion counter is updated to $n$, and the
new sub-queue node becomes the tail of the corresponding sub-queue.

Since bits 62 and 63 in access keys are reserved for flag bits, the
maximum insertion count per insertion key is thus $2^{62} - 1$.


<a name="@Dequeue_order_preservation_19"></a>

### Dequeue order preservation


Removals can take place from anywhere inside of a crit-queue, with
the specified dequeue order preserved among remaining elements.
For example, consider the elements in an ascending crit-queue
with the following dequeue sequence:

1. $k_{0, 6}$
2. $k_{2, 5}$
3. $k_{2, 8}$
4. $k_{4, 5}$
5. $k_{5, 0}$

Here, removing $k_{2, 5}$ simply updates the dequeue sequence to:

1. $k_{0, 6}$
2. $k_{2, 8}$
3. $k_{4, 5}$
4. $k_{5, 0}$


<a name="@Sub-queue_removal_updates_20"></a>

### Sub-queue removal updates


Removal via access key lookup in the sub-queue node hash table leads
to an update within the corresponding sub-queue.

For example, consider the following crit-queue:

>                                          64th
>                                         /    \
>                       000...000000...000      000...001000...000
>       [k_{0, 0} -> k_{0, 1} -> k_{0, 2}]      [k_{1, 0}]
>        ^ sub-queue head

Removal of $k_{0, 1}$ produces:

>                             64th
>                            /    \
>          000...000000...000      000...001000...000
>      [k_{0, 0} -> k_{0, 2}]      [k_{1, 0}]

And similarly for $k_{0, 0}$:

>                        64th
>                       /    \
>     000...000000...000      000...001000...000
>             [k_{0, 2}]      [k_{1, 0}]

Here, if ${k_{0, 2}}$ were to be removed, the tree would then have a
single leaf at its root:

>     000...001000...000 (root)
>         [k_{1, 0}]

Notably, however, the leaf corresponding to insertion key 0 is not
deallocated, but rather, is converted to a "free leaf" with an
empty sub-queue.


<a name="@Free_leaves_21"></a>

### Free leaves


Free leaves are leaf nodes with an empty sub-queue.

Free leaves track insertion counts in case another key-value
insertion pair, having the insertion key encoded in the free leaf
key, is inserted. Here, the free leaf is added back to the crit-bit
tree and the new sub-queue node becomes the head of the leaf's
sub-queue. Continuing the example, inserting another key-value pair
with insertion key 0, $k_{0, 3}$, produces:

>                        64th
>                       /    \
>     000...000000...000      000...001000...000
>             [k_{0, 3}]      [k_{1, 0}]


<a name="@Dequeues_22"></a>

### Dequeues


Dequeues are processed as removals from the crit-queue head, a field
that stores:

* The minimum access key in an ascending crit-queue, or
* The maximum access key in a descending crit-queue.

After all elements in the corresponding sub-queue have been dequeued
in order of ascending insertion count, dequeuing proceeds with the
head of the sub-queue in the next leaf, which is accessed by either:

* Inorder successor traversal if an ascending crit-queue, or
* Inorder predecessor traversal if a descending crit-queue.


<a name="@Implementation_analysis_23"></a>

## Implementation analysis



<a name="@Core_functionality_24"></a>

### Core functionality


In the present implementation, key-value insertion pairs are
inserted via <code><a href="critqueue.md#0xc0deb00c_critqueue_insert">insert</a>()</code>, which accepts a <code>u64</code> insertion key and an
insertion value of type <code>V</code>. A corresponding <code>u128</code> access key is
returned, which can be used for subsequent access key lookup via
<code><a href="critqueue.md#0xc0deb00c_critqueue_borrow">borrow</a>()</code>, <code><a href="critqueue.md#0xc0deb00c_critqueue_borrow_mut">borrow_mut</a>()</code>, or <code><a href="critqueue.md#0xc0deb00c_critqueue_remove">remove</a>()</code>, with the latter returning
the corresponding insertion value after it has been removed from the
crit-queue. No access key is required for <code><a href="critqueue.md#0xc0deb00c_critqueue_dequeue">dequeue</a>()</code>, which simply
removes and returns the insertion value at the head of the
crit-queue, if there is one.


<a name="@Inserting_25"></a>

### Inserting


Insertions are, like a crit-bit tree, $O(k)$ in the worst case,
where $k = 64$ (the number of variable bits in an insertion key),
since a new leaf node has to be inserted into the crit-bit tree.
In the intermediate case where a new leaf node has to be inserted
into the crit-bit tree but the tree is generally balanced,
insertions improve to $O(log_2(n))$, where $n$ is the number of
leaves in the tree. In the best case, where the corresponding
sub-queue already has a leaf in the crit-bit tree and a new
sub-queue node simply has to be inserted at the tail of the
sub-queue, insertions improve to $O(1)$.

Insertions are parallelizable in the general case where:

1. They do not alter the head of the crit-queue.
2. They do not write to overlapping crit-bit tree edges.
3. They do not write to overlapping sub-queue edges.
4. They alter neither the head nor the tail of the same sub-queue.
5. They do not write to the same sub-queue.

The final parallelism constraint is a result of insertion count
updates, and may potentially be eliminated in the case of a
parallelized insertion count aggregator.


<a name="@Removing_26"></a>

### Removing


With sub-queue nodes stored in a hash table, removal operations via
access key are are thus $O(1)$, and are parallelizable in the
general case where:

1. They do not alter the head of the crit-queue.
2. They do not write to overlapping crit-bit tree edges.
3. They do not write to overlapping sub-queue edges.
4. They alter neither the head nor the tail of the same sub-queue.


<a name="@Dequeuing_27"></a>

### Dequeuing


Dequeues are iterable, and, as a form of removal, are $O(1)$, but
since they alter the head of the queue, they are not parallelizable.


<a name="@Functions_28"></a>

## Functions



<a name="@Public_function_index_29"></a>

### Public function index


* <code><a href="critqueue.md#0xc0deb00c_critqueue_borrow">borrow</a>()</code>
* <code><a href="critqueue.md#0xc0deb00c_critqueue_borrow_mut">borrow_mut</a>()</code>
* <code><a href="critqueue.md#0xc0deb00c_critqueue_dequeue">dequeue</a>()</code>
* <code><a href="critqueue.md#0xc0deb00c_critqueue_get_head_access_key">get_head_access_key</a>()</code>
* <code><a href="critqueue.md#0xc0deb00c_critqueue_has_access_key">has_access_key</a>()</code>
* <code><a href="critqueue.md#0xc0deb00c_critqueue_insert">insert</a>()</code>
* <code><a href="critqueue.md#0xc0deb00c_critqueue_is_empty">is_empty</a>()</code>
* <code><a href="critqueue.md#0xc0deb00c_critqueue_new">new</a>()</code>
* <code><a href="critqueue.md#0xc0deb00c_critqueue_remove">remove</a>()</code>
* <code><a href="critqueue.md#0xc0deb00c_critqueue_would_become_new_head">would_become_new_head</a>()</code>
* <code><a href="critqueue.md#0xc0deb00c_critqueue_would_trail_head">would_trail_head</a>()</code>


<a name="@Dependency_charts_30"></a>

### Dependency charts


The below dependency charts use <code>mermaid.js</code> syntax, which can be
automatically rendered into a diagram (depending on the browser)
when viewing the documentation file generated from source code. If
a browser renders the diagrams with coloring that makes it difficult
to read, try a different browser.

* <code><a href="critqueue.md#0xc0deb00c_critqueue_insert">insert</a>()</code>:

```mermaid

flowchart LR

insert --> insert_update_subqueue
insert --> insert_allocate_leaf
insert --> insert_check_head
insert --> insert_leaf

insert_leaf --> search
insert_leaf --> get_critical_bitmask
insert_leaf --> insert_leaf_above_root_node
insert_leaf --> insert_leaf_below_anchor_node

```

* <code><a href="critqueue.md#0xc0deb00c_critqueue_dequeue">dequeue</a>()</code> and <code><a href="critqueue.md#0xc0deb00c_critqueue_remove">remove</a>()</code>:

```mermaid

flowchart LR

dequeue --> remove

remove --> remove_subqueue_node
remove --> traverse
remove --> remove_free_leaf

```


<a name="@Complete_docgen_index_31"></a>

## Complete docgen index


The below index is automatically generated from source code:


-  [General overview sections](#@General_overview_sections_0)
-  [Bit conventions](#@Bit_conventions_1)
    -  [Number](#@Number_2)
    -  [Status](#@Status_3)
    -  [Masking](#@Masking_4)
-  [Crit-bit trees](#@Crit-bit_trees_5)
    -  [General](#@General_6)
    -  [Structure](#@Structure_7)
    -  [Insertions](#@Insertions_8)
    -  [Removals](#@Removals_9)
    -  [As a map](#@As_a_map_10)
    -  [References](#@References_11)
-  [Crit-queues](#@Crit-queues_12)
    -  [Key storage multiplicity](#@Key_storage_multiplicity_13)
    -  [Sorting order](#@Sorting_order_14)
    -  [Leaves](#@Leaves_15)
    -  [Sub-queue nodes](#@Sub-queue_nodes_16)
    -  [Inner keys](#@Inner_keys_17)
    -  [Insertion counts](#@Insertion_counts_18)
    -  [Dequeue order preservation](#@Dequeue_order_preservation_19)
    -  [Sub-queue removal updates](#@Sub-queue_removal_updates_20)
    -  [Free leaves](#@Free_leaves_21)
    -  [Dequeues](#@Dequeues_22)
-  [Implementation analysis](#@Implementation_analysis_23)
    -  [Core functionality](#@Core_functionality_24)
    -  [Inserting](#@Inserting_25)
    -  [Removing](#@Removing_26)
    -  [Dequeuing](#@Dequeuing_27)
-  [Functions](#@Functions_28)
    -  [Public function index](#@Public_function_index_29)
    -  [Dependency charts](#@Dependency_charts_30)
-  [Complete docgen index](#@Complete_docgen_index_31)
-  [Struct `CritQueue`](#0xc0deb00c_critqueue_CritQueue)
-  [Struct `Inner`](#0xc0deb00c_critqueue_Inner)
-  [Struct `Leaf`](#0xc0deb00c_critqueue_Leaf)
-  [Struct `SubQueueNode`](#0xc0deb00c_critqueue_SubQueueNode)
-  [Constants](#@Constants_32)
-  [Function `borrow`](#0xc0deb00c_critqueue_borrow)
    -  [Testing](#@Testing_33)
-  [Function `borrow_mut`](#0xc0deb00c_critqueue_borrow_mut)
    -  [Testing](#@Testing_34)
-  [Function `dequeue`](#0xc0deb00c_critqueue_dequeue)
    -  [Example](#@Example_35)
    -  [Testing](#@Testing_36)
-  [Function `get_head_access_key`](#0xc0deb00c_critqueue_get_head_access_key)
    -  [Testing](#@Testing_37)
-  [Function `has_access_key`](#0xc0deb00c_critqueue_has_access_key)
    -  [Testing](#@Testing_38)
-  [Function `insert`](#0xc0deb00c_critqueue_insert)
    -  [Parameters](#@Parameters_39)
    -  [Returns](#@Returns_40)
    -  [Reference diagrams](#@Reference_diagrams_41)
        -  [Conventions](#@Conventions_42)
        -  [Insertion sequence](#@Insertion_sequence_43)
    -  [Testing](#@Testing_44)
-  [Function `is_empty`](#0xc0deb00c_critqueue_is_empty)
    -  [Testing](#@Testing_45)
-  [Function `new`](#0xc0deb00c_critqueue_new)
-  [Function `remove`](#0xc0deb00c_critqueue_remove)
    -  [Reference diagrams](#@Reference_diagrams_46)
        -  [Conventions](#@Conventions_47)
        -  [Insertion preposition sequence](#@Insertion_preposition_sequence_48)
        -  [No sub-queue head update](#@No_sub-queue_head_update_49)
        -  [Sub-queue head update, no free leaf](#@Sub-queue_head_update,_no_free_leaf_50)
        -  [Free leaf generation 1](#@Free_leaf_generation_1_51)
        -  [Free leaf generation 2](#@Free_leaf_generation_2_52)
    -  [Testing](#@Testing_53)
-  [Function `would_become_new_head`](#0xc0deb00c_critqueue_would_become_new_head)
    -  [Testing](#@Testing_54)
-  [Function `would_trail_head`](#0xc0deb00c_critqueue_would_trail_head)
    -  [Testing](#@Testing_55)
-  [Function `get_critical_bitmask`](#0xc0deb00c_critqueue_get_critical_bitmask)
    -  [<code>XOR</code>/<code>AND</code> method](#@<code>XOR</code>/<code>AND</code>_method_56)
    -  [Binary search method](#@Binary_search_method_57)
    -  [Testing](#@Testing_58)
-  [Function `insert_allocate_leaf`](#0xc0deb00c_critqueue_insert_allocate_leaf)
    -  [Returns](#@Returns_59)
    -  [Assumptions](#@Assumptions_60)
    -  [Testing](#@Testing_61)
-  [Function `insert_check_head`](#0xc0deb00c_critqueue_insert_check_head)
    -  [Testing](#@Testing_62)
-  [Function `insert_leaf`](#0xc0deb00c_critqueue_insert_leaf)
    -  [Parameters](#@Parameters_63)
    -  [Assumptions](#@Assumptions_64)
    -  [Diagrams](#@Diagrams_65)
    -  [Testing](#@Testing_66)
-  [Function `insert_leaf_above_root_node`](#0xc0deb00c_critqueue_insert_leaf_above_root_node)
    -  [Parameters](#@Parameters_67)
    -  [Assumptions](#@Assumptions_68)
    -  [Reference diagrams](#@Reference_diagrams_69)
        -  [Inserting above a leaf](#@Inserting_above_a_leaf_70)
        -  [Inserting above an inner node](#@Inserting_above_an_inner_node_73)
    -  [Testing](#@Testing_76)
-  [Function `insert_leaf_below_anchor_node`](#0xc0deb00c_critqueue_insert_leaf_below_anchor_node)
    -  [Parameters](#@Parameters_77)
    -  [Assumptions](#@Assumptions_78)
    -  [Reference diagrams](#@Reference_diagrams_79)
        -  [Anchor node children polarity](#@Anchor_node_children_polarity_80)
        -  [Child displacement](#@Child_displacement_81)
        -  [New inner node children polarity](#@New_inner_node_children_polarity_82)
    -  [Testing](#@Testing_83)
-  [Function `insert_update_subqueue`](#0xc0deb00c_critqueue_insert_update_subqueue)
    -  [Returns](#@Returns_84)
    -  [Aborts](#@Aborts_85)
    -  [Assumptions](#@Assumptions_86)
    -  [Testing](#@Testing_87)
-  [Function `remove_free_leaf`](#0xc0deb00c_critqueue_remove_free_leaf)
    -  [Assumptions](#@Assumptions_88)
    -  [Removing the root](#@Removing_the_root_89)
    -  [Reference diagrams](#@Reference_diagrams_90)
        -  [With grandparent](#@With_grandparent_91)
        -  [Without grandparent](#@Without_grandparent_94)
    -  [Testing](#@Testing_97)
-  [Function `remove_subqueue_node`](#0xc0deb00c_critqueue_remove_subqueue_node)
    -  [Parameters](#@Parameters_98)
    -  [Returns](#@Returns_99)
    -  [Reference diagrams](#@Reference_diagrams_100)
        -  [Conventions](#@Conventions_101)
        -  [Removal sequence](#@Removal_sequence_102)
    -  [Testing](#@Testing_103)
-  [Function `search`](#0xc0deb00c_critqueue_search)
    -  [Returns](#@Returns_104)
    -  [Assumptions](#@Assumptions_105)
    -  [Reference diagrams](#@Reference_diagrams_106)
        -  [Leaf at root](#@Leaf_at_root_107)
        -  [Inner node at root](#@Inner_node_at_root_108)
    -  [Testing](#@Testing_109)
-  [Function `traverse`](#0xc0deb00c_critqueue_traverse)
    -  [Parameters](#@Parameters_110)
    -  [Returns](#@Returns_111)
    -  [Membership considerations](#@Membership_considerations_112)
    -  [Reference diagram](#@Reference_diagram_113)
        -  [Conventions](#@Conventions_114)
        -  [Inorder predecessor](#@Inorder_predecessor_115)
        -  [Inorder successor](#@Inorder_successor_116)
    -  [Testing](#@Testing_117)


<pre><code><b>use</b> <a href="">0x1::option</a>;
<b>use</b> <a href="">0x1::table</a>;
</code></pre>



<a name="0xc0deb00c_critqueue_CritQueue"></a>

## Struct `CritQueue`

Hybrid between a crit-bit tree and a queue. See above.


<pre><code><b>struct</b> <a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a>&lt;V&gt; <b>has</b> store
</code></pre>



<details>
<summary>Fields</summary>


<dl>
<dt>
<code>order: bool</code>
</dt>
<dd>
 Crit-queue sort order, <code><a href="critqueue.md#0xc0deb00c_critqueue_ASCENDING">ASCENDING</a></code> or <code><a href="critqueue.md#0xc0deb00c_critqueue_DESCENDING">DESCENDING</a></code>.
</dd>
<dt>
<code>root: <a href="_Option">option::Option</a>&lt;u128&gt;</code>
</dt>
<dd>
 Node key of crit-bit tree root. None if crit-queue is empty.
</dd>
<dt>
<code>head: <a href="_Option">option::Option</a>&lt;u128&gt;</code>
</dt>
<dd>
 Access key of crit-queue head. None if crit-queue is empty,
 else minimum access key if ascending crit-queue, and
 maximum access key if descending crit-queue.
</dd>
<dt>
<code>inners: <a href="_Table">table::Table</a>&lt;u128, <a href="critqueue.md#0xc0deb00c_critqueue_Inner">critqueue::Inner</a>&gt;</code>
</dt>
<dd>
 Map from inner key to inner node.
</dd>
<dt>
<code>leaves: <a href="_Table">table::Table</a>&lt;u128, <a href="critqueue.md#0xc0deb00c_critqueue_Leaf">critqueue::Leaf</a>&gt;</code>
</dt>
<dd>
 Map from leaf key to leaf node.
</dd>
<dt>
<code>subqueue_nodes: <a href="_Table">table::Table</a>&lt;u128, <a href="critqueue.md#0xc0deb00c_critqueue_SubQueueNode">critqueue::SubQueueNode</a>&lt;V&gt;&gt;</code>
</dt>
<dd>
 Map from access key to sub-queue node.
</dd>
</dl>


</details>

<a name="0xc0deb00c_critqueue_Inner"></a>

## Struct `Inner`

A crit-bit tree inner node.


<pre><code><b>struct</b> <a href="critqueue.md#0xc0deb00c_critqueue_Inner">Inner</a> <b>has</b> store
</code></pre>



<details>
<summary>Fields</summary>


<dl>
<dt>
<code>bitmask: u128</code>
</dt>
<dd>
 Bitmask set at critical bit.
</dd>
<dt>
<code>parent: <a href="_Option">option::Option</a>&lt;u128&gt;</code>
</dt>
<dd>
 If none, node is root. Else parent key.
</dd>
<dt>
<code>left: u128</code>
</dt>
<dd>
 Left child key.
</dd>
<dt>
<code>right: u128</code>
</dt>
<dd>
 Right child key.
</dd>
</dl>


</details>

<a name="0xc0deb00c_critqueue_Leaf"></a>

## Struct `Leaf`

A crit-bit tree leaf node. A free leaf if no sub-queue head.
Else the root of the crit-bit tree if no parent.


<pre><code><b>struct</b> <a href="critqueue.md#0xc0deb00c_critqueue_Leaf">Leaf</a> <b>has</b> store
</code></pre>



<details>
<summary>Fields</summary>


<dl>
<dt>
<code>count: u64</code>
</dt>
<dd>
 0-indexed insertion count for corresponding insertion key.
</dd>
<dt>
<code>parent: <a href="_Option">option::Option</a>&lt;u128&gt;</code>
</dt>
<dd>
 If no sub-queue head or tail, should also be none, since
 leaf is a free leaf. Else corresponds to the inner key of
 the leaf's parent node, none when leaf is the root of the
 crit-bit tree.
</dd>
<dt>
<code>head: <a href="_Option">option::Option</a>&lt;u128&gt;</code>
</dt>
<dd>
 If none, node is a free leaf. Else the access key of the
 sub-queue head.
</dd>
<dt>
<code>tail: <a href="_Option">option::Option</a>&lt;u128&gt;</code>
</dt>
<dd>
 If none, node is a free leaf. Else the access key of the
 sub-queue tail.
</dd>
</dl>


</details>

<a name="0xc0deb00c_critqueue_SubQueueNode"></a>

## Struct `SubQueueNode`

A node in a sub-queue.


<pre><code><b>struct</b> <a href="critqueue.md#0xc0deb00c_critqueue_SubQueueNode">SubQueueNode</a>&lt;V&gt; <b>has</b> store
</code></pre>



<details>
<summary>Fields</summary>


<dl>
<dt>
<code>insertion_value: V</code>
</dt>
<dd>
 Insertion value.
</dd>
<dt>
<code>previous: <a href="_Option">option::Option</a>&lt;u128&gt;</code>
</dt>
<dd>
 Access key of previous sub-queue node, if any.
</dd>
<dt>
<code>next: <a href="_Option">option::Option</a>&lt;u128&gt;</code>
</dt>
<dd>
 Access key of next sub-queue node, if any.
</dd>
</dl>


</details>

<a name="@Constants_32"></a>

## Constants


<a name="0xc0deb00c_critqueue_ACCESS_KEY_TO_INNER_KEY"></a>

<code>u128</code> bitmask set at bit 63, for converting an access key
to an inner key via bitwise <code>OR</code>. Generated in Python via
<code>hex(int('1' + '0' * 63, 2))</code>.


<pre><code><b>const</b> <a href="critqueue.md#0xc0deb00c_critqueue_ACCESS_KEY_TO_INNER_KEY">ACCESS_KEY_TO_INNER_KEY</a>: u128 = 0x8000000000000000;
</code></pre>



<a name="0xc0deb00c_critqueue_ACCESS_KEY_TO_LEAF_KEY"></a>

<code>u128</code> bitmask set at bits 64-127, for converting an access key
to a leaf key via bitwise <code>AND</code>. Generated in Python via
<code>hex(int('1' * 64 + '0' * 64, 2))</code>.


<pre><code><b>const</b> <a href="critqueue.md#0xc0deb00c_critqueue_ACCESS_KEY_TO_LEAF_KEY">ACCESS_KEY_TO_LEAF_KEY</a>: u128 = 0xffffffffffffffff0000000000000000;
</code></pre>



<a name="0xc0deb00c_critqueue_ASCENDING"></a>

Ascending crit-queue flag.


<pre><code><b>const</b> <a href="critqueue.md#0xc0deb00c_critqueue_ASCENDING">ASCENDING</a>: bool = <b>false</b>;
</code></pre>



<a name="0xc0deb00c_critqueue_DESCENDING"></a>

Descending crit-queue flag.


<pre><code><b>const</b> <a href="critqueue.md#0xc0deb00c_critqueue_DESCENDING">DESCENDING</a>: bool = <b>true</b>;
</code></pre>



<a name="0xc0deb00c_critqueue_E_TOO_MANY_INSERTIONS"></a>

When an insertion key has been inserted too many times.


<pre><code><b>const</b> <a href="critqueue.md#0xc0deb00c_critqueue_E_TOO_MANY_INSERTIONS">E_TOO_MANY_INSERTIONS</a>: u64 = 0;
</code></pre>



<a name="0xc0deb00c_critqueue_HI_128"></a>

<code>u128</code> bitmask with all bits set, generated in Python via
<code>hex(int('1' * 128, 2))</code>.


<pre><code><b>const</b> <a href="critqueue.md#0xc0deb00c_critqueue_HI_128">HI_128</a>: u128 = 0xffffffffffffffffffffffffffffffff;
</code></pre>



<a name="0xc0deb00c_critqueue_HI_64"></a>

<code>u64</code> bitmask with all bits set, generated in Python via
<code>hex(int('1' * 64, 2))</code>.


<pre><code><b>const</b> <a href="critqueue.md#0xc0deb00c_critqueue_HI_64">HI_64</a>: u64 = 0xffffffffffffffff;
</code></pre>



<a name="0xc0deb00c_critqueue_INSERTION_KEY"></a>

Number of bits that insertion key is shifted in a <code>u128</code> key.


<pre><code><b>const</b> <a href="critqueue.md#0xc0deb00c_critqueue_INSERTION_KEY">INSERTION_KEY</a>: u8 = 64;
</code></pre>



<a name="0xc0deb00c_critqueue_MAX_INSERTION_COUNT"></a>

Maximum number of times a given insertion key can be inserted.
A <code>u64</code> bitmask with all bits set except 62 and 63, generated
in Python via <code>hex(int('1' * 62, 2))</code>.


<pre><code><b>const</b> <a href="critqueue.md#0xc0deb00c_critqueue_MAX_INSERTION_COUNT">MAX_INSERTION_COUNT</a>: u64 = 0x3fffffffffffffff;
</code></pre>



<a name="0xc0deb00c_critqueue_MSB_u128"></a>

Most significant bit number for a <code>u128</code>


<pre><code><b>const</b> <a href="critqueue.md#0xc0deb00c_critqueue_MSB_u128">MSB_u128</a>: u8 = 127;
</code></pre>



<a name="0xc0deb00c_critqueue_NOT_INSERTION_COUNT_DESCENDING"></a>

<code>XOR</code> bitmask for flipping insertion count bits 0-61 and
setting bit 62 high in the case of a descending crit-queue.
<code>u64</code> bitmask with all bits set except bit 63, cast to a <code>u128</code>.
Generated in Python via <code>hex(int('1' * 63, 2))</code>.


<pre><code><b>const</b> <a href="critqueue.md#0xc0deb00c_critqueue_NOT_INSERTION_COUNT_DESCENDING">NOT_INSERTION_COUNT_DESCENDING</a>: u128 = 0x7fffffffffffffff;
</code></pre>



<a name="0xc0deb00c_critqueue_PREDECESSOR"></a>

Flag for inorder predecessor traversal.


<pre><code><b>const</b> <a href="critqueue.md#0xc0deb00c_critqueue_PREDECESSOR">PREDECESSOR</a>: bool = <b>true</b>;
</code></pre>



<a name="0xc0deb00c_critqueue_SUCCESSOR"></a>

Flag for inorder successor traversal.


<pre><code><b>const</b> <a href="critqueue.md#0xc0deb00c_critqueue_SUCCESSOR">SUCCESSOR</a>: bool = <b>false</b>;
</code></pre>



<a name="0xc0deb00c_critqueue_TREE_NODE_INNER"></a>

Result of bitwise crit-bit tree node key <code>AND</code> with
<code><a href="critqueue.md#0xc0deb00c_critqueue_TREE_NODE_TYPE">TREE_NODE_TYPE</a></code>, indicating that the key is set at bit 63 and
is thus an inner key. Generated in Python via
<code>hex(int('1' + '0' * 63, 2))</code>.


<pre><code><b>const</b> <a href="critqueue.md#0xc0deb00c_critqueue_TREE_NODE_INNER">TREE_NODE_INNER</a>: u128 = 0x8000000000000000;
</code></pre>



<a name="0xc0deb00c_critqueue_TREE_NODE_LEAF"></a>

Result of bitwise crit-bit tree node key <code>AND</code> with
<code><a href="critqueue.md#0xc0deb00c_critqueue_TREE_NODE_TYPE">TREE_NODE_TYPE</a></code>, indicating that the key is unset at bit 63
and is thus a leaf key.


<pre><code><b>const</b> <a href="critqueue.md#0xc0deb00c_critqueue_TREE_NODE_LEAF">TREE_NODE_LEAF</a>: u128 = 0;
</code></pre>



<a name="0xc0deb00c_critqueue_TREE_NODE_TYPE"></a>

<code>u128</code> bitmask set at bit 63, the crit-bit tree node type
bit flag, generated in Python via <code>hex(int('1' + '0' * 63, 2))</code>.


<pre><code><b>const</b> <a href="critqueue.md#0xc0deb00c_critqueue_TREE_NODE_TYPE">TREE_NODE_TYPE</a>: u128 = 0x8000000000000000;
</code></pre>



<a name="0xc0deb00c_critqueue_borrow"></a>

## Function `borrow`

Borrow insertion value corresponding to <code>access_key</code> in given
<code><a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a></code>, aborting if no such access key.


<a name="@Testing_33"></a>

### Testing


* <code>test_borrowers()</code>


<pre><code><b>public</b> <b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_borrow">borrow</a>&lt;V&gt;(critqueue_ref: &<a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">critqueue::CritQueue</a>&lt;V&gt;, access_key: u128): &V
</code></pre>



<details>
<summary>Implementation</summary>


<pre><code><b>public</b> <b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_borrow">borrow</a>&lt;V&gt;(
    critqueue_ref: &<a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a>&lt;V&gt;,
    access_key: u128
): &V {
    &<a href="_borrow">table::borrow</a>(
        &critqueue_ref.subqueue_nodes, access_key).insertion_value
}
</code></pre>



</details>

<a name="0xc0deb00c_critqueue_borrow_mut"></a>

## Function `borrow_mut`

Mutably borrow insertion value corresponding to <code>access_key</code>
<code><a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a></code>, aborting if no such access key


<a name="@Testing_34"></a>

### Testing


* <code>test_borrowers()</code>


<pre><code><b>public</b> <b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_borrow_mut">borrow_mut</a>&lt;V&gt;(critqueue_ref_mut: &<b>mut</b> <a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">critqueue::CritQueue</a>&lt;V&gt;, access_key: u128): &<b>mut</b> V
</code></pre>



<details>
<summary>Implementation</summary>


<pre><code><b>public</b> <b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_borrow_mut">borrow_mut</a>&lt;V&gt;(
    critqueue_ref_mut: &<b>mut</b> <a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a>&lt;V&gt;,
    access_key: u128
): &<b>mut</b> V {
    &<b>mut</b> <a href="_borrow_mut">table::borrow_mut</a>(
        &<b>mut</b> critqueue_ref_mut.subqueue_nodes, access_key).insertion_value
}
</code></pre>



</details>

<a name="0xc0deb00c_critqueue_dequeue"></a>

## Function `dequeue`

Dequeue the insertion value at the head of the given
<code><a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a></code>, aborting if empty.

See <code>test_dequeue()</code> for iterated dequeue syntax.


<a name="@Example_35"></a>

### Example


Consider the following insertion keys and sub-queues:

| Insertion key | Sub-queue insertion values |
|---------------|----------------------------|
| 0             | [0 -> 10 -> 20 -> ... 90]  |
| 1             | [1 -> 11 -> 21 -> ... 91]  |
| ...           | ...                        |
| 9             | [9 -> 19 -> 29 -> ... 99]  |

Here, insertion key <code>0</code> has the insertion value <code>0</code> at the head
of its corresponding sub-queue, followed by insertion value
<code>10</code>, <code>20</code>, and so on, up until <code>90</code>.

Dequeuing from an ascending crit-queue yields the insertion
value sequence:

* <code>0</code>
* <code>10</code>
* ...
* <code>90</code>
* <code>1</code>
* <code>11</code>
* ...
* <code>91</code>
* ...
* <code>9</code>
* <code>19</code>
* ...
* <code>99</code>

Dequeuing from a descending crit-queue yields:

* <code>9</code>
* <code>19</code>
* ...
* <code>99</code>
* <code>8</code>
* <code>18</code>
* ...
* <code>98</code>
* ...
* <code>0</code>
* <code>10</code>
* ...
* <code>90</code>


<a name="@Testing_36"></a>

### Testing


* <code>test_dequeue()</code>


<pre><code><b>public</b> <b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_dequeue">dequeue</a>&lt;V&gt;(critqueue_ref_mut: &<b>mut</b> <a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">critqueue::CritQueue</a>&lt;V&gt;): V
</code></pre>



<details>
<summary>Implementation</summary>


<pre><code><b>public</b> <b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_dequeue">dequeue</a>&lt;V&gt;(
    critqueue_ref_mut: &<b>mut</b> <a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a>&lt;V&gt;,
): V {
    // Get the access key of the crit-queue head.
    <b>let</b> head_access_key = *<a href="_borrow">option::borrow</a>(&critqueue_ref_mut.head);
    // Remove and <b>return</b> the corresponding insertion value.
    <a href="critqueue.md#0xc0deb00c_critqueue_remove">remove</a>(critqueue_ref_mut, head_access_key)
}
</code></pre>



</details>

<a name="0xc0deb00c_critqueue_get_head_access_key"></a>

## Function `get_head_access_key`

Return access key of given <code><a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a></code> head, if any.


<a name="@Testing_37"></a>

### Testing


* <code>test_get_head_access_key()</code>


<pre><code><b>public</b> <b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_get_head_access_key">get_head_access_key</a>&lt;V&gt;(critqueue_ref: &<a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">critqueue::CritQueue</a>&lt;V&gt;): <a href="_Option">option::Option</a>&lt;u128&gt;
</code></pre>



<details>
<summary>Implementation</summary>


<pre><code><b>public</b> <b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_get_head_access_key">get_head_access_key</a>&lt;V&gt;(
    critqueue_ref: &<a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a>&lt;V&gt;,
): Option&lt;u128&gt; {
    critqueue_ref.head
}
</code></pre>



</details>

<a name="0xc0deb00c_critqueue_has_access_key"></a>

## Function `has_access_key`

Return <code><b>true</b></code> if given <code><a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a></code> has the given <code>access_key</code>.


<a name="@Testing_38"></a>

### Testing


* <code>test_has_access_key()</code>


<pre><code><b>public</b> <b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_has_access_key">has_access_key</a>&lt;V&gt;(critqueue_ref: &<a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">critqueue::CritQueue</a>&lt;V&gt;, access_key: u128): bool
</code></pre>



<details>
<summary>Implementation</summary>


<pre><code><b>public</b> <b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_has_access_key">has_access_key</a>&lt;V&gt;(
    critqueue_ref: &<a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a>&lt;V&gt;,
    access_key: u128
): bool {
    <a href="_contains">table::contains</a>(&critqueue_ref.subqueue_nodes, access_key)
}
</code></pre>



</details>

<a name="0xc0deb00c_critqueue_insert"></a>

## Function `insert`

Insert the given key-value insertion pair into the given
<code><a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a></code>, returning an access key.


<a name="@Parameters_39"></a>

### Parameters


* <code>critqueue_ref_mut</code>: Mutable reference to crit-queue.
* <code>insertion_key</code>: Key to insert.
* <code>insertion_value</code>: Value to insert.


<a name="@Returns_40"></a>

### Returns


* <code>u128</code>: Access key for given key-value pair.


<a name="@Reference_diagrams_41"></a>

### Reference diagrams



<a name="@Conventions_42"></a>

#### Conventions


For ease of illustration, critical bitmasks and leaf keys are
depicted relative to bit 64, but tested with correspondingly
bitshifted amounts. Insertion keys are given in binary, while
insertion values and insertion counts are given in decimal:

>     101
>     [n_0{7} -> n_1{8}]

Here, <code>101</code> refers to a crit-bit tree leaf for insertion key
<code>101</code>, which has a sub-queue with node <code>n_0{7}</code> at its head,
having insertion count 0 and an insertion value of 7. The
next sub-queue node <code>n_1{8}</code>, the sub-queue tail, has insertion
count 1 and insertion value 8.


<a name="@Insertion_sequence_43"></a>

#### Insertion sequence


1. Insert <code>{010, 4}</code>:

>     010 <- new leaf
>     [n_0{4}]
>      ^ new sub-queue node

2. Insert <code>{101, 9}</code>, for <code>001</code> as a free leaf with insertion
count 4 prior to the insertion:

>             2nd <- new inner node
>            /   \
>          010   101 <- new leaf node
>     [n_0{4}]   [n_5{9}]
>                 ^ new sub-queue node

3. Insert <code>{010, 6}</code>:

>                       2nd
>                      /   \
>                    010   101
>     [n_0{4} -> n_1{6}]   [n_5{9}]
>                ^ new sub-queue node

4. Insert <code>{000, 8}</code>, for <code>000</code> as a free leaf with insertion
count 2 prior to the insertion:

>                          2nd
>                         /   \
>     new inner node -> 1st    101
>                      /   \   [n_5{9}]
>        new leaf -> 000   010
>               [n_3{8}]   [n_0{4} -> n_1{6}]
>                ^ new sub-queue node


<a name="@Testing_44"></a>

### Testing


* <code>test_insert_ascending()</code>
* <code>test_insert_descending()</code>


<pre><code><b>public</b> <b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_insert">insert</a>&lt;V&gt;(critqueue_ref_mut: &<b>mut</b> <a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">critqueue::CritQueue</a>&lt;V&gt;, insertion_key: u64, insertion_value: V): u128
</code></pre>



<details>
<summary>Implementation</summary>


<pre><code><b>public</b> <b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_insert">insert</a>&lt;V&gt;(
    critqueue_ref_mut: &<b>mut</b> <a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a>&lt;V&gt;,
    insertion_key: u64,
    insertion_value: V
): u128 {
    // Initialize a sub-queue node <b>with</b> the insertion value,
    // assuming it is the sole sub-queue node in a free leaf.
    <b>let</b> subqueue_node = <a href="critqueue.md#0xc0deb00c_critqueue_SubQueueNode">SubQueueNode</a>{insertion_value,
        previous: <a href="_none">option::none</a>(), next: <a href="_none">option::none</a>()};
    // Get leaf key from insertion key.
    <b>let</b> leaf_key = (insertion_key <b>as</b> u128) &lt;&lt; <a href="critqueue.md#0xc0deb00c_critqueue_INSERTION_KEY">INSERTION_KEY</a>;
    // Borrow mutable reference <b>to</b> leaves <a href="">table</a>.
    <b>let</b> leaves_ref_mut = &<b>mut</b> critqueue_ref_mut.leaves;
    // Determine <b>if</b> corresponding leaf <b>has</b> already been allocated.
    <b>let</b> leaf_already_allocated = <a href="_contains">table::contains</a>(leaves_ref_mut, leaf_key);
    // Get access key for new sub-queue node, and <b>if</b> corresponding
    // leaf node is a free leaf. If leaf is already allocated:
    <b>let</b> (access_key, free_leaf) = <b>if</b> (leaf_already_allocated)
        // Update its sub-queue and the new sub-queue node, storing
        // the access key of the new sub-queue node and <b>if</b> the
        // corresponding leaf is free.
        <a href="critqueue.md#0xc0deb00c_critqueue_insert_update_subqueue">insert_update_subqueue</a>(
            critqueue_ref_mut, &<b>mut</b> subqueue_node, leaf_key)
        // Otherwise, store access key of the new sub-queue node,
        // found in a newly-allocated free leaf.
        <b>else</b> (<a href="critqueue.md#0xc0deb00c_critqueue_insert_allocate_leaf">insert_allocate_leaf</a>(critqueue_ref_mut, leaf_key), <b>true</b>);
    // Borrow mutable reference <b>to</b> sub-queue nodes <a href="">table</a>.
    <b>let</b> subqueue_nodes_ref_mut = &<b>mut</b> critqueue_ref_mut.subqueue_nodes;
    // Add new sub-queue node <b>to</b> the sub-queue nodes <a href="">table</a>.
    <a href="_add">table::add</a>(subqueue_nodes_ref_mut, access_key, subqueue_node);
    // Check the crit-queue head, updating <b>as</b> necessary.
    <a href="critqueue.md#0xc0deb00c_critqueue_insert_check_head">insert_check_head</a>(critqueue_ref_mut, access_key);
    // If leaf is free, insert it <b>to</b> the crit-bit tree.
    <b>if</b> (free_leaf) <a href="critqueue.md#0xc0deb00c_critqueue_insert_leaf">insert_leaf</a>(critqueue_ref_mut, access_key);
    access_key // Return access key.
}
</code></pre>



</details>

<a name="0xc0deb00c_critqueue_is_empty"></a>

## Function `is_empty`

Return <code><b>true</b></code> if given <code><a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a></code> is empty.


<a name="@Testing_45"></a>

### Testing


* <code>test_is_empty()</code>


<pre><code><b>public</b> <b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_is_empty">is_empty</a>&lt;V&gt;(critqueue_ref: &<a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">critqueue::CritQueue</a>&lt;V&gt;): bool
</code></pre>



<details>
<summary>Implementation</summary>


<pre><code><b>public</b> <b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_is_empty">is_empty</a>&lt;V&gt;(
    critqueue_ref: &<a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a>&lt;V&gt;,
): bool {
    <a href="_is_none">option::is_none</a>(&critqueue_ref.root)
}
</code></pre>



</details>

<a name="0xc0deb00c_critqueue_new"></a>

## Function `new`

Return <code><a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a></code> of sort <code>order</code> <code><a href="critqueue.md#0xc0deb00c_critqueue_ASCENDING">ASCENDING</a></code> or <code><a href="critqueue.md#0xc0deb00c_critqueue_DESCENDING">DESCENDING</a></code>.


<pre><code><b>public</b> <b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_new">new</a>&lt;V: store&gt;(order: bool): <a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">critqueue::CritQueue</a>&lt;V&gt;
</code></pre>



<details>
<summary>Implementation</summary>


<pre><code><b>public</b> <b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_new">new</a>&lt;V: store&gt;(
    order: bool
): <a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a>&lt;V&gt; {
    <a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a>{
        order,
        root: <a href="_none">option::none</a>(),
        head: <a href="_none">option::none</a>(),
        inners: <a href="_new">table::new</a>(),
        leaves: <a href="_new">table::new</a>(),
        subqueue_nodes: <a href="_new">table::new</a>()
    }
}
</code></pre>



</details>

<a name="0xc0deb00c_critqueue_remove"></a>

## Function `remove`

Remove and return from given <code><a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a></code> the insertion value
corresponding to <code>access_key</code>, aborting if no such entry.


<a name="@Reference_diagrams_46"></a>

### Reference diagrams



<a name="@Conventions_47"></a>

#### Conventions


For ease of illustration, critical bitmasks and leaf keys are
depicted relative to bit 64, but tested with correspondingly
bitshifted amounts. Insertion keys are given in binary, while
insertion values and insertion counts are given in decimal:

>     101
>     [n_0{7} -> n_1{8}]

Here, <code>101</code> refers to a crit-bit tree leaf for insertion key
<code>101</code>, which has a sub-queue with node <code>n_0{7}</code> at its head,
having insertion count 0 and an insertion value of 7. The
next sub-queue node <code>n_1{8}</code>, the sub-queue tail, has insertion
count 1 and insertion value 8.


<a name="@Insertion_preposition_sequence_48"></a>

#### Insertion preposition sequence


The below preposition sequence is applicable for all tests
other than <code>test_remove_root()</code>, for removals relative to the
tree after the final preposition insertion.

1. Insert <code>{010, 4}</code>:

>     010 <- new leaf
>     [n_0{4}]
>      ^ new sub-queue node

2. Insert <code>{101, 9}</code>:

>             2nd <- new inner node
>            /   \
>          010   101 <- new leaf node
>     [n_0{4}]   [n_0{9}]
>                 ^ new sub-queue node

3. Insert <code>{101, 2}</code>:

>             2nd
>            /   \
>          010   101
>     [n_0{4}]   [n_0{9} -> n_1{2}]
>                           ^ new sub-queue node

4. Insert <code>{000, 8}</code>:

>                          2nd
>                         /   \
>     new inner node -> 1st    101
>                      /   \   [n_0{9} -> n_1{2}]
>        new leaf -> 000   010
>               [n_0{8}]   [n_0{4}]
>                ^ new sub-queue node


<a name="@No_sub-queue_head_update_49"></a>

#### No sub-queue head update


Remove <code>n_1{2}</code> from <code>101</code>:

>                2nd
>               /   \
>             1st    101
>            /   \   [n_0{9}]
>          000   010  ^ sub-queue head is now tail too
>     [n_0{8}]   [n_0{4}]
>


<a name="@Sub-queue_head_update,_no_free_leaf_50"></a>

#### Sub-queue head update, no free leaf


Remove <code>n_0{9}</code> from <code>101</code>, which results in a crit-queue head
update for a descending crit-queue:

>                2nd
>               /   \
>             1st    101
>            /   \   [n_1{2}]
>          000   010  ^ new crit-queue head (descending)
>     [n_0{8}]   [n_0{4}]


<a name="@Free_leaf_generation_1_51"></a>

#### Free leaf generation 1


Remove <code>n_0{8}</code> from <code>000</code>, which results in a crit-queue head
update for an ascending crit-queue:

>             2nd
>            /   \
>          010    101
>     [n_0{4}]    [n_0{9} -> n_1{2}]
>      ^ new crit-queue head (ascending)


<a name="@Free_leaf_generation_2_52"></a>

#### Free leaf generation 2


Remove <code>n_0{9}</code> and <code>n_1{2}</code> from <code>101</code>, which results in a
crit-queue head update for a descending crit-queue:

>             1st
>            /   \
>          000   010
>     [n_0{8}]   [n_0{4}]
>                 ^ new crit-queue head (descending)


<a name="@Testing_53"></a>

### Testing


* <code>test_remove_no_subqueue_head_update()</code>
* <code>test_remove_subqueue_head_update_no_free_leaf_ascending()</code>
* <code>test_remove_subqueue_head_update_no_free_leaf_descending()</code>
* <code>test_remove_free_leaf_1_ascending()</code>
* <code>test_remove_free_leaf_1_descending()</code>
* <code>test_remove_free_leaf_2_ascending()</code>
* <code>test_remove_free_leaf_2_descending()</code>
* <code>test_remove_root()</code>


<pre><code><b>public</b> <b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_remove">remove</a>&lt;V&gt;(critqueue_ref_mut: &<b>mut</b> <a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">critqueue::CritQueue</a>&lt;V&gt;, access_key: u128): V
</code></pre>



<details>
<summary>Implementation</summary>


<pre><code><b>public</b> <b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_remove">remove</a>&lt;V&gt;(
    critqueue_ref_mut: &<b>mut</b> <a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a>&lt;V&gt;,
    access_key: u128
): V {
    // Get requested insertion value by removing the sub-queue node
    // it was stored in, storing the optional new head field of the
    // corresponding sub-queue.
    <b>let</b> (insertion_value, optional_new_subqueue_head_field) =
        <a href="critqueue.md#0xc0deb00c_critqueue_remove_subqueue_node">remove_subqueue_node</a>(critqueue_ref_mut, access_key);
    // Check <b>if</b> removed sub-queue node was head of the crit-queue.
    <b>let</b> just_removed_critqueue_head = access_key ==
        *<a href="_borrow">option::borrow</a>(&critqueue_ref_mut.head);
    // If the sub-queue <b>has</b> a new head field:
    <b>if</b> (<a href="_is_some">option::is_some</a>(&optional_new_subqueue_head_field)) {
        // Get the new sub-queue head field.
        <b>let</b> new_subqueue_head_field =
            *<a href="_borrow">option::borrow</a>(&optional_new_subqueue_head_field);
        // If the new sub-queue head field is none, then the
        // sub-queue is now empty, so the leaf needs <b>to</b> be freed.
        <b>if</b> (<a href="_is_none">option::is_none</a>(&new_subqueue_head_field)) {
            // Get the leaf's key from the access key.
            <b>let</b> leaf_key = access_key & <a href="critqueue.md#0xc0deb00c_critqueue_ACCESS_KEY_TO_LEAF_KEY">ACCESS_KEY_TO_LEAF_KEY</a>;
            // If the crit-queue head was just removed, <b>update</b> it
            // before freeing the leaf:
            <b>if</b> (just_removed_critqueue_head) {
                // Get crit-queue insertion key sort order.
                <b>let</b> order = critqueue_ref_mut.order;
                // If ascending crit-queue, try traversing <b>to</b> the
                // successor of the leaf <b>to</b> free,
                <b>let</b> target = <b>if</b> (order == <a href="critqueue.md#0xc0deb00c_critqueue_ASCENDING">ASCENDING</a>) <a href="critqueue.md#0xc0deb00c_critqueue_SUCCESSOR">SUCCESSOR</a> <b>else</b>
                    <a href="critqueue.md#0xc0deb00c_critqueue_PREDECESSOR">PREDECESSOR</a>; // Otherwise the predecessor.
                // Traverse, storing optional next leaf key.
                <b>let</b> optional_next_leaf_key =
                    <a href="critqueue.md#0xc0deb00c_critqueue_traverse">traverse</a>(critqueue_ref_mut, leaf_key, target);
                // Crit-queue is empty <b>if</b> there is no next key.
                <b>let</b> now_empty = <a href="_is_none">option::is_none</a>(&optional_next_leaf_key);
                // If crit-queue now empty, it <b>has</b> no head.
                critqueue_ref_mut.head = <b>if</b> (now_empty) <a href="_none">option::none</a>()
                    // Otherwise, the new crit-queue head is the
                    // head of the leaf just traversed <b>to</b>.
                    <b>else</b> <a href="_borrow">table::borrow</a>(&critqueue_ref_mut.leaves,
                        *<a href="_borrow">option::borrow</a>(&optional_next_leaf_key)).head;
            }; // The crit-queue head is now updated.
            // Free the leaf by removing it from the crit-bit tree.
            <a href="critqueue.md#0xc0deb00c_critqueue_remove_free_leaf">remove_free_leaf</a>(critqueue_ref_mut, leaf_key);
        // Otherwise, <b>if</b> the new sub-queue head field indicates a
        // different sub-queue node then the one just removed:
        } <b>else</b> {
            // If the crit-queue head was just removed, then the
            // new sub-queue head is also the new crit-queue head.
            <b>if</b> (just_removed_critqueue_head)
                critqueue_ref_mut.head = new_subqueue_head_field;
        }
    };
    insertion_value // Return insertion value.
}
</code></pre>



</details>

<a name="0xc0deb00c_critqueue_would_become_new_head"></a>

## Function `would_become_new_head`

Return <code><b>true</b></code> if, were <code>insertion_key</code> to be inserted, its
access key would become the new head of the given <code><a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a></code>.


<a name="@Testing_54"></a>

### Testing


* <code>test_would_become_trail_head()</code>


<pre><code><b>public</b> <b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_would_become_new_head">would_become_new_head</a>&lt;V&gt;(critqueue_ref: &<a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">critqueue::CritQueue</a>&lt;V&gt;, insertion_key: u64): bool
</code></pre>



<details>
<summary>Implementation</summary>


<pre><code><b>public</b> <b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_would_become_new_head">would_become_new_head</a>&lt;V&gt;(
    critqueue_ref: &<a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a>&lt;V&gt;,
    insertion_key: u64
): bool {
    // If the crit-queue is empty and thus <b>has</b> no head:
    <b>if</b> (<a href="_is_none">option::is_none</a>(&critqueue_ref.head)) {
        // Return that insertion key would become new head.
        <b>return</b> <b>true</b>
    } <b>else</b> { // Otherwise, <b>if</b> crit-queue is not empty:
        // Get insertion key of crit-queue head.
        <b>let</b> head_insertion_key = (*<a href="_borrow">option::borrow</a>(&critqueue_ref.head) &gt;&gt;
            <a href="critqueue.md#0xc0deb00c_critqueue_INSERTION_KEY">INSERTION_KEY</a> <b>as</b> u64);
        // If an ascending crit-queue, <b>return</b> <b>true</b> <b>if</b> insertion key
        // is less than insertion key of crit-queue head.
        <b>return</b> <b>if</b> (critqueue_ref.order == <a href="critqueue.md#0xc0deb00c_critqueue_ASCENDING">ASCENDING</a>)
            insertion_key &lt; head_insertion_key <b>else</b>
            // If a descending crit-queue, <b>return</b> <b>true</b> <b>if</b> insertion
            // key is greater than insertion key of crit-queue head.
            insertion_key &gt; head_insertion_key
    }
}
</code></pre>



</details>

<a name="0xc0deb00c_critqueue_would_trail_head"></a>

## Function `would_trail_head`

Return <code><b>true</b></code> if, were <code>insertion_key</code> to be inserted, its
access key would trail behind the head of the given <code><a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a></code>.


<a name="@Testing_55"></a>

### Testing


* <code>test_would_become_trail_head()</code>


<pre><code><b>public</b> <b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_would_trail_head">would_trail_head</a>&lt;V&gt;(critqueue_ref: &<a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">critqueue::CritQueue</a>&lt;V&gt;, insertion_key: u64): bool
</code></pre>



<details>
<summary>Implementation</summary>


<pre><code><b>public</b> <b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_would_trail_head">would_trail_head</a>&lt;V&gt;(
    critqueue_ref: &<a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a>&lt;V&gt;,
    insertion_key: u64
): bool {
    !<a href="critqueue.md#0xc0deb00c_critqueue_would_become_new_head">would_become_new_head</a>(critqueue_ref, insertion_key)
}
</code></pre>



</details>

<a name="0xc0deb00c_critqueue_get_critical_bitmask"></a>

## Function `get_critical_bitmask`

Return a bitmask set at the most significant bit at which two
unequal bitstrings, <code>s1</code> and <code>s2</code>, vary.


<a name="@<code>XOR</code>/<code>AND</code>_method_56"></a>

### <code>XOR</code>/<code>AND</code> method


First, a bitwise <code>XOR</code> is used to flag all differing bits:

>              s1: 11110001
>              s2: 11011100
>     x = s1 ^ s2: 00101101
>                    ^ critical bit = 5

Here, the critical bit is equivalent to the bit number of the
most significant set bit in the bitwise <code>XOR</code> result
<code>x = s1 ^ s2</code>. At this point, [Langley 2008] notes that <code>x</code>
bitwise <code>AND</code> <code>x - 1</code> will be nonzero so long as <code>x</code> contains
at least some bits set which are of lesser significance than the
critical bit:

>                   x: 00101101
>               x - 1: 00101100
>     x = x & (x - 1): 00101100

Thus he suggests repeating <code>x & (x - 1)</code> while the new result
<code>x = x & (x - 1)</code> is not equal to zero, because such a loop will
eventually reduce <code>x</code> to a power of two (excepting the trivial
case where <code>x</code> starts as all 0 except bit 0 set, for which the
loop never enters past the initial conditional check). Per this
method, using the new <code>x</code> value for the current example, the
second iteration proceeds as follows:

>                   x: 00101100
>               x - 1: 00101011
>     x = x & (x - 1): 00101000

The third iteration:

>                   x: 00101000
>               x - 1: 00100111
>     x = x & (x - 1): 00100000
Now, <code>x & x - 1</code> will equal zero and the loop will not begin a
fourth iteration:

>                 x: 00100000
>             x - 1: 00011111
>     x AND (x - 1): 00000000

Thus after three iterations a corresponding critical bitmask
has been determined. However, in the case where the two input
strings vary at all bits of lesser significance than the
critical bit, there may be required as many as <code>k - 1</code>
iterations, where <code>k</code> is the number of bits in each string under
comparison. For instance, consider the case of the two 8-bit
strings <code>s1</code> and <code>s2</code> as follows:

>                  s1: 10101010
>                  s2: 01010101
>         x = s1 ^ s2: 11111111
>                      ^ critical bit = 7
>     x = x & (x - 1): 11111110 [iteration 1]
>     x = x & (x - 1): 11111100 [iteration 2]
>     x = x & (x - 1): 11111000 [iteration 3]
>     ...

Notably, this method is only suggested after already having
identified the varying byte between the two strings, thus
limiting <code>x & (x - 1)</code> operations to at most 7 iterations.


<a name="@Binary_search_method_57"></a>

### Binary search method


For the present implementation, unlike in [Langley 2008],
strings are not partitioned into a multi-byte array, rather,
they are stored as <code>u128</code> integers, so a binary search is
instead proposed. Here, the same <code>x = s1 ^ s2</code> operation is
first used to identify all differing bits, before iterating on
an upper (<code>u</code>) and lower bound (<code>l</code>) for the critical bit
number:

>              s1: 10101010
>              s2: 01010101
>     x = s1 ^ s2: 11111111
>            u = 7 ^      ^ l = 0

The upper bound <code>u</code> is initialized to the length of the
bitstring (7 in this example, but 127 for a <code>u128</code>), and the
lower bound <code>l</code> is initialized to 0. Next the midpoint <code>m</code> is
calculated as the average of <code>u</code> and <code>l</code>, in this case
<code>m = (7 + 0) / 2 = 3</code>, per truncating integer division. Finally,
the shifted compare value <code>s = x &gt;&gt; m</code> is calculated, with the
result having three potential outcomes:

| Shift result | Outcome                              |
|--------------|--------------------------------------|
| <code>s == 1</code>     | The critical bit <code>c</code> is equal to <code>m</code> |
| <code>s == 0</code>     | <code>c &lt; m</code>, so set <code>u</code> to <code>m - 1</code>       |
| <code>s &gt; 1</code>      | <code>c &gt; m</code>, so set <code>l</code> to <code>m + 1</code>       |

Hence, continuing the current example:

>              x: 11111111
>     s = x >> m: 00011111

<code>s &gt; 1</code>, so <code>l = m + 1 = 4</code>, and the search window has shrunk:

>     x = s1 ^ s2: 11111111
>            u = 7 ^  ^ l = 4

Updating the midpoint yields <code>m = (7 + 4) / 2 = 5</code>:

>              x: 11111111
>     s = x >> m: 00000111

Again <code>s &gt; 1</code>, so update <code>l = m + 1 = 6</code>, and the window
shrinks again:

>     x = s1 ^ s2: 11111111
>            u = 7 ^^ l = 6
>      s = x >> m: 00000011

Again <code>s &gt; 1</code>, so update <code>l = m + 1 = 7</code>, the final iteration:

>     x = s1 ^ s2: 11111111
>            u = 7 ^ l = 7
>      s = x >> m: 00000001

Here, <code>s == 1</code>, which means that <code>c = m = 7</code>, and the
corresponding critical bitmask <code>1 &lt;&lt; c</code> is returned:

>         s1: 10101010
>         s2: 01010101
>     1 << c: 10000000

Notably this search has converged after only 3 iterations, as
opposed to 7 for the linear search proposed above, and in
general such a search converges after $log_2(k)$ iterations at
most, where $k$ is the number of bits in each of the strings
<code>s1</code> and <code>s2</code> under comparison. Hence this search method
improves the $O(k)$ search proposed by [Langley 2008] to
$O(log_2(k))$.


<a name="@Testing_58"></a>

### Testing


* <code>test_get_critical_bitmask()</code>


<pre><code><b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_get_critical_bitmask">get_critical_bitmask</a>(s1: u128, s2: u128): u128
</code></pre>



<details>
<summary>Implementation</summary>


<pre><code><b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_get_critical_bitmask">get_critical_bitmask</a>(
    s1: u128,
    s2: u128,
): u128 {
    <b>let</b> x = s1 ^ s2; // XOR result marked 1 at bits that differ.
    <b>let</b> l = 0; // Lower bound on critical bit search.
    <b>let</b> u = <a href="critqueue.md#0xc0deb00c_critqueue_MSB_u128">MSB_u128</a>; // Upper bound on critical bit search.
    <b>loop</b> { // Begin binary search.
        <b>let</b> m = (l + u) / 2; // Calculate midpoint of search window.
        <b>let</b> s = x &gt;&gt; m; // Calculate midpoint shift of XOR result.
        <b>if</b> (s == 1) <b>return</b> 1 &lt;&lt; m; // If shift equals 1, c = m.
        // Update search bounds.
        <b>if</b> (s &gt; 1) l = m + 1 <b>else</b> u = m - 1;
    }
}
</code></pre>



</details>

<a name="0xc0deb00c_critqueue_insert_allocate_leaf"></a>

## Function `insert_allocate_leaf`

Allocate a leaf during insertion.

Inner function for <code><a href="critqueue.md#0xc0deb00c_critqueue_insert">insert</a>()</code>.


<a name="@Returns_59"></a>

### Returns


* <code>u128</code>: Access key of new sub-queue node.


<a name="@Assumptions_60"></a>

### Assumptions


* <code>critqueue_ref_mut</code> indicates a <code><a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a></code> that does not have
an allocated leaf with the given <code>leaf_key</code>.
* A <code><a href="critqueue.md#0xc0deb00c_critqueue_SubQueueNode">SubQueueNode</a></code> with the appropriate access key has been
initialized as if it were the sole sub-queue node in a free
leaf.


<a name="@Testing_61"></a>

### Testing


* <code>test_insert_allocate_leaf()</code>


<pre><code><b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_insert_allocate_leaf">insert_allocate_leaf</a>&lt;V&gt;(critqueue_ref_mut: &<b>mut</b> <a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">critqueue::CritQueue</a>&lt;V&gt;, leaf_key: u128): u128
</code></pre>



<details>
<summary>Implementation</summary>


<pre><code><b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_insert_allocate_leaf">insert_allocate_leaf</a>&lt;V&gt;(
    critqueue_ref_mut: &<b>mut</b> <a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a>&lt;V&gt;,
    leaf_key: u128
): u128 {
    // Get the sort order of the crit-queue.
    <b>let</b> order = critqueue_ref_mut.order;
    // Get access key. If ascending, is identical <b>to</b> leaf key, which
    // <b>has</b> insertion count in bits 64-127 and 0 for bits 0-63.
    <b>let</b> access_key = <b>if</b> (order == <a href="critqueue.md#0xc0deb00c_critqueue_ASCENDING">ASCENDING</a>) leaf_key <b>else</b>
        // Else same, but flipped insertion count and bit 63 set.
        leaf_key  ^ <a href="critqueue.md#0xc0deb00c_critqueue_NOT_INSERTION_COUNT_DESCENDING">NOT_INSERTION_COUNT_DESCENDING</a>;
    // Declare leaf <b>with</b> insertion count 0, no parent, and new
    // sub-queue node <b>as</b> both head and tail.
    <b>let</b> leaf = <a href="critqueue.md#0xc0deb00c_critqueue_Leaf">Leaf</a>{count: 0, parent: <a href="_none">option::none</a>(), head:
        <a href="_some">option::some</a>(access_key), tail: <a href="_some">option::some</a>(access_key)};
    // Borrow mutable reference <b>to</b> leaves <a href="">table</a>.
    <b>let</b> leaves_ref_mut = &<b>mut</b> critqueue_ref_mut.leaves;
    // Add the leaf <b>to</b> the leaves <a href="">table</a>.
    <a href="_add">table::add</a>(leaves_ref_mut, leaf_key, leaf);
    access_key // Return access key.
}
</code></pre>



</details>

<a name="0xc0deb00c_critqueue_insert_check_head"></a>

## Function `insert_check_head`

Check head of given <code><a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a></code>, optionally setting it to the
<code>access_key</code> of a new key-value insertion pair.

Inner function for <code><a href="critqueue.md#0xc0deb00c_critqueue_insert">insert</a>()</code>.


<a name="@Testing_62"></a>

### Testing


* <code>test_insert_check_head()</code>


<pre><code><b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_insert_check_head">insert_check_head</a>&lt;V&gt;(critqueue_ref_mut: &<b>mut</b> <a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">critqueue::CritQueue</a>&lt;V&gt;, access_key: u128)
</code></pre>



<details>
<summary>Implementation</summary>


<pre><code><b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_insert_check_head">insert_check_head</a>&lt;V&gt;(
    critqueue_ref_mut: &<b>mut</b> <a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a>&lt;V&gt;,
    access_key: u128
) {
    // Get crit-queue sort order.
    <b>let</b> order = critqueue_ref_mut.order;
    // Get mutable reference <b>to</b> crit-queue head field.
    <b>let</b> head_ref_mut = &<b>mut</b> critqueue_ref_mut.head;
    <b>if</b> (<a href="_is_none">option::is_none</a>(head_ref_mut)) { // If an empty crit-queue:
        // Set the head <b>to</b> be the new access key.
        <a href="_fill">option::fill</a>(head_ref_mut, access_key);
    } <b>else</b> { // If crit-queue is not empty:
        // If the sort order is ascending and new access key is less
        // than that of the crit-queue head, or
        <b>if</b> ((order == <a href="critqueue.md#0xc0deb00c_critqueue_ASCENDING">ASCENDING</a> &&
                access_key &lt; *<a href="_borrow">option::borrow</a>(head_ref_mut)) ||
            // If descending sort order and new access key is
            // greater than that of crit-queue head:
            (order == <a href="critqueue.md#0xc0deb00c_critqueue_DESCENDING">DESCENDING</a> &&
                access_key &gt; *<a href="_borrow">option::borrow</a>(head_ref_mut)))
            // Set new access key <b>as</b> the crit-queue head.
            _ = <a href="_swap">option::swap</a>(head_ref_mut, access_key);
    };
}
</code></pre>



</details>

<a name="0xc0deb00c_critqueue_insert_leaf"></a>

## Function `insert_leaf`

Insert a free leaf to the crit-bit tree in a crit-queue.

Inner function for <code><a href="critqueue.md#0xc0deb00c_critqueue_insert">insert</a>()</code>.

If crit-queue is empty, set root as new leaf key and return,
otherwise search for the closest leaf in the crit-bit tree. Then
get the critical bit between the new leaf key and the search
match, which becomes the critical bitmask of a new inner node to
insert. Start walking up the tree, inserting the new inner node
below the first walked node with a larger critical bit, or above
the root of the tree, whichever comes first.


<a name="@Parameters_63"></a>

### Parameters


* <code>critqueue_ref_mut</code>: Mutable reference to crit-queue.
* <code>access_key</code>: Unique access key of key-value insertion pair
just inserted, from which is derived the new leaf key
corresponding to the free leaf to insert, and optionally, a
new inner key for a new inner node to insert.


<a name="@Assumptions_64"></a>

### Assumptions


* Given <code><a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a></code> has a free leaf with <code>leaf_key</code>.


<a name="@Diagrams_65"></a>

### Diagrams


For ease of illustration, critical bitmasks and leaf keys are
depicted relative to bit 64, but tested with correspondingly
bitshifted amounts, for inner keys that are additionally
encoded with a mock insertion key, mock insertion count,
and inner node bit flag.

1. Start by inserting <code>0011</code> to an empty tree:

>     0011 <- new leaf

2. Then insert <code>0100</code>:

>         2nd <- new inner node
>        /   \
>     0011   0100 <- new leaf

3. Insert <code>0101</code>:

>         2nd
>        /   \
>     0011   0th <- new inner node
>           /   \
>        0100   0101 <- new leaf

4. Insert <code>1000</code>:

>            3rd <- new inner node
>           /   \
>         2nd   1000 <- new leaf
>        /   \
>     0011   0th
>           /   \
>        0100   0101

5. Insert <code>0110</code>:

>            3rd
>           /   \
>         2nd   1000
>        /   \
>     0011   1st <- new inner node
>           /   \
>         0th   0110 <- new leaf
>        /   \
>     0100   0101


<a name="@Testing_66"></a>

### Testing


* <code>test_insert_leaf()</code>


<pre><code><b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_insert_leaf">insert_leaf</a>&lt;V&gt;(critqueue_ref_mut: &<b>mut</b> <a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">critqueue::CritQueue</a>&lt;V&gt;, access_key: u128)
</code></pre>



<details>
<summary>Implementation</summary>


<pre><code><b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_insert_leaf">insert_leaf</a>&lt;V&gt;(
    critqueue_ref_mut: &<b>mut</b> <a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a>&lt;V&gt;,
    access_key: u128
) {
    // Get leaf key for free leaf inserted <b>to</b> tree.
    <b>let</b> leaf_key = access_key & <a href="critqueue.md#0xc0deb00c_critqueue_ACCESS_KEY_TO_LEAF_KEY">ACCESS_KEY_TO_LEAF_KEY</a>;
    // If crit-queue is empty, set root <b>as</b> new leaf key and <b>return</b>.
    <b>if</b> (<a href="_is_none">option::is_none</a>(&critqueue_ref_mut.root))
        <b>return</b> <a href="_fill">option::fill</a>(&<b>mut</b> critqueue_ref_mut.root, leaf_key);
    // Get inner key for new inner node corresponding <b>to</b> access key.
    <b>let</b> new_inner_node_key = access_key | <a href="critqueue.md#0xc0deb00c_critqueue_ACCESS_KEY_TO_INNER_KEY">ACCESS_KEY_TO_INNER_KEY</a>;
    // Search for closest leaf, returning its leaf key, the inner
    // key of its parent, and the parent's critical bitmask.
    <b>let</b> (walk_node_key, optional_parent_key, optional_parent_bitmask) =
        <a href="critqueue.md#0xc0deb00c_critqueue_search">search</a>(critqueue_ref_mut, leaf_key);
    // Get critical bitmask between new leaf key and walk leaf key.
    <b>let</b> critical_bitmask = <a href="critqueue.md#0xc0deb00c_critqueue_get_critical_bitmask">get_critical_bitmask</a>(leaf_key, walk_node_key);
    <b>loop</b> { // Start walking up tree from search leaf.
        // Return <b>if</b> walk node is root and <b>has</b> no parent,
        <b>if</b> (<a href="_is_none">option::is_none</a>(&optional_parent_key)) <b>return</b>
            // By inserting free leaf above it.
            <a href="critqueue.md#0xc0deb00c_critqueue_insert_leaf_above_root_node">insert_leaf_above_root_node</a>(critqueue_ref_mut,
                new_inner_node_key, critical_bitmask, leaf_key);
        // Otherwise there is a parent, so get its key and bitmask.
        <b>let</b> (parent_key, parent_bitmask) =
            (*<a href="_borrow">option::borrow</a>(&optional_parent_key),
             *<a href="_borrow">option::borrow</a>(&optional_parent_bitmask));
        // Return <b>if</b> critical bitmask is less than that of parent,
        <b>if</b> (critical_bitmask &lt; parent_bitmask) <b>return</b>
            // By inserting free leaf below the parent.
            <a href="critqueue.md#0xc0deb00c_critqueue_insert_leaf_below_anchor_node">insert_leaf_below_anchor_node</a>(critqueue_ref_mut, parent_key,
                new_inner_node_key, critical_bitmask, leaf_key);
        // If have not returned, walk continues at parent node.
        walk_node_key = parent_key;
        // Immutably borrow inner nodes <a href="">table</a>.
        <b>let</b> inners_ref = &critqueue_ref_mut.inners;
        // Immutably borrow the new inner node <b>to</b> walk.
        <b>let</b> walk_node_ref = <a href="_borrow">table::borrow</a>(inners_ref, walk_node_key);
        // Get its optional parent key.
        optional_parent_key = walk_node_ref.parent;
        // If new node <b>to</b> walk <b>has</b> no parent:
        <b>if</b> (<a href="_is_none">option::is_none</a>(&optional_parent_key)) {
            // Set optional parent bitmask <b>to</b> none.
            optional_parent_bitmask = <a href="_none">option::none</a>();
        } <b>else</b> { // If new node <b>to</b> walk does have a parent:
            // Get the parent's key.
            parent_key = *(<a href="_borrow">option::borrow</a>(&optional_parent_key));
            // Immutably borrow the parent.
            <b>let</b> parent_ref = <a href="_borrow">table::borrow</a>(inners_ref, parent_key);
            // Pack its bitmask in an <a href="">option</a>.
            optional_parent_bitmask = <a href="_some">option::some</a>(parent_ref.bitmask);
        }
    }
}
</code></pre>



</details>

<a name="0xc0deb00c_critqueue_insert_leaf_above_root_node"></a>

## Function `insert_leaf_above_root_node`

Insert new leaf and inner node above the root node.

Inner function for <code><a href="critqueue.md#0xc0deb00c_critqueue_insert_leaf">insert_leaf</a>()</code>.


<a name="@Parameters_67"></a>

### Parameters


* <code>critqueue_ref_mut</code>: Mutable reference to crit-queue.
* <code>new_inner_node_key</code>: Inner key of new inner node to insert.
* <code>new_inner_node_bitmask</code>: Critical bitmask for new inner node.
* <code>new_leaf_key</code>: Leaf key of free leaf to insert to the tree.


<a name="@Assumptions_68"></a>

### Assumptions


* Given <code><a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a></code> has a free leaf with <code>new_leaf_key</code>.
* <code>new_inner_node_bitmask</code> is greater than that of root node,
which has been reached via upward walk in <code><a href="critqueue.md#0xc0deb00c_critqueue_insert_leaf">insert_leaf</a>()</code>.


<a name="@Reference_diagrams_69"></a>

### Reference diagrams


For ease of illustration, critical bitmasks and leaf keys are
depicted relative to bit 64, but tested with correspondingly
bitshifted amounts, for inner keys that are additionally
encoded with a mock insertion key, mock insertion count,
and inner node bit flag.


<a name="@Inserting_above_a_leaf_70"></a>

#### Inserting above a leaf


Both examples reference the following diagram:

>     0100


<a name="@New_leaf_as_left_child_71"></a>

##### New leaf as left child


Here, inserting <code>0000</code> yields:

>                     2nd <- new inner node
>                    /   \
>     new leaf -> 0000   0100 <- old root


<a name="@New_leaf_as_right_child_72"></a>

##### New leaf as right child


If <code>1111</code> were to be inserted instead:

>                     3rd <- new inner node
>                    /   \
>     old root -> 0100   1111 <- new leaf


<a name="@Inserting_above_an_inner_node_73"></a>

#### Inserting above an inner node


Both examples reference the following diagram:

>         1st
>        /   \
>     1001   1011


<a name="@New_leaf_as_left_child_74"></a>

##### New leaf as left child


Here, inserting <code>0001</code> yields:

>                     3rd <- new inner node
>                    /   \
>     new leaf -> 0001   1st <- old root node
>                       /   \
>                    1001   1011


<a name="@New_leaf_as_right_child_75"></a>

##### New leaf as right child


If <code>1100</code> were to be inserted instead:

>       new inner node -> 2nd
>                        /   \
>     old root node -> 1st   1100 <- new leaf
>                     /   \
>                  1001   1011


<a name="@Testing_76"></a>

### Testing


* <code>test_insert_leaf_above_root_node_1()</code>
* <code>test_insert_leaf_above_root_node_2()</code>
* <code>test_insert_leaf_above_root_node_3()</code>
* <code>test_insert_leaf_above_root_node_4()</code>


<pre><code><b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_insert_leaf_above_root_node">insert_leaf_above_root_node</a>&lt;V&gt;(critqueue_ref_mut: &<b>mut</b> <a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">critqueue::CritQueue</a>&lt;V&gt;, new_inner_node_key: u128, new_inner_node_bitmask: u128, new_leaf_key: u128)
</code></pre>



<details>
<summary>Implementation</summary>


<pre><code><b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_insert_leaf_above_root_node">insert_leaf_above_root_node</a>&lt;V&gt;(
    critqueue_ref_mut: &<b>mut</b> <a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a>&lt;V&gt;,
    new_inner_node_key: u128,
    new_inner_node_bitmask: u128,
    new_leaf_key: u128
) {
    // Borrow mutable reference <b>to</b> inner nodes <a href="">table</a>.
    <b>let</b> inners_ref_mut = &<b>mut</b> critqueue_ref_mut.inners;
    // Borrow mutable reference <b>to</b> leaves <a href="">table</a>.
    <b>let</b> leaves_ref_mut = &<b>mut</b> critqueue_ref_mut.leaves;
    // Swap out root node key for new inner key, storing <b>old</b> key.
    <b>let</b> old_root_node_key =
        <a href="_swap">option::swap</a>(&<b>mut</b> critqueue_ref_mut.root, new_inner_node_key);
    <b>let</b> old_root_is_leaf = // Determine <b>if</b> <b>old</b> root is a leaf.
        old_root_node_key & <a href="critqueue.md#0xc0deb00c_critqueue_TREE_NODE_TYPE">TREE_NODE_TYPE</a> == <a href="critqueue.md#0xc0deb00c_critqueue_TREE_NODE_LEAF">TREE_NODE_LEAF</a>;
    // Get mutable reference <b>to</b> <b>old</b> root's parent field:
    <b>let</b> old_root_parent_field_ref_mut = <b>if</b> (old_root_is_leaf)
        // If <b>old</b> root is a leaf, borrow from leaves <a href="">table</a>.
        &<b>mut</b> <a href="_borrow_mut">table::borrow_mut</a>(leaves_ref_mut, old_root_node_key).parent
            <b>else</b> // Else borrow from inner nodes <a href="">table</a>.
        &<b>mut</b> <a href="_borrow_mut">table::borrow_mut</a>(inners_ref_mut, old_root_node_key).parent;
    // Set <b>old</b> root node <b>to</b> have new inner node <b>as</b> its parent.
    <a href="_fill">option::fill</a>(old_root_parent_field_ref_mut, new_inner_node_key);
    <b>let</b> new_leaf_ref_mut = // Borrow mutable reference <b>to</b> new leaf.
        <a href="_borrow_mut">table::borrow_mut</a>(leaves_ref_mut, new_leaf_key);
    // Set new leaf <b>to</b> have new inner node <b>as</b> its parent.
    <a href="_fill">option::fill</a>(&<b>mut</b> new_leaf_ref_mut.parent, new_inner_node_key);
    // If leaf key AND new inner node's critical bitmask is 0,
    // free leaf is unset at new inner node's critical bit and
    // should thus go on its left, <b>with</b> the <b>old</b> root on new inner
    // node's right. Else the opposite.
    <b>let</b> (left, right) = <b>if</b> (new_leaf_key & new_inner_node_bitmask == 0)
        (new_leaf_key, old_root_node_key) <b>else</b>
        (old_root_node_key, new_leaf_key);
    // Add <b>to</b> inner nodes <a href="">table</a> the new inner node <b>as</b> root.
    <a href="_add">table::add</a>(inners_ref_mut, new_inner_node_key, <a href="critqueue.md#0xc0deb00c_critqueue_Inner">Inner</a>{left, right,
        bitmask: new_inner_node_bitmask, parent: <a href="_none">option::none</a>()});
}
</code></pre>



</details>

<a name="0xc0deb00c_critqueue_insert_leaf_below_anchor_node"></a>

## Function `insert_leaf_below_anchor_node`

Insert new free leaf and inner node below anchor node.

Inner function for <code><a href="critqueue.md#0xc0deb00c_critqueue_insert_leaf">insert_leaf</a>()</code>.


<a name="@Parameters_77"></a>

### Parameters


* <code>critqueue_ref_mut</code>: Mutable reference to crit-queue.
* <code>anchor_node_key</code>: Key of the inner node to insert below, the
"anchor node".
* <code>new_inner_node_key</code>: Inner key of new inner node to insert.
* <code>new_inner_node_bitmask</code>: Critical bitmask for new inner node.
* <code>new_leaf_key</code>: Leaf key of free leaf to insert to the tree.


<a name="@Assumptions_78"></a>

### Assumptions


* Given <code><a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a></code> has a free leaf with <code>new_leaf_key</code>.
* <code>new_inner_node_bitmask</code> is less than that of anchor node,
which has been reached via upward walk in <code><a href="critqueue.md#0xc0deb00c_critqueue_insert_leaf">insert_leaf</a>()</code>.
* <code>new_inner_node_bitmask</code> is greater than that of the displaced
child, if the displaced child is an inner node (see below).


<a name="@Reference_diagrams_79"></a>

### Reference diagrams


For ease of illustration, critical bitmasks and leaf keys are
depicted relative to bit 64, but tested with correspondingly
bitshifted amounts, for inner keys that are additionally
encoded with a mock insertion key, mock insertion count,
and inner node bit flag.

Both insertion examples reference the following diagram:

>         3rd
>        /   \
>     0001   1st
>           /   \
>        1001   1011


<a name="@Anchor_node_children_polarity_80"></a>

#### Anchor node children polarity


The anchor node is the node below which the free leaf and new
inner node should be inserted, for example either <code>3rd</code> or
<code>1st</code>. The free leaf key can be inserted to either the left or
the right of the anchor node, depending on its whether it is
unset or set, respectively, at the anchor node's critical bit.
For example, per below, a free leaf key of <code>1000</code> would
be inserted to the left of <code>1st</code>, while a free leaf key of
<code>1111</code> would be inserted to the right of <code>3rd</code>.


<a name="@Child_displacement_81"></a>

#### Child displacement


When a leaf key is inserted, a new inner node is generated,
which displaces either the left or the right child of the anchor
node, based on the side that the leaf key should be inserted.
For example, inserting free leaf key <code>1000</code> displaces <code>1001</code>:

>                       3rd
>                      /   \
>                   0001   1st <- anchor node
>                         /   \
>     new inner node -> 0th   1011
>                      /   \
>       new leaf -> 1000   1001 <- displaced child

Both leaves and inner nodes can be displaced. For example,
were free leaf key <code>1111</code> to be inserted instead, it would
displace <code>1st</code>:

>                        3rd <- anchor node
>                       /   \
>                    0001   2nd <- new inner node
>                          /   \
>     displaced child -> 1st   1111 <- new leaf
>                       /   \
>                    1001   1011


<a name="@New_inner_node_children_polarity_82"></a>

#### New inner node children polarity


The new inner node can have the new leaf as either its left or
right child, depending on the new inner node's critical bit.
As in the first example above, where the new leaf is unset at
the new inner node's critical bit, the new inner node's left
child is the new leaf and new inner node's right child is the
displaced child. Conversely, as in the second example above,
where the new leaf is set at the new inner node's critical bit,
the new inner node's left child is the displaced child and the
new inner node's right child is the new leaf.


<a name="@Testing_83"></a>

### Testing


* <code>test_insert_leaf_below_anchor_node_1()</code>
* <code>test_insert_leaf_below_anchor_node_2()</code>


<pre><code><b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_insert_leaf_below_anchor_node">insert_leaf_below_anchor_node</a>&lt;V&gt;(critqueue_ref_mut: &<b>mut</b> <a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">critqueue::CritQueue</a>&lt;V&gt;, anchor_node_key: u128, new_inner_node_key: u128, new_inner_node_bitmask: u128, new_leaf_key: u128)
</code></pre>



<details>
<summary>Implementation</summary>


<pre><code><b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_insert_leaf_below_anchor_node">insert_leaf_below_anchor_node</a>&lt;V&gt;(
    critqueue_ref_mut: &<b>mut</b> <a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a>&lt;V&gt;,
    anchor_node_key: u128,
    new_inner_node_key: u128,
    new_inner_node_bitmask: u128,
    new_leaf_key: u128
) {
    // Borrow mutable reference <b>to</b> inner nodes <a href="">table</a>.
    <b>let</b> inners_ref_mut = &<b>mut</b> critqueue_ref_mut.inners;
    // Borrow mutable reference <b>to</b> leaves <a href="">table</a>.
    <b>let</b> leaves_ref_mut = &<b>mut</b> critqueue_ref_mut.leaves;
    <b>let</b> anchor_node_ref_mut = // Mutably borrow anchor node.
        <a href="_borrow_mut">table::borrow_mut</a>(inners_ref_mut, anchor_node_key);
    // Get anchor node critical bitmask.
    <b>let</b> anchor_bitmask = anchor_node_ref_mut.bitmask;
    <b>let</b> displaced_child_key; // Declare displaced child key.
    // If new leaf key AND anchor bitmask is 0, new leaf is unset
    // at anchor node's critical bit and should thus go on its left:
    <b>if</b> (new_leaf_key & anchor_bitmask == 0) {
        // Displaced child is thus on anchor's left.
        displaced_child_key = anchor_node_ref_mut.left;
        // Anchor node now <b>has</b> <b>as</b> its left child the new inner node.
        anchor_node_ref_mut.left = new_inner_node_key;
    } <b>else</b> { // If new leaf goes <b>to</b> right of anchor node:
        // Displaced child is thus on anchor's right.
        displaced_child_key = anchor_node_ref_mut.right;
        // Anchor node now <b>has</b> the new inner node <b>as</b> right child.
        anchor_node_ref_mut.right = new_inner_node_key;
    };
    // Determine <b>if</b> displaced child is a leaf.
    <b>let</b> displaced_child_is_leaf =
        displaced_child_key & <a href="critqueue.md#0xc0deb00c_critqueue_TREE_NODE_TYPE">TREE_NODE_TYPE</a> == <a href="critqueue.md#0xc0deb00c_critqueue_TREE_NODE_LEAF">TREE_NODE_LEAF</a>;
    // Get mutable reference <b>to</b> displaced child's parent field:
    <b>let</b> displaced_child_parent_field_ref_mut = <b>if</b> (displaced_child_is_leaf)
        // If displaced child is a leaf, borrow from leaves <a href="">table</a>.
        &<b>mut</b> <a href="_borrow_mut">table::borrow_mut</a>(leaves_ref_mut, displaced_child_key).parent
            <b>else</b> // Else borrow from inner nodes <a href="">table</a>.
        &<b>mut</b> <a href="_borrow_mut">table::borrow_mut</a>(inners_ref_mut, displaced_child_key).parent;
    // Swap anchor node key in displaced child's parent field <b>with</b>
    // the new inner node key, discarding the anchor node key.
    <a href="_swap">option::swap</a>(displaced_child_parent_field_ref_mut, new_inner_node_key);
    // If new leaf key AND new inner node's critical bitmask is 0,
    // free leaf is unset at new inner node's critical bit and
    // should thus go on its left, <b>with</b> displaced child on new
    // inner node's right. Else the opposite.
    <b>let</b> (left, right) = <b>if</b> (new_leaf_key & new_inner_node_bitmask == 0)
        (new_leaf_key, displaced_child_key) <b>else</b>
        (displaced_child_key, new_leaf_key);
    // Add <b>to</b> inner nodes <a href="">table</a> the new inner node.
    <a href="_add">table::add</a>(inners_ref_mut, new_inner_node_key, <a href="critqueue.md#0xc0deb00c_critqueue_Inner">Inner</a>{left, right,
        bitmask: new_inner_node_bitmask,
        parent: <a href="_some">option::some</a>(anchor_node_key)});
    <b>let</b> new_leaf_ref_mut = // Borrow mutable reference <b>to</b> new leaf.
        <a href="_borrow_mut">table::borrow_mut</a>(leaves_ref_mut, new_leaf_key);
    // Set new leaf <b>to</b> have <b>as</b> its parent the new inner node.
    <a href="_fill">option::fill</a>(&<b>mut</b> new_leaf_ref_mut.parent, new_inner_node_key);
}
</code></pre>



</details>

<a name="0xc0deb00c_critqueue_insert_update_subqueue"></a>

## Function `insert_update_subqueue`

Update a sub-queue, inside an allocated leaf, during insertion.

Inner function for <code><a href="critqueue.md#0xc0deb00c_critqueue_insert">insert</a>()</code>.


<a name="@Returns_84"></a>

### Returns


* <code>u128</code>: Access key of new sub-queue node.
* <code>bool</code>: <code><b>true</b></code> if allocated leaf is a free leaf, else <code><b>false</b></code>.


<a name="@Aborts_85"></a>

### Aborts


* <code><a href="critqueue.md#0xc0deb00c_critqueue_E_TOO_MANY_INSERTIONS">E_TOO_MANY_INSERTIONS</a></code>: Insertion key encoded in <code>leaf_key</code>
has already been inserted the maximum number of times.


<a name="@Assumptions_86"></a>

### Assumptions


* <code>critqueue_ref_mut</code> indicates a <code><a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a></code> that already
contains an allocated leaf with the given <code>leaf_key</code>.
* <code>subqueue_node_ref_mut</code> indicates a <code><a href="critqueue.md#0xc0deb00c_critqueue_SubQueueNode">SubQueueNode</a></code> with the
appropriate access key, which has been initialized as if it
were the sole sub-queue node in a free leaf.


<a name="@Testing_87"></a>

### Testing


* <code>test_insert_update_subqueue_ascending_free()</code>
* <code>test_insert_update_subqueue_ascending_not_free()</code>
* <code>test_insert_update_subqueue_descending_free()</code>
* <code>test_insert_update_subqueue_descending_not_free()</code>
* <code>test_insert_update_subqueue_max_failure()</code>
* <code>test_insert_update_subqueue_max_success()</code>


<pre><code><b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_insert_update_subqueue">insert_update_subqueue</a>&lt;V&gt;(critqueue_ref_mut: &<b>mut</b> <a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">critqueue::CritQueue</a>&lt;V&gt;, subqueue_node_ref_mut: &<b>mut</b> <a href="critqueue.md#0xc0deb00c_critqueue_SubQueueNode">critqueue::SubQueueNode</a>&lt;V&gt;, leaf_key: u128): (u128, bool)
</code></pre>



<details>
<summary>Implementation</summary>


<pre><code><b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_insert_update_subqueue">insert_update_subqueue</a>&lt;V&gt;(
    critqueue_ref_mut: &<b>mut</b> <a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a>&lt;V&gt;,
    subqueue_node_ref_mut: &<b>mut</b> <a href="critqueue.md#0xc0deb00c_critqueue_SubQueueNode">SubQueueNode</a>&lt;V&gt;,
    leaf_key: u128,
): (
    u128,
    bool
) {
    // Get the sort order of the crit-queue.
    <b>let</b> order = critqueue_ref_mut.order;
    // Borrow mutable reference <b>to</b> leaves <a href="">table</a>.
    <b>let</b> leaves_ref_mut = &<b>mut</b> critqueue_ref_mut.leaves;
    // Borrow mutable reference <b>to</b> leaf.
    <b>let</b> leaf_ref_mut = <a href="_borrow_mut">table::borrow_mut</a>(leaves_ref_mut, leaf_key);
    // Get insertion count of new insertion key.
    <b>let</b> count = leaf_ref_mut.count + 1;
    // Assert max insertion count is not exceeded.
    <b>assert</b>!(count &lt;= <a href="critqueue.md#0xc0deb00c_critqueue_MAX_INSERTION_COUNT">MAX_INSERTION_COUNT</a>, <a href="critqueue.md#0xc0deb00c_critqueue_E_TOO_MANY_INSERTIONS">E_TOO_MANY_INSERTIONS</a>);
    // Update leaf insertion counter.
    leaf_ref_mut.count = count;
    // Get access key. If ascending, bits 64-127 are the same <b>as</b> the
    // leaf key, and bits 0-63 are the insertion count.
    <b>let</b> access_key = <b>if</b> (order == <a href="critqueue.md#0xc0deb00c_critqueue_ASCENDING">ASCENDING</a>) leaf_key | (count <b>as</b> u128)
        // Else same, but flipped insertion count and bit 63 set.
        <b>else</b> (leaf_key | (count <b>as</b> u128)) ^ <a href="critqueue.md#0xc0deb00c_critqueue_NOT_INSERTION_COUNT_DESCENDING">NOT_INSERTION_COUNT_DESCENDING</a>;
    // Get <b>old</b> sub-queue tail field.
    <b>let</b> old_tail = leaf_ref_mut.tail;
    // Set sub-queue <b>to</b> have new sub-queue node <b>as</b> its tail.
    leaf_ref_mut.tail = <a href="_some">option::some</a>(access_key);
    <b>let</b> free_leaf = <b>true</b>; // Assume free leaf.
    <b>if</b> (<a href="_is_none">option::is_none</a>(&old_tail)) { // If leaf is a free leaf:
        // Update sub-queue <b>to</b> have new node <b>as</b> its head.
        leaf_ref_mut.head = <a href="_some">option::some</a>(access_key);
    } <b>else</b> { // If not a free leaf:
        free_leaf = <b>false</b>; // Flag <b>as</b> such.
        // Get the access key of the <b>old</b> sub-queue tail.
        <b>let</b> old_tail_access_key = *<a href="_borrow">option::borrow</a>(&old_tail);
        // Borrow mutable reference <b>to</b> the <b>old</b> sub-queue tail.
        <b>let</b> old_tail_ref_mut = <a href="_borrow_mut">table::borrow_mut</a>(
            &<b>mut</b> critqueue_ref_mut.subqueue_nodes, old_tail_access_key);
        // Set <b>old</b> sub-queue tail <b>to</b> have <b>as</b> its next sub-queue
        // node the new sub-queue node.
        old_tail_ref_mut.next = <a href="_some">option::some</a>(access_key);
        // Set the new sub-queue node <b>to</b> have <b>as</b> its previous
        // sub-queue node the <b>old</b> sub-queue tail.
        subqueue_node_ref_mut.previous = old_tail;
    };
    (access_key, free_leaf) // Return access key and <b>if</b> free leaf.
}
</code></pre>



</details>

<a name="0xc0deb00c_critqueue_remove_free_leaf"></a>

## Function `remove_free_leaf`

Remove from the crit-queue crit-bit tree the leaf having the
given key, aborting if no such leaf in the tree.

Inner function for <code><a href="critqueue.md#0xc0deb00c_critqueue_remove">remove</a>()</code> that only gets called if the
leaf's sub-queue has been emptied, which means that the leaf
should be freed from the tree. Free leaves are not deallocated,
however, since their insertion counter is still required to
generate unique access keys for any future insertions. Rather,
their parent field is simply set to none.


<a name="@Assumptions_88"></a>

### Assumptions


* Given <code><a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a></code> has a crit-bit tree with at least one node,
which is the indicated leaf in the case of a singleton tree.


<a name="@Removing_the_root_89"></a>

### Removing the root


If removing the root, the crit-queue root field should be
updated to none. The leaf's field should already be none, since
it was the root.


<a name="@Reference_diagrams_90"></a>

### Reference diagrams


For ease of illustration, critical bitmasks and leaf keys are
depicted relative to bit 64, but tested with correspondingly
bitshifted amounts, generated via <code><a href="critqueue.md#0xc0deb00c_critqueue_insert">insert</a>()</code>.


<a name="@With_grandparent_91"></a>

#### With grandparent


1. The sibling to the freed leaf must be updated to have as its
parent the grandparent to the freed leaf.
2. The grandparent to the freed leaf must be updated to have as
its child the sibling to the freed leaf, on the same side
that the removed parent was a child on.
3. The removed parent node must be destroyed.
4. The freed leaf must have its parent field set to none.


<a name="@Inner_node_sibling_92"></a>

##### Inner node sibling


>           grandparent -> 2nd
>                         /   \
>     removed parent -> 1st   111
>                      /   \
>      freed leaf -> 000   0th <- sibling
>                         /   \
>                       010   011

Becomes

>     old grandparent -> 2nd
>                       /   \
>      old sibling -> 0th   111
>                    /   \
>                  010   011


<a name="@Leaf_sibling_93"></a>

##### Leaf sibling


>                2nd <- grandparent
>               /   \
>             001   1st <- removed parent
>                  /   \
>     sibling -> 101   111 <- freed leaf

Becomes

>        2nd <- old grandparent
>       /   \
>     001   101 <- old sibling


<a name="@Without_grandparent_94"></a>

#### Without grandparent


1. The sibling to the freed leaf must be updated to indicate
that it has no parent.
2. The crit-queue root must be updated to have as its root the
sibling to the freed leaf.
3. The removed parent node must be destroyed.
4. The freed leaf must have its parent field set to none.


<a name="@Inner_node_sibling_95"></a>

##### Inner node sibling


>         parent -> 2nd
>                  /   \
>     sibling -> 0th   111 <- freed leaf
>               /   \
>             010   011

Becomes

>        0th <- old sibling
>       /   \
>     010   011


<a name="@Leaf_sibling_96"></a>

##### Leaf sibling


>                      2nd <- parent
>                     /   \
>     freed leaf -> 001   101 <- sibling

Becomes

>     101 <- old sibling


<a name="@Testing_97"></a>

### Testing


* <code>test_remove_free_leaf_inner_sibling()</code>
* <code>test_remove_free_leaf_leaf_sibling_root()</code>


<pre><code><b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_remove_free_leaf">remove_free_leaf</a>&lt;V&gt;(critqueue_ref_mut: &<b>mut</b> <a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">critqueue::CritQueue</a>&lt;V&gt;, leaf_key: u128)
</code></pre>



<details>
<summary>Implementation</summary>


<pre><code><b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_remove_free_leaf">remove_free_leaf</a>&lt;V&gt;(
    critqueue_ref_mut: &<b>mut</b> <a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a>&lt;V&gt;,
    leaf_key: u128,
) {
    // If the indicated leaf is the root of the tree, then set the
    // crit-bit root <b>to</b> none.
    <b>if</b> (leaf_key == *<a href="_borrow">option::borrow</a>(&critqueue_ref_mut.root)) <b>return</b>
        critqueue_ref_mut.root = <a href="_none">option::none</a>();
    // Mutably borrow inner nodes <a href="">table</a>.
    <b>let</b> inners_ref_mut = &<b>mut</b> critqueue_ref_mut.inners;
    // Mutably borrow leaves <a href="">table</a>.
    <b>let</b> leaves_ref_mut = &<b>mut</b> critqueue_ref_mut.leaves;
    // Mutably borrow leaf <b>to</b> remove.
    <b>let</b> leaf_ref_mut = <a href="_borrow_mut">table::borrow_mut</a>(leaves_ref_mut, leaf_key);
    // Get the inner key of its parent.
    <b>let</b> parent_key = *<a href="_borrow">option::borrow</a>(&leaf_ref_mut.parent);
    // Set freed leaf <b>to</b> indicate it <b>has</b> no parent.
    leaf_ref_mut.parent = <a href="_none">option::none</a>();
    // Immutably borrow the parent <b>to</b> remove.
    <b>let</b> parent_ref = <a href="_borrow">table::borrow</a>(inners_ref_mut, parent_key);
    // If freed leaf key AND parent's bitmask is 0, freed leaf key
    // was not set at parent's critical bit, thus was a left child.
    <b>let</b> free_leaf_was_left_child = leaf_key & parent_ref.bitmask == 0;
    // Sibling key is from field on opposite side of free leaf.
    <b>let</b> sibling_key = <b>if</b> (free_leaf_was_left_child) parent_ref.right <b>else</b>
        parent_ref.left;
    // Declare sibling's new parent field.
    <b>let</b> sibling_new_parent_field;
    <b>if</b> (<a href="_is_none">option::is_none</a>(&parent_ref.parent)) { // If parent is root:
        // Sibling is now root, and thus <b>has</b> none for parent field.
        sibling_new_parent_field = <a href="_none">option::none</a>();
        // Update crit-queue root field <b>to</b> indicate the sibling.
        critqueue_ref_mut.root = <a href="_some">option::some</a>(sibling_key);
    } <b>else</b> { // If freed leaf <b>has</b> a grandparent:
        // Get grandparent's inner key.
        <b>let</b> grandparent_key = *<a href="_borrow">option::borrow</a>(&parent_ref.parent);
        // Set it <b>as</b> sibling's new parent field.
        sibling_new_parent_field = <a href="_some">option::some</a>(grandparent_key);
        <b>let</b> grandparent_ref_mut = // Mutably borrow grandparent.
            <a href="_borrow_mut">table::borrow_mut</a>(inners_ref_mut, grandparent_key);
        // If freed leaf key AND grandparent's bitmask is 0, freed
        // leaf key was not set at grandparent's critical bit, and
        // was thus located <b>to</b> its left.
        <b>let</b> free_leaf_was_left_grandchild = leaf_key &
            grandparent_ref_mut.bitmask == 0;
        // Update grandparent <b>to</b> have <b>as</b> its child the freed leaf's
        // sibling on the same side.
        <b>if</b> (free_leaf_was_left_grandchild) grandparent_ref_mut.left =
            sibling_key <b>else</b> grandparent_ref_mut.right = sibling_key;
    };
    <b>let</b> sibling_is_leaf = // Determine <b>if</b> sibling is a leaf.
        sibling_key & <a href="critqueue.md#0xc0deb00c_critqueue_TREE_NODE_TYPE">TREE_NODE_TYPE</a> == <a href="critqueue.md#0xc0deb00c_critqueue_TREE_NODE_LEAF">TREE_NODE_LEAF</a>;
    // Get mutable reference <b>to</b> sibling's parent field:
    <b>let</b> sibling_parent_field_ref_mut = <b>if</b> (sibling_is_leaf)
        // If sibling is a leaf, borrow from leaves <a href="">table</a>.
        &<b>mut</b> <a href="_borrow_mut">table::borrow_mut</a>(leaves_ref_mut, sibling_key).parent
            <b>else</b> // Else borrow from inner nodes <a href="">table</a>.
        &<b>mut</b> <a href="_borrow_mut">table::borrow_mut</a>(inners_ref_mut, sibling_key).parent;
    // Set sibling's parent field <b>to</b> the new parent field.
    *sibling_parent_field_ref_mut = sibling_new_parent_field;
    <b>let</b> <a href="critqueue.md#0xc0deb00c_critqueue_Inner">Inner</a>{bitmask: _, parent: _, left: _, right: _} = <a href="_remove">table::remove</a>(
        inners_ref_mut, parent_key); // Destroy parent inner node.
}
</code></pre>



</details>

<a name="0xc0deb00c_critqueue_remove_subqueue_node"></a>

## Function `remove_subqueue_node`

Remove insertion value corresponding to given access key,
aborting if no such access key in crit-queue.

Inner function for <code><a href="critqueue.md#0xc0deb00c_critqueue_remove">remove</a>()</code>.

Does not update parent field of corresponding leaf if its
sub-queue is emptied, as the parent field may be required for
traversal in <code><a href="critqueue.md#0xc0deb00c_critqueue_remove">remove</a>()</code> before the leaf is freed from the tree
in <code><a href="critqueue.md#0xc0deb00c_critqueue_remove_free_leaf">remove_free_leaf</a>()</code>.


<a name="@Parameters_98"></a>

### Parameters


* <code>critqueue_ref_mut</code>: Mutable reference to given <code><a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a></code>.
* <code>access_key</code>: Access key corresponding to insertion value.


<a name="@Returns_99"></a>

### Returns


* <code>V</code>: Insertion value corresponding to <code>access_key</code>.
* <code>Option&lt;Option&lt;u128&gt;&gt;</code>: The new sub-queue head field indicated
by the corresponding leaf (<code><a href="critqueue.md#0xc0deb00c_critqueue_Leaf">Leaf</a>.head</code>), if removal resulted
in an update to it.


<a name="@Reference_diagrams_100"></a>

### Reference diagrams



<a name="@Conventions_101"></a>

#### Conventions


For ease of illustration, sub-queue node leaf keys are depicted
relative to bit 64, but tested with correspondingly bitshifted
amounts. Insertion counts and values are given in decimal:

>                                           1st
>                                          /   \
>                                        101   111
>     [n_0{7} -> n_1{8} -> n_2{4} -> n_3{5}]   [n_0{9}]

Here, all sub-queue nodes in the left sub-queue have insertion
key <code>101</code>, for <code>n_i{j}</code> indicating a sub-queue node with
insertion count <code>i</code> and insertion value <code>j</code>.


<a name="@Removal_sequence_102"></a>

#### Removal sequence


1. Remove <code>n_2{4}</code>, neither the sub-queue head nor tail,
yielding:

>                                 1st
>                                /   \
>                              101   111
>     [n_0{7} -> n_1{8} -> n_3{5}]   [n_0{9}]

2. Remove <code>n_3{5}</code>, the sub-queue tail:

>                       1st
>                      /   \
>                    101   111
>     [n_0{7} -> n_1{8}]   [n_0{9}]

3. Remove <code>n_0{7}</code>, the sub-queue head:

>             1st
>            /   \
>          101   111
>     [n_1{8}]   [n_0{9}]

4. Remove <code>n_1{8}</code>, the sub-queue head and tail, yielding a
leaf that is still in the crit-bit tree but has an empty
sub-queue:

>        1st
>       /   \
>     101   111
>     []    [n_0{9}]


<a name="@Testing_103"></a>

### Testing


* <code>test_remove_subqueue_node()</code>


<pre><code><b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_remove_subqueue_node">remove_subqueue_node</a>&lt;V&gt;(critqueue_ref_mut: &<b>mut</b> <a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">critqueue::CritQueue</a>&lt;V&gt;, access_key: u128): (V, <a href="_Option">option::Option</a>&lt;<a href="_Option">option::Option</a>&lt;u128&gt;&gt;)
</code></pre>



<details>
<summary>Implementation</summary>


<pre><code><b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_remove_subqueue_node">remove_subqueue_node</a>&lt;V&gt;(
    critqueue_ref_mut: &<b>mut</b> <a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a>&lt;V&gt;,
    access_key: u128
): (
    V,
    Option&lt;Option&lt;u128&gt;&gt;
) {
    // Mutably borrow sub-queue nodes <a href="">table</a>.
    <b>let</b> subqueue_nodes_ref_mut = &<b>mut</b> critqueue_ref_mut.subqueue_nodes;
    // Mutably borrow leaves <a href="">table</a>.
    <b>let</b> leaves_ref_mut = &<b>mut</b> critqueue_ref_mut.leaves;
    // Mutably borrow leaf containing corresponding sub-queue.
    <b>let</b> leaf_ref_mut = <a href="_borrow_mut">table::borrow_mut</a>(leaves_ref_mut, access_key &
        <a href="critqueue.md#0xc0deb00c_critqueue_ACCESS_KEY_TO_LEAF_KEY">ACCESS_KEY_TO_LEAF_KEY</a>);
    // Remove sub-queue node from <a href="">table</a> and unpack its fields.
    <b>let</b> <a href="critqueue.md#0xc0deb00c_critqueue_SubQueueNode">SubQueueNode</a>{insertion_value, previous, next} = <a href="_remove">table::remove</a>(
        subqueue_nodes_ref_mut, access_key);
    // Assume sub-queue head is unaltered by removal.
    <b>let</b> optional_new_subqueue_head_field = <a href="_none">option::none</a>();
    <b>if</b> (<a href="_is_none">option::is_none</a>(&previous)) { // If node was sub-queue head:
        // Set <b>as</b> sub-queue head field the node's next field.
        leaf_ref_mut.head = next;
        // Update optional new sub-queue head field <b>return</b> value.
        optional_new_subqueue_head_field = <a href="_some">option::some</a>(next);
    } <b>else</b> { // If node was not sub-queue head:
        // Update the node having the previous access key <b>to</b> have <b>as</b>
        // its next field the next field of the removed node.
        <a href="_borrow_mut">table::borrow_mut</a>(subqueue_nodes_ref_mut,
            *<a href="_borrow">option::borrow</a>(&previous)).next = next;
    };
    <b>if</b> (<a href="_is_none">option::is_none</a>(&next)) { // If node was sub-queue tail:
        // Set <b>as</b> sub-queue tail field the node's previous field.
        leaf_ref_mut.tail = previous;
    } <b>else</b> { // If node was not sub-queue tail:
        // Update the node having the next access key <b>to</b> have <b>as</b> its
        // previous field the previous field of the removed node.
        <a href="_borrow_mut">table::borrow_mut</a>(subqueue_nodes_ref_mut,
            *<a href="_borrow">option::borrow</a>(&next)).previous = previous;
    };
    // Return insertion value and optional new sub-queue head field.
    (insertion_value, optional_new_subqueue_head_field)
}
</code></pre>



</details>

<a name="0xc0deb00c_critqueue_search"></a>

## Function `search`

Search in given <code><a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a></code> for closest match to <code>seed_key</code>.

If root is a leaf, return immediately. Otherwise, starting at
the root, walk down from inner node to inner node, branching
left whenever <code>seed_key</code> is unset at an inner node's critical
bit, and right whenever <code>seed_key</code> is set at an inner node's
critical bit. After arriving at a leaf, known as the "match
leaf", return its leaf key, the inner key of its parent, and the
parent's critical bitmask.


<a name="@Returns_104"></a>

### Returns


* <code>u128</code>: Match leaf key.
* <code>Option&lt;u128&gt;</code>: Match parent inner key, if any.
* <code>Option&lt;u128&gt;</code>: Match parent's critical bitmask, if any.


<a name="@Assumptions_105"></a>

### Assumptions


* Given <code><a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a></code> does not have an empty crit-bit tree.


<a name="@Reference_diagrams_106"></a>

### Reference diagrams


For ease of illustration, critical bitmasks and leaf keys are
depicted relative to bit 64, but tested with correspondingly
bitshifted amounts, for inner keys that are additionally
encoded with a mock insertion key, mock insertion count,
and inner node bit flag.


<a name="@Leaf_at_root_107"></a>

#### Leaf at root


>     111

| <code>seed_key</code> | Match leaf key | Match parent bitmask |
|------------|----------------|----------------------|
| Any        | <code>111</code>          | None                 |


<a name="@Inner_node_at_root_108"></a>

#### Inner node at root


>        2nd
>       /   \
>     001   1st
>          /   \
>        101   111

| <code>seed_key</code> | Match leaf key  | Match parent bitmask  |
|------------|-----------------|-----------------------|
| <code>011</code>      | <code>001</code>           | <code>2nd</code>                 |
| <code>100</code>      | <code>101</code>           | <code>1st</code>                 |
| <code>111</code>      | <code>111</code>           | <code>1st</code>                 |


<a name="@Testing_109"></a>

### Testing


* <code>test_search()</code>


<pre><code><b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_search">search</a>&lt;V&gt;(critqueue_ref_mut: &<b>mut</b> <a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">critqueue::CritQueue</a>&lt;V&gt;, seed_key: u128): (u128, <a href="_Option">option::Option</a>&lt;u128&gt;, <a href="_Option">option::Option</a>&lt;u128&gt;)
</code></pre>



<details>
<summary>Implementation</summary>


<pre><code><b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_search">search</a>&lt;V&gt;(
    critqueue_ref_mut: &<b>mut</b> <a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a>&lt;V&gt;,
    seed_key: u128
): (
    u128,
    Option&lt;u128&gt;,
    Option&lt;u128&gt;
) {
    // Get crit-queue root key.
    <b>let</b> root_key = *<a href="_borrow">option::borrow</a>(&critqueue_ref_mut.root);
    // If root is a leaf, <b>return</b> its leaf key, and indicate that
    // it does not have a parent.
    <b>if</b> (root_key & <a href="critqueue.md#0xc0deb00c_critqueue_TREE_NODE_TYPE">TREE_NODE_TYPE</a> == <a href="critqueue.md#0xc0deb00c_critqueue_TREE_NODE_LEAF">TREE_NODE_LEAF</a>) <b>return</b>
        (root_key, <a href="_none">option::none</a>(), <a href="_none">option::none</a>());
    <b>let</b> parent_key = root_key; // Otherwise begin walk from root.
    // Borrow mutable reference <b>to</b> <a href="">table</a> of inner nodes.
    <b>let</b> inners_ref_mut = &<b>mut</b> critqueue_ref_mut.inners;
    // Initialize match parent <b>to</b> corresponding node.
    <b>let</b> parent_ref_mut = <a href="_borrow_mut">table::borrow_mut</a>(inners_ref_mut, parent_key);
    <b>loop</b> { // Loop over inner nodes until arriving at a leaf:
        // Get bitmask of inner node for current iteration.
        <b>let</b> parent_bitmask = parent_ref_mut.bitmask;
        // If leaf key AND inner node's critical bitmask is 0, then
        // the leaf key is unset at the critical bit, so branch <b>to</b>
        // the inner node's left child. Else the right child.
        <b>let</b> child_key = <b>if</b> (seed_key & parent_bitmask == 0)
            parent_ref_mut.left <b>else</b> parent_ref_mut.right;
        // If child is a leaf, have arrived at the match leaf.
        <b>if</b> (child_key & <a href="critqueue.md#0xc0deb00c_critqueue_TREE_NODE_TYPE">TREE_NODE_TYPE</a> == <a href="critqueue.md#0xc0deb00c_critqueue_TREE_NODE_LEAF">TREE_NODE_LEAF</a>) <b>return</b>
            // So <b>return</b> the match leaf key, the inner key of the
            // match parent, and the match parent's bitmask, <b>with</b>
            // the latter two packed in an <a href="">option</a>
            (child_key, <a href="_some">option::some</a>(parent_key),
                <a href="_some">option::some</a>(parent_bitmask));
        // If have not returned, child is an inner node, so inner
        // key for next iteration becomes parent key.
        parent_key = child_key;
        // Borrow mutable reference <b>to</b> new inner node <b>to</b> check.
        parent_ref_mut = <a href="_borrow_mut">table::borrow_mut</a>(inners_ref_mut, parent_key);
    }
}
</code></pre>



</details>

<a name="0xc0deb00c_critqueue_traverse"></a>

## Function `traverse`

Traverse from leaf to inorder predecessor or successor.


<a name="@Parameters_110"></a>

### Parameters


* <code>critqueue_ref</code>: Immutable reference to given <code><a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a></code>.
* <code>start_leaf_key</code>: Leaf key of leaf to traverse from.
* <code>target</code>: Either <code><a href="critqueue.md#0xc0deb00c_critqueue_PREDECESSOR">PREDECESSOR</a></code> or <code><a href="critqueue.md#0xc0deb00c_critqueue_SUCCESSOR">SUCCESSOR</a></code>.


<a name="@Returns_111"></a>

### Returns


* <code>Option&lt;u128&gt;</code>: Leaf key of either inorder predecessor or
successor to leaf having <code>start_leaf_key</code>, if any.


<a name="@Membership_considerations_112"></a>

### Membership considerations


* Aborts if no leaf in crit-queue with given <code>start_leaf_key</code>.
* Returns none if <code>start_leaf_key</code> indicates a free leaf.
* Returns none if <code>start_leaf_key</code> indicates crit-bit root.


<a name="@Reference_diagram_113"></a>

### Reference diagram



<a name="@Conventions_114"></a>

#### Conventions


For ease of illustration, critical bitmasks and leaf keys are
depicted relative to bit 64, but tested with correspondingly
bitshifted amounts. Insertion keys are given in binary:

>            3rd
>           /   \
>         2nd   1000
>        /   \
>     0011   1st
>           /   \
>         0th   0110
>        /   \
>     0100   0101

Traversal starts at the "start leaf", walks to an "apex node",
then ends at the "target leaf", if any.


<a name="@Inorder_predecessor_115"></a>

#### Inorder predecessor


1. Walk up from the start leaf until arriving at the inner node
that has the start leaf key as the minimum key in its right
subtree, the apex node: walk up until arriving at a parent
that has the last walked node as its right child.
2. Walk down to the maximum key in the apex node's left subtree,
the target leaf: walk to apex node's left child, then walk
along right children, breaking out at a leaf.

| Start key | Apex node | Target leaf |
|-----------|-----------|-------------|
| <code>1000</code>    | <code>3rd</code>     | <code>0110</code>      |
| <code>0110</code>    | <code>1st</code>     | <code>0101</code>      |
| <code>0101</code>    | <code>0th</code>     | <code>0100</code>      |
| <code>0100</code>    | <code>2nd</code>     | <code>0011</code>      |
| <code>0011</code>    | None      | None        |


<a name="@Inorder_successor_116"></a>

#### Inorder successor


1. Walk up from the start leaf until arriving at the inner node
that has the start leaf key as the maximum key in its left
subtree, the apex node: walk up until arriving at a parent
that has the last walked node as its left child.
2. Walk down to the minimum key in the apex node's right
subtree, the target leaf: walk to apex node's right child,
then walk along left children, breaking out at a leaf.

| Start key | Apex node | Target leaf |
|-----------|-----------|-------------|
| <code>0011</code>    | <code>2nd</code>     | <code>0100</code>      |
| <code>0100</code>    | <code>0th</code>     | <code>0101</code>      |
| <code>0101</code>    | <code>1st</code>     | <code>0110</code>      |
| <code>0110</code>    | <code>3rd</code>     | <code>1000</code>      |
| <code>1000</code>    | None      | None        |


<a name="@Testing_117"></a>

### Testing


* <code>test_traverse()</code>


<pre><code><b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_traverse">traverse</a>&lt;V&gt;(critqueue_ref: &<a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">critqueue::CritQueue</a>&lt;V&gt;, start_leaf_key: u128, target: bool): <a href="_Option">option::Option</a>&lt;u128&gt;
</code></pre>



<details>
<summary>Implementation</summary>


<pre><code><b>fun</b> <a href="critqueue.md#0xc0deb00c_critqueue_traverse">traverse</a>&lt;V&gt;(
    critqueue_ref: &<a href="critqueue.md#0xc0deb00c_critqueue_CritQueue">CritQueue</a>&lt;V&gt;,
    start_leaf_key: u128,
    target: bool
): Option&lt;u128&gt; {
    // Immutably borrow leaves <a href="">table</a>.
    <b>let</b> leaves_ref = &critqueue_ref.leaves;
    // Immutably borrow inner nodes <a href="">table</a>.
    <b>let</b> inners_ref = &critqueue_ref.inners;
    // Immutably borrow start leaf, first child for upward walk.
    <b>let</b> start_leaf_ref = <a href="_borrow">table::borrow</a>(leaves_ref, start_leaf_key);
    // Immutably borrow optional parent key.
    <b>let</b> optional_parent_key_ref = &start_leaf_ref.parent;
    <b>let</b> parent_ref; // Declare reference <b>to</b> parent node.
    <b>loop</b> { // Begin upward walk <b>to</b> apex node.
        // If no parent <b>to</b> walk <b>to</b>, <b>return</b> no target node.
        <b>if</b> (<a href="_is_none">option::is_none</a>(optional_parent_key_ref)) <b>return</b>
            <a href="_none">option::none</a>();
        // Get inner key of parent.
        <b>let</b> parent_key = *<a href="_borrow">option::borrow</a>(optional_parent_key_ref);
        // Immutably borrow parent.
        parent_ref = <a href="_borrow">table::borrow</a>(inners_ref, parent_key);
        <b>let</b> bitmask = parent_ref.bitmask; // Get parent's bitmask.
        // If predecessor traversal and leaf key is set at critical
        // bit (<b>if</b> upward walk <b>has</b> reached parent via right child),
        <b>if</b> ((target == <a href="critqueue.md#0xc0deb00c_critqueue_PREDECESSOR">PREDECESSOR</a> && bitmask & start_leaf_key != 0) ||
             // or <b>if</b> successor traversal and leaf key is unset at
             // critical bit (<b>if</b> upward walk <b>has</b> reached parent via
             // left child):
            (target == <a href="critqueue.md#0xc0deb00c_critqueue_SUCCESSOR">SUCCESSOR</a>   && bitmask & start_leaf_key == 0))
             <b>break</b>; // Then <b>break</b>, since apex node <b>has</b> been reached.
        // Otherwise keep looping, checking the parent's parent.
        optional_parent_key_ref = &parent_ref.parent;
    }; // Now at apex node.
    // If predecessor traversal review apex node's left child next,
    <b>let</b> child_key = <b>if</b> (target == <a href="critqueue.md#0xc0deb00c_critqueue_PREDECESSOR">PREDECESSOR</a>) parent_ref.left <b>else</b>
        parent_ref.right; // Else the right child.
    // While the child under review is an inner node:
    <b>while</b> (child_key & <a href="critqueue.md#0xc0deb00c_critqueue_TREE_NODE_TYPE">TREE_NODE_TYPE</a> == <a href="critqueue.md#0xc0deb00c_critqueue_TREE_NODE_INNER">TREE_NODE_INNER</a>) {
        // Immutably borrow the child.
        <b>let</b> child_ref = <a href="_borrow">table::borrow</a>(inners_ref, child_key);
        // For the next iteration, review the child's right child
        // <b>if</b> predecessor traversal, <b>else</b> the left child.
        child_key = <b>if</b> (target == <a href="critqueue.md#0xc0deb00c_critqueue_PREDECESSOR">PREDECESSOR</a>) child_ref.right <b>else</b>
            child_ref.left;
    }; // Have arrived at a leaf.
    <a href="_some">option::some</a>(child_key) // Return <a href="">option</a>-packed target leaf key.
}
</code></pre>



</details>
