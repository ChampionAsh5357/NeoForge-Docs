---
sidebar_position: 1
---
# Features

Features are the final decorators generated within each chunk. Features define how ores, ice spikes, lava lakes, etc. generate. Placements further vary where a feature is generated within each chunk.

## Creating a Feature

There are two parts to create a new feature: the `Feature<FC>` subclass that defines how a feature, given a starting position, is generated; and the `FeatureConfiguration` implementation that defines a configuration to use to generate the feature (e.g., the size of an ore vein, what block state to spawn, etc.). `FC` is the type of the `FeatureConfiguration`.

A `FeatureConfiguration` can be implemented like any other data object, or, if no configuration is required, then a `NoneFeatureConfiguration` can be used. The `FeatureConfiguration` implementation must also have a [`Codec`][codec] to encode and decode the data. If a `FeatureConfiguration` takes in other features to generate, then `FeatureConfiguration#getFeatures` must be overridden to a list of [configured features][configuredfeature] that could be used during the generation of this feature.

```java
public record ExampleConfiguration(int height, BlockState state) implements FeatureConfiguration {

    public static final Codec<ExampleConfiguration> CODEC = RecordCodecBuilder.create(
        instance -> instance.group(
            ExtraCodecs.POSITIVE_INT.fieldOf("height").forGetter(ExampleConfiguration::height),
            BlockState.CODEC.fieldOf("state").forGetter(ExampleConfiguration::state)
        ).apply(instance, ExampleConfiguration::new)
    );
}
```

The `Feature<FC>`, where `FC` is the type of the `FeatureConfiguration`, subclass takes in the codec used to encode and decode the associated `FeatureConfiguration`. `Feature` only has one method to implement: `#place`. The `place` method takes in a `FeaturePlaceContext` that contains the level this feature will be generated within, the generator of the chunk, a random instance, the starting `BlockPos` to generate the feature at, and the configuration object. If the feature was successfully generated, then `place` should return `true`.

:::warning
A feature must only operate within a 16x16 horizontal block centered on the starting position. Features are only allowed to set or get blocks within a 3x3 chunk centered around the chunk the feature is generating within. Any operations outside this 16x16 radius will likely cause an exception to be thrown at a random point in time during chunk generation.
:::

`Feature`s must be [statically registered][staticregistration].

```java
public class ExampleFeature extends Feature<ExampleConfiguration> {

    public ExampleFeature() {
        // Pass in the configuration codec
        super(ExampleConfiguration.CODEC);
    }

    @Override
    public boolean place(FeaturePlaceContext<ExampleConfiguration> ctx) {
        WorldGenLevel level = ctx.level();
        ExampleConfiguration config = ctx.config();

        for (int i = 0; i < config.height(), i++) {
            level.setBlock(config.state(), ctx.origin().above(i), Block.UPDATE_CLIENTS);
        }

        return true;
    }
}

// In some class where you register your registry objects
public static final DeferredRegister<Feature<?>> FEATURES =
    DeferredRegister.create(Registries.FEATURE, MOD_ID);

public static final DeferredHolder<Feature<?>, ExampleFeature> EXAMPLE_FEATURE =
    FEATURES.register("example", ExampleFeature::new);
```

## Configuring a Feature

Once you have chosen or created a [registered feature][feature] to generate in the level, the feature must be configured with any specified settings. A feature with a set configuration is known as a `ConfiguredFeature<FC, F>`, where `FC` is the type of the `FeatureConfiguration` and `F` is the type of the `Feature<FC>`.

A `ConfiguredFeature` needs to be [registered dynamically][dynamicregistration] either by data generation or directly writing the JSON. The JSONs are located at `data/<mod_id>/worldgen/configured_feature/<path>.json`.

```java
// Define keys for datapack registry objects

// An example using a custom feature
public static final ResourceKey<ConfiguredFeature<?, ?>> EXAMPLE_CONFIGURED_FEATURE =
    ResourceKey.create(
        Registries.CONFIGURED_FEATURE, // The registry this key is for
        ResourceLocation.fromNamespaceAndPath(MOD_ID, "example") // The registry name
    );
// An example using a vanilla feature
public static final ResourceKey<ConfiguredFeature<?, ?>> EXAMPLE_ORE_CONFIGURED_FEATURE =
    ResourceKey.create(
        Registries.CONFIGURED_FEATURE,
        ResourceLocation.fromNamespaceAndPath(MOD_ID, "example_ore")
    );

// For some RegistrySetBuilder BUILDER
//   being passed to DatapackBuiltinEntriesProvider
//   in a listener for GatherDataEvent
BUILDER.add(Registries.CONFIGURED_FEATURE, bootstrap -> {
    // Lookup any necessary registries
    // Static registries only need to be looked up if you need to grab the tag data

    // Register the configured features

    bootstrap.register(EXAMPLE_CONFIGURED_FEATURE, new ConfiguredFeature<>(
        EXAMPLE_FEATURE.value(), // The feature being configured
        // The configuration
        new ExampleConfiguration(
            3,
            Blocks.DIAMOND_BLOCK.defaultBlockState()
        )
    ));

    boostrap.register(EXAMPLE_ORE_CONFIGURED_FEATURE, new ConfiguredFeature<>(
        Feature.ORE,
        new OreConfiguration(
            // A test which returns true if the block can be replaced with ore
            new TagMatchTest(BlockTags.STONE_ORE_REPLACEABLES),
            // The ore block
            Blocks.DIAMOND_BLOCK.defaultBlockState(),
            // The size of the vein
            6
        )
    ));
})
```

```json5
// In data/examplemod/worldgen/configured_feature/example.json
{
    "type": "examplemod:example",
    "config": {
        "height": 3,
        "state": {
            "Name": "minecraft:diamond_block"
        }
    }
}

// In data/examplemod/worldgen/configured_feature/example_ore.json
{
    "type": "minecraft:ore",
    "config": {
        "discard_chance_on_air_exposure": 0.0,
        "size": 6,
        "targets": [
            {
                "state": {
                    "Name": "minecraft:diamond_block"
                },
                "target": {
                    "predicate_type": "minecraft:tag_match",
                    "tag": "minecraft:stone_ore_replaceables"
                }
            }
        ]
    }
}
```

See the [Minecraft Wiki][configuredwiki] for a list of vanilla configured features.

## Placing a Feature

If a [configured feature][configuredfeature] was added as-is to a level, then every feature would generate in the exact same location within each chunk once. To vary the location as well as how many times a configured feature is generated within a chunk, a configured feature is given a list of `PlacementModifier`s to modify the starting placement. A configured feature with a list of placements is known as a `PlacedFeature`, and this is what is provided to the chunk generator to generate the feature.

### Placement Modifiers

A `PlacementModifier` subclass contains two methods: `#getPositions`, which returns a stream of `BlockPos` representing the starting locations of the `ConfiguredFeature`; and `#type`, which returns a `PlacementModifierType`: a `MapCodec` that encodes and decodes the `PlacementModifier`.

`getPositions` has three parameters: the `PlacementContext` that contains the level this feature will be generated within, the generator of the chunk, and the current feature (or parent if this feature is generated via another feature) being generated; a random instance, and the starting position of the feature. The method returns a stream of `BlockPos` that will indicate all starting positions for the feature in this chunk.

The `PlacementModifierType` must be [statically registered][staticregistration].

```java
public class ExamplePlacementModifier extends PlacementModifier {

    public static final MapCodec<ExamplePlacementModifier> CODEC =
        ExtraCodecs.POSITIVE_INT.fieldOf("chance")
            .xmap(ExamplePlacementModifier::new, ExamplePlacementModifier::getChance);

    public ExamplePlacementModifier(int chance) {
        // ...
    }

    @Override
    public Stream<BlockPos> getPositions(PlacementContext ctx, RandomSource random, BlockPos pos) {
        return random.nextFloat() < 1f / (float) (this.chance + pos.getY())
            ? Stream.of(pos)
            : Stream.of();
    }

    @Override
    public PlacementModifierType<?> type() {
        return EXAMPLE_PLACEMENT_MODIFIER_TYPE.value();
    }
}

// In some class where you register your registry objects
public static final DeferredRegister<PlacementModifierType<?>> PLACEMENT_MODIFIER_TYPES =
    DeferredRegister.create(Registries.PLACEMENT_MODIFIER_TYPE, MOD_ID);

public static final DeferredHolder<
        PlacementModifierType<?>,
        PlacementModifierType<ExamplePlacementModifier>
    > EXAMPLE_PLACEMENT_MODIFIER_TYPE =
        FEATPLACEMENT_MODIFIER_TYPESURES.register("example", () -> () -> CODEC);
```

See the [Minecraft Wiki][placedwiki] for a list of placement modifiers.

### Generating the `PlacedFeature`

A `PlacedFeature` is a combination of a `ConfiguredFeature` and a list of `PlacementModifier`s. The `PlacementModifier`s are applied in order to some initial position to generate all starting locations for the `ConfiguredFeature` to generate. As such, the order of the placement modifiers affect how a feature will generate.

For example, let's say I have the following two features:

```json
[
    {
        // Adds up to 40 instances of this feature
        "type": "minecraft:count",
        "count": 40
    },
    {
        // Spreads this feature randomly horizontally within a 16x16 area
        "type": "minecraft:in_square"
    }
]
```

In the current order, the initial position will be duplicated up to 40 times and then offset randomly, creating up to 40 unique locations for the feature to generate.

If the order of the placements were flipped, then the initial position will be offset randomly and then duplicated up to 40 times, creating one unique location for the feature to generate.

As such, there are generally three phases of placements to apply: the initial count, random spread, and post filters.

The initial count phase determines how many initial starting positions should there be for a given chunk. For example, a `CountPlacement` is used to duplicate the initial starting position some number of times randomly. On the other side, a `RarityFilter` that is used to set some chance of this feature spawning in a chunk.

The random spread phase takes all initial positions and diversifies their location. `InSquarePlacement` randomly offsets the horizontal position within a 16x16 area. `HeightRangePlacement` or `HeightmapPlacement` can offset the vertical position. `RandomOffsetPlacement` can do both at the same time!

Finally, the post filter phase applies any final modifications to the starting locations that may be invalid due to the random spread phase. `BiomeFilter` removes any starting locations that are not in a `Biome` the feature can generate in. `BlockPredicateFilter` removes any starting locations where the current block does not pass the given predicate.

:::note
These phases are just a suggestion. You can apply placements in any order; jsut be aware that the interaction between placements may produce unintended side effects.
:::


A `PlacedFeature` needs to be [registered dynamically][dynamicregistration] either by data generation or directly writing the JSON. The JSONs are located at `data/<mod_id>/worldgen/placed_feature/<path>.json`.

```java
// Define keys for datapack registry objects

public static final ResourceKey<PlacedFeature> EXAMPLE_PLACED_FEATURE =
    ResourceKey.create(
        Registries.PLACED_FEATURE,
        ResourceLocation.fromNamespaceAndPath(MOD_ID, "example")
    );
public static final ResourceKey<PlacedFeature> EXAMPLE_ORE_PLACED_FEATURE =
    ResourceKey.create(
        Registries.PLACED_FEATURE,
        ResourceLocation.fromNamespaceAndPath(MOD_ID, "example_ore")
    );

// For some RegistrySetBuilder BUILDER
//   being passed to DatapackBuiltinEntriesProvider
//   in a listener for GatherDataEvent
BUILDER.add(Registries.PLACED_FEATURE, bootstrap -> {
    // Lookup any necessary registries
    // Static registries only need to be looked up if you need to grab the tag data
    HolderGetter<ConfiguredFeature<?, ?>> configuredFeatures = bootstrap.lookup(
        Registries.CONFIGURED_FEATURE
    );

    // Register the placed features

    bootstrap.register(EXAMPLE_PLACED_FEATURE, new PlacedFeature(
        // The configured feature to place
        configuredFeatures.getOrThrow(EXAMPLE_CONFIGURED_FEATURE),
        // A list of placement modifiers
        List.of(
            // Initial Count Phase
            new ExamplePlacementModifier(5),

            // Random Spread Phase
            RandomOffsetPlacement.of(
                // Horizontal Spread
                ConstantInt.of(10),
                // Vertical Spread
                UniformInt.of(-5, 10)
            ),

            // Post Filter Phase
            BiomeFilter.biome()
        )
    ));

    boostrap.register(EXAMPLE_ORE_PLACED_FEATURE, new PlacedFeature(
        configuredFeatures.getOrThrow(EXAMPLE_ORE_CONFIGURED_FEATURE),
        List.of(
            // Initial Count Phase
            CountPlacement.of(90),

            // Random Spread Phase

            // 16x16 horizontal spread
            InSquarePlacement.spread(),

            // Randomly selects a height between the two values using two randomizations
            // - The first randomizes a value between the minimum and plateau value
            // - The second randomizes a value between the plateau and maximum value
            // - The two values are added to the minimum height to get the final Y
            HeightRangePlacement.triangle(
                // Minimum height
                // - 80 below the minimum generation height (-144 for overworld)
                VerticalAnchor.aboveBottom(-80),
                // Maximum Height
                // - 80 above the minimum generation height (16 for overworld)
                VerticalAnchor.aboveBottom(80)
            ),

            // Post Filter Phase
            BiomeFilter.biome()
        )
    ));
})
```

