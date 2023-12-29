+++
title = "The road to the perfect plugin system"
author = "D3lta"
date = "2023-09-27"

:::::description = "We are starting a new project: a Minecraft server written in Rust!"

[banner]
image = "plugin.jpg"
caption = "random picture to illustrate the post"
+++

Minecraft's java edition became so popular thanks to its modding community, and the ability to create your own server, and to customize it with plugins.
On curse Forge, there is more than 144.5K mods at this date, and this number is growing every day.
More than 8K plugin on modrinth. Modding is a big part of Minecraft, maybe even the biggest part of it. So we can't ignore it.
Some recent minecraft projects like Minestom are just a mod loader, the base game is also considered as mod !

<!--more-->

# What is a plugin/mod for Archipel?

Following Minestom choices, we want all content of our software to be a mod/plugin, leaving only core features in the server itself. Even minecraft vanilla features will be a plugin. Like this you can always choose want you want and what you don't want.
But what is a plugin ? It's a piece of code that can be loaded without a full recompilation of the server.
It should allow you to add new features to the server, or modify existing ones. You could create new blocks, new fluids, new Entities, extend the world generation, etc.


# So what are the constraint for a good plugin system?

A good plugin system should be:
* fast, to many mod should not slow down the server, ideally, it should safely handle multithreading
* sand-boxed, a plugin should not be able to crash the server or be able to access data it shouldn't, like sensitive host files
* easy to use, a plugin should be easy to create (for the writer), and easy to use (for the user)
* cross_platform, it should work on all platform supported by the server, it will be a pain to have to recompile a plugin for each platform, like minecraft does with java, you can run the same mod independently on windows, linux and mac, even on x86 and arm
* use a stable ABI (Application Binary Interface), to allow plugins to be compiled with different compilers (or version of a compiler), without breaking the ABI
* able to import symbols from other plugins, to allow plugins to interact with each other. Core mod are in way the philosophy of the fabric modding community. They avoid to get to many features in the core of the server and decrease work need to update to a newer version ! They can also reduce memory usage by limiting the number of binaries loaded in ram

It's hard to get all of this, and we will have to make some compromises...

# So what did you try?
* lua/wren, they are both very popular scripting language, and are very easy to embed in a C/C++ application https://www.lua.org/, https://wren.io/
* rust with dylib (dynamic library) using Rust ABI
* rust with dylib using FFI (Foreign Function Interface) also known as C ABI
* rust compiled to wasm (Web Assembly) with a WASM runtime like wastime or wasmer, https://wasmer.io/
* using java with JNI, https://en.wikipedia.org/wiki/Java_Native_Interface
* using C# and .NET with hostfxr, hosting the .NET runtime directly in the server, https://learn.microsoft.com/en-us/dotnet/core/tutorials/netcore-hosting

# First thoughts : lua/wren

Like every other plugin system, we thought about using lua. It's a very popular language, and it's used in many games.
It's also very easy to embed in a C/C++ application. LuaJit is also the FastVm.
Wren is a little slower but solve some problems of lua, like the lack of classes and add static typing. It's also very easy to embed.
Both are very lightweight and easy to embed, sandbox-able, but they suffer from a big problem, they don't support any kind of multithreading. So we are forced to use a single thread or synchronize multiple VMs, which is not ideal because the user can always store data on specific VMs, and can't access it from another VM.
* **fast : 2/5,** loosing any kind of multithreading is a big problem
* **sand-boxed : 4/5**, they both can be sandboxed easily
* **easy to use : ?/5**, they look easy to use, but implementing a 25k lines of code game in lua/wren can really become a mess very quickly, they don't provide good enough abstraction tools to handle a big project
* **cross_platform : 5/5**, they are both cross-platform and can run on almost any platform
* **stable ABI : 5/5**, they both have a stable ABI
* **able to import symbols from other plugins : 3/5**, interaction would be very tricky to implement, since in lua importing a file is executing it, getting a function of another plugin would always result in a Core call.

why did I look for alternative, I find lua/wren to primitive for our use case, and I don't want to write a 25k lines of code game in lua/wren,
Interaction with an ECS is also very tricky to implement in lua/wren (a Component is basically a static-sized, statically typed structure),
which will look like very verbose in lua/wren.

