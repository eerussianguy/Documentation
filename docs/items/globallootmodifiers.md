Global Loot Modifiers
===========

Global Loot Modifiers are a data-driven method of handling modification of harvested drops without the need to overwrite dozens to hundreds of vanilla loot tables or to handle effects that would require interactions with another mod's loot tables without knowing what mods may be loaded. Global Loot Modifiers are also stacking, rather than last-load-wins, similar to tags.

Registering a Global Loot Modifier
-------------------------------

You will need 4 things:
1. Create a `global_loot_modifiers.json` file at `/data/forge/loot_modifiers/`.
    This will tell Forge about your modifiers and works similar to [tags][].
2. A serialized json representing your modifier.
    This will contain all of the data about your modification and allows data packs to tweak your effect.
3. A class that extends `IGlobalLootModifier`.
    The operational code that makes your modifier work. Most modders can extend `LootModifier` as it supplies base functionality.
4. Finally, the serializer for your operational class.
    This is [registered] as any other `IForgeRegistryEntry`.

The `global_loot_modifiers.json`
-------------------------------

All you need to add here are the file names of your loot modifiers. The [ResourceLocation][resloc] specified points to `data/<namespace>/loot_modifiers/<path>.json`.

```json
{
  "replace": false,
  "entries": [
    "global_loot_test:silk_touch_bamboo",
    "global_loot_test:smelting",
    "global_loot_test:wheat_harvest"
  ]
}
```

`replace` causes the cache of modifiers to be cleared fully when this asset loads (mods are loaded in an order that may be specified by a data pack). For modders, you will want to use `false`. Data pack makers may want to specify their overrides with `true`.

`entries` is an *ordered list* of the modifiers that will be loaded. This means that any modifier that not listed will not be loaded, and the ones listed are their written order. This is primarily relevant to data pack makers for resolving conflicts between modifiers from separate mods.

The Serialized JSON
-------------------------------

This file contains all of the potential variables related to your modifier, including the conditions that must be met prior to modifying any loot. Avoid hard-coded values wherever possible so that data pack makers can adjust balance if they wish to.
```json
{
  "conditions": [
    {
      "condition": "minecraft:match_tool",
      "predicate": {
        "item": "minecraft:shears"
      }
    },
    {
      "condition": "block_state_property",
      "block":"minecraft:wheat"
    }
  ],
  "seedItem": "minecraft:wheat_seeds",
  "numSeeds": 3,
  "replacement": "minecraft:wheat"
}
```

In the above example, the modification only happens if an entity harvests wheat when using shears (specified by the two `conditions` which are automatically `AND`ed together). The `seedsItem` and `numSeeds` values are then used to count how many seeds were generated by the vanilla loot table, and if matched, are substituted for an additional `replacement` item instead. The operation code will be shown below.
`conditions` is the only object needed by the system specification, everything else is the mod maker's data.

The `LootModifier` Subclass and Serializer
-------------------------------

You will also need a class that extends `GlobalLootModifierSerializer<T>` where `T` is your `LootModifier` subclass in order to deserialize your json data file into operational code.

```java
public class WheatSeedsConverterModifier extends LootModifier {
    private final int numSeedsToConvert;
    private final Item itemToCheck;
    private final Item itemReward;
    public WheatSeedsConverterModifier(LootItemCondition[] conditions, int numSeeds, Item itemCheck, Item reward) {
        super(conditions);
        numSeedsToConvert = numSeeds;
        itemToCheck = itemCheck;
        itemReward = reward;
    }

    @Nonnull
    @Override
    public List<ItemStack> doApply(List<ItemStack> generatedLoot, LootContext context) {
        //
        // Additional conditions can be checked, though as much as possible should be parameterized via JSON data.
        // It is better to write a new ILootCondition implementation than to do things here.
        //
        int numSeeds = 0;
        for (ItemStack stack : generatedLoot) {
            if (stack.getItem() == itemToCheck)
                numSeeds += stack.getCount();
        }
        if (numSeeds >= numSeedsToConvert) {
            generatedLoot.removeIf(x -> x.getItem() == itemToCheck);
            generatedLoot.add(new ItemStack(itemReward, (numSeeds / numSeedsToConvert)));
            numSeeds = numSeeds % numSeedsToConvert;
            if (numSeeds > 0)
                generatedLoot.add(new ItemStack(itemToCheck, numSeeds));
        }
        return generatedLoot;
    }

    public static class Serializer extends GlobalLootModifierSerializer<WheatSeedsConverterModifier> {

        @Override
        public WheatSeedsConverterModifier read(ResourceLocation name, JsonObject object, LootItemCondition[] conditions) {
            int numSeeds = GsonHelper.getAsInt(object, "numSeeds");
            Item seed = ForgeRegistries.ITEMS.getValue(new ResourceLocation((GsonHelper.getAsString(object, "seedItem"))));
            Item wheat = ForgeRegistries.ITEMS.getValue(new ResourceLocation(GsonHelper.getAsString(object, "replacement")));
            return new WheatSeedsConverterModifier(conditions, numSeeds, seed, wheat);
        }

        @Override
        public JsonObject write(WheatSeedsConverterModifier instance) {
            JsonObject json = makeConditions(instance.conditions);
            json.addProperty("numSeeds", instance.numSeedsToConvert);
            json.addProperty("seedItem", ForgeRegistries.ITEMS.getKey(instance.itemToCheck).toString());
            json.addProperty("replacement", ForgeRegistries.ITEMS.getKey(instance.itemReward).toString());
            return json;
        }
    }
}
```

The critical portion is the `doApply` method.

This method is only called if the `conditions` specified return `true`. If so, the modder is now able to make the modifications they desire. In this case, we can see that the number of `itemToCheck` meets or exceeds the `numSeedsToConvert` before modifying the list by adding an `itemReward` and removing any excess `itemToCheck` stacks, matching the previously mentioned effects: *When a wheat block is harvested with shears, if enough seeds are generated as loot, they are converted to additional wheat instead*.

Also take note of the `read` method in the serializer. The conditions are already deserialized for you and if you have no other data, simply `return new MyModifier(conditions)`. However, the full `JsonObject` is available if needed. The `write` method, on the other hand, is used for if you want to utilize `GlobalLootModifierProvider` for [data generation][datagen].

Additional [examples] can be found on the Forge Git repository, including silk touch and smelting effects.

[tags]: ../utilities/tags.md
[resloc]: ../concepts/resources.md#ResourceLocation
[registered]: ../concepts/registries.md#registering-things
[datagen]: ../datagen/intro.md
[examples]: https://github.com/MinecraftForge/MinecraftForge/blob/1.17.x/src/test/java/net/minecraftforge/debug/gameplay/loot/GlobalLootModifiersTest.java
