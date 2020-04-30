# Implementation Details

## Table of Contents <!-- omit in toc -->

- [Terms](#terms)
- [Requirements](#requirements)
  - [Must](#must)
  - [Should](#should)
  - [Could](#could)
- [Structure](#structure)
  - [Asset Type](#asset-type)
    - [Asset Type Registry](#asset-type-registry)
    - [Asset Type Id](#asset-type-id)
    - [Asset Type Data](#asset-type-data)
  - [Asset](#asset)
    - [Asset Id](#asset-id)
    - [Asset Path](#asset-path)
    - [Asset Reference](#asset-reference)
    - [Asset Storage](#asset-storage)
    - [Asset Handle](#asset-handle)
  - [Asset Loader](#asset-loader)
    - [Loader Registry](#loader-registry)
    - [Loader](#loader)
    - [Loader Id](#loader-id)
  - [Asset Importer](#asset-importer)
    - [Importer Registry](#importer-registry)
    - [Importer](#importer)
    - [Importer Id](#importer-id)
- [Problems and Possible Solutions](#problems-and-possible-solutions)
  - [Dynamic Registration but Constant Ids](#dynamic-registration-but-constant-ids)
    - [How do you keep ids the same when all things are registered dynamically?](#how-do-you-keep-ids-the-same-when-all-things-are-registered-dynamically)
    - [We only need same ids for static/internal types](#we-only-need-same-ids-for-staticinternal-types)
  - [Dynamically Registered Types and Deserialization](#dynamically-registered-types-and-deserialization)
    - [How do you know what type the data should be deserialized into?](#how-do-you-know-what-type-the-data-should-be-deserialized-into)
    - [Already solved for static/internal types, but not for dynamic ones](#already-solved-for-staticinternal-types-but-not-for-dynamic-ones)
    - [Example for Assets](#example-for-assets)
    - [Example for Loaders](#example-for-loaders)
    - [Example for Importers](#example-for-importers)

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

### Should

1. Automatic metadata generation should be possible to lift the burden from the developers. It's computers' job, not humans.
1. Async asset loaders should be supported. This is especially important for network sources.
1. Automatic unloading of assets that are not being used should be implemented. It also needs to be configurable for individual instances.

### Could

1. All loaded assets could be able to be cloned and produce a separate instance. Referncing this new instance can only be done by using the new handle and not by using the original asset id value.
1. The serialized assets could be built into packed forms when ordered so in development. This will also do things like swapping path values to id values for references to other assets. Each registered loader can represent one "bundle" of assets that can be loaded separately from each other.

## Structure

Below are the parts that will make up the whole asset managing process.

### Asset Type

#### Asset Type Registry

This is where all the fun starts. The registry holds and manages all asset types to be used. It is somewhat like a map with asset type id as the key and the asset type data as the value.

Asset type registries should be serializable and deserializable. While serializing, only static types will be taken into consideration. By making the registry serializable, all asset types can be managed conveniently, and the production build of the game just needs to deserialize the asset types data to populate initial state.

<details>
<summary>Example implementation code</summary>

```rust
use serde::{Deserialize, Serialize, Serializer};
use std::collections::HashMap;

#[derive(Clone, Debug, Deserialize)]
#[serde(transparent)]
pub struct AssetTypeRegistry {
    types: HashMap<AssetTypeId, AssetTypeData>,
}

impl Serialize for AssetTypeRegistry {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        let statics: Vec<_> = self
          .types
          .iter() // Iterator<Item = (&AssetTypeId, &AssetTypeData)>
          .filter(|(_, data)| data.is_static)
          .collect();
        let mut map_ser = serializer.serialize_map(Some(statics.len()))?;
        for (id, data) in statics {
            map_ser.serialize_entry(id, data)?;
        }
        map_ser.end()
    }
}
```

</details>

#### Asset Type Id

A unique identifier of each asset type. This is needed to store what types of assets there need to be. It should be serializable and deserializable to a constant value for static asset types.

<details>
<summary>Example implementation code</summary>

```rust
use serde::{Deserialize, Serialize};
use uuid::Uuid;

#[derive(Clone, Copy, Debug, Eq, PartialEq, Hash, Deserialize, Serialize)]
#[serde(transparent)]
pub struct AssetTypeId(Uuid);

impl AssetTypeId {
    pub fn new_random() -> Self {
        AssetTypeId(Uuid::new_v4())
    }
}
```

</details>

#### Asset Type Data

Some information about an asset type. There should be a name and types of formats that assets with this type can be serialized as.

<details>
<summary>Example implementation code</summary>

```rust
use serde::{Deserialize, Serialize};

pub type AssetSerializeFormat = Cow<'static, str>;

#[derive(Clone, Debug, Deserialize, Serialize)]
pub struct AssetTypeData {
    name: Cow<'static, str>,
    formats: Vec<AssetSerializeFormat>,
    #[serde(skip)]
    is_static: bool,
}
```

</details>

### Asset

#### Asset Id

Just as an asset type id, this is a unique identifier for a single asset data stored in a source. This should stay constant for static assets.

<details>
<summary>Example implementation code</summary>

```rust
use serde::{Deserialize, Serialize};
use uuid::Uuid;

#[derive(Clone, Copy, Debug, Eq, PartialEq, Hash, Deserialize, Serialize)]
#[serde(transparent)]
pub struct AssetId(Uuid);

impl AssetId {
    pub fn new_random() -> Self {
        AssetId(Uuid::new_v4())
    }
}
```

</details>

#### Asset Path

Loaders must be able to identify the requested asset from the asset id or a path to the asset. The path must include information of which loader to use and how to locate the asset. In order to leave all details of locating the asset to each loader while keeping the path data simple, URIs can be used. For example, a loader that loads assets in a directory in the file system can claim that it can load the asset when the requested path starts with `file` scheme.

<details>
<summary>Example implementation code</summary>

```rust
use serde::{Deserialize, Serialize};
use url::Url;

#[derive(Clone, Debug, Eq, PartialEq, Hash, Deserialize, Serialize)]
#[serde(transparent)]
pub struct AssetPath(Url);
```

</details>

#### Asset Reference

Since an asset can be identified with either its id or its path, we can combine these two and call it the reference to the asset. This can be used to reference other assets in serialized assets. For ease of use, the serialized format should be a plain string in human-readable formats.

<details>
<summary>Example implementation code</summary>

```rust
use serde::{Deserialize, Deserializer, Serialize, Serializer};

#[derive(Clone, Debug, Eq, PartialEq, Hash)]
pub enum AssetRef {
    Id(AssetId),
    Path(AssetPath),
}

impl Serialize for AssetRef {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        match self {
            AssetRef::Id(id) => id.serialize(serializer),
            AssetRef::Path(path) => path.serialize(serializer),
        }
    }
}

impl<'de> Deserialize<'de> for AssetRef {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: Deserializer<'de>,
    {
        // try deserializing as `AssetId` and if it fails try `AssetPath` next
    }
}
```

</details>

#### Asset Storage

A place where all loaded assets are stored as well as processed for import. This is conceptually a map with an opaque handle as the key and the asset itself as the value. Any additional data associated to the assets are also stored together.

For separation of concern and some other reasons (including what my gut is telling me), each asset type will get its own storage instance. Since the processing of assets in their serialized forms will take place in the storage, it is beneficial to have separate storages so that different types of assets can get processed at the same time.

The reason behind the assets getting processed in the storage is that it is better to have a handle as early as possible than to keep waiting for the requested asset. Working with the handle will be a lot easier to both humans and computers.

<details>
<summary>Example implementation code</summary>

```rust
use futures::io::AsyncRead;
use std::collections::HashMap;
use std::sync::Arc;

pub trait LoadedAsset: mopa::Any {
    ...
}
mopa::mopafy!(LoadedAsset);

impl<T: mopa::Any> LoadedAsset for T {
    ...
}

enum StoredAsset {
    NewId(AssetId),
    NewPath(AssetPath),
    Loading(Box<dyn AsyncRead>, AssetData),
    Importing,
    Imported(Box<dyn LoadedAsset>, AssetData),
    Error(String),
}

pub struct AssetStorage {
    type_id: AssetTypeId,
    assets: HashMap<AssetHandle, StoredAsset>,
    to_remove: Vec<AssetHandle>,
}

impl AssetStorage {
    fn new_handle(&self, asset_ref: AssetRef) -> AssetHandle {
        AssetHandle::new(self.type_id, Arc::new(asset_ref))
    }

    pub fn new_asset_id(&mut self, id: AssetId) -> AssetHandle {
        // check if it already exists and return the existing one if it does

        let handle = self.new_handle(AssetRef::Id(id));
        self.assets.insert(handle.clone(), StoredAsset::NewId(id));
        handle
    }

    pub fn new_asset_path(&mut self, path: AssetPath) -> AssetHandle {
        // check if it already exists and return the existing one if it does

        let handle = self.new_handle(AssetRef::Path(path));
        self.assets.insert(handle.clone(), StoredAsset::NewPath(path));
        handle
    }

    pub fn process(
        &mut self,
        loaders: &LoaderRegistry,
        importers: &ImporterRegistry,
    ) {
        // start loading assets with `New*` variants
        // import

        // check all handles have more than 1 strong references
        for handle in self.assets.keys() {
            if !handle.is_used() {
                self.to_remove.push(handle.clone());
            }
        }

        // get rid of those with handles with only 1 reference
        for handle in self.to_remove.drain(..) {
            self.assets.remove(&handle);
        }
    }
}
```

</details>

#### Asset Handle

An opaque handle to a loaded asset. This is needed to access the assets stored in the asset storage. This can also be used to check if the asset is actually being used through the reference count of `Arc`.

<details>
<summary>Example implementation code</summary>

```rust
use std::sync::Arc;

#[derive(Clone, Debug, Eq, PartialEq, Hash)]
pub struct AssetHandle {
    type_id: AssetTypeId,
    asset_ref: Arc<AssetRef>,
}

impl AssetHandle {
    pub fn new(type_id: AssetTypeId, asset_ref: Arc<AssetRef>) -> Self {
        AssetHandle { type_id, asset_ref }
    }

    pub fn is_used(&self) -> bool {
        Arc::strong_count(&self.asset_ref) > 1
    }
}
```

</details>

### Asset Loader

#### Loader Registry

The place where all loaders for assets are managed. Just as an asset type registry, this should be serializable and deserializable in order to make the management easier.

<details>
<summary>Example implementation code</summary>

For (de)serialization implementation, see [Dynamically Registered Types and Deserialization].

```rust
use std::collections::HashMap;

#[derive(Clone)]
pub struct LoaderRegisty {
    loaders: HashMap<LoaderId, Box<dyn Loader>>,
    schemes: HashMap<Cow<'static, str>, Vec<LoaderId>>,
}
```

</details>

#### Loader

A loader reads the requested assets from its source. It can optionally provide write operations to commit generated metadata, too. Loaders must be able to be serialized and deserialized.

<details>
<summary>Example implementation code</summary>

For (de)serialization implementation, see [Dynamically Registered Types and Deserialization].

```rust
use futures::future::BoxFuture;
use futures::io::{AsyncRead, AsyncWrite};

pub type LoaderFuture<'a, T> = BoxFuture<'a, Result<T, LoaderError>>;

pub enum LoaderError {
    ...
}

pub trait Loader {
    fn supported_schemes(&self) -> Vec<Cow<'static, str>>;

    fn check_path_supported(&self, path: &AssetPath) -> bool;

    fn read_metadata(&self, asset_ref: &AssetRef) -> LoaderFuture<'_, Box<dyn AsyncRead>>;

    fn read_asset(&self, asset_ref: &AssetRef) -> LoaderFuture<'_, Box<dyn AsyncRead>>;

    fn write_metadata(
        &self,
        asset_id: AssetId,
        asset_path: &AssetPath,
    ) -> LoaderFuture<'_, Box<dyn AsyncWrite>>;
}
```

</details>

#### Loader Id

An identification for registered loaders. This should be used to receive the registered instance of a loader.

<details>
<summary>Example implementation code</summary>

```rust
use serde::{Deserialize, Serialize};
use uuid::Uuid;

#[derive(Clone, Copy, Debug, Eq, PartialEq, Hash, Deserialize, Serialize)]
#[serde(transparent)]
pub struct LoaderId(Uuid);
```

</details>

### Asset Importer

#### Importer Registry

A place to register and retrieve importers. This can be thought of as a map from importer ids to importer instances, but the importers should also be able to be queried by supported save types and asset types. Just like an asset type registry and a loader registry, this should be able to be serialized and deserialized while skipping over any importers that are not from the internal code.

<details>
<summary>Example implementation code</summary>

For (de)serialization implementation, see [Dynamically Registered Types and Deserialization].

```rust
pub struct ImporterRegistry {
    importers: HashMap<ImporterId, Box<dyn Importer>>,
    asset_types: HashMap<AssetTypeId, Vec<ImporterId>>,
}
```

</details>

#### Importer

This converts loaded serialized assets to assets as usable data. There are also associated data with it like supported asset types and what kinds of serialized data it can handle. For example, a serializable struct and be saved as both `ron` format and `json` format. A `RonImporter`, for example, is only able to handle `ron` files while a `JsonImporter` can only convert `json` files. Thus, the information of supported types (of assets and serialized assets) are the mandatory things an importer should specify.

<details>
<summary>Example implementation code</summary>

For (de)serialization implementation, see [Dynamically Registered Types and Deserialization].

```rust
use futures::future::BoxFuture;
use futures::io::AsyncRead;

pub type ImporterFuture<'a, T> = BoxFuture<'a, Result<T, ImporterError>>;

pub enum ImporterError {
    ..
}

pub trait Importer {
    fn asset_type(&self) -> AssetTypeId;

    fn supported_serialized_formats(&self) -> &[AssetSerializeFormat];

    fn import(&self, bytes: Box<dyn AsyncRead>)
        -> ImporterFuture<'_, Vec<Box<dyn LoadedAsset + 'static>>>;
}
```

</details>

#### Importer Id

A unique identifier for each importer instance. This should be serializable and deserializable, and the value should remain consistent for static/internal importers.

<details>
<summary>Example implementation code</summary>

```rust
use serde::{Deserialize, Serialize};
use uuid::Uuid;

#[derive(Clone, Copy, Debug, Eq, PartialEq, Hash, Deserialize, Serialize)]
#[serde(transparent)]
pub struct ImporterId(Uuid);
```

</details>

## Problems and Possible Solutions

### Dynamic Registration but Constant Ids

#### How do you keep ids the same when all things are registered dynamically?

With the very dynamic nature of this system, keeping constant id values across builds can be a very daunting task to do. Is there a way to have a fixed value for all of those types?

#### We only need same ids for static/internal types

It is impossible to keep everything that will be dynamically registered the same everywhere. Instead, only those from the internal code or scripts need consistent values, and we can instead expose some restricted and tailored API to the scripts that will come from outside.

For types from Rust code, we can use what [`atelier-assets`] has implemented. It uses `TypeUuid` trait from [`type-uuid`] crate to assign the given uuid to each type that implements `TypeUuid` trait. This is different from the registration in this asset managing system in that it is decided at compile time.

Example code:

```rust
use serde::{Deserialize, Serialize};
use type_uuid::{TypeUuid, TypeUuidDynamic};

#[derive(Clone, Copy, Debug, Eq, PartialEq, Deserialize, Serialize, TypeUuid)]
#[uuid = "d2b38e6d-5a2e-4978-9569-0482b0690739"]
pub struct CombatStat {
    max_health: u32,
    attack: u32,
    defense: u32,
}

let boxed: Box<dyn TypeUuidDynamic> = Box::new(CombatStat::new());
assert_eq!(boxed.uuid(), "d2b38e6d-5a2e-4978-9569-0482b0690739");
assert_eq!(<CombatStat as TypeUuid>::UUID, "d2b38e6d-5a2e-4978-9569-0482b0690739");
```

For this to work, there should be separate APIs to register static types and dynamic types. Other than that, it seems like a perfect solution.

```rust
use type_uuid::TypeUuid;
use uuid::Uuid;

impl AssetTypeId {
    pub fn of<T: TypeUuid>() -> Self {
        AssetTypeId(Uuid::from_bytes(T::UUID))
    }
}

impl AssetTypeRegistry {
    pub fn register_static_type<T: TypeUuid>(
        &mut self,
        name: impl Into<Cow<'static, str>>,
        formats: impl Into<Vec<AssetSerializeFormat>>,
    ) -> AssetTypeId {
        let id = AssetTypeId::of::<T>();
        self.types.insert(
            id,
            AssetTypeData {
                name: name.into(),
                formats: formats.into(),
                is_static: true,
            },
        );
        id
    }

    pub fn register_dynamic_type(
        &mut self,
        name: impl Into<Cow<'static, str>>,
        formats: impl Into<Vec<AssetSerializeFormat>>,
    ) -> AssetTypeId {
        let id = AssetTypeId::new_random();
        self.types.insert(
            id,
            AssetTypeData {
                name: name.into(),
                formats: formats.into(),
                is_static: false,
            },
        );
        id
    }
}
```

### Dynamically Registered Types and Deserialization

#### How do you know what type the data should be deserialized into?

While registering asset types and other things freely and dynamically sounds all good, it has its own limitations. Serialization can be easily achieved by [`erased-serde`] crate, but what about deserialization? The specific type needs to be specified for Rust to understand what function (that it has generated from its type system) to call for the deserialization. That is not possible only with the registries explain above. Thus for anything to be deserialized, the solid type must be specified. This requires tedious things; for example, a new importer must be registered for every type in Rust that represents an asset type and should be deserialized.

#### Already solved for static/internal types, but not for dynamic ones

As of now, [`atelier-assets`] has solved this problem with [`typetag`] crate. (And [`typetag`] uses [`erased-serde`] and [`inventory`] crates internally.) `SerdeImportable` trait from [`atelier-assets`] uses [`typetag`] to enable deserializing into the correct asset type. The trait is used by `RonImporter` that deserializes any deserializable assets in `ron` format.

Like `TypeUuid`, this only works for types that are defined in the Rust code only. For any other types that needs deserialization, a dedicated type should be written to act as an intermediate representation between the serialized data and the acutally wanted data type. This would be one of the responsibilities of the glue code between the game engine and a scripting engine.

#### Example for Assets

```rust
use type_uuid::TypeUuidDynamic;

#[typetag::serde(tag = "type", content = "value")]
pub trait SerdeAsset: erased_serde::Serialize + TypeUuidDynamic + LoadedAsset {}
erased_serde::serialize_trait_object!(SerdeAsset);

#[typetag::serde(name = "d2b38e6d-5a2e-4978-9569-0482b0690739")]
impl SerdeAsset for CombatStat {}
```

##### Dynamic Asset Type Registration

But we also need to think about how to deserialize assets of dynamically registered types. Let's say, for example, that there is this intermediate representation of any object in a scripting engine.

```rust
use serde::{Deserialize, Serialize};

#[derive(Clone, Debug, PartialEq, Deserialize, Serialize, TypeUuid)]
#[serde(untagged)]
#[uuid = "af5eee1f-a49f-4f9f-8dc6-46386e8b7604"]
pub enum ScriptObject {
    Number(f64),
    String(String),
    Map(HashMap<String, ScriptObject>),
    Array(Vec<ScriptObject>),
}

#[typetag::serde(name = "af5eee1f-a49f-4f9f-8dc6-46386e8b7604")]
impl SerdeAsset for ScriptObject {}
```

Then in the scripting engine, some value of a type like this ...

```ts
interface CombatStat {
  attackType: String,
  maxHealth: Number,
  attack: Number,
  defense: Number,
}
```

... will be serialized into a `json` string like below when being serialized as `Box<dyn SerdeAsset>`.

```json
{
  "type": "af5eee1f-a49f-4f9f-8dc6-46386e8b7604",
  "value": {
    "attackType": "Melee",
    "maxHealth": 1000,
    "attack": 300,
    "defense": 200
  }
}
```

When we try to register this kind of struct as an asset type, it might work as intended for the first time.

```rust
asset_type_registry.register_static_type::<ScriptObject>("CombatStat", vec!["json".into()]);
```

But when we try to register another type from the same scripting engine, ...

```rust
asset_type_registry.register_static_type::<ScriptObject>("OtherStuff", vec!["ron".into()]);
```

... we are actually overriding the first call of `register_static_type::<ScriptObject>` since both `CombatStat` and `OtherStuff` have the same type uuid in Rust's point of view. We can solve this problem using `register_dynamic_type` instead. But to keep more information for later, we can revise the asset type data a little bit to add extra information about if this type is represented as a static type in Rust runtime but acutally should be considered as a new unique asset type.

```rust
pub struct AssetTypeData {
    id: AssetTypeId,
    name: Cow<'static, str>,
    formats: Vec<AssetSerializeFormat>,
    static_type_id: Option<AssetTypeId>,
}

impl AssetTypeData {
    pub fn is_static(&self) -> bool {
        // `true` only if the id and the value inside static_type_id are equal
        self.static_type_id.map(|id| id == self.id).unwrap_or(false)
    }

    pub fn is_from_static(&self) -> bool {
        self.static_type_id.is_some()
    }
}

impl AssetTypeRegistry {
    pub fn register_static<T: TypeUuid>(
        &mut self,
        name: impl Into<Cow<'static, str>>,
        formats: impl Into<Vec<AssetSerializeFormat>>,
    ) -> AssetTypeId {
        let id = AssetTypeId::of::<T>(),
        self.types.insert(
            id,
            AssetTypeData {
                id,
                name: name.into(),
                formats: formats.into(),
                static_type_id: Some(id),
            },
        );
        id
    }

    pub fn register_dynamic_from_static<T: TypeUuid>(
        &mut self,
        name: impl Into<Cow<'static, str>>,
        formats: impl Into<Vec<AssetSerializeFormat>>,
    ) -> AssetTypeId {
        let id = AssetTypeId::new_random();
        self.types.insert(
            id,
            AssetTypeData {
                id,
                name: name.into(),
                formats: formats.into(),
                static_type_id: Some(AssetTypeId::of::<T>()),
            },
        );
        id
    }

    pub fn register_dynamic(
        &mut self,
        name: impl Into<Cow<'static, str>>,
        formats: impl Into<Vec<AssetSerializeFormat>>,
    ) -> AssetTypeId {
        let id = AssetTypeId::new_random();
        self.types.insert(
            id,
            AssetTypeData {
                id,
                name: name.into(),
                formats: formats.into(),
                static_type_id: None,
            },
        );
        id
    }
}
```

##### Importing Deserializable Assets

To make importers not tied to a solid asset type, the importer trait needs some revision.

```rust
pub type ImporterResult<T> = Result<T, ImporterError>;

pub trait Importer {
    fn supported_serialized_formats(&self) -> &[AssetSerializeFormat];

    fn import(&self, data: &AssetTypeData, bytes: Box<dyn AsyncRead>)
        -> ImporterFuture<'_, Vec<Box<dyn LoadedAsset + 'static>>>;
}
```

Now we can create and register an importer that deserializes assets from `json` format.

```rust
use serde::{Deserialize, Serialize};

const JSON_FORMAT: &[Cow<'static, str>] = &[Cow::Borrowed("json")];

#[derive(Clone, Copy, Debug, Eq, PartialEq, Deserialize, Serialize)]
pub struct JsonImporter;

impl Importer for JsonImporter {
    fn supported_serialized_formats(&self) -> &[AssetSerializeFormat] {
        JSON_FORMAT
    }

    fn import(
        &self,
        data: &AssetTypeData,
        bytes: Box<dyn AsyncRead>,
    ) -> ImporterFuture<'_, Vec<Box<dyn LoadedAsset + 'static>>> {
        Box::pin(import_json(data, bytes)) // not sure if the type is right here
    }
}

async fn import_json<'a>(
    data: &'a AssetTypeData,
    mut bytes: Box<dyn AsyncRead + 'a>,
) -> ImporterResult<Vec<Box<dyn LoadedAsset + 'static>>> {
    use futures::io::AsyncReadExt;
    let mut content = Vec::new();
    bytes.read_to_end(&mut content).await?;
    let deserialized: Box<dyn SerdeAsset> = serde_json::de::from_bytes(&content)?;
    Ok(deserialized as Box<dyn LoadedAsset>)
}

let json_importer_id = importer_registry.register(Box::new(JsonImporter));
importer_registry.add_asset_type(json_importer_id, some_asset_type_id);
```

#### Example for Loaders

Loaders do not need to have same ids across builds. However, their type ids are needed in order to deserialize to a right instance.

```rust
use serde::{Deserialize, Serialize, Serializer};
use type_uuid::TypeUuidDynamic;

#[typetag::serde(tag = "type", content = "loader")]
pub trait Loader: erased_serde::Serialize + TypeUuidDynamic {
    ...
}
erased_serde::serialize_trait_object!(Loader);

pub struct LoaderData {
    is_static: bool,
}

impl LoaderData {
    pub fn new_static() -> Self {
        LoaderData {
            is_static: true,
        }
    }

    pub fn new_dynamic() -> Self {
        LoaderData {
            is_static: false,
        }
    }
}

#[derive(Clone, Deserialize, Serialize)]
pub struct LoaderRegisty {
    #[serde(serialize_with = "loader_registry_loaders_serialize")]
    loaders: HashMap<LoaderId, (Box<dyn Loader>, LoaderData)>,
    schemes: HashMap<Cow<'static, str>, Vec<LoaderId>>,
}

impl LoaderRegistry {
    pub fn register_static(&mut self, loader: impl Loader) -> LoaderId {
        let id = LoaderId::new_random();
        for scheme in loader.supported_schemes() {
            self.schemes
                .entry(scheme)
                .or_insert_with(Vec::new)
                .push(id);
        }
        self.loaders.insert(id, (Box::new(loader), LoaderData::new_static()));
        id
    }

    pub fn register_dynamic(&mut self, loader: impl Loader) -> LoaderId {
        ... // same as `register_static`
        self.loaders.insert(id, (Box::new(loader), LoaderData::new_dynamic()));
        id
    }
}

fn loader_registry_loaders_serialize<S>(
    loaders: &HashMap<LoaderId, (Box<dyn Loader>, LoaderData)>,
    serializer: S
) -> Result<S::Ok, S::Error>
where
    S: Serializer,
{
    // serialize the map, omitting dynamically registered loaders
}
```

Example of a struct implementing the `Loader` trait:

```rust
use std::path::PathBuf;
use type_uuid::TypeUuid;

#[derive(Clone, Debug, Eq, PartialEq, Deserialize, Serialize, TypeUuid)]
#[uuid = "d114388a-baff-491b-a0de-1c1f2a68eae7"]
pub struct DirectoryLoader {
    name: Cow<'static, str>,
    dir_path: PathBuf,
}

#[typetag::serde(name = "d114388a-baff-491b-a0de-1c1f2a68eae7")]
impl Loader for DirectoryLoader {
    fn supported_schemes(&self) -> Vec<Cow<'static, str>> {
        vec!["file".into(), "x-moya-asset-directory".into()]
    }

    ...
}
```

#### Example for Importers

Unlike loaders, ids of importers will be used in the asset metadata to cache which importer is needed for each asset. Therefore, the ids for static importers need to stay consistent. Also, they need to be serializable and deserializable.

```rust
use type_uuid::TypeUuidDynamic;

#[typetag::serde(tag = "type", content = "importer")]
pub trait Importer: erased_serde::Serialize + TypeUuidDynamic {
    fn supported_serialized_formats(&self) -> &[AssetSerializeFormat];

    fn import(&self, data: &AssetTypeData, bytes: Box<dyn AsyncRead>)
        -> ImporterFuture<'_, Vec<Box<dyn LoadedAsset + 'static>>>;
}
erased_serde::serialize_trait_object!(Importer);
```

Example for a static importer that deserializes assets from `ron` format:

```rust
use serde::{Deserialize, Serialize};

const RON_FORMAT: &[Cow<'static, str>] = &[Cow::Borrowed("ron")];

#[derive(Clone, Copy, Debug, Deserialize, Serialize, TypeUuid)]
#[uuid = "9e82ef2d-1044-41e7-8e68-b70dc5066f5f"]
pub struct RonImporter;

#[typetag::serde(name = "9e82ef2d-1044-41e7-8e68-b70dc5066f5f")]
impl Importer for RonImporter {
    fn supported_serialized_formats(&self) -> &[AssetSerializeFormat] {
        RON_FORMAT
    }

    fn import(
        &self,
        data: &AssetTypeData,
        bytes: Box<dyn AsyncRead>,
    ) -> ImporterFuture<'_, Vec<Box<dyn LoadedAsset + 'static>>> {
        ... // similar to `JsonImporter` above
    }
}
```

<!-- Links -->
[`atelier-assets`]: https://github.com/amethyst/atelier-assets
[`erased-serde`]: https://github.com/dtolnay/erased-serde
[`inventory`]: https://github.com/dtolnay/inventory
[`type-uuid`]: https://github.com/randomPoison/type-uuid
[`typetag`]: https://github.com/dtolnay/typetag
[Dynamically Registered Types and Deserialization]: #dynamically-registered-types-and-deserialization
