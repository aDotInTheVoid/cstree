<div align=center>
  <h1><code>cstree</code></h1>
  <p>
    <strong>A library for generic lossless syntax trees</strong>
  </p>

  <p>
    <a href="https://github.com/domenicquirl/cstree/actions?query=workflow%3ACI"> <img src="https://github.com/domenicquirl/cstree/workflows/CI/badge.svg" alt="build status" /></a>
    <a href="https://docs.rs/cstree/"> <img src="https://docs.rs/cstree/badge.svg" alt="documentation" /></a>
    <a href="https://crates.io/crates/cstree"> <img src="https://img.shields.io/crates/v/cstree.svg" alt="crates.io" /></a>
  </p>
</div>

`cstree` is a library for creating and working with concrete syntax trees (CSTs).
"Traditional" abstract syntax trees (ASTs) usually contain different types of nodes which represent information about the source text of a document and reduce this information to the minimal amount necessary to correctly interpret it.
In contrast, CSTs are lossless representations of the entire input where all tree nodes are represented uniformly (i.e. the nodes are _untyped_), but include a `SyntaxKind` field to determine the kind of node.
One of the big advantages of this representation is not only that it can recreate the original source exactly, but also that it lends itself very well to the representation of _incomplete or erroneous_ trees and is thus very suited for usage in contexts such as IDEs.

The concept of and the data structures for CSTs are inspired in part by Swift's [libsyntax](https://github.com/apple/swift/tree/5e2c815edfd758f9b1309ce07bfc01c4bc20ec23/lib/Syntax).
Trees consist of two layers: the inner tree (called _green_ tree) contains the actual source text in _position independent_ green nodes. 
Tokens and nodes that appear identically at multiple places in the source text are _deduplicated_ in this representation in order to store the tree efficiently.
This means that the green tree may not structurally be a tree.
To remedy this, the actual syntax tree is constructed on top of the green tree as a secondary tree (called _red_ tree), which models the exact source structure.

The `cstree` implementation is a fork of the excellent [`rowan`](https://github.com/rust-analyzer/rowan/), developed by the authors of [rust-analyzer](https://github.com/rust-analyzer/rust-analyzer/) who wrote up a conceptual overview of their implementation in [their repository](https://github.com/rust-analyzer/rust-analyzer/blob/master/docs/dev/syntax.md#trees).
Notable differences of `cstree` compared to `rowan`:
  - Syntax trees (red trees) are created lazily, but are persistent. Once a node has been created, it will remain allocated, while `rowan` re-creates the red layer on the fly. Apart from the trade-off discussed [here](https://github.com/rust-analyzer/rust-analyzer/blob/master/docs/dev/syntax.md#memoized-rednodes), this helps to achieve good tree traversal speed while providing the next points:
  - Syntax (red) nodes are `Send` and `Sync`, allowing to share realized trees across threads. This is achieved by atomically reference counting syntax trees as a whole, which also gets rid of the need to reference count individual nodes (helping with the point above).
  - Syntax nodes can hold custom data.
  - `cstree` trees are trees over interned strings. This means `cstree` will deduplicate the text of tokens such as identifiers with the same name. In this position, `rowan` stores each string, with a small string optimization (see [`SmolStr`](https://crates.io/crates/smol_str)).
  - Performance optimizations for tree creation: only allocate new nodes on the heap if they are not in cache, avoid recursively hashing subtrees
  - Performance optimizations for tree traversal: persisting red nodes allows tree traversal methods to return references. You can still `clone` to obtain an owned node, but you only pay that cost when you need to.

## Getting Started
The main entry points for constructing syntax trees are `GreenNodeBuilder` and `SyntaxNode::new_root` for green and red trees respectively.
See `examples/s_expressions` for a guided tutorial to `cstree`.

## AST Layer
While `cstree` is built for concrete syntax trees, applications are quite easily able to work with either a CST or an AST representation, or freely switch between them.
To do so, use `cstree` to build syntax and underlying green tree and provide AST wrappers for your different kinds of nodes.
An example of how this is done can be seen [here](https://github.com/rust-analyzer/rust-analyzer/blob/master/crates/syntax/src/ast/generated.rs) and [here](https://github.com/rust-analyzer/rust-analyzer/blob/master/crates/syntax/src/ast/generated/nodes.rs) (note that the latter file is automatically generated by a task).

## License

`cstree` is primarily distributed under the terms of both the MIT license and the Apache License (Version 2.0).

See `LICENSE-APACHE` and `LICENSE-MIT` for details.