# all try to fit rust in a plugin system, using dylib and laziness, I mean Rust ABI

Bevy uses this approach, and it's very easy to use, you just have to add a dependency to your Cargo.toml, and you can use it like any other rust library.
The core of the project will be written in rust, and having a homogenous project looks like very conformable, no need to learn a new language, no need to learn a new build system, no need to learn a new IDE, etc.
Compiled rust for native platform is the fastest option, we can also use multithreading, and any other more advanced features of rust. But we will lose some keypoint here:
* **fast : 6/5**, rust is one of the fastest language, we are talking about a 2x to 10x the speed of any other alternative.
* **sand-boxed : 0/5**, a DLL get the same level of permission as the server, it can access any file, it can crash the server, it can do anything the server can do, which is not ideal
* **easy to use : 4/5**, rust is not the easiest language to learn, but it's not the hardest either, and it's very well documented, and will fit perfectly with the rest of the codebase ! sharing symbols between the core and a plugin is also quite easy, may be even easier than in lua/wren
* **cross_platform : 1/5**, rust is cross-platform, and can run on almost any platform, but need a recompilation for each platform, and we can't use the same binary on different platform
* **stable ABI : 0/5**, rust ABI is not stable, changing the flag of the compiler can change the padding of a structure, and break the ABI, plugin would need to be compiled with the same compiler, the same version of the compiler and the same flags, which is not ideal
* **able to import symbols from other plugins : 2/5**, the nightmare start from here, importing symbols from other plugin isn't really a thing, the "host plugin" must expose an array of static void pointer to every desired functions, and the "guest plugin" must cast the void pointer to the desired function pointer, and call it. This is not ideal, and will result in a lot of boilerplate code, and a lot of unsafe code.

# Rust the 2nd round, using C ABI, a bit better but still far from perfect

This solution is really similar to the precedent, but using C abi instead of Rust ABI. This will solve problems of ABI stability, like being constrained to use the same compiler, the same version of the compiler and the same flags. But this come with a trade-off, more boilerplate code, and more unsafe code. to map all to a C ABI, we will need to use a lot of unsafe code, and we will need to use a lot of boilerplate code to map rust types to C types, and to map rust functions to C functions. We will also need to use a lot of unsafe code to call the C functions, and to cast the C types to rust types. This is not ideal, but it's the best we can do with C ABI. We can't even use a trait to abstract the function pointer, because the trait will be defined in the "host plugin", and the "guest plugin" will not be able to implement it.
* **fast : 6/5**, same as the precedent
* **sand-boxed : 0/5**, same as the precedent
* **easy to use : 3/5**, same as the precedent, but with more boilerplate code
* **cross_platform : 2/5**, a bit better than the precedent, but we can still forget about using the same binary on different OS/cpu architectures
* **stable ABI : 5/5**, we are using C ABI/FFI, which is perfectly stable
* **able to import symbols from other plugins : 1/5** even worse than the precedent, we can't even use a trait to abstract the function pointer, because the trait will be defined in the "host plugin", and the "guest plugin" will not be able to implement it.

Both of these solutions allow to get a reference over any object passed by the core, It means we can iterate over components of our ECS without any copy ! This is a very good point, and will allow us to write very efficient plugins.

# Rust the 3rd round, using WASM, a bit slower but good enough, or not ?

Webassembly is a new kind of bytecode, designed to be close of native performances but way more portability, it can be compared to JAVA or C# bytecode, but more minimalistic. Wasm run in a browser when JavaScrip is too slow, or run with a dedicated runtime like Wastime or Wasmer.
Wasm have very cool trade-off, its speed is between native speed and luajit (the fastest Jit language), easy to use, the easiest to sand_box (it's designed for that), very cross-platform, stable ABI, and can import symbols from other plugins.
calling a function in wasm in very easy, at least with primitive... concurrently there is no offical way to pass complexe object from the host to the guest, but there is a hacky way to pass a pointer to the guest, and the guest can read/write to this pointer. This is not ideal, but it's the best we can do with wasm.
Passing a complex object require some serialisation, which come with a big performance cost, and a lot of boilerplate code. It also requires a lot of handmade VM memory management, and mean for us, reimplementing the whole ECS in WasmTime VM memory, or copying each component to the vm memory during entity query, which also come with a cost.

* **fast : 2/5**, wasm is fast, but the cost of serialisation and handmade VM memory management is very high
* **sand-boxed : 5/5**, wasm is designed for that
* **easy to use : -1/5**, wasm is very easy to use, but the handmade VM memory management is very tricky to implement (I have tried, and almost jumped from the window)
* **cross_platform : 5/5**, wasm is designed for that
* **stable ABI : 5/5**, wasm is designed for that
* **able to import symbols from other plugins : 2/5**, concurrently the one of the best solutions we have seen so far, but still not perfect, complex object are very tricky to pass

Feather (https://feathermc.org/) uses this approach, and they failed to implement it, because the heavy sand-boxing and handmade VM memory management is very tricky to implement, and come with a big performance cost. They also failed to implement the ECS in the VM memory, and they are copying each component to the VM memory during entity query, which also come with a big performance cost.

# Talking of bytecode, what about Java

Java (or any alternative running on the JVM like kotlin) is one of the most popular language today, and used to create minecraft. It provides good speeds, portability, and the targeted community already used it

* **fast : 3/5**, java is fast, but the OO with deep inheritance is not the best for cache coherency, and the memory management is not the best (java is hungry for memory)
* **sand-boxed : ?/5**, I didn't see any way to sandbox java, because I find big issues for my use case, I didn't dig too much
* **easy to use : 4/5**, java is very easy to use and to lean, JNI is also surprisingly easy to use
* **cross_platform : 5/5**, java is designed for that
* **stable ABI : 4/5**, provide some bytecode versions and securities for that
* **able to import symbols from other plugins : 5/5**, allow symbol resolution at runtime, it mean it allow to import symbols from other plugins out of the box, this is the best solution we have seen so far
* **little extra**, the minecraft community already know it !

But java don't have any Native function pointer (each "function container") is a polymorphic object, making a huge overhead for our project.
java don't support "inline classes", "primitives classes" or "structure" (call that like you want), I mean an in stack datastructures with compile time size, easy to put in array, making the ECS iteration very impractical.
The JEP402 may change that : https://openjdk.org/jeps/402

# and what about java's little brother made by microsoft, C# ?

C# is a very good language, with a lot of good features, and a very good community, and come with very nice feature, like struct (in stack data object), references (with some kind of mutability control). Like java, C# run a VM, a single binary can be run everywhere while dotNet is available
* **fast : 4/5**, C# is really fast, also comme with OO and deep inheritance, but you can also use struct and more functional programming
* **sand-boxed : ?/5**, I see some project talking about sand_boxing C#, but I didn't dig too much
* **easy to use : 4.5/5**, C# is very easy to use and to learn, and the C# community is very active (-0.5 because I don't like PascalCase on Methods)
* **cross_platform : 5/5**, C# can run the same binaries everywhere while dotNet is available (windows, linux, mac, on x86 or arm, arm64 with the anycpu target)
* **stable ABI : 4/5**, stable across major version
* **able to import symbols from other plugins : 5/5**, allow symbol resolution at runtime, it mean it allow to import symbols from other plugins out of the box, this is the best solution we have seen so far

C# have "unmanaged struct" (struct with a fixed size, and no reference), which is very good for ECS and cache coherency, It looks like C# is the best solution so far
, but C# suffers from the same problem as java, native function pointer are hard to get, but existent, a function can be declared as [UnmanagedCallersOnly] and can be called from a function pointer, but it's not very ergonomic, and require a lot of boilerplate code.
I didn't find any way to use references from hostfxr, it may be tricky to iterate over ECS without any copy.
Threading doesn't work out of the box but should be doable with some work !

HostFxr is really minimalistic, it exposes at most 10 functions, we also looked to use Mono like the unity engine, but mono is slowly dying, and the mono community is not very active, mono is still stuck with C# 8.0 while dotNet is already at C# 12.0 !!

At this time we are working to use C# as our language of choice, for modding ! We are working on a C# library to make modding easier !

