+++
title = "The dawn of a new project"
author = "Bricklou, D3lta"
date = "2023-09-27"
categories = ["news"]
tags = ["what-is", "news"]
description = "We are starting a new project: a Minecraft server written in Rust!"

[banner]
image = "minecraft_ingame_screenshot.png"
caption = "Minecraft In-Game screenshot"
+++

Did you hear about Minecraft? You know, this game where you can build stuff with blocks? Well, we are trying to do the same thing, but with a twist: we will rewrite all the server in Rust.

Why? Because we can. And because we want to learn Rust and how Minecraft works.

So, let us present you our project: **Archipel Project**!

<!--more-->

# What is Archipel Project?

Like we said, it is a Minecraft server written in Rust. The goal is to have a server fast, secure and (almost) easy to use.

But is it just another Minecraft server? Can it compete with big servers like Spigot or Paper? Well, no. At least, not at the beginning.
but we have more than one trick up our sleeves to make it better than the others.
starting by a full rewrite in rust, which will make it faster and more secure than the others java alternatives.
more data oriented patterns, like ECS, will also make it faster, more scalable and easier to extend than the traditional OOP patterns used by the others servers. See https://github.com/SanderMertens/ecs-faq for more details!

Wait but Valence and feather already have these kind of features, why not use them instead of writing a new server?

Archipel also target a more distributed work over nodes, It will be built on a microservice architecture, with a lot of small services that will communicate with each other: some will store the world data,
some will run the logic, and others will handle the player connections and authentication.

We will try to make it as modular as possible, so you can use only the services you need.

# Will it still be compatible with Minecraft?

Yes, it will! It needs to stay compatible with the official and latest java client! We are also thinking about supporting the Bedrock client too, but it is not our priority.
To support multiple clients version, and server side modding, we are thinking of using a layer Item translation, some mod or plugin already do that, like ViaVersion or polymer (https://polymer.pb4.eu/latest/),
but we want to push the concept further, and make it more generic, so it can be used by any mod or plugin.

# When will it be ready?

We don't know. At the time to write this post, we haven't even started to write the code. We are still in the design phase, where some parts still
need some clarification and where some of us are still learning Rust. We will try to keep you updated on the progress of the project.

# So, what's this blog about?

This blog will be used to keep you updated on the progress of the project. But it's not the main reason we created it. We also to share with you
our experience and all the accumulated knowledge by publishing scientific articles on various subjects, like Rust, Minecraft, distributed systems, etc.

# How can I help?

You can help us by sharing this blog and our project with your friends, or by contributing to the project. We will soon publish a list of issues
we must work on, so you can pick one and start working on it. We will also publish a list of features we want to implement, so you can
pick one and start working on it too.

# Conclusion

We hope you will enjoy this project as much as we do. We will try to keep you updated on the progress of the project, so stay tuned!

See you soon!
