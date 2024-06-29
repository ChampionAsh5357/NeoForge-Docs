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

Biome modifiers are made up of three parts:

- The [datapack registered][datareg] `BiomeModifier` used to modify the biome builder.
- The [statically registered][staticreg] `MapCodec` that encodes and decodes the modifiers.
- The JSON that constructs the `BiomeModifier`, using the registered id of the `MapCodec` as the indexable type.

### The `BiomeModifier` Implementation

A `BiomeModifier` contains two methods: `#modify` and `#codec`. `modify` takes in the current `Biome` being constructor, the modifier `BiomeModifier.Phase`, and the builder of the biome to modify. Every `BiomeModifier` is called once per `Phase` to organize when certain modifications to the biome should occur:

| Phase               | Description                                                              |
|:-------------------:|:-------------------------------------------------------------------------|
| `BEFORE_EVERYTHING` | A catch-all for everything that needs to run before the standard phases. |
| `ADD`               | Adding features, mob spawns, etc.                                        |
| `REMOVE`            | Removing features, mob spawns, etc.                                      |
| `MODIFY`            | Modifying single values (e.g., climate, colors).                         |
| `AFTER_EVERYTHING`  | A catch-all for everything that needs to run after the standard phases.  |

`codec` takes in the `MapCodec` that encodes and decodes the modifiers. This `MapCodec` is [statically registered][staticreg], with its id used as the type of the biome modifier.

```java
public record ExampleBiomeModifier(HolderSet<Biome> biomes, int value) implements BiomeModifier {
    
    @Override
    public void modify(Holder<Biome> biome, Phase phase, ModifiableBiomeInfo.BiomeInfo.Builder builder) {
        if (phase == /* Pick the phase that best matches what your want to modify */) {
            // Modify the 'builder', checking any information about the biome itself
        }
    }

    @Override
    public MapCodec<? extends BiomeModifier> codec() {
        return EXAMPLE_BIOME_MODIFIER.value();
    }
}

// In some registration class
private static final DeferredRegister<MapCodec<? extends BiomeModifier>> REGISTRAR =
    DeferredRegister.create(NeoForgeRegistries.Keys.BIOME_MODIFIER_SERIALIZERS, MOD_ID);

public static final Holder<MapCodec<? extends BiomeModifier>> EXAMPLE_BIOME_MODIFIER =
    REGISTRAR.register("example_biome_modifier", () -> RecordCodecBuilder.mapCodec(instance ->
        instance.group(
            Biome.LIST_CODEC.fieldOf("biomes").forGetter(ExampleBiomeModifier::biomes),
            Codec.INT.fieldOf("value").forGetter(ExampleBiomeModifier::value)
        ).apply(instance, ExampleBiomeModifier::new)
    ));
```

### Generating the Biome Modifier

A `BiomeModifier` can be created in two ways: writing the JSON manually or via [data generation][datagen] by passing a `RegistrySetBuilder` to `DatapackBuiltinEntriesProvider`. Biome Modifiers are located within `data/<modid>/neoforge/biome_modifier/<path>.json` All biome modifiers contain a `type` key that references the id of the `MapCodec` used for the biome modifier. All other settings provided by the biome modifier are added as additional keys on the root object.

```java
// Define keys for datapack registry objects

public static final ResourceKey<BiomeModifier> EXAMPLE_MODIFIER =
    ResourceKey.create(
        NeoForgeRegistries.Keys.BIOME_MODIFIERS, // The registry this key is for
        ResourceLocation.fromNamespaceAndPath(MOD_ID, "example_modifier") // The registry name
    );

// For some RegistrySetBuilder BUILDER
//   being passed to DatapackBuiltinEntriesProvider
//   in a listener for GatherDataEvent
BUILDER.add(NeoForgeRegistries.Keys.BIOME_MODIFIERS, bootstrap -> {
    // Lookup any necessary registries
    // Static registries only need to be looked up if you need to grab the tag data
    HolderGetter<Biome> biomes = bootstrap.lookup(Registries.BIOME);

    // Register the biome modifiers

    bootstrap.register(EXAMPLE_MODIFIER,
        new ExampleBiomeModifier(
            biomes.getOrThrow(Tags.Biomes.IS_OVERWORLD),
            20
        )
    );
})
```

```json5
// In data/examplemod/neoforge/biome_modifier/example_modifier.json
{
    // The registy key of the MapCodec for the modifier
    "type": "examplemod:example_biome_modifier",
    // All additional settings are applied to the root object
    "biomes": "#c:is_overworld",
    "value": 20
}
```