```json5
// In data/examplemod/worldgen/placed_feature/example.json
{
    "feature": "examplemod:example",
    "placement": [
        {
            "type": "examplemod:example",
            "chance": 5
        },
        {
            "type": "minecraft:random_offset",
            "xz_spread": 10,
            "y_spread": {
                "type": "minecraft:uniform",
                "max_inclusive": 10,
                "min_inclusive": -5
            }
        },
        {
            "type": "minecraft:biome"
        }
    ]
}

// In data/examplemod/worldgen/placed_feature/example_ore.json
{
    "feature": "examplemod:example_ore",
    "placement": [
        {
            "type": "minecraft:count",
            "count": 90
        },
        {
            "type": "minecraft:in_square"
        },
        {
            "type": "minecraft:height_range",
            "height": {
                "type": "minecraft:trapezoid",
                "max_inclusive": {
                    "above_bottom": 80
                },
                "min_inclusive": {
                    "above_bottom": -80
                }
            }
        },
        {
            "type": "minecraft:biome"
        }
    ]
}
```

## Adding a Feature to a Biome

[Placed features][placedfeature] are generated within a chunk by adding it to a list for the associated `GenerationStep.Decoration` in a biome. Each feature list within a `GenerationStep.Decoration` is generated in oridinal order (e.g., features added to the `UNDERGROUND_ORES` decoration will be generated before `VEGETAL_DECORATIONS`). The following is the list of decorations in the order they are generated:

| `GenerationStep.Decoration` | Description                                                                    |
|:---------------------------:|:-------------------------------------------------------------------------------|
| `RAW_GENERATION`            | For blobs of terrain (e.g., small end islands)                                 |
| `LAKES`                     | For bodies of liquid (e.g., lava lakes)                                        |
| `LOCAL_MODIFICATIONS`       | For miscellaneous terrain modifications (e.g., icebergs)                       |
| `UNDERGROUND_STRUCTURES`    | For underground structures (e.g., monster rooms)                               |
| `SURFACE_STRUCTURES`        | For structures above the surface (e.g., desert wells)                          |
| `STRONGHOLDS`               | Only for strongholds, but goes unused                                          |
| `UNDERGROUND_ORES`          | Blobs of ore underground (e.g., iron, clay disks)                              |
| `UNDERGROUND_DECORATION`    | Additional underground decorations (e.g., infested block blobs, nether gravel) |
| `FLUID_SPRINGS`             | Single liquid units (e.g., water springs)                                      |
| `VEGETAL_DECORATION`        | Above surface vegetation (e.g., trees, bamboo)                                 |
| `TOP_LAYER_MODIFICATION`    | Post surface modifications (e.g., freeze top layer)                            |

:::note
While you can choose the decoration step your feature should go within via their names, it is not a requirement. All you need to be aware of is that features further down the list will generate using the state of the level with any prior features already generated.
:::

### Your Modded Biomes

If you are the creator of the biome, then you can add the feature using the [`BiomeGenerationSettings.Builder`][biomebuilder] by data generation or directly writing the JSON. The JSONs are located at `data/<mod_id>/worldgen/biome/<path>.json`.

```java
// Define keys for datapack registry objects

public static final ResourceKey<Biome> EXAMPLE_BIOME =
    ResourceKey.create(
        Registries.BIOME, // The registry this key is for
        ResourceLocation.fromNamespaceAndPath(MOD_ID, "example") // The registry name
    );

// For some RegistrySetBuilder BUILDER
//   being passed to DatapackBuiltinEntriesProvider
//   in a listener for GatherDataEvent
BUILDER.add(Registries.BIOME, bootstrap -> {
    // Lookup any necessary registries
    // Static registries only need to be looked up if you need to grab the tag data
    HolderGetter<PlacedFeature> placedFeatures = bootstrap.lookup(Registries.PLACED_FEATURE);
    HolderGetter<ConfiguredWorldCarver<?>> configuredWorldCarvers = bootstrap.lookup(Registries.CONFIGURED_CARVER);

    // Register the biomes

    bootstrap.register(EXAMPLE_BIOME,
        new Biome.BiomeBuilder().generationSettings(
            new BiomeGenerationSettings.Builder(
                placedFeatures, configuredWorldCarvers
            ).addFeature( // Add features in generation step
                GenerationStep.Decoration.LOCAL_MODIFICATIONS, EXAMPLE_PLACED_FEATURE
            ).addFeature(
                GenerationStep.Decoration.UNDERGROUND_ORES, EXAMPLE_ORE_PLACED_FEATURE
            )
        ).temperature(2f).downfall(0f)
        .specialEffects(new BiomeSpecialEffects.Builder()
            .fogColor(0).waterColor(0).waterFogColor(0).skyColor(0).build()
        ).mobSpawnSettings(new MobSpawnSettings.Builder().build())
        .build()
    );
})
```

