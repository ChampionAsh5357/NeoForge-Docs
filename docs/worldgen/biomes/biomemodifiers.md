# Biome Modifiers

Biome Modifiers are a data-driven system for modifying a biome. A modifier may modify the following information about a biome:

- Adding or removing [features] and carvers
- Adding or removing mob spawns and mob spawn costs
- Modifying the climate of a biome
- Modifying rendering effects of the biome
    - Colors of the water, fog, sky, foliage, and grass
    - Ambient particles to spawn
    - Background music and ambient sounds

## Creating a Biome Modifier

TODO

## Existing Biome Modifiers

NeoForge provides some basic biome modifiers for common usecases. Both the datagen and JSON implementation will be shown.

### `AddFeaturesBiomeModifier`

`AddFeaturesBiomeModifier` adds features, such as ore generation, to biomes. The modifier takes in the `Biome` id or tag of the biomes the features are added to, a [`PlacedFeature`][placedfeature] id or tag of the features to add to the selected biomes, and the [`GenerationStep.Decoration`][decoration] the features will be generated within.

```java
// Assume we have some PlacedFeature EXAMPLE_PLACED_FEATURE

// Define keys for datapack registry objects

public static final ResourceKey<BiomeModifier> ADD_FEATURES_EXAMPLE =
    ResourceKey.create(
        NeoForgeRegistries.Keys.BIOME_MODIFIERS, // The registry this key is for
        ResourceLocation.fromNamespaceAndPath(MOD_ID, "add_features_example") // The registry name
    );

// For some RegistrySetBuilder BUILDER
//   being passed to DatapackBuiltinEntriesProvider
//   in a listener for GatherDataEvent
BUILDER.add(NeoForgeRegistries.Keys.BIOME_MODIFIERS, bootstrap -> {
    // Lookup any necessary registries
    // Static registries only need to be looked up if you need to grab the tag data
    HolderGetter<Biome> biomes = bootstrap.lookup(Registries.BIOME);
    HolderGetter<PlacedFeature> placedFeatures = bootstrap.lookup(Registries.PLACED_FEATURE);

    // Register the biome modifiers

    bootstrap.register(ADD_FEATURES_EXAMPLE,
        new AddFeaturesBiomeModifier(
            // The biome(s) to generate within
            HolderSet.direct(biomes.getOrThrow(Biomes.PLAINS)),
            // The feature(s) to generate within the biomes
            HolderSet.direct(placedFeatures.getOrThrow(EXAMPLE_PLACED_FEATURE)),
            // The generation step
            GenerationStep.Decoration.LOCAL_MODIFICATIONS
        )
    );
})
```

```json5
// In data/examplemod/neoforge/biome_modifier/add_features_example.json
{
    "type": "neoforge:add_features",
    // Can either be an id "minecraft:plains"
    //   List of ids ["minecraft:plains", "minecraft:badlands", ...]
    //   Or a tag "#c:is_overworld"
    "biomes": "minecraft:plains",
    // Can either be an id "examplemod:add_features_example"
    //   List of ids ["examplemod:add_features_example", "minecraft:ice_spike", ...]
    //   Or a tag "#examplemod:placed_feature_tag"
    "features": "examplemod:add_features_example",
    // See GenerationStep.Decoration for a list of valid enum names
    "step": "local_modifications"
}
```

:::warning
Avoid adding vanilla placed features using biome modifiers, as this may cause a feature cycle violation. A feature cycle violation is when two biomes have the same feature in the lists but in different orders.

A feature cycle violation may also occur if the same feature is added using more than one biome modifier, so only use one biome modifier to add any given feature.
:::

### `RemoveFeaturesBiomeModifier`

`RemoveFeaturesBiomeModifier` removes features from biomes. The modifier takes in the `Biome` id or tag of the biomes the features are removed from, a [`PlacedFeature`][placedfeature] id or tag of the features to remove from the selected biomes, and the [`GenerationStep.Decoration`s][decoration] that the features will be removed from.

