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

The nodes of the tree are either encrypted keys, references to (encrypted) keys
or references to the `main_key`.

Each leaf of the tree is either an encrypted key in the key set or a reference
to a key in the key set. The width of the tree is thus the size of the key set.

If a node is a ciphertext, it is encrypted either by the key encrypted in the
node's parent node, or the key referenced by the parent node.

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
new `main_key`.

If there is a blank leaf, the direct path is set as described in
{{setting-a-direct-path}} with the new key as the new `main_key` and the index
of the blank leaf as the affected index.

~~~ ascii-art
                                                      k2_ref                                    k3_ref                     
                                           _____________|                            _____________|_____________           
                                          /              \                          /                           \          
         k1_ref                       {k1}_k2             \                     {k1}_k3                       k3_ref       
     ______|______                 ______|______           \                 ______|______                ______|______    
    /             \               /             \           \               /             \              /             \   
{k0}_k1         k1_ref        {k0}_k1         k1_ref      k2_ref        {k0}_k1         k1_ref       {k2}_k3         k3_ref
~~~



## Removing keys

When removing a key in a specific index, the new `main_key` is freshly sampled
and the direct path of the index is set as described in
{{setting-a-direct-path}}.

## Updating keys

When updating a key in a specific index, the new key is set to be the new
`main_key` and the direct path of the index is set as described in
{{setting-a-direct-path}}.

## Decryption

Any party in possession of the current `main_key` can decrypt a copy of the key
tree using the following algorithm starting at the root.

- If the node is an encrypted key, decrypt it using the key in the parent node
  (or the `main_key` if the node is the root) and add it to the output set.
- If the node is a reference do nothing.
- If the node is a parent node, execute the algorithm for both children of the
  node.

After executing the algorithm, the output set contains all keys in the key set.

## Security properties

TreeWrap provides the following security guarantees, analogous to
forward-secrecy and post-compromise security:

- An adversary in possession of the current `main_key` and key tree, as well as
  any number of past key trees can decrypt the current key set, but will not
  gain access to keys that were updated or removed in the past.

- An adversary in possession of the current `main_key` and key tree, as well as
  any number of past key trees that observes key tree updates in the future will
  not gain access to newly added keys or new values of updated keys.