## Existing Biome Modifiers

NeoForge provides some basic biome modifiers for common usecases. Both the datagen and JSON implementation will be shown. All biome modifiers can be found in `BiomeModifiers`.

### Disabling Biome Modifiers

Datapacks can disable mod-added biome modifiers by overriding the associated JSON and setting the type to `neoforge:none`.

```json
// For some existing data/examplemod/neoforge/biome_modifier/add_features_example.json
// In another datapack
{
    "type": "neoforge:none"
}
```

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

`RemoveSpawnsBiomeModifier` removes mob spawning information from biomes. The modifier takes in the `Biome` id or tag of the biomes the spawning information are removed from and the `EntityType` id or tag of the mobs to remove.

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

`AddSpawnCostsBiomeModifier` adds spawn costs for mobs to biomes. The modifier takes in the `Biome` id or tag of the biomes the spawn costs are added to, the `EntityType` id or tag of the mobs to add spawn costs for, and the `MobSpawnSettings.MobSpawnCost` of the mob. The `MobSpawnCost` contains the energy budget, which indicates the maximum number of entities that can spawn in a location based upon the charge provided for each entity spawned. 

```java
// Define keys for datapack registry objects

public static final ResourceKey<BiomeModifier> ADD_SPAWN_COSTS_EXAMPLE =
    ResourceKey.create(
        NeoForgeRegistries.Keys.BIOME_MODIFIERS, // The registry this key is for
        ResourceLocation.fromNamespaceAndPath(MOD_ID, "add_spawn_costs_example") // The registry name
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

    bootstrap.register(ADD_SPAWN_COSTS_EXAMPLE,
        new AddSpawnCostsBiomeModifier(
            // The biome(s) to add the spawn costs to
            biomes.getOrThrow(Tags.Biomes.IS_OVERWORLD),
            // The entities to add the spawn costs for
            entities.getOrThrow(EntityTypeTags.SKELETONS),
            new MobSpawnSettings.MobSpawnCost(
                1.0, // The energy budget
                0.1  // The amount of charge each entity takes up from the budget
            )
        )
    );
})
```

```json5
// In data/examplemod/neoforge/biome_modifier/add_spawn_costs_example.json
{
    "type": "neoforge:add_spawn_costs",
    // Can either be an id "minecraft:plains"
    //   List of ids ["minecraft:plains", "minecraft:badlands", ...]
    //   Or a tag "#c:is_overworld"
    "biomes": "#c:is_overworld",
    // Can either be an id "minecraft:ghast"
    //   List of ids ["minecraft:ghast", "minecraft:skeleton", ...]
    //   Or a tag "#minecraft:skeletons"
    "entity_types": "#minecraft:skeletons",
    "spawn_cost": {
        // The energy budget
        "energy_budget": 1.0,
        // The amount of charge each entity takes up from the budget
        "charge": 0.1
    }
}
```

### `RemoveSpawnCostsBiomeModifier`

`RemoveSpawnsBiomeModifier` removes spawn costs for mobs from biomes. The modifier takes in the `Biome` id or tag of the biomes the spawn costs are removed from and the `EntityType` id or tag of the mobs to remove the spawn cost for.

```java
// Define keys for datapack registry objects

public static final ResourceKey<BiomeModifier> REMOVE_SPAWN_COSTS_EXAMPLE =
    ResourceKey.create(
        NeoForgeRegistries.Keys.BIOME_MODIFIERS, // The registry this key is for
        ResourceLocation.fromNamespaceAndPath(MOD_ID, "remove_spawn_costs_example") // The registry name
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

    bootstrap.register(REMOVE_SPAWN_COSTS_EXAMPLE,
        new RemoveSpawnCostsBiomeModifier(
            // The biome(s) to remove the spawnc costs from
            biomes.getOrThrow(Tags.Biomes.IS_OVERWORLD),
            // The entities to remove spawn costs for
            entities.getOrThrow(EntityTypeTags.SKELETONS)
        )
    );
})
```

```json5
// In data/examplemod/neoforge/biome_modifier/remove_spawn_costs_example.json
{
    "type": "neoforge:remove_spawn_costs",
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


[features]: ../features.md
[datareg]: ../../concepts/registries.md#datapack-registries
[staticreg]: ../../concepts/registries.md#methods-for-registering
[datagen]: ../../resources/index.md#data-generation
[placedfeature]: ../features.md#placing-a-feature
[decoration]: ../features.md#adding-a-feature-to-a-biome
