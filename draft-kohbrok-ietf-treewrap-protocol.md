---
title: "TreeWrap Protocol"
category: info

docname: draft-kohbrok-ietf-treewrap-protocol-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: ""
workgroup: "IETF"
keyword:
 - encryption
 - forward secrecy
 - post-compromise security
venue:
  group: "IETF"
  type: "Internet Engineering Task Force"
  mail: ""
  arch: ""
  github: "kkohbrok/ietf-treewrap-protocol"
  latest: "https://kkohbrok.github.io/ietf-treewrap-protocol/draft-kohbrok-ietf-treewrap-protocol.html"

author:
 -
    fullname: Raphael Robert
    organization: Phoenix R&D
    email: ietf@raphaelrobert.com
 -
    fullname: Konrad Kohbrok
    organization: Phoenix R&D
    email: konrad.kohbrok@datashrine.de
 -
    fullname: Joël Alwen
    email: alwenjo@amazon.com
 -
    fullname: Marta Mularczyk
    email: mulmarta@amazon.com

normative:

informative:

--- abstract

TreeWrap is a protocol to distribute and manage a set of keys. Keys can be
added, removed (to provide forward secrecy) and updated (to provide
post-compromise security). TreeWrap key material can be stored and distributed
efficiently through untrusted infrastructure.

--- middle

# Introduction

# Protocol overview

A set of keys managed through the TreeWrap protocol can be distributed and
stored on untrusted infrastructure as a binary tree called the _key tree_.

The key that allows the recovery of the set of keys from a key tree is called the
tree's `main_key`.

The nodes of the tree are either references the parent key or encrypted keys.

~~~ tls
enum {
  reserved(0),
  parent_key_ref(1),
  encrypted_key(2),
} NodeType

struct {
  NodeType type;
  select (Node.type) {
    case parent_key_ref:
      struct{};
    case encrypted_key:
      opaque ciphertext<V>;
  }
} Node

struct {
  Node content;
  bool blank;
} LeafNode
~~~

Each leaf of the tree is either blank (if `blank = true`), an encrypted key in
the key set or a reference to a key in the key set. The width of the tree minus
the number of blanks is thus the size of the key set.

If a node is an encrypted key, it was encrypted either under the key encrypted
in the node's parent node, the key referenced by the parent node, or the root
node (if the node in question is the root node).

If the root node is of type `parent_key_ref`, it's a reference to the key tree's
`main_key`.

## Initializing a key tree

A key tree is initialized by setting the root (and only node) of the tree to be
a reference to the `main_key`.

## Setting a direct path

All operations (add, remove and update) use the following basic algorithm that
take as input a the current `main_key`, the new `main_key`, as well as the
direct path (including the leaf) and the copath of the leaf index that is the
target of the operation.

- Recover the keys in the direct path starting with the root node (which is
  always a reference to the current `main_key`)
  - If the node is a reference to a key, replace the node with the key
  - If the node is an encrypted key, use the key in the parent node to decrypt
    the key
- Decrypt the encrypted keys in all copath nodes of the leaf index with the keys
  in their respective parent nodes
- Replace the nodes in the direct path with references to the new `main_key`
- Encrypt the keys in the copath with the new `main_key`

The outputs of the operation are the new nodes in the direct path and copath of
the affected index. The outputs consist only of key references and ciphertexts
and can be sent to untrusted parties to update copies of the key tree.

## Adding keys

When a key is added to the tree, the new key either fills a blank leaf (see
{{removing-keys}}) or, if there are no blank leaves, the tree is extended.

If there are no blank leaves, the tree is extended using the following
algorithm.

- If the tree is full, two new nodes are added: A new root and a new leaf, where
  the previous root and the new leaf are the children of the new root.
- If the tree is not full, two new nodes are added: A new parent node and a new
  leaf. The new parent node is added as the leaf of the right-most parent node,
  where the children are roots of sub-trees of different debth. The left child
  of the new parent node is the former right child of its parent node. The right
  child of the new parent node is the new leaf node.

In both cases, both the new parent and the new leaf node are references to the
new `main_key`. Leaf nodes added in this way have `blank` set to `false`.

If there is a blank leaf in the tree, the direct path is set as described in
{{setting-a-direct-path}} with the new key as the new `main_key` and the index
of the blank leaf as the affected index.