```java
// Define keys for datapack registry objects

public static final ResourceKey<BiomeModifier> REMOVE_FEATURES_EXAMPLE =
    ResourceKey.create(
        NeoForgeRegistries.Keys.BIOME_MODIFIERS, // The registry this key is for
        ResourceLocation.fromNamespaceAndPath(MOD_ID, "remove_features_example") // The registry name
    );

// For some RegistrySetBuilder BUILDER
//   being passed to DatapackBuiltinEntriesProvider
//   in a listener for GatherDataEvent
BUILDER.add(NeoForgeRegistries.Keys.BIOME_MODIFIERS, bootstrap -> {
    // Lookup any necessary registries
    // Static registries only need to be looked up if you need to grab the tag data
    HolderGetter<Biome> biomes = bootstrap.lookup(Registries.BIOME);
    HolderGetter<PlacedFeature> placedFeatures = bootstrap.lookup(Registries.PLACED_FEATURE);

    // Register the biome modifiers

    bootstrap.register(REMOVE_FEATURES_EXAMPLE,
        new RemoveFeaturesBiomeModifier(
            // The biome(s) to remove from
            biomes.getOrThrow(Tags.Biomes.IS_OVERWORLD),
            // The feature(s) to remove from the biomes
            HolderSet.direct(placedFeatures.getOrThrow(OrePlacements.ORE_DIAMOND)),
            // The generation steps to remove from
            Set.of(
                GenerationStep.Decoration.LOCAL_MODIFICATIONS,
                GenerationStep.Decoration.UNDERGROUND_ORES
            )
        )
    );
})
```

```json5
// In data/examplemod/neoforge/biome_modifier/remove_features_example.json
{
    "type": "neoforge:remove_features",
    // Can either be an id "minecraft:plains"
    //   List of ids ["minecraft:plains", "minecraft:badlands", ...]
    //   Or a tag "#c:is_overworld"
    "biomes": "#c:is_overworld",
    // Can either be an id "minecraft:ore_diamond"
    //   List of ids ["minecraft:ore_diamond", "minecraft:ice_spike", ...]
    //   Or a tag "#examplemod:placed_feature_tag"
    "features": "minecraft:ore_diamond",
    // Can either be a single step "local_modifications"
    //   Or a list ["local_modifications", "underground_ores"]
    // See GenerationStep.Decoration for a list of valid enum names
    "steps": [
        "local_modifications",
        "underground_ores"
    ]
}
```

### `AddSpawnsBiomeModifier`

`AddSpawnsBiomeModifier` adds mob spawning information to be used by `NaturalSpawner` to biomes. The modifier takes in the `Biome` id or tag of the biomes the spawning information are added to and the `SpawnerData` of the mobs to add. Each `SpawnerData` contains the mob id, the spawn weight, and the minimum/maximum number of mobs to spawn at a given time.

```java
// Assume we have some EntityType EXAMPLE_ENTITY

// Define keys for datapack registry objects

public static final ResourceKey<BiomeModifier> ADD_SPAWNS_EXAMPLE =
    ResourceKey.create(
        NeoForgeRegistries.Keys.BIOME_MODIFIERS, // The registry this key is for
        ResourceLocation.fromNamespaceAndPath(MOD_ID, "add_spawns_example") // The registry name
    );

// For some RegistrySetBuilder BUILDER
//   being passed to DatapackBuiltinEntriesProvider
//   in a listener for GatherDataEvent
BUILDER.add(NeoForgeRegistries.Keys.BIOME_MODIFIERS, bootstrap -> {
    // Lookup any necessary registries
    // Static registries only need to be looked up if you need to grab the tag data
    HolderGetter<Biome> biomes = bootstrap.lookup(Registries.BIOME);

    // Register the biome modifiers

    bootstrap.register(ADD_SPAWNS_EXAMPLE,
        new AddSpawnsBiomeModifier(
            // The biome(s) to spawn the mobs within
            HolderSet.direct(biomes.getOrThrow(Biomes.PLAINS)),
            // The spawners of the entities to add
            List.of(
                new SpawnerData(EXAMPLE_ENTITY, 100, 1, 4),
                new SpawnerData(EntityType.GHAST, 1, 5, 10)
            )
        )
    );
})
```

