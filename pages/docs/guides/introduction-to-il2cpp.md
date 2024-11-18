# Introduction to IL2CPP

Before Gang Beasts v1.22, we used to be able to do a lot more with modding, such as full Harmony support (explained later on), out-of-the-box importing of more Unity packages, accurate source decompilation of code using DnSpy, etc... This is because the game used to use Unity's Mono backend for code compilation. This backend would compile C# code to lower level instruction code known as CIL, or simply IL, which stands for Common Intermediate Language or just Intermediate Language. These instructions were changeable at runtime, which made modding a breeze using patching tools like MonoMod and Harmony, and decompilation even easier.  
Unfortunately, once Gang Beasts began receiving updates again in 2023, Boneloaf (the developers of Gang Beasts) switched the game over to Unity's IL2CPP backend for performance and compatibility reasons. This adds an extra step to compilation, converting the IL to even lower level C++ code (which then compiles to machine code), making the code potentially faster but also much harder to decompile and modify.  
Thanks to MelonLoader and its libraries, however, its still possible to modify the game almost the same way we could before, just with a few janky bits.

> [!TIP]
> Further reading about modding with C# in a Unity IL2CPP domain is covered on the [MelonLoader wiki](https://melonwiki.xyz) and [Il2CppInterop docs](https://github.com/BepInEx/Il2CppInterop/tree/master/Documentation). It can also be helpful to ask in community chats such as our [Cement Discord](https://discord.gg/fCwXc5k43w), the [MelonLoader Discord](https://discord.gg/2Wn3N2P), or the [BepInEx](https://discord.gg/MpFEDAg) Discord.

## Notable IL2CPP Differences

> [!NOTE]
> This section explains in detail some **intermediate**
 concepts a beginner may not fully understand or require. To get straight into making your first mod, jump to [Getting Started](getting-started.md).

### Harmony

> [!IMPORTANT]
> Harmony is explained in full in [the Harmony docs](https://harmony.pardeike.net/). Note that MelonLoader uses a fork of Harmony called [HarmonyX](https://github.com/BepInEx/HarmonyX/wiki) that has a small difference in workflow, however the original Harmony docs should still be relevant.
> For ease of explanation, we highly recommend you read these docs for more information.

When you work with Harmony in IL2CPP, you're not able to manipulate the runtime code (IL) of the game like in Mono. Instead, you're basically hooking into codeless generated "dummy" assemblies that only contains the method signature for the original native method.  
What this means is Harmony's "transpilers" are no longer possible entirely, as there isn't any actual instructions to patch. You can only patch a method using a prefix or a postfix. The recommended way of creating a Harmony patch is explained as follows:
> [!IMPORTANT]
> Make sure to read the comments.

```csharp
using System;
using HarmonyLib;
using MelonLoader;

namespace MyFirstMod;

internal static class VanillaTypePatches // It is recommended to follow these naming conventions (PascalCase + Vanilla type name + "Patches" at the end)
{
    [HarmonyPatch(typeof(VanillaType), nameof(VanillaType.VanillaMethod))] // Replace VanillaType and VanillaMethod with the type and method you want to patch
    private static class VanillaMethodPatch // This is a class because you can put both a prefix and a postfix on the same method (also recommended to follow the same naming conventions, this time with vanilla method name and non-plural "Patch")
    {
        /* Called just before VanillaMethod. Returns a bool to decide whether to run the original method or not: false = skip original method, true = don't skip. Can also be void if you want it to always run the original method.
        Try to prefer using Postfix as this can prevent other patches from running. MUST BE CALLED "Prefix" OR HAVE THE [HarmonyPrefix] ANNOTATION! */
        private static bool Prefix()
        {
            // Can be anything
            return true;
        }

        private static void Postfix() // Called after VanillaMethod. Recommended for most patching cases to ensure mod compatibility. MUST BE CALLED "Postfix" OR HAVE THE [HarmonyPostfix] ANNOTATION!
        {
            // Can be anything
        
            Melon<Core>.Logger.Msg(ConsoleColor.Green, "Patch worked!") // This will be called after VanillaMethod is finished but before it returns a value, allowing you to modify said value if it exists (explained later)
        }
    }
}
```

> [!TODO]
> Provide IL2CPP-specific info about [Harmony injections](https://harmony.pardeike.net/articles/patching-injections.html)

> [!WARNING]
> IL2CPP Harmony patches do not work well for constructors OR generics. Do not be fooled by `MethodType.Constructor`!

### Unity-Serialized Fields

> [!TIP]
> Some useful information about how this system works in Unity itself can be found in the [Unity docs](https://docs.unity.com/), starting from the [`SerializeField` attribute documentation](https://docs.unity3d.com/ScriptReference/SerializeField.html).

Explanations for this in modding are hard to come by, but we'll try our best to summarize. Basically, Unity's serialized `MonoBehaviour` fields (such as non-hidden public fields and private fields with the `SerializeField` attribute) are saved in a separate object associated with that script's `GameObject` and `Assembly`. In Mono, it was possible, without any extra effort, to make custom scripts inside the Unity Editor with these serialized fields and later inject the object the script is attached to via [`AssetBundle`](https://docs.unity3d.com/ScriptReference/AssetBundle.html) into the game with all (potentially modified) editor-assigned fields preserved. With IL2CPP this becomes slightly harder.

> [!TIP]
> The following concepts are taken from [this Il2CppInterop pull request](https://github.com/BepInEx/Il2CppInterop/pull/24) and further explained.
