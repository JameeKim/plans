# Implementation Details (WIP)

## Table of Contents <!-- omit in toc -->

- [Terms](#terms)
- [Requirements](#requirements)
  - [Must](#must)
  - [Should](#should)
  - [Could](#could)
- [Structure](#structure)
  - [Parts](#parts)
    - [Asset Type Registry](#asset-type-registry)
    - [Asset Type Id](#asset-type-id)
    - [Asset Type Data](#asset-type-data)
    - [Asset Storage](#asset-storage)
    - [Asset Id](#asset-id)
    - [Asset Handle](#asset-handle)
    - [Loader Registry](#loader-registry)
    - [Loader](#loader)
    - [Loader Id](#loader-id)
    - [Importer Registry](#importer-registry)
    - [Importer](#importer)
    - [Importer Id](#importer-id)
- [Problems and Possible Solutions](#problems-and-possible-solutions)
  - [Dynamic Registration but Constant Ids](#dynamic-registration-but-constant-ids)
    - [How do you keep ids the same when all things are registered dynamically?](#how-do-you-keep-ids-the-same-when-all-things-are-registered-dynamically)
    - [We only need same ids for static/internal types](#we-only-need-same-ids-for-staticinternal-types)
  - [Dynamic Asset Types and Deserialization](#dynamic-asset-types-and-deserialization)
    - [How do you know what type this data should be deserialized into?](#how-do-you-know-what-type-this-data-should-be-deserialized-into)
    - [Already solved for static/internal types, but not for dynamic ones](#already-solved-for-staticinternal-types-but-not-for-dynamic-ones)
- [Integration with the Engine](#integration-with-the-engine)

---

## Terms

There are some words I will be using here that can mean many different things without clarification. Below is a list of those.

<dl>
  <dt>asset</dt>
  <dd>
    Any data that would be loaded from anywhere outside of the compiled source code.<br> This includes but is not limited to audio files on the disk, additional sprites from the server on the internet, and prefab data stored in a database. Another point worth noting is that assets tend to have multiple instances of the same type. And about the types of assets, the next one is...
  </dd>
  <dt>asset type</dt>
  <dd>
    Unique identifier of a group of assets that provide the same functionality.<br> This is more easily understandable by examples. Audio, 3-dimensional mesh to be rendered, and texture for sprites are a few of numerous examples. Assets with different types are used for different purposes because they are meant to be different kinds of data. There could be many more types, and actually a large portion of them can be very specific to the application or game it is used for.
  </dd>
  <dt>static asset types</dt>
  <dd>
    Asset types with ids statically assgined to them, in order to be used for assets provided from the game itself.<br> Asset types that will be dynamically registered from external code (e.g. not Rust) will be called dynamic asset types.
  </dd>
  <dt>asset storage</dt>
  <dd>
    Where all loaded assets are stored to and can be retrieved from.<br> This can be thought of as a map data structure with an identifier or handle as the key and the loaded data of the asset as the value. It is convenient to have separate asset storages for different types of assets. With separate storages, there is an improvement in asset lookup operation as well as the flow of the operations.
  </dd>
  <dt>serialized asset</dt>
  <dd>
    An asset in its saved, or serialized, form.<br> It will be just an array of bytes. But the contents can have various formats such as <code>json</code>, <code>ron</code>, <code>cbor</code>, plain text vertext shaders, <code>spv</code> shaders, <code>ogg</code>, and <code>mp3</code>.
  </dd>
  <dt>loader</dt>
  <dd>
    A source of saved assets that provides reading and optionally writing.<br> A single loader represents a single source. For example, there could be 2 directories you want to read assets from. Each directory is a single source, so there would be 2 loaders, one for each. There can be many types of loaders, including a directory, a database, or the internet. Write operations are meant to be used in development, where asset ids can be generated for new assets and saved with other metadata.
  </dd>
  <dt>importer/processor</dt>
  <dd>
    Converts a serialized asset into a usable data.<br> This is responsible for parsing the incoming (probably) byte stream and turning it into a form that can be used as data. The most prominent example would be deserializing a struct or enum type. Some importers can have users change some options to customize how the serialized data is parsed and converted. For example, a sprite sheet should be parsed with the sprite data, because otherwise it is not clear how to slice it.
  </dd>
</dl>

## Requirements

### Must

1. All things must be capable of dynamic registering. "All things" include asset types, asset storages (which go together with asset types anyways), importers, and loaders.
1. At the same time, all things must be represented as data as much as possible. This means that as many things as possible should be able to be serialized and deserialized.
1. All asset types and importers that are to be predefined in Rust code must have consistent ids associated to them so that the ids remain the same across platforms and builds. This is so that the id value stored as metadata for every asset remains the same. This requirement is not applied to those that are meant to be registered dynamically, such as new types of assets registered from a script of an external mod that provides its own assets to be used by itself.
1. All references to other assets in a serialized asset must be able to be specified as either the asset ids or the paths to the assets. The former is for machine-generated values, while the latter is for manually inserted values.
1. All requests to load the same asset must result in only one actual loading of the asset. In other words, every asset should be loaded only once unless explicitly stated otherwise. This can be accompanied by making the loaded assets immutable, but that has not been solidly decided on for now.
1. All loaded assets must be able to be cloned and produce a separate instance. Referncing this new instance can only be done by using the new handle and not by using the original asset id value.

### Should

1. Automatic metadata generation should be possible to lift the burden from the developers. It's computers' job, not humans.
1. Async asset loaders should be supported. This is especially important for network sources.
1. Automatic unloading of assets that are not being used should be implemented. It also needs to be configurable for individual instances.

### Could

1. The serialized assets could be built into packed forms when ordered so in development. This will also do things like swapping path values to id values for references to other assets. Each registered loader can represent one "bundle" of assets that can be loaded separately from each other.

## Structure

*This section needs more details.*

### Parts

Below are the parts that will make up the whole asset managing process.

#### Asset Type Registry

This is where all the fun starts. The registry holds and manages all asset types to be used. It is somewhat like a map with asset type id as the key and the asset type data as the value.

Asset type registries should be serializable and deserializable. While serializing, only static types will be taken into consideration. By making the registry serializable, all asset types can be managed conveniently, and the production build of the game just needs to deserialize the asset types data to populate initial state.

#### Asset Type Id

A unique identifier of each asset type. This is needed to store what types of assets there need to be. It should be serializable and deserializable to a constant value for static asset types.

#### Asset Type Data

Some information about an asset type. There should be a name and types of formats that assets with this type can be serialized as.

#### Asset Storage

A place where all loaded assets are stored as well as processed for import. This is conceptually a map with an opaque handle as the key and the asset itself as the value. Any additional data associated to the assets are also stored together.

For separation of concern and some other reasons (including what my gut is telling me), each asset type will get its own storage instance. Since the processing of assets in their serialized forms will take place in the storage, it is beneficial to have separate storages so that different types of assets can get processed at the same time.

The reason behind the assets getting processed in the storage is that it is better to have a handle as early as possible than to keep waiting for the requested asset. Working with the handle will be a lot easier to both humans and computers.

#### Asset Id

Just as an asset type id, this is a unique identifier for a single asset data stored in a source. This should stay constant for static assets.

#### Asset Handle

An opaque handle to a loaded asset. This is needed to access the assets stored in the asset storage. This can also be used to check if the asset is actually being used through the reference count of `Arc`.

#### Loader Registry

The place where all loaders for assets are managed. Just as an asset type registry, this should be serializable and deserializable in order to make the management easier.

#### Loader

A loader reads the requested assets from its source. It can optionally provide write operations to commit generated metadata, too. Loaders should be able to be serialized and deserialized if they are meant to be registered as static loaders, meaning that they will always be used in the game no matter what things happen from external scripts.

#### Loader Id

An identification for registered loaders. This should be used to receive the registered instance of a loader. The value should stay constant for internally registered loaders.

#### Importer Registry

A place to register and retrieve importers. This can be thought of as a map from importer ids to importer instances, but the importers should also be able to be queried by supported save types and asset types. Just like an asset type registry and a loader registry, this should be able to be serialized and deserialized while skipping over any importers that are not from the internal code.

#### Importer

This converts loaded serialized assets to assets as usable data. There are also associated data with it like supported asset types and what kinds of serialized data it can handle. For example, a serializable struct and be saved as both `ron` format and `json` format. A `RonImporter`, for example, is only able to handle `ron` files while a `JsonImporter` can only convert `json` files. Thus, the information of supported types (of assets and serialized assets) are the mandatory things an importer should specify.

#### Importer Id

A unique identifier for each importer instance.

## Problems and Possible Solutions

### Dynamic Registration but Constant Ids

#### How do you keep ids the same when all things are registered dynamically?

With the very dynamic nature of this system, keeping constant id values across builds can be a very daunting task to do. Is there any way to have a fixed value for all of those types?

#### We only need same ids for static/internal types

It is impossible to keep everything that will be dynamically registered the same everywhere. Instead, only those from the internal code or scripts need consistent values, and we can instead expose some restricted and tailored API to the scripts that will come from outside.

For types from Rust code, we can use what [`atelier-assets`] has implemented. It uses `TypeUuid` trait from [`type-uuid`] crate to register static uuids of the desired types using [`inventory`] crate. This registration is different from those in this asset managing system in that it takes place when starting the Rust runtime and thus is relatively very static.

For this to work, there should be separate APIs to register static types and dynamic types. Other than that, it seems like a perfect solution.

### Dynamic Asset Types and Deserialization

#### How do you know what type this data should be deserialized into?

While registering asset types freely and dynamically sounds all good, it has its own limitations. Serialization can be easily achieved by `erased_serde` crate, but what about deserialization? The specific type needs to be specified for Rust to understand what function (that it has generated from its type system) to call for the deserialization. That is not possible only with the asset type registry. Thus for anything to be deserialized, the solid type must be specified. This requires registering a new importer for every type in Rust that represents an asset type and should be deserialized.

#### Already solved for static/internal types, but not for dynamic ones

As of now, [`atelier-assets`] has solved this problem with [`typetag`] crate. (And [`typetag`] uses [`erased-serde`] internally.) `SerdeImportable` trait from [`atelier-assets`] uses [`typetag`] to enable deserializing into the correct type. The trait used by `RonImporter` that deserializes any deserializable assets in `ron` format.

Like `TypeUuid`, this only works for types that are defined in the Rust code only. For any other types that needs deserialization, a dedicated type should be written to act as an intermediatory representation from the serialized data and the acutally wanted data type. This would be one of the responsibilities of the glue code between the game engine and a scripting engine.

## Integration with the Engine

*This section is currently WIP.*

- All registries are inserted into the world as resources.
- `AssetLoadSystem` polls on the futures from the loaders.
- `AssetImportSystem` polls on the futures from the importers.

[`atelier-assets`]: https://github.com/amethyst/atelier-assets
[`erased-serde`]: https://github.com/dtolnay/erased-serde
[`inventory`]: https://github.com/dtolnay/inventory
[`type-uuid`]: https://github.com/randomPoison/type-uuid
[`typetag`]: https://github.com/dtolnay/typetag