```json5
// In data/examplemod/neoforge/biome_modifier/add_spawns_example.json
{
    "type": "neoforge:add_spawns",
    // Can either be an id "minecraft:plains"
    //   List of ids ["minecraft:plains", "minecraft:badlands", ...]
    //   Or a tag "#c:is_overworld"
    "biomes": "minecraft:plains",
    // Can be either a single object or a list of objects
    "spawners": [
        {
            // The entity id
            "type": "examplemod:example_entity",
            // The weighted spawn value
            "weight": 100,
            // The minimum number to spawn at once
            "minCount": 1,
            // The maxmimum number to spawn at once
            "maxCount": 4
        },
        {
            "type": "minecraft:ghast",
            "weight": 1,
            "minCount": 5,
            "maxCount": 10
        }
    ]
}
```

### `RemoveSpawnsBiomeModifier`

`RemoveSpawnsBiomeModifier` removes mob spawning information from biomes. The modifier takes in the `Biome` id or tag of the biomes the spawning information are removed from and the `EntityTypes` of the mobs to remove.

```java
// Define keys for datapack registry objects

public static final ResourceKey<BiomeModifier> REMOVE_SPAWNS_EXAMPLE =
    ResourceKey.create(
        NeoForgeRegistries.Keys.BIOME_MODIFIERS, // The registry this key is for
        ResourceLocation.fromNamespaceAndPath(MOD_ID, "remove_spawns_example") // The registry name
    );

// For some RegistrySetBuilder BUILDER
//   being passed to DatapackBuiltinEntriesProvider
//   in a listener for GatherDataEvent
BUILDER.add(NeoForgeRegistries.Keys.BIOME_MODIFIERS, bootstrap -> {
    // Lookup any necessary registries
    // Static registries only need to be looked up if you need to grab the tag data
    HolderGetter<Biome> biomes = bootstrap.lookup(Registries.BIOME);
    HolderGetter<EntityType<?>> entities = bootstrap.lookup(Registries.ENTITY_TYPE);

    // Register the biome modifiers

    bootstrap.register(REMOVE_SPAWNS_EXAMPLE,
        new RemoveSpawnsBiomeModifier(
            // The biome(s) to remove the spawns from
            biomes.getOrThrow(Tags.Biomes.IS_OVERWORLD),
            // The entities to remove spawns for
            entities.getOrThrow(EntityTypeTags.SKELETONS)
        )
    );
})
```

```json5
// In data/examplemod/neoforge/biome_modifier/remove_spawns_example.json
{
    "type": "neoforge:remove_spawns",
    // Can either be an id "minecraft:plains"
    //   List of ids ["minecraft:plains", "minecraft:badlands", ...]
    //   Or a tag "#c:is_overworld"
    "biomes": "#c:is_overworld",
    // Can either be an id "minecraft:ghast"
    //   List of ids ["minecraft:ghast", "minecraft:skeleton", ...]
    //   Or a tag "#minecraft:skeletons"
    "entity_types": "#minecraft:skeletons"
}
```

### `AddCarversBiomeModifier`

`AddCarversBiomeModifier` adds carvers, such as caves, to biomes. The modifier takes in the `Biome` id or tag of the biomes the carvers are added to, a `ConfiguredWorldCarver` id or tag of the carvers to add to the selected biomes, and the `GenerationStep.Carving` the carvers will be generated within.

```java
// Assume we have some ConfiguredWorldCarver EXAMPLE_CARVER

// Define keys for datapack registry objects

public static final ResourceKey<BiomeModifier> ADD_CARVERS_EXAMPLE =
    ResourceKey.create(
        NeoForgeRegistries.Keys.BIOME_MODIFIERS, // The registry this key is for
        ResourceLocation.fromNamespaceAndPath(MOD_ID, "add_carvers_example") // The registry name
    );

// For some RegistrySetBuilder BUILDER
//   being passed to DatapackBuiltinEntriesProvider
//   in a listener for GatherDataEvent
BUILDER.add(NeoForgeRegistries.Keys.BIOME_MODIFIERS, bootstrap -> {
    // Lookup any necessary registries
    // Static registries only need to be looked up if you need to grab the tag data
    HolderGetter<Biome> biomes = bootstrap.lookup(Registries.BIOME);
    HolderGetter<ConfiguredWorldCarver<?>> carvers = bootstrap.lookup(Registries.CONFIGURED_CARVER);

    // Register the biome modifiers

    bootstrap.register(ADD_CARVERS_EXAMPLE,
        new AddCarversBiomeModifier(
            // The biome(s) to generate within
            HolderSet.direct(biomes.getOrThrow(Biomes.PLAINS)),
            // The carver(s) to generate within the biomes
            HolderSet.direct(carvers.getOrThrow(EXAMPLE_CARVER)),
            // The generation step
            GenerationStep.Carving.AIR
        )
    );
})
```