```json
// In data/examplemod/worldgen/biome/example.json
{
  "carvers": {},
  "downfall": 0.0,
  "effects": {
    "fog_color": 0,
    "sky_color": 0,
    "water_color": 0,
    "water_fog_color": 0
  },
  "features": [
    // RAW_GENERATION
    [],
    // LAKES
    [],
    // LOCAL_MODIFICATIONS
    [
        "examplemod:example"
    ],
    // UNDERGROUND_STRUCTURES
    [],
    // SURFACE_STRUCTURES
    [],
    // STRONGHOLDS
    [],
    // UNDERGROUND_ORES
    [
        "examplemod:example_ore"
    ],
    // UNDERGROUND_DECORATION
    [],
    // FLUID_SPRINGS
    [],
    // VEGETAL_DECORATION
    [],
    // TOP_LAYER_MODIFICATION
    []
  ],
  "has_precipitation": true,
  "spawn_costs": {},
  "spawners": {},
  "temperature": 2.0
}
```

### Biome Modifiers

If you are **not** the creator of the biome, then you can add the feature using [biome modifiers][biomemodifiers]. The simplest way to do so is to use `AddFeaturesBiomeModifier`. However, if more complex logic is needed, a custom biome modifier can be created.

`AddFeaturesBiomeModifier` can be created either by data generation or directly writing the JSON. The JSONs are located at `data/<mod_id>/neoforge/biome_modifier/<path>.json`.

```java
// Define keys for datapack registry objects

public static final ResourceKey<Biome> EXAMPLE_BIOME_MODIFIER =
    ResourceKey.create(
        NeoForgeRegistries.Keys.BIOME_MODIFIERS, // The registry this key is for
        ResourceLocation.fromNamespaceAndPath(MOD_ID, "example") // The registry name
    );
public static final ResourceKey<Biome> EXAMPLE_ORE_BIOME_MODIFIER =
    ResourceKey.create(
        NeoForgeRegistries.Keys.BIOME_MODIFIERS, // The registry this key is for
        ResourceLocation.fromNamespaceAndPath(MOD_ID, "example_ore") // The registry name
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

    bootstrap.register(EXAMPLE_BIOME_MODIFIER,
        new AddFeaturesBiomeModifier(
            // A single biome to generate within
            HolderSet.direct(biomes.getOrThrow(Biomes.PLAINS)),
            // A single feature to generate within the biomes
            HolderSet.direct(placedFeatures.getOrThrow(EXAMPLE_PLACED_FEATURE)),
            // The generation step
            GenerationStep.Decoration.LOCAL_MODIFICATIONS
        )
    );

    bootstrap.register(EXAMPLE_ORE_BIOME_MODIFIER,
        new AddFeaturesBiomeModifier(
            // The biomes this feature can generate within
            biomes.getOrThrow(Tags.Biomes.IS_OVERWORLD),
            HolderSet.direct(placedFeatures.getOrThrow(EXAMPLE_ORE_PLACED_FEATURE)),
            GenerationStep.Decoration.UNDERGROUND_ORES
        )
    );
})
```

```json5
// In data/examplemod/neoforge/biome_modifier/example.json
{
  "type": "neoforge:add_features",
  "biomes": "minecraft:plains",
  "features": "examplemod:example",
  "step": "local_modifications"
}

// In data/examplemod/neoforge/biome_modifier/example_ore.json
{
  "type": "neoforge:add_features",
  "biomes": "#c:is_overworld",
  "features": "examplemod:example_ore",
  "step": "underground_ores"
}
```


[codec]: ../datastorage/codecs.md
[staticregistration]: ../concepts/registries.md#methods-for-registering
[feature]: #creating-a-feature
[dynamicregistration]: ../concepts/registries.md#data-generation-for-datapack-registries
[configuredwiki]: https://minecraft.wiki/w/Configured_feature#Feature_types
[configuredfeature]: #configuring-a-feature
[placedwiki]: https://minecraft.wiki/w/Placed_feature#Placement_Modifiers
[placedfeature]: #placing-a-feature
<!--TODO: Add correct link-->
[biomebuilder]: #
[biomemodifiers]: ./biomes/biomemodifiers.md
