# Crate ideas

These are ideas for crates. Most of them are focused around game development and high-performance
computing.

## List

| Name                      | Description                               | Status                        |
|---------------------------|-------------------------------------------|-------------------------------|
| [rkyv](#rkyv)             | Zero-copy deserialization framework       | ✅ [Done][rkyv]               |
| [bytecheck](#bytecheck)   | Byte validation framework                 | ✅ [Done][bytecheck]          |
| [ptr_meta](#ptr_meta)     | Stabilization of the `ptr_meta` RFC       | ✅ [Done][ptr_meta]           |
| [skema](#skema)           | Dynamic schemas and foreign types         | 💭 Planning                   |
| [assets](#assets)         | Asset management and bundling             | 💭 Planning                   |
| [vectorize](#vectorize)   | SIMD vectorization for aggregate types    | 💭 Planning                   |
| [splinter](#splinter)     | Proven and performant spline library      | 🚧 [In progress][splinter]    |
| [protoss](#protoss)       | Protocol type generation                  | 💭 Planning                   |

[rkyv]: https://github.com/djkoloski/rkyv
[bytecheck]: https://github.com/djkoloski/rkyv
[ptr_meta]: https://github.com/djkoloski/ptr_meta
[splinter]: https://github.com/djkoloski/splinter

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
