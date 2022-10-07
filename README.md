# `codec_generator` PROSE format

The following is a guide and specification for **Prose**, a custom output format. It is native to the `codec_generator` compiler project, meaning that it is not an established format with extant documentation. This document is intended to serve such a purpose.


## Background

[`codec_generator`](https://gitlab.com/archaephyrryx/codec_generator) is a project within the broad Tezos ecosystem, whose main product is a compiler framework written in OCaml. This compiler takes, as input, a set of `data-encoding` *schema* values, corresponding to the protocol-specific types defined throughout Octez. As output, it produces *codecs*.

A codec, in the general case, is a source-code module that defines a particular data-type and associated serialization/deserialization methods, intended to have a similar structure to the original type, and with the same encoding and decoding semantics and nuances as the original schema. Currently, support has been implemented for outputting Rust codecs, with support for TypeScript under active development as well.

A codec may also be a semi-formal *description* of the structural layout of a schema type, as is the case with the **Prose** output target.


### Motivation

At the time that **Prose** was originally implemented, the compiler framework itself was hard-coded for producing only Rust codecs. In light of this, it was important to add a new output target, to guard against over-specialization in the design of the compiler as it evolved further. As the process of implementing support for a completely new programming language would have been prohibitive, it was sufficient to instead invent a simple-enough target format, to serve as a proof-of-concept for the extensibility of the compiler beyond just Rust.

Beyond this, there are two practical purposes that Prose serves in its own right:

A Prose codec can be used as a prototype for development of hand-written code in languages currently unsupported by `codec_generator`; in this way, it can partially obviate the need for consulting the original schema, which can be difficult to decipher, and relies on familiarity with both OCaml and the `data-encoding` library.

Secondly, for the purposes of internal development of the `codec_generator` pipeline, Prose codec output can be used to debug the compiler itself: if there is an input schema that triggers an exception, or produces unexpected output for a different target language, the generated Prose codec is instrumental to determine the exact combination of properties that caused the failure.

### How to Use

For developers working outside of `codec_generator`, there are two primary ways in which **Prose** codecs can be useful.

The first is as a diagnostic aid, through the script [`whatchanged.sh`](https://gitlab.com/archaephyrryx/codec_generator/-/blob/master/whatchanged.sh), packaged with the `codec_generator` repository. This script can be used to discover the concrete changes visible in the binary encoding of a type, across different protocol versions. If a field is added to a record, a variant is added to an enumerated type or discriminated union, or otherwise, some aspect of the serialization format is substantively different between two protocol versions, this script can report both what schemata were changed, and what those changes were.

Beyond this, if a new type is introduced, either to the protocol itself or to the scope of a Tezos client library, a Prose codec for that new type can be used as a model to implement both the type itself, and its implicit encoding/decoding semantics.

As a caveat, it is worth mentioning that not all properties of an Octez schema are preserved by the `codec_generator` tool, and certain high-level mappings from an underlying schema type to its intended Octez type cannot be introspected or represented in any target language, Prose or otherwise. Therefore, while Prose codecs can be used to implement the fundamental structure of a type, the Octez definition should still be consulted to fill in any major gaps.

For a more specific guide as to how to write code to decode, encode, and process a schema type (based on the structure revealed by a Prose codec) please see the `codec_generator` [Target Implementation Guide][howtomd]. This document is primarily focused on informing future development of target-language modules for `codec_generator`, but can also be used as a guide to hand-writing codec libraries.

## Specification

**Prose** is a custom target language, and its corresponding output files are given the `.prose` or `.prose~` extensions.

It expresses a compact model of the `data-encoding` schema language, operating on values already *simplified* by the `codec_generator` compiler.

(Note that this document is being written based on the Prose model for pre-[0.7][dev7] versions of `data-encoding`, and may have to be updated once the Prose model is restabilized onto `v0.7`)

Though it is potentially subject to reorganization, the definition of the Prose generator and AST are unified into a single module, hosted on the project repo  under the path [`src/virtual/prose.ml`][prose-ml].

The underlying AST used to model **Prose** is the following:

{%gist archaephyrryx/7ed664ccfbc096a157e09baf7f004851 %}

These structural elements are a transient model, and are compiled into human-readable descriptions for each given component.

### Unit

A `Unit` element corresponds to almost all schemata of type `unit Encoding.t`. Such an element can either be thought of as being 'omitted' from the binary serialization layer, or as being encoded as a zero-width binary value.

It can arise when the source schema includes one of the following `data-encoding` combinators at some layer (Cf. [encoding.ml][encoding-ml]):


* `null`
* `empty`
* `unit`
* `constant _str` (`_str`: descriptive string presented only in the JSON layer, often for giving names to tagged variants in a discriminated union)

In **Prose** output, `Unit` is represented by the following string:

```ocaml
"zero-width value (null or unit)"
```

### Bool

A `Bool` element corresponds to a `bool Encoding.t`,  which represents a boolean value as a single byte, whose bits are either all set (`0xff := true`) or all unset (`0x00 := false`).

Though similar flagging can be found at the binary layer for `Dft` or `Opt` (tagged) elements, these are treated separately in the **Prose** model.

Almost every instance of `Bool` will correspond to the `Encoding.bool` combinator occurring in the source schema.

In **Prose** output, `Bool` is represented by the following string:

```ocaml
"boolean value"
```

### Num

A `Num _` element is an aggregate over all `num_t` cases defined in [`simplified.ml`][simplified-ml]:

{%gist archaephyrryx/acb04c69c2312ea2706d2634f63d0f97 %}

These correspond to the respective combinators:

* `int8`
* `uint8`
* `uint16`
* `int16`
* `int31`
* `int32`
* `int64`
* `ranged_int`
* `float`
* `ranged_float`

Note that `Uint30` appears as a first-class value in `simplfied.ml`, while in `data-encoding` it does not appear; this is due to the fact that, while 30-bit unsigned integer values are commonly used for enum tag-values, dynamic length prefixes, and other composite constructs, a raw integer value representing the same range as `uint30` can only be modeled as a `ranged_int` from `0x0` to `0x3fffffff`. However, such a `RangedInt` value is re-associated as `Uint30` when it has those bounds, during the Simplification phase of the compiler run.

Integer-valued `Num _` are rendered as:

```ocaml
"<N>-bit (un)?signed integer ()"
```

for the appropriate value of `N` and the correct indicator of signedness.

For specific (non-equivalent to a NativeInt) `RangedInt` values, an additional suffix of

```ocaml
" between <MIN> and <MAX>"
```
is appended.

For `Double` and `RangedDouble`, the following string is emitted:

```ocaml
"IEEE-754 double-precision float"
```

with `RangedDouble` adding the suffix

```ocaml
" between MIN and MAX"
```

as appropriate.

#### `Int31/Uint30` vs `Int32`

There is a technical distinction that should be raised on the subject of the division of 4-byte integer types into `Int32`, `Int31`, and `Uint30`.

On 32-bit OCaml platforms, in-memory values are represented as 32-bit machine words whose MSB is a one-bit flag indicating whether the remaining 31-bits are an unboxed value (e.g. Ocaml `int`, `bool`, `unit`, certain ADT discriminants, etc.) or the pointer to the address of a boxed value, which may take up more than one machine word to store (`list`, `string`, recursive types, records, tuples, etc.).

As a consequence, 32-bit OCaml platforms have only 31 bits to store an unboxed integer value, reserving the highest of the non-flag bits for sign and the remaining 30 bits for the value, using two's-complement representation as normal.

This means that in order to be compatible with OCaml builds on 32-bit systems, `data-encoding` treats `int` as a 31-bit signed value, hence `Int31`. Within this range, `Uint30` is a virtual sub-type consisting of all non-negative `Int31` values.

As an alternative, there is a standard library module in OCaml, `Int32`, that offers portable 32-bit integers on all platforms, either raw on 64-bit systems or boxed on 32-bit systems. This is the underlying type used for the `int32` combinator. (Similarly, an `Int64` module exists, corresponding to the `int64` combinator). Though guaranteed-portable, the memory footprint of such values, and the worse performance of operations on such values compared to unboxed `int`, means that `Int32` is typically used only in contexts where all the extra bit of precision cannot be sacrificed. Similarly, `Int64` incurs a performance penalty on even 64-bit platforms compared to `int`, and is only used when representing values over the full 64-bit range (e.g. time-related values in Octez, as in [lib_base/time.ml](https://gitlab.com/tezos/tezos/-/blob/0e7a0e9a67a381916b8e23cd704f43ac4d770920/src/lib_base/time.ml#L28)). Therefore, the majority of fixed-precision integer schemata use `int` as their underlying type, using `Int32.t` and `Int64.t` for the extra variants `int32` and `int64` only.


Due to the fact that there is no standard `Uint32` module in use by `data-encoding`, all unsigned 4-byte integers can be assumed to be 30-bit maximal, meaning that any value `0x4000_0000` ($2^{30}$) or above is illegal for that component or sub-component.

In practice, due to the fact that most usages of `Uint30`-valued binary elements are for unsigned values strictly less than $2^{30}$ (e.g. binary length of strings, arrays; tags for enumerated types), this should not be a major issue, but to the extent that bounds-checking does not significantly reduce performance, it is advised to ensure that values greater than $2^{30} - 1$ are never serialized into a context that expects a `Uint30` value. Similarly, for the case of `Int31`, values less than $-(2^{31})$ or greater than $2^{30} - 1$ should be filtered out.

As a final note, it is important to clarify that, while values outside of the range $[0, 2^{30} - 1]$ (for `Uint30`) or $[-(2^{30}), 2^{30} - 1]$ (for `Int31`) are invalid, the actual **encoding** of both types is equivalent to that of a full-range `Int32` with the same numeric value. That is, negative `Int31` values will still use the MSB as their sign-bit, rather than use the 32-bit OCaml in-memory representation. All this means is that `Int31` and `Uint30` should be encoded and decoded as if they were `Int32` values, but that post-decode and pre-encode bounds-checking are necessary to filter out illegal values.

### `N`/`Z`

An `N` or `Z` element represents an arbitrary-precision natural ($\mathbb{N}_0$) or integer ($\mathbb{Z}$). These are generally referred to as `Zarith` values, as the underlying type of the encoding is the `Z.t` type defined in the OCaml [Zarith][zarith-lib] library.

`N` corresponds to the `n` combinator, and `Z` corresponds to the `z` combinator, in `data-encoding`.

In Prose output, `N` is emitted as

```ocaml
"arbitrary-precision natural (non-negative) integer"
```

and `Z` as

```ocaml
"arbitrary-precision integer"
```

#### Dynamic-Size Integer Encoding

A custom binary serialization/deserialization strategy is used for the `data-encoding` primitives `N` and `Z`.

As of `data-encoding` [v0.7][dev7], this strategy may be used for bounded-precision integer values, via `uint_like_n` and `int_like_z`, and so it applies more broadly than just for Zarith-based arbitrary-precision integers.

Due to the nuance and complexity of the encoding strategy in question, it is sufficient to state here that the encoding is a self-terminating byte-sequence where the high bit for each byte is `1` whenever there is at least one more byte left in the value, and `0` when the byte in question is to be considered terminal. The remaining lower seven bits in each byte are to be interpreted in little-endian byte-order as the constituent bits of the binary representation of the value, with `Z` and `uint_like_z` using the second highest bit in the initial byte to signal the sign, as proper negation rather than two's-complement representation.

A detailed explanation of the Zarith encoding strategy can be found in the [Target Implementation Guide][howtozarith] (a more detailed explanation of several other schema/codec types can be found in the same document as well).


### `String` and `Bytes`

The unified `String _` element represents two intrinsically related, but technically distinct schema kinds in `data-encoding`:

- `String (Some n)` is a transcription of `Fixed.string n`
- `String None` is a transcription of `Variable.string` 

Similarly, `Bytes (Some n | None)` models `Fixed.bytes n` and `Variable.bytes`


`String _` is emitted with the base string-value:

```ocaml
"character string"
```

with `Bytes _` emitting a similar:

```ocaml
"byte sequence"
```


with the conditional suffix (only for `(String | Bytes) (Some n)`) of

```ocaml
" (fixed length: n)"
```

In both cases, the length stored in the AST constructor, and emitted in the codec output, is the **number of bytes** (even for `String`), rather than the number of characters, due to the nature of `string` in OCaml as outlined below:

#### OCaml `string` type

The OCaml `string` type is far more permissive in its binary contents than string types in certain other languages. Because OCaml strings are immutable, fixed-length sequences of bytes, there is no ironclad guarantee as to whether the byte contents of a `string` will be valid UTF-8, or be valid as strings in other languages.

It is important, therefore, to understand that `String` and `Bytes` are virtually interchangeable, with the only real difference being that a value typed as `string` is thereby giving a hint that it contains text-oriented data, and might be useful to present as a sequence of characters rather than a raw binary blob.

In fact, since `v0.6` of `data-encoding`, there is a representation-hint value passed to the newly defined `string'` and `bytes'` constructors, taking a value within the new type

```ocaml
type string_json_repr = Hex | Plain
```

(Cf. [encoding.ml](https://gitlab.com/nomadic-labs/data-encoding/-/blob/2d51c28280e780725e201ecdeb68df485c17578e/src/encoding.ml#L92))

which indicates whether the JSON represntation of such a value should be a plaintext string encoding, or presented as a hexadecimal encoding of its constituent bytes. Accordingly, this version redefines `string` and `bytes`:

```ocaml
let string = string' Plain

let bytes = bytes' Hex
```

(This refactoring applies both to the top-level definitions, as well as for the `Fixed` and `Variable` module definitions of `string` and `bytes`, each with their own corresponding prime-variant).

### `Dyn`

An element `Dyn (lp, t)` corresponds to the `dynamic_size` schema wrapper in `data-encoding`, with `lp` being of type

```ocaml
type lenpref = [`Uint30 | `Uint16 | `Uint8]
```

This value indicates the encoding used for a *length prefix* (measuring length-in-bytes, even for payload values with another natural concept of *length*)

:::warning
In v0.7 of `data-encoding`, a new variant is added to this collection, namely <code>`N</code>.

This new variant specifies a dynamically sized length-prefix, using the encoding for the `Zarith` schema-type `N`; however, it is understood that such values ultimately model `int` and not `Zarith.Z.t`, and so they are still constrained to be expressible as `uint30` values. The only difference, therefore, between the `Uint30` and `N` length-prefix encodings is whether to optimize for smaller values, which can cut down the overhead of the byte prefix according to this table:

| Value Range              | N bytes | vs Uint8 | vs Uint16 | vs Uint30 |
|--------------------------|---------|----------|-----------|-----------|
| $0$--$127$               | 1       | +0%      | -50%      | -75%      |
| $128$--$255$             | 2       | +100%    | +0%       | -50%      |
| $256$--$(2^{14} - 1)$    | 2       | N/A      | +0%       | -50%      |
| $2^{14}$--$(2^{16} - 1)$ | 3       | N/A      | +50%      | -25%      |
| $2^{16}$--$(2^{21} - 1)$ | 3       | N/A      | N/A       | -25%      |
| $2^{21}$--$(2^{28} - 1)$ | 4       | N/A      | N/A       | +0%       |
| $2^{28}$--$(2^{30} - 1)$ | 5       | N/A      | N/A       | +25%      |

:::




### `Seq`

An element `Seq (t, limit)` corresponds to a unified notion of *sequence* type, which models both `list`- and `array`-based schema components. The constructor itself is typed as `Seq of t * limit`, where `t` is a self-reference to the Prose AST type, and `limit` is a mirrored definition from `Data_encoding`:

```ocaml
type limit = Data_encoding__Encoding.limit =
  | No_limit
  | At_most of int
  | Exactly of int
```

The encoding *format* does not change based on what limit is enforced, but the operational semantics of decoding each kind of sequence is slightly different, and so each limit-variant will be explored.

#### `Seq (t, No_limit)`

An element `Seq (t, No_limit)` represents a sequence whose cardinality (number of elements) is fully variable.

The d


(Note that with the release of [v0.7][dev7])



### `Tuple`

An element `Tuple ts` represents an OCaml tuple whose in-order positional arguments are given by the list `ts`. Each element is, in turn, the translation of the corresponding positional argument in the source schema, into a Prose AST type.

For example,

```ocaml
Data_encoding.(tup3 bool Variable.string int64)
```

would be translated (approximately) to

```ocaml
Tuple [Bool; String None; Num (Int (NativeInt Int64))]
```

Tuples, as product-types, yield more complex output than simple elements.

As a rule, a tuple consisting of `N` positional argument will be emitted with a header of


```ocaml
"N-tuple : "
```

followed by a sequence of items, indented by one layer compared to the header, each of which will be:

```ocaml
"i: <output for element at positional index i>"
```

For example, the above 3-tuple would yield

```text
3-tuple : 
    0: boolean value
    1: character string
    2: 64-bit signed integer
```

### Record

An element `Record fs` is similar to an element `Tuple _`, but models a record type rather than a tuple type. As such, it stores a list of named fields, as `(string * t)`.

Much like `Tuple _`, the emitted string consists of a header line:

```ocaml
"Record : "
```

followed by a sequence of items, indented one layer; each of these items follows the pattern

```ocaml
"`<field name>`: <output for corresponding field>"
```

For example, the following schema:

```ocaml=
obj3 (req "first" string) (opt "middle" string) (req "last" string))
```

would be converted to Prose and emitted as

```text!
Record : 
    `first`: length-prefixed (prefix width: 4 bytes): character string
    `middle`: [tagged] nullable of: length-prefixed (prefix width: 4 bytes): character string
    `last`: length-prefixed (prefix width: 4 bytes): character string
```







```
    | Seq of t * limit
    | Pad of int * t
    | Opt of bool * t
    | Dft of t * string

    | Enum0 of enum_size * (string * int) list
    | EnumPlus of tag_size * (int * (string * t)) list
    | SubDef of string * t
    | ExtRef of string
```



[howtozarith]: https://gitlab.com/archaephyrryx/codec_generator/-/blob/master/experience_src/HOWTO.md#zarithzzarithn
[howtomd]: https://gitlab.com/archaephyrryx/codec_generator/-/blob/master/experience_src/HOWTO.md
[zarith-lib]: https://opam.ocaml.org/packages/zarith/
[prose-ml]: https://gitlab.com/archaephyrryx/codec_generator/-/blob/master/src/virtual/prose.ml
[encoding-ml]: https://gitlab.com/nomadic-labs/data-encoding/-/blob/master/src/encoding.ml
[simplified-ml]: https://gitlab.com/archaephyrryx/codec_generator/-/blob/master/src/common/simplified.ml
[dev7]: https://ocaml.org/p/data-encoding/0.7/doc/Data_encoding/index.html