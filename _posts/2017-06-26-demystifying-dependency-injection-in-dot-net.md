---
layout: post
title:  "Demystifying Dependency Injection in .NET"
tags: [SOLID Design, Dependency Injection] 
comments: true
excerpt_separator: <!--more-->
---

<h2>Intro</h2>

I decided to start this blog about 8 months ago and I have yet to find anything to write about.  I have had ideas for blog posts but I swiftly reject them because I feel they are too basic or that no one would really benefit from _my_ knowledge.  Basically I was stricken with Imposters Syndrome.

I have been spending some time on Stack Overflow lately and I have done my best to craft very detailed answers (complete with fiddles where applicable). I have found that my answers are typically well received, leading me to the realization that my knowledge and experience _may actually make a difference in helping other programmers_.

 So I've decided that the potential to help just one person outweighs any anxiety I may have about my skill or knowledge.

 With that being said, I have come across some questions about SOLID design, specifically on Dependency Injection.  This was a topic that took me a while to truly grasp and I thought maybe I could try and break it down for anyone out there still struggling with it.  I think I may write more about the principles of SOLID design in the future (albeit out of order).

<h2>Demystifying Dependency Injection</h2>

So what is "dependency injection" anyway?

Dependency injection is one of the five principles described by Robert C. Martin in his [SOLID](https://en.wikipedia.org/wiki/SOLID_(object-oriented_design)) design proposal.  It basically states that:

 - High-level modules should not depend on low-level modules. Both should depend on abstractions.
 - Abstractions should not depend on details. Details should depend on abstractions.

 After reading that it should make _perfect sense_ right?

 <!--more-->

Basically what that means is you should write code that does not care about what objects are, only that they have a certain <strong>shape</strong>.

<h3>Consider the following</h3>

Lets say you need to create a web application that searches a 3rd party music service.  It seems very simple so you will likely layout your solution like this:

```
MusicSearch.sln
    MusicSearch.Web
```        

One project that contains all the code you need, everything from gathering the data from the 3rd party music service to surfacing that data in an MVC application.  The one stop shop for all your needs, _right_?

What happens when you need to unit test functionality specific to the MVC application (i.e testing routes, designing the layout, etc)? What happens when you decide to switch to another 3rd party music catalog service?  In either of those cases you will have to perform some pretty heavy lifting in your already completed and working application.  What you need is a way to make your code more modular, and easier to maintain.  Thats where dependency injection comes in.

<h3>If I knew then what I know now</h3>

With dependency injection in mind let's look at our solution structure again:

```
MusicSearch.sln
    MusicSearch.Service.Infrastructure
    MusicSearch.Service.Spotify
    MusicSearch.Web
```

As you can see we added two new projects.  

 - **MusicSearch.Service.Infrastructure** will contain interfaces on what our service _looks like_.
 - **MusicSearch.Service.Spotify** will be how we will specifically implement the interfaces for use with Spotify.

 So let's define our service.  For our needs we simply want to get artist, albums and songs based on a query.

{% highlight csharp %}

public interface IMusicSearchService
{
    IEnumerable<Album> SearchAlbums(string query);
    IEnumerable<Artist> SearchArtists(string query);
    IEnumerable<Song> SearchSongs(string query);
}

{% endhighlight %}

This interface will live in the <strong>MusicSearch.Service.Infrastructure</strong> project.  This project will also contain our definitions for `Album`, `Artist` and `Song` and anything else that is a shared piece of our applications infrastructure.

Now, in our <strong>MusicSearch.Service.Spotify</strong> project we will write a _concrete_ class that _implements_ our `IMusicSearchService` interface.

{% highlight csharp %}

public class SpotifyMusicSearchService : IMusicSearchService
{
    public IEnumerable<Album> SearchAlbums(string query)
    {
        //Some code specific to Spotify
    }

    public IEnumerable<Artist> SearchArtists(string query)
    {
        //Some code specific to Spotify
    }

    public IEnumerable<Song> SearchSongs(string query)
    {
        //Some code specific to Spotify
    }
}

{% endhighlight %}

So there we have a concrete class that takes the _shape_ of our interface. What now?

In our MVC project <strong>MusicSearch.Web</strong>, we can now code everything off our interface without caring at all how its implemented.  We all know interfaces are contracts for our code so lets use them as such.  Take a look at our controller now.


{% highlight csharp %}

public class SearchController : Controller
{
    private readonly IMusicSearchService _musicSearchService;

    public ActionResult AlbumSearch(string query)
    {
        var results = _musicSearchService.SearchAlbums(query);
        //more code
    }

    public ActionResult ArtistSearch(string query)
    {
        var results = _musicSearchService.SearchArtists(query);
        //more code
    }

    public ActionResult SongSearch(string query)
    {
        var results = _musicSearchService.SearchSongs(query);
        //more code
    }
}

{% endhighlight %}

You see how our code now only references the `IMusicSearchService` interface instead of a concrete class?  It is indifferent to what `_service` ultimately is, all it needs to care about is whatever our interface tells it to.  Now we have the flexibility to to make any version of our service (so long as it takes the shape we have defined).  The only thing we have left to do is tell our MVC code what concrete class to use.  For that we need to _inject_ our dependency.

<h3>This is where the magic happens</h3>

You simply need to create a constructor that takes an argument of type `IMusicSearchService`.  Seriously, _that's it_.

{% highlight csharp %}

public class SearchController : Controller
{
    private readonly IMusicSearchService _musicSearchService;

    public SearchController(IMusicSearchService musicSearchService)
    {
        _musicSearchService = musicSearchService; //magic!
    }

    public ActionResult AlbumSearch(string query)
    {
        var results = _musicSearchService.SearchAlbums(query);
        //more code
    }

    public ActionResult ArtistSearch(string query)
    {
        var results = _musicSearchService.SearchArtists(query);
        //more code
    }

    public ActionResult SongSearch(string query)
    {
        var results = _musicSearchService.SearchSongs(query);
        //more code
    }
}

{% endhighlight %}

Anytime an instance of your controller is created, an argument of type `IMusicSearchService` <strong>has</strong> to be passed in.  This could be done by simply creating a new object and passing it as a parameter, but then we <strong>still</strong> have our code dictating what concrete to use.  Thankfully there is another way; [IoC containers](https://github.com/quozd/awesome-dotnet#ioc).  IoC containers are complex, but for our purposes today all you need to know is that _it takes care of instantiating objects for you_.  We just have to tell it how.

***

I personally use [Ninject](https://github.com/ninject/ninject), and the following examples will work under the assumption that the <strong>Ninject.MVC5</strong> Nuget package is installed.

***

After <strong>Ninject.MVC5</strong> is installed in your project, you will see a new file: `App_Start/NinjectWebCommon.cs`.  When you open it you should see a pretty weird looking class, something like this:

{% highlight csharp %}

[assembly: WebActivatorEx.PreApplicationStartMethod(typeof(MusicSearch.Web.App_Start.NinjectWebCommon), "Start")]
[assembly: WebActivatorEx.ApplicationShutdownMethodAttribute(typeof(MusicSearch.Web.App_Start.NinjectWebCommon), "Stop")]

namespace MusicSearch.Web.App_Start
{
    using System;
    using System.Web;

    using Microsoft.Web.Infrastructure.DynamicModuleHelper;    
    using Ninject;
    using Ninject.Web.Common;

    public static class NinjectWebCommon 
    {
        private static readonly Bootstrapper bootstrapper = new Bootstrapper();

        /// <summary>
        /// Starts the application
        /// </summary>
        public static void Start() 
        {
            DynamicModuleUtility.RegisterModule(typeof(OnePerRequestHttpModule));
            DynamicModuleUtility.RegisterModule(typeof(NinjectHttpModule));
            bootstrapper.Initialize(CreateKernel);
        }
        
        /// <summary>
        /// Stops the application.
        /// </summary>
        public static void Stop()
        {
            bootstrapper.ShutDown();
        }
        
        /// <summary>
        /// Creates the kernel that will manage your application.
        /// </summary>
        /// <returns>The created kernel.</returns>
        private static IKernel CreateKernel()
        {
            var kernel = new StandardKernel();
            try
            {
                kernel.Bind<Func<IKernel>>().ToMethod(ctx => () => new Bootstrapper().Kernel);
                kernel.Bind<IHttpModule>().To<HttpApplicationInitializationHttpModule>();

                RegisterServices(kernel);
                return kernel;
            }
            catch
            {
                kernel.Dispose();
                throw;
            }
        }

        /// <summary>
        /// Load your modules or register your services here!
        /// </summary>
        /// <param name="kernel">The kernel.</param>
        private static void RegisterServices(IKernel kernel)
        {

        }        
    }
}

{% endhighlight %}

The only part you need to be concerned with is the `RegisterServices()` method.  That is where we will define our _bindings_.

{% highlight csharp %}

private static void RegisterServices(IKernel kernel)
{
    //Anytime our code needs a "IMusicSearchService", use "SpotifyMusicSearchService"
    kernel.Bind<IMusicSearchService>().To<SpotifyMusicSearchService>();
} 

{% endhighlight %} 

Now, <strong>that's it</strong>.

<h3>Endless possibilities</h3>

Remember the problems we dreamt up earlier?

 - What happens when you need to unit test functionality specific to the MVC app? 
 - What happens when you decide to switch to another 3rd party music catalog service?

Both of those are quite easily solved now.  We can now create "mock" services that simply just return some dummy data for the purpose of testing our MVC app.  If we decide to switch from Spotify to another 3rd party service, we simply create a new project with the new service's specific code.  All thats needed is to change our bindings in the IoC container.

As long as we always _code against abstracts_, we make our code <strong>better</strong>.