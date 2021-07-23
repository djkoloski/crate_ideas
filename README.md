# Crate ideas

These are ideas for crates. Most of them are focused around game development and high-performance
computing.

## List

| Name                              | Description                               | Status                        |
|-----------------------------------|-------------------------------------------|-------------------------------|
| [rkyv](#rkyv)                     | Zero-copy deserialization framework       | ✅ [Done][rkyv]               |
| [bytecheck](#bytecheck)           | Byte validation framework                 | ✅ [Done][bytecheck]          |
| [ptr_meta](#ptr_meta)             | Stabilization of the `ptr_meta` RFC       | ✅ [Done][ptr_meta]           |
| [skema](#skema)                   | Dynamic schemas and foreign types         | 💭 Planning                   |
| [assets](#assets)                 | Asset management and bundling             | 💭 Planning                   |
| [vectorize](#vectorize)           | SIMD vectorization for aggregate types    | 💭 Planning                   |
| [splinter](#splinter)             | Proven and performant spline library      | 🚧 [In progress][splinter]    |
| [protoss](#protoss)               | Protocol type generation                  | 🚧 [In progress][protoss]     |
| [rkyv_compress](#rkyv_compress)   | Compression types for rkyv                | 💭 Planning                   |
| [rkyv_mmap](#rkyv_mmap)           | Memory mapping helpers for rkyv           | 💭 Planning                   |
| [rkyv_intern](#rkyv_intern)       | Value interning for rkyv                  | 💭 Planning                   |
| [rend](#rend)                     | Archive-aware primitives for Rust         | ✅ [Done][rend]               |
| [rkyv_stream](#rkyv_stream)       | Streaming serialization for rkyv          | 💭 Planning                   |
| [structver](#structver)           | Automatic struct versioning               | 💭 Planning                   |
| [rkyv_dow](#rkyv_dow)             | Deserialize-on-write structures for rkyv  | 💭 Planning                   |
| [progresso](#progresso)           | Lazy validation adapter for bytecheck     | 💭 Planning                   |

[rkyv]: https://github.com/djkoloski/rkyv
[bytecheck]: https://github.com/djkoloski/rkyv
[ptr_meta]: https://github.com/djkoloski/ptr_meta
[splinter]: https://github.com/djkoloski/splinter
[protoss]: https://github.com/djkoloski/protoss
[rend]: https://github.com/djkoloski/rend

## `rkyv`

`rkyv` is a crate that enables zero-copy deserialization for rust types, even complex ones. All
archivable data can be loaded with `mmap` and a pointer cast.

## `bytecheck`

`bytecheck` is a crate that can verify the integrity of some in-memory data without security
vulnerabilities.

## `skema`

`skema` is a crate that provides utilities for dealing with foreign types.

## `assets`

`assets` builds on `rkyv` and adds a more formal bundle system. It introduces the concept of assets
and bundles, and provides implementations of asset graphs and asset loaders.

- An asset is the smallest possible individual piece of data. Assets can be moved between bundles.
- Bundles are a group of assets determined by some process. Bundles contain all the information
  needed to load and maintain dependencies at runtime.

Asset references function similarly to shared pointers with a strong and weak variety, but have
different serialization rules:

- Neither strong nor weak asset references cause assets to be bundled together
- Strong references guarantee that the referenced asset will be loaded before the referencing asset,
  weak references do not
- Weak references can be upgraded to strong references, which will cause any dependent bundles to be
  loaded

Should asset references be resolved ahead-of-time or just-in-time?

Ahead-of-time:
- No performance impact from asset lookups, may cause issues with `mmap` since the underlying memory
  is immutable
Just-in-time:
- No issues with immutable backing memory, not sure of the performance impact

Compromise? Archived asset references perform just-in-time lookup, and they resolve statically
during deserialization?

Cannot allow assets with strong references to be loaded, deserialized, and then have the backing
bundle unloaded since it could break its strong references.

But it should be allowed to load a bundle, deserialize some assets and use them
(e.g. upload textures to GPU), then unload the bundle.

Automatic vs. manual bundling

Manual bundling: The bundle for each asset is decided when the asset is created. The asset can be
manually moved between bundles and the 

Automatic bundling: Some mysterious clustering criteria used to determine bundling

Loading: everything should be async

Asset post-processing? How to indicate that some assets need to hold the bundle in memory but others
deserialize into a standalone form?

Support copied memory (`fread`) and memory mapping (`mmap`) because which one is better will depend
on the application.

## `vectorize`

Vectorize operations on aggregate types with just a `#[derive]`.

The idea is an expansion of the `Wec` idea that combines four `Vec`s into a single wide type that
has the four components packed together. Instead of just vectors, this could be used for component
updates in entity-component systems.

Unsolved problems:
- Mixed vectorizable and unvectorizable fields: should the unvectorizable fields fall back to an
  array and do the operations manually?
- Aggregate containers (i.e. storage) that has partially-initialized vectorized types: shouldn't the
  uninitialized slots be filled with a `Default` value or something?
- Structs with different levels of vectorizability: one field is 2-vectorizable and another is
  4-vectorizable. Should a 4-vectorized type be built? What about things like 128-vectorizable
  bools?

## `splinter`

High-performance spline library with proofs for its algorithms.

Needs more support for approximate algorithms with error bounds and for larger structures (like full
splines rather than curves).

## `protoss`

Cap'n Proto and FlatBuffers support schema evolution by restricting how types can change:

- New fields can only be added at the ends of types
- Old fields can't be deleted, only deprecated

These rules could be enforced pretty easily, and some macros could be used to get the same style of
declaration as .proto and .capnp files:

```rust
use protoss::Proto;

#[derive(Proto)]
pub struct MyMessage {
  #[id = 0]
  pub a: i32;
  #[id = 2]
  pub b: u32;
  #[id = 1]
  pub c: String;
}
```

would generate:

```rust
use protoss::Proto;

pub struct MyMessage {
  pub a: i32,
  pub b: u32,
  pub c: String,
}

#[repr(C)]
pub struct ArchivedMyMessage {
  bytes: [u8],
}

const _: () = {
  impl ArchivedMyMessage {
    pub fn a(&self) -> Option<&Archived<i32>> {
      // ...
    }
    pub fn b(&self) -> Option<&Archived<u32>> {
      // ...
    }
    pub fn c(&self) -> Option<&Archived<String>> {
      // ...
    }
    pub fn a_pin(self: Pin<&mut Self>) -> Pin<&mut Archived<i32>> {
      // ...
    }
    pub fn b_pin(self: Pin<&mut Self>) -> Pin<&mut Archived<u32>> {
      // ...
    }
    pub fn c_pin(self: Pin<&mut Self>) -> Pin<&mut Archived<String>> {
      // ...
    }
  }

  #[repr(C)]
  struct ArchivedMyMessageData
  where
  {
    a: rkyv::Archived<i32>,
    c: rkyv::Archived<String>,
    b: rkyv::Archived<u32>,
  }

  use rkyv::{ArchivedMetadata, Serialize, Serializer, SerializeUnsized};

  impl ArchivePointee for ArchivedMyMessage {
    type ArchivedMetadata = ArchivedUSize;

    fn pointer_metadata(archived: &Self::ArchivedMetadata) -> <Self as Pointee>::Metadata {
      archived as usize
    }
  }

  impl ArchiveUnsized for ArchivedMyMessage {
    type Archived = ArchivedMyMessage;
    type MetadataResolver = ();

    fn resolve_metadata(&self, _: usize, _: Self::MetadataResolver) -> ArchivedMetadata<Self> {
      core::mem::size_of::<ArchivedMyMessageData>() as ArchivedUSize
    }
  }

  impl<S: Serializer + ?Sized> SerializeUnsized<S> for MyMessage
  where
    i32: Serialize<S>,
    String: Serialize<S>,
    u32: Serialize<S>,
  {
    // serialize fields, build archived data type, write to serializer
  }
};
```

## `rkyv_compress`

Compression wrapper types for rkyv. For example:

```rust
use rkyv::{Archive, Deserialize, Serialize};
use rkyv_compress::LZ4;

#[derive(Archive, Serialize, Deserialize)]
struct Texture {
    width: usize,
    height: usize,
    data: LZ4<Vec<u8>>,
}

let buf = ...
let texture = unsafe { archived_root::<Texture>(buf) };

// able to access width and height
println!("width: {}, height: {}", texture.width, texture.height);
// can access compressed bytes
println!("compressed length: {}", texture.data.compressed_len());
// deserializes normally
println!("uncompressed length: {}", texture.data.deserialize().len());
```

### Main concepts

Compression wrapper types:
- `LZ4<T>`, `Zip<T>`, etc.
- Act like thin wrappers around the inner type
- Serialize boxy and write compressed data to the serializer and point to it with a `RawRelPtr`

Serializer bounds:
- `pub trait CompressionSerializer<C>`
- Needed so that compressed values can be serialized into a separate "temporary" archive and then
  compressed and written to the "permanent" archive
- Could be how dictionary training occurs

Look at `lzzzz`.

## `rkyv_mmap`

Some helper functions for `mmap`-ing files with rkyv. Nothing too fancy, just to help users get
something working end-to-end easily.

## `rkyv_intern`

Interning values for better compression. For example:

```rust
use rkyv::{Archive, Deserialize, Serialize};
use rkyv_intern::Intern;

#[derive(Archive, Serialize, Deserialize)]
pub struct Log {
    request_type: Intern<String>,
    path: Intern<String>,
    response: u16,
    time: String,
}

let mut serializer = InternSerializer::new(WriteSerializer::new(Vec::new()));
// serialize value
let buf = // ...
let value = unsafe { archived_root::<Log>(buf) };
```

- Interned objects can't be mutated through `Pin`s
- Inner type must be `Eq`
- Transparently dereferences to the inner type

## `rend`

Most methods of handling endianness are just functions to swap between little- and big-endian. But
for storage formats like `rkyv`, it's desirable for these functions to hide and become more or less
implicit. [Another crate](https://docs.rs/simple_endian/0.2.0/simple_endian/) has a similar
approach.

This basically entails creating little- and big-endian variants of all primitive types and having
them perform the necessary conversions on the fly.

```rust
use rend::*;

let little_int = i32_le::new(0x12345678);
// Internal representation is little-endian
assert_eq!([0x78, 0x56, 0x34, 0x12], unsafe { ::core::mem::transmute::<_, [u8; 4]>(little_int) });

// Can also be made with `.into()`
let little_int: i32_le = 0x12345678.into();
// Still formats correctly
assert_eq!("305419896", format!("{}", little_int));
assert_eq!("0x12345678", format!("0x{:x}", little_int));

let big_int = i32_be::new(0x12345678);
// Internal representation is big-endian
assert_eq!([0x12, 0x34, 0x56, 0x78], unsafe { ::core::mem::transmute::<_, [u8; 4]>(big_int) });

// Can also be made with `.into()`
let big_int: i32_be = 0x12345678.into();
// Still formats correctly
assert_eq!("305419896", format!("{}", big_int));
assert_eq!("0x12345678", format!("0x{:x}", big_int));
```

## `rkyv_stream`

Streaming serialization and deserialization would enable a lot of use cases where items need to be
serialized and deserialized in order, but don't need to be randomly accessed. In these cases, the
memory footprints can be brought down to a constant by serializing and deserializing them
standalone. In order to do so with dependent data of variable size, the stream needs to be split
into packets with each packet containing some small number of items (probably just one).

```
+----------------------------+
| packet header (length)     |
+----------------------------+
| payload (aligned properly) |
+----------------------------+
| ...                        |
```

The serializer / deserializer needs to know whether the stream has ended or not, so an additional
zero-length packet will probably need to be placed at the end. However, using ZLPs as terminators
means that there will be confusion with streamed ZSTs. This can be resolved by having a separate
format for zero-sized types that just writes one byte for each ZST. This is as opposed to writing
the number of ZSTs and sending that because if users really want that they can just use regular rkyv
serialization with a `Vec<ZST>`.

## `structver`

This would basically just be a new trait:

```rust
pub trait Versioned {
    const VERSION_HASH: [u8; 32];
}
```

Which would allow:

```rust
#[derive(Versioned)]
#[version = 2]
pub struct A {
    a: i32,
}
```

to compile to:

```rust
pub struct A {
    a: i32,
}

impl Versioned for A {
    const VERSION_HASH: [u8; 32] = Sha256::new()
        .update(&2u32.to_le_bytes())
        .update(&i32::VERSION_HASH)
        .finalize();
}
```

using [sha2_const](https://docs.rs/sha2-const/0.1.1/sha2_const/). This would effectively bump the
version hash whenever the struct or any dependencies change, you'd just have to bump the version for
each struct when you change it.

It's possible that this could also be extended to perform automatic versioning without requiring
bumping in an attribute by hashing the struct definition as a string instead of an explicit version.
I'm not sure what the capabilities of the sha2_const crate are.

I think it would be safe to write the hash of the unarchived types instead of the hash of the
archived types. If rkyv updates and changes some of its structures that could cause an issue, but
you could always just manually bump the version in that case.

## `rkyv_dow`

Deserialize-on-write structures are essentially just types that have enough space for either an
archived type or an unarchived type.

All DOWs start out serialized. Read operations on a DOW keep them in the serialized form, but write
operations perform a deserialization and switch to the deserialized form. Once the desired
modifications are complete, the structure can be re-serialized to disk.

This would require some library support, since every archived type would additionally need to
archive to itself. However, they wouldn't necessarily be `ArchiveCopy` because they would still have
non-ZST resolvers and perform work.

## `progresso`

One of the major puzzle pieces that remains unsolved for rkyv is progressive validation. This is essentially navigating through an archive and only validating the portions that you actually use. Solving this for the extreme general case of mutable access to progressively validated data is daunting at best and impossible at worst. Some example tricky cases:

- Shared pointers cannot be verified to be located in a valid memory range because the first instance of the shared pointer may not have been encountered yet
- Mutating the underlying memory is not safe unless you have a guarantee that there will no other references (`const` or `mut`) to the same region
  - This requires that all subobjects be located in disjoint memory regions (which is part of what `bytecheck` ensures)

API:
- Obtain a progressive validator by wrapping a pointer to the root object
- Progressive validators can be either `const` or `mut`, but neither can provide progressive mutable access
- You can check the whole subobject with `bytecheck` and get back a reference to the archived object
- Checking the whole subobject allows mutable access if checked with a mutable progressive validator
- Accessing a specific field returns a subobject with the validation stepped and preserves the outer validator