```json5
// In data/examplemod/neoforge/biome_modifier/add_carvers_example.json
{
    "type": "neoforge:add_carvers",
    // Can either be an id "minecraft:plains"
    //   List of ids ["minecraft:plains", "minecraft:badlands", ...]
    //   Or a tag "#c:is_overworld"
    "biomes": "minecraft:plains",
    // Can either be an id "examplemod:add_carvers_example"
    //   List of ids ["examplemod:add_carvers_example", "minecraft:canyon", ...]
    //   Or a tag "#examplemod:configured_carver_tag"
    "carvers": "examplemod:add_carvers_example",
    // See GenerationStep.Carving for a list of valid enum names
    "step": "air"
}
```

### `RemoveCarversBiomeModifier`

`RemoveCarversBiomeModifier` removes carvers from biomes. The modifier takes in the `Biome` id or tag of the biomes the carvers are removed from, a `ConfiguredWorldCarver` id or tag of the carvers to remove from the selected biomes, and the `GenerationStep.Carving`s that the carvers will be removed from.

```java
// Define keys for datapack registry objects

public static final ResourceKey<BiomeModifier> REMOVE_CARVERS_EXAMPLE =
    ResourceKey.create(
        NeoForgeRegistries.Keys.BIOME_MODIFIERS, // The registry this key is for
        ResourceLocation.fromNamespaceAndPath(MOD_ID, "remove_carvers_example") // The registry name
    );

// For some RegistrySetBuilder BUILDER
//   being passed to DatapackBuiltinEntriesProvider
//   in a listener for GatherDataEvent
BUILDER.add(NeoForgeRegistries.Keys.BIOME_MODIFIERS, bootstrap -> {
    // Lookup any necessary registries
    // Static registries only need to be looked up if you need to grab the tag data
    HolderGetter<Biome> biomes = bootstrap.lookup(Registries.BIOME);
    HolderGetter<ConfiguredWorldCarver<?>> carvers = bootstrap.lookup(Registries.CONFIGURED_CARVER);

    // Register the biome modifiers

    bootstrap.register(REMOVE_CARVERS_EXAMPLE,
        new AddFeaturesBiomeModifier(
            // The biome(s) to remove from
            biomes.getOrThrow(Tags.Biomes.IS_OVERWORLD),
            // The carver(s) to remove from the biomes
            HolderSet.direct(carvers.getOrThrow(Carvers.CAVE)),
            // The generation steps to remove from
            Set.of(
                GenerationStep.Carving.AIR,
                GenerationStep.Carving.LIQUID
            )
        )
    );
})
```

```json5
// In data/examplemod/neoforge/biome_modifier/remove_carvers_example.json
{
    "type": "neoforge:remove_carvers",
    // Can either be an id "minecraft:plains"
    //   List of ids ["minecraft:plains", "minecraft:badlands", ...]
    //   Or a tag "#c:is_overworld"
    "biomes": "#c:is_overworld",
    // Can either be an id "minecraft:cave"
    //   List of ids ["minecraft:cave", "minecraft:canyon", ...]
    //   Or a tag "#examplemod:configured_carver_tag"
    "carvers": "minecraft:cave",
    // Can either be a single step "air"
    //   Or a list ["air", "liquid"]
    // See GenerationStep.Carving for a list of valid enum names
    "steps": [
        "air",
        "liquid"
    ]
}
```

### `AddSpawnCostsBiomeModifier`

TODO

### `RemoveSpawnCostsBiomeModifier`

TODO


[features]: ../features.md
[placedfeature]: ../features.md#placing-a-feature
[decoration]: ../features.md#adding-a-feature-to-a-biome
