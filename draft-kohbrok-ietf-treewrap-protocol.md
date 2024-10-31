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

TreeWrap is a protocol to encrypt a vector of records that allows individual
records be added, updated and removed. The key material used for encryption is
managed to provide forward-secrecy and post-compromise security. Operations on
the vector of records are efficient both in terms of computational effort, as
well as size of the resulting update to the encrypted records.

--- middle

# Introduction

# Protocol overview

The protocol state consists of four elements:

- Vector of records: The vector of records to be encrypted
- AEAD Scheme: The AEAD scheme to encrypt the records and key material
- `main_key`: A key that allows the holder to decrypt the remaining state
- _Encrypted records_: A vector where each element is either _blank_ (i.e.
  empty) or an encrypted record
- _Key tree_: A binary tree as wide as the size of the ciphertext vector where
  nodes are either encrypted keys or references to (encrypted) keys

The AEAD scheme, the encrypted records and the key tree represent the public
state of the protocol. All three can be stored by untrusted parties.

Holders of the `main_key` can use it along with the public state of the protocol
to obtain the vector of records.

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
  opaque encrypted_record<V>;
} EncryptedRecord

struct {
  AEADScheme aead_scheme;
  Node key_tree<V>;
  Option<EncryptedRecord> encrypted_records<V>;
} TreeWrapPublicState
~~~

If a node is an encrypted key, it was encrypted either under the key encrypted
in the node's parent node, the key referenced by the parent node, or the
`main_key` (if the node in question is the root node).

If the root node is of type `parent_key_ref`, it's a reference to the key tree's
`main_key`.

The key in each individual leaf of the key tree (either encrypted or referenced)
is used to encrypt a record in the vector of records. Key and record are paired
according to their leaf index and vector index respectively.

## Initialization

The initial `main_key` is taken as input, but MUST be of the appropriate length
for use with the AEAD scheme. The key tree is initialized by setting the root
(and only node) of the tree to be of type `parent_key_ref`.

## Setting a direct path

All key tree operations (add, remove and update) use the following basic
algorithm that takes as input a new `main_key`, the current `main_key`, as well
as the direct path (including the leaf) and the copath of the target key tree
leaf.

- Recover the keys in the direct path starting with the root node (which is
  always a reference to the current `main_key`)
  - If the node is a reference to a key, replace the node with the key
  - If the node is an encrypted key, use the key in the parent node to decrypt
    the key
- Decrypt the encrypted keys in all copath nodes of the leaf index with the keys
  in their respective parent nodes
- Replace the nodes in the direct path with references to the new `main_key`
- Encrypt the keys in the copath with the new `main_key`

The outputs of the operation are the affected index and the new nodes in its
copath. The outputs are all public and can be used to update a key tree.

~~~ tls
struct {
  uint32 leaf_index;
  Node copath<V>;
} KeyTreeUpdate
~~~

## Adding records

The input to the to an add operation is the new record, as well as a new
`main_key`.

When a record is added to the vector of records, the new record either fills the
blank with the lowest index (see {{removing-records}}) or, if there are no
blanks, the vector, as well as the key tree are extended.

If there are no blanks in the vector of records, the vector is expanded by one
and the key tree is extended using the following algorithm.

- If the tree is full, two new nodes are added: A new root and a new leaf, where
  the previous root and the new leaf are the children of the new root.
- If the tree is not full, two new nodes are added: A new parent node and a new
  leaf. The new parent node is added as the leaf of the right-most parent node,
  where the children are roots of sub-trees of different debth. The left child
  of the new parent node is the former right child of its parent node. The right
  child of the new parent node is the new leaf node.

In both cases both the new parent and the new leaf node are references to the
new `main_key`.

If there is a blank leaf in the vector of records, the direct path of the key
tree leaf with the same index is set as described in {{setting-a-direct-path}}.

Finally, the newly added record is encrypted under the new `main_key`.

~~~ tls
struct {
  KeyTreeUpdate key_tree_update;
  EncryptedRecord added_record;
} AddResult
~~~

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

## Updating records

The input to an update operation is the index of the record to be updated, the
new record, as well as a new `main_key`.

When updating a record in a specific index the direct path of the key tree is
set as described in {{setting-a-direct-path}}. The updated record is then
encrypted under the new `main_key` with the resulting encrypting record
replacing the old encrypted record in the target index.

~~~ tls
struct {
  KeyTreeUpdate key_tree_update;
  EncryptedRecord updated_record;
} UpdateResult
~~~

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

## Removing records

A remove operation takes as input the index of the record to be removed, as well
as a new `main_key`.

A record that is targeted for removal is replaced with a blank before setting
the direct path of the corresponding leaf node in the key tree as described in
{{setting-a-direct-path}}.

If the target record was the record at the end of the vector, the vector is
shortened by one. Similarly, the record's corresponding leaf, as well as its
parent node are removed from the key tree. If the removed parent node was not
the root node, its other child becomes the right child of the removed parent's
parent node.

~~~ tls
struct {
  KeyTreeUpdate key_tree_update;
} RemoveResult
~~~

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

## Processing operation results

The result of an operation can be processed by both parties holding only the
public protocol state and by parties holding the vector of records.

### Updating a vector of records

For remove operations, a party that wants to update the vector of records needs
only the affected index.

For add and update operations, the party additionally needs the new `main_key`,
as well as the (new) encrypted record. The party then decrypts the encrypted
record using the new `main_key`. In case of an update the party replaces the
record in the affected index with the result. In case of an add, the party uses
the result to replace the blank with the lowest index or to extend the vector.

### Updating a public protocol state

A party that wants to update a public protocol state, needs the -Result struct
of the respective operation. It can then use the KeyTreeUpdate to replace the
copath nodes of the affected leaf and set the leaf's direct path to be
`parent_node_ref`s (potentially extending the tree in case of adds).

In case of updates, the encrypted record is replaced with the new one. In case
of adds, the new record either replaces a blank or the vector is extended. In
case of removes, the affected record is replaced with a blank and both key tree
and vector are shortened if the new blank is at the end of the vector.

## Decryption

Any party in possession of the current `main_key` and copy of the protocol
public state can decrypt first the keys in the key tree and then the encrypted
records to recover the vector of records.

The key tree is decrypted using the following algorithm starting at the root.
Decryption will yield a vector of keys where each key can be used to decrypt the
record at the same index.

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

# Key generation and distribution

All protocol operations take a new `main_key` as input. That key can be freshly
sampled and distributed as required by the application. Alternatively a
previously agreed-upon key can be used. The latter can be facilitated, for
example, by establishing an MLS group to export the necessary key material. In
that case, TreeWrap operations can be paired with MLS group management
operations to coordinate access to new and updated records, as well as restrict
access to deleted records.

# Security properties

TreeWrap provides the following security guarantees, analogous to
forward-secrecy and post-compromise security:

- An adversary in possession of the current `main_key` and public protocol
  state, as well as any number of past key trees can recover the current vector
  of records, but will not gain access to records that were updated or removed
  in the past.

- An adversary in possession in the situation described above that also observes
  -Results of operations in the future will not gain access to newly added or
  changed records.