{{add-operation}} depicts the key tree through two add operations. `{k0}_k1`
denotes the key `k0` encrypted under `k1`.

~~~ ascii-art
         k1_ref
     ______|______
    /             \
{k0}_k1         k1_ref



                        k2_ref
             _____________|
            /              \
        {k1}_k2             \
     ______|______           \
    /             \           \
{k0}_k1         k1_ref      k2_ref



                        k3_ref
             _____________|_____________
            /                           \
        {k1}_k3                       k3_ref
     ______|______                ______|______
    /             \              /             \
{k0}_k1         k1_ref       {k2}_k3         k3_ref
~~~
{: #add-operation title="A sequence of two add operations" }

## Updating keys

When updating a key in a specific index, the new key is set to be the new
`main_key` and the direct path of the target key is set as described in
{{setting-a-direct-path}}.

~~~ ascii-art
                        k3_ref
             _____________|_____________
            /                           \
        {k1}_k3                       k3_ref
     ______|______                ______|______
    /             \              /             \
{k0}_k1         k1_ref       {k2}_k3         k3_ref

                        k4_ref
             _____________|_____________
            /                           \
         k4_ref                      {k3}_k4
     ______|______                ______|______
    /             \              /             \
{k0}_k4         k4_ref       {k2}_k3         k3_ref
~~~
{: #update-operation title="Updating k1 to k4" }

## Removing keys

A leaf that is targeted for removal and that are not the right-most leaf of the
tree is not so much removed as overwritten by a new leaf marked as blank.

When removing a key, a fresh `main_key` is stampled and the target leaf and the
direct path of the target leaf's index is set as described in
{{setting-a-direct-path}}. The target leaf is additionally marked as blank by
setting `blank = true`.

If the target leaf is the right-most leaf of the tree, that node, as well as its
parent node are removed from the tree. If the removed parent node was not the
root node, its other child becomes the right child of the removed parent's
parent node.

~~~ ascii-art
                        k3_ref
             _____________|_____________
            /                           \
        {k1}_k3                       k3_ref
     ______|______                ______|______
    /             \              /             \
{k0}_k1         k1_ref       {k2}_k3         k3_ref

                        k5_ref
             _____________|_
            /               \
        {k4}_k5              \
     ______|______            \
    /             \            \
{k0}_k4         k4_ref       {k2}_k5

        {k4}_k6
     ______|______
    /             \
{k0}_k4         k4_ref
~~~
{: #remove-operations title="Removing first k3 and then k2." }

Note that in the sequence of operations shown in {{remove-operations}}, neither
k5 nor k6 are part of the key set. Both are generated as part of a remove
operation as a nessecity to purge the target keys from the tree. As a
consequence, they will not be in the key set that results from decrypting the
key tree as described in {{decryption}}.

## Decryption

Any party in possession of the current `main_key` can decrypt a copy of the key
tree using the following algorithm starting at the root. Decryption will yield a
vector of keys.

- If the node is an encrypted key, decrypt it using the key in the parent node
  (or the `main_key` if the node is the root).
  - If the node is a non-blank leaf node, add the key to the output set at the
    index of the leaf, otherwise store the key and continue.
  - If the node is a blank leaf node, terminate this execution of the algorithm.
  - If the node is a parent node, store the key and execute this algorithm with
    both of the node's children.
- If the node is a reference proceed as follows:
  - If the node is a leaf node, follow the references until a node
    with a key is reached.
    - If the root is reached without finding a key, add the `main_key` to the
      output set at the index of the leaf node.
    - If a parent node with a key is reached, add that key to the output set at
      the index of the leaf node.
  - If the node is a parent node execute the algorithm for both children of the
    node.

After executing the algorithm, the output set contains all keys in the key set.

A party that is already in possession of the key set and the `main_key` of a key
tree can perform any operation on the key tree from just the new direct path and
copath nodes.

TODO: Write up an algorithm to do this.

## Security properties

TreeWrap provides the following security guarantees, analogous to
forward-secrecy and post-compromise security:

- An adversary in possession of the current `main_key` and key tree, as well as
  any number of past key trees can decrypt the current key set, but will not
  gain access to keys that were updated or removed in the past.

- An adversary in possession of the current `main_key` and key tree, as well as
  any number of past key trees that observes key tree updates in the future will
  not gain access to newly added keys or new values of updated keys.
