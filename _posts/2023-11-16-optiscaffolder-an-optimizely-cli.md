---
layout: post
title: OptiScaffolder - an Optimizely Developer CLI
tags: [Optimizely]
comments: true
excerpt_separator: <!--more-->
---

Back when Optimizely 12 was introduced, support for the <a href="https://marketplace.visualstudio.com/items?itemName=EPiServer.EpiserverCMSVisualStudioExtension" target="_blank">Visual Studio developer extension</a> was dropped. This tool was extremely helpful in scaffolding out the required files when making blocks or pages, and its absence was immediately missed. After a couple weeks into a new project I finally had enough and spent a long evening coming up with a tool to help make this easier and in the process, discovering that this was just the first step needed in a potentially better way of delivering Optimizely projects...

<!--more-->

Earlier this week I decided to make <a href="https://github.com/maccettura/OptiScaffold" target="_blank">this project</a> public, and add the tool <a href="https://www.nuget.org/packages/OptiScaffold/" target="_blank">to nuget</a> so that anyone can use it.

### Quick overview

Essentially I just needed a way to create all the files needed for blocks and pages. This meant something that created the `*.cshtml` files, the various `*.cs` files, getting the Optimizely content GUIDs inserted and matched up, adding the right Optmizely attributes to the models, etc. I assumed at some point the team at Opti would make their own version of this tool to replace the VS extension they abandoned, but to this day they haven't (despite saying it would be coming soon). 

Even if they got around to building this thing, I eventually realized that the way I prefer to build out Opti projects is a bit opinionated. I like to enforce some architecture standards like: 

- using custom abstract base classes for pages/blocks as an extra layer of abstraction so our block and pages types don't inherit from `BlockData` and `PageData` directly
- Always using block & page view model classes, even when they're redundant
- - There is no overhead as far as I can tell to doing this, but with the constant changes throughout a project lifecycle I prefer to have the foundation there already if a block or page becomes more _involved_.
- In addition to the last point, always using custom base classes for block/page view models.

I believe these standards improve my ability to deliver consistent, high-quality projects to clients and since the tool was originally just for me, I built these standards (opinions) into it. 

### Expanding on the idea

In the year or so since I built this tool for my internal use, I started to take notice of the various standards, common classes, paradigms, etc that I built into every Opti project. I decided that I should gather my thoughts and formalize a _lightly_ opinionated base/starter project that incorporates some of these libraries, extensions and common paradigms so that I can make nuget package(s) and a Github repository template to share with others. Stay tuned for more posts on that idea in the future.