+++
title = "The road to the perfect plugin system"
author = "D3lta-2-1"
date = "2023-12-29"

categories = ["dev logs"]
tags = ["plugins", "core", "research"]

description = "Exploring Modding possibilities and finding the ideal solution."

[banner]
image = "plugin.jpg"
caption = "Plugins are like legos, there are modular and allow creativity"
+++

Minecraft Java Edition became very popular thanks to its modding community, and the ability to create your own server, by customizing with plugins and mods.

For example, there are more than 144.5K mods to this date on CurseForge, more than 8K plugin on Modrinth, and this number is growing every day.

<!--more-->

# What is a plugin/mod for Archipel?

Modding is a big (maybe the biggest) part of Minecraft, so we can't ignore it. Some recent Minecraft projects like Minestom
are just a mod loaders, where the base game is also considered as mod!

Following Minestom choices, we want all content of our software to be a mod/plugin, leaving only core features in the server itself.
Even Minecraft Vanilla features will be considered as a plugin. This way, you can always choose to keep only the parts you are interested in.

> _But what is a plugin?_

A plugin is a piece of code that can be loaded to modify the program behavior, without a full recompilation or modification of the server binaries.
Plugins are the solutions we provide to developers to modify and improve the game. Allowing them to add new blocks or items, imagine new dimensions
and challenge their players with creative mechanics.

# What is a good plugin system?

To meet our needs, a good plugin system would be:

- **fast**: too many mods should not slow down the server. Ideally, it would safely handle multi-threading.
- **sand-boxed**: a plugin should not makes the server crash or access data on the host system (like sensitive files).
- **easy to use**: a plugin should be easy to create and maintain (for the developer), and easy to use (for the user).
- **cross-platform**: it has to work everywhere the server works, without recompilation, like Minecraft does with Java.
  You can run the same mod independently on Windows, Linux and Mac, even on x86 and ARM.
- **stable ABI** (Application Binary Interface): this is a must-have, plugins should be able to be compiled with different compilers (or version of a compiler)
  and keep the same output without breaking the ABI.
- **symbols importation between plugins**: have a plugin is nice, but being able to interact with other plugins is even better. Plugins should be able
  to import symbols from other plugins, like [loom](https://fabricmc.net/wiki/documentation:fabric_loom)'s `modimplementation`.
  Core mods align with the philosophy of the fabric modding community. They aim to avoid bloating the server core with excessive features, reducing the
  effort required for updates between versions. Additionally, core mods can optimize memory usage by limiting the number of loaded binaries in RAM.

Obviously, it is going to be hard, if not impossible to find a solution that meets all these criteria, we might even have to make some compromises...

In the following sections, we will explore some solutions we have considered, and why we didn't choose them.

# First thoughts: Lua/wren

Like every other plugin system, we thought about using lua. It's a very popular language that is used in many games and is very easy to embed into C/C++ applications.

The 2 mosts popular implementations are [LuaJit](https://luajit.org/) and [Wren](https://wren.io/). Wren is a little slower, but solves many problems from Lua, like the lack of classes or static typings.

Both are very lightweight and easy to use, sandbox-able but, they suffer from a big problem: they don't support any kind of multi-threading. Developers are forced to working inside a single thread or
to work with multiples VMs in parallel (Not the smartest idea, right?). This means that this is not an idea solution, since user can always store data on a specific VM, but
will not be able to access it from another.

To resume:

| Criteria                      | Note | Summary                                                                                                                                                          |
| ----------------------------- | :--: | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Fast                          | 2/5  | Lacks multithreading.                                                                                                                                            |
| Sand-boxed                    | Yes  | Both provides sand-boxing capabilities.                                                                                                                          |
| Easy to use                   | 3/5  | Looks easy to use, but might quickly become messy on large projects.                                                                                             |
| Cross-platform                | Yes  | Can run literally everywhere.                                                                                                                                    |
| Stable ABI                    | Yes  | _Nothing to comment_                                                                                                                                             |
| Cross-plugins symbols imports | 3/5  | Interactions would be very tricky to implement, since Lua imports means executing it. Getting a function from another plugin would always result in a core call. |

Why did I look for alternatives? Because Lua/wren are too primitive for our use case, and we can't rely on 25k lines of code written with it just to ensure everything run smootly.
Others points like interactions with an ECS are very tricky to implement: a component is basically a static-sized and statically typed structure, and this notion doesn't exists at all in these
languages due to their dynamic typing nature.

# all try to fit rust in a plugin system, using dylib and laziness, I mean Rust ABI

Bevy uses this approach, and it's very easy to use, you just have to add a dependency to your Cargo.toml, and you can use it like any other rust library.
The core of the project will be written in rust, and having a homogenous project looks like very conformable, no need to learn a new language, no need to learn a new build system, no need to learn a new IDE, etc.
Compiled rust for native platform is the fastest option, we can also use multithreading, and any other more advanced features of rust. But we will lose some keypoint here: -**fast: 5/5**, [AOT](https://en.wikipedia.org/wiki/Ahead-of-time_compilation) is the fastest way to execute code, we are talking about 2x to 10x the speed of any other alternative. -**sand-boxed: no**, a DLL get the same level of permission as the server, it can access any file, it can crash the server, it can do anything the server can do, which is not ideal -**easy to use: 4/5**, rust is not the easiest language to learn, but it's not the hardest either, and it's very well documented, and will fit perfectly with the rest of the codebase! -**cross-platform: 1/5**, rust code is cross-platform, and can run on almost any platform, but need to be recompiled for each platform, and we can't use the same binary on different platform -**stable ABI: 0/5**, rust ABI is not stable, changing the flag of the compiler can change the padding of a structure, and break the ABI, plugin would need to be compiled with the same compiler, the same version of the compiler and the same flags, which is not ideal -**able to import symbols from other plugins: 2/5**, the nightmare start from here, importing symbols from other plugin isn't really a thing, the "host plugin" must expose an array of static void pointers to every desired functions, and the "guest plugin" must cast the void pointer to the desired function pointer, and call it. This is not ideal, and will result in a lot of boilerplate code, and a lot of unsafe code.

# Rust the 2nd round, using C ABI, a bit better but still far from perfect

This solution is really similar to the precedent, but using C abi instead of Rust ABI. This will solve problems of ABI stability, like being constrained to use the same compiler, the same version of the compiler and the same flags. But this come with a trade-off, more boilerplate code, and more unsafe code. to map all to a C ABI, we will need to use a lot of unsafe code, and we will need to use a lot of boilerplate code to map rust types to C types, and to map rust functions to C functions. We will also need to use a lot of unsafe code to call the C functions, and to cast the C types to rust types. This is not ideal, but it's the best we can do with C ABI. We can't even use a trait to abstract the function pointer, because the trait will be defined in the "host plugin", and the "guest plugin" will not be able to implement it. -**fast: 5/5**, same as the precedent -**sand-boxed: no**, same as the precedent -**easy to use: 3/5**, same as the precedent, but with more boilerplate code -**cross-platform: 2/5**, a bit better than the precedent, but we can still forget about using the same binary on different OS/cpu architectures -**stable ABI: 5/5**, we are using C ABI/FFI, which is perfectly stable -**able to import symbols from other plugins: 1/5** even worse than the precedent, we can't even use a trait to abstract the function pointer, because the trait will be defined in the "host plugin", and the "guest plugin" will not be able to implement it.

Both of these solutions allow to get a reference over any object passed by the core, It means we can iterate over components of our ECS without any copy! This is a very good point, and will allow us to write very efficient plugins.

# Rust the 3rd round, using WASM, a bit slower but good enough, or not ?

Webassembly is a new kind of bytecode, designed to be close of native performances but way more portability, it can be compared to JAVA or C# bytecode, but more minimalistic. Wasm run in a browser when JavaScrip is too slow, or run with a dedicated runtime like Wastime or Wasmer.
Wasm have very cool trade-off, its speed is between native speed and Luajit (the fastest Jit language), easy to use, the easiest to sand_box (it's designed for that), very cross-platform, stable ABI, and can import symbols from other plugins.
calling a function in Wasm in very easy, at least with primitive... concurrently there is no offical way to pass complexe object from the host to the guest, but there is a hacky way to pass a pointer to the guest, and the guest can read/write to this pointer. This is not ideal, but it's the best we can do with wasm.
Passing a complex object require some serialisation, which come with a big performance cost, and a lot of boilerplate code. It also requires a lot of handmade VM memory management, and mean for us, reimplementing the whole ECS in WasmTime VM memory, or copying each component to the vm memory during entity query, which also come with a cost.

-**fast: 2/5**, Wasm is fast, but the cost of serialisation and handmade VM memory management is very high -**sand-boxed: yes**, Wasm is designed for that -**easy to use: 0/5**, Wasm is very easy to use, but the handmade VM memory management is very tricky to implement (I have tried and almost jumped from the window) -**cross-platform: 5/5**, Wasm is designed for that -**stable ABI: 5/5**, Wasm is designed for that -**able to import symbols from other plugins: 2/5**, concurrently one of the best solutions we have seen so far, but still not perfect, complex object are very tricky to pass

[Feather](https://feathermc.org/) uses this approach, and they failed to implement it, because the heavy sand-boxing and handmade VM memory management is very tricky to implement, and come with a big performance cost. They also failed to implement the ECS in the VM memory, and they are copying each component to the VM memory during entity query, which also come with a big performance cost.

# Talking of bytecode, what about Java

Java (or any alternative running on the JVM like kotlin) is one of the most popular language today, and used to create Minecraft. It provides good speeds, portability, and the targeted community already used it

-**fast: 3/5**, Java is fast, but the OO with deep inheritance is not the best for cache coherency, and the memory management is not the best (Java is hungry for memory) -**sand-boxed: no**, I didn't see any way to sandbox java, because I find big issues for my use case, I didn't dig too much -**easy to use: 4/5**, java is very easy to use and to lean, JNI is also surprisingly easy to use -**cross-platform: 5/5**, java is designed for that -**stable ABI: 4/5**, provide some bytecode versions and securities for that -**able to import symbols from other plugins: 5/5**, allow symbol resolution at runtime, it mean it allow to import symbols from other plugins out of the box, this is the best solution we have seen so far -**bonus**, the Minecraft community already know it!

But java don't have any Native function pointer (each "function container") is a polymorphic object, making a huge overhead for our project.
java don't support "inline classes", "primitives classes" or "structure" (call that like you want), I mean an in stack datastructures with compile time size, easy to put in array, making the ECS iteration very impractical.
The [JEP402](https://openjdk.org/jeps/402) may change that.

# and what about java's little brother made by microsoft, C# ?

C# is a very good language, with a lot of good features, and a very good community, and come with very nice feature, like struct (in stack data object), references (with some kind of mutability control). Like java, C# run a VM, a single binary can be run everywhere while dotNet is available -**fast: 4/5**, C# is really fast, also comme with OO and deep inheritance, but you can also use struct and more functional programming -**sand-boxed: no**, I see some project talking about sand_boxing C#, but I didn't dig too much -**easy to use: 4.5/5**, C# is very easy to use and to learn, and the C# community is very active (-0.5 because I don't like PascalCase on Methods) -**cross-platform: 5/5**, C# can run the same binaries everywhere while dotNet is available (windows, linux, mac, on x86 or arm, arm64 with the anycpu target) -**stable ABI: 4/5**, stable across major version -**able to import symbols from other plugins: 5/5**, allow symbol resolution at runtime, it mean it allow to import symbols from other plugins out of the box, this is the best solution we have seen so far

C# have "unmanaged struct" (struct with a fixed size, and no reference), which is very good for ECS and cache coherency, It looks like C# is the best solution so far,
but C# suffers from the same problem as java, native function pointer are hard to get, but existent, a function can be declared as [UnmanagedCallersOnly] and can be called from a function pointer, but it's not very ergonomic, and require a lot of boilerplate code.
I didn't find any way to use references from `hostfxr`, it may be tricky to iterate over ECS without any copy.
Threading doesn't work out of the box but should be doable with some work!

HostFxr is really minimalistic, it exposes at most 10 functions, we also looked to use Mono like the unity engine, but mono is slowly dying, and the mono community is not very active, mono is still stuck with C# 8.0 while dotNet is already at C# 12.0 !!

At this time we are working to use C# as our language of choice, for modding! We are working on a C# library to make modding easier!
