---
layout: post
title:  "2021 Advent of Code"
tags: [Advent of Code] 
comments: true
excerpt_separator: <!--more-->
---

It's almost December, and that means a new Advent of Code begins. I am freezing and its late so this is a quick one.

<!--more-->

I <a href="https://github.com/maccettura/AdventOfCode" target="_blank">maintain a Github template</a> that I update yearly to reflect the latest C#/.NET features available.

The biggest changes to the template this year is the addition of <a href="https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-10#global-using-directives" target="_blank">global usings</a> and <a href="https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-10#file-scoped-namespace-declaration" target="_blank">file scoped namespace declarations</a>.

This isn't going to affect performance, but it will let me delete a bunch of (now) unnecessary code.

For example:


{% highlight csharp %}

using System;
using System.Collections.Generic;
using AdventOfCode.Properties;

namespace AdventOfCode.Day01
{
    // the rest
}

{% endhighlight %}

Can now simply be:

{% highlight csharp %}

namespace AdventOfCode.Day01;

// the rest

{% endhighlight %}

We reduced the file size a bit, cleaned up one level of (now) redundant indentation and got the easiest win of the "learning the latest C# features" endeavor.

However, those using directives still need to go somewhere...

Since the 3 using directives I removed are all needed across the board in this project, I added them to the Program.cs file, along with the new `global` modifier. (**Note** This was a logical central point to me, its not required to go specifically here; the modifier can be used anywhere!)

{% highlight csharp %}

global using AdventOfCode;
global using AdventOfCode.Properties;
global using System;
global using System.Collections.Generic;

// rest of Program.cs

{% endhighlight %}

The way I have structured this project means I have one folder for every day in the Advent of Code calendar.  Removing `n` of using directives, across 25 folders gave me a lot of pre-holiday joy.

The C# language team have been making wonder & welcome additions to the language yearly. This isn't Steve Ballmer's Microsoft anymore...
<img src="https://media.giphy.com/media/Y4v5zIFWE34PsiCPRs/giphy.gif" />

Happy holidays!

