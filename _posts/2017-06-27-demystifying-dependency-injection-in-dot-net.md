---
layout: post
title:  "Demystifying Dependency Injection in .NET"
tags: [SOLID Design, Dependency Injection] 
comments: true
excerpt_separator: <!--more-->
---

<h2>Demystifying Dependency Injection</h2>

So what is "dependency injection" anyway?

Dependency injection is one of the five principles described by Robert C. Martin in his [SOLID](https://en.wikipedia.org/wiki/SOLID_(object-oriented_design)) design proposal.  It basically states that:

 - High-level modules should not depend on low-level modules. Both should depend on abstractions.
 - Abstractions should not depend on details. Details should depend on abstractions.

 After reading that it should make _perfect sense_ right?

Basically what that means is you should write code that does not care about what objects are, only that they have a certain <strong>shape</strong>.

 <!--more-->

<h3>Consider the following</h3>

Lets say you need to create a web application that searches a 3rd party music service.  It seems very simple so you will likely layout your solution like this:

```
MusicSearch.sln
    MusicSearch.Web
```        

One project that contains all the code you need, everything from gathering the data from the 3rd party music service to surfacing that data in an MVC application.  The one stop shop for all your needs, _right_?

What happens when you need to unit test functionality specific to the MVC application (i.e testing routes, designing the layout, etc)? What happens when you decide to switch to another 3rd party music service?  In either of those cases you will have to perform some pretty heavy lifting in your already completed and working application.  What you need is a way to make your code more modular, and easier to maintain.  Thats where dependency injection comes in.

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

    public IActionResult AlbumSearch(string query)
    {
        var results = _musicSearchService.SearchAlbums(query);
        //more code
    }

    public IActionResult ArtistSearch(string query)
    {
        var results = _musicSearchService.SearchArtists(query);
        //more code
    }

    public IActionResult SongSearch(string query)
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

    public IActionResult AlbumSearch(string query)
    {
        var results = _musicSearchService.SearchAlbums(query);
        //more code
    }

    public IActionResult ArtistSearch(string query)
    {
        var results = _musicSearchService.SearchArtists(query);
        //more code
    }

    public IActionResult SongSearch(string query)
    {
        var results = _musicSearchService.SearchSongs(query);
        //more code
    }
}

{% endhighlight %}

Anytime an instance of your controller is created, an argument of type `IMusicSearchService` <strong>has</strong> to be passed in.  This could be done by simply creating a new object and passing it as a parameter, but then we <strong>still</strong> have our code dictating what concrete to use.  Thankfully there is another way; [IoC containers](https://github.com/quozd/awesome-dotnet#ioc).  IoC containers are complex, but for our purposes today all you need to know is that _it takes care of instantiating objects for you_.  We just have to tell it how.

Microsoft has included an IoC container in ASP.NET Core, for the purpose of this post I will be showing how to inject dependencies using the built in tool. 

Navigate to the `Startup.cs` class and look for a method called: `ConfigureServices()`, this is where we will define our _bindings_.

{% highlight csharp %}

public void ConfigureServices(IServiceCollection services)
{
    /*
         There will be other services registered here, the code may vary so I am excluding it
    */
        
    //This is the where the IoC magic happens
    services.AddTransient<IMusicSearchService, SpotifyMusicSearchService>();
}

{% endhighlight %}

Basically this line of code says that anytime our code needs a "IMusicSearchService", use "SpotifyMusicSearchService".  We do have other options for scope besides "Transient" (Scoped, Singleton), but for our purposes we will use "Transient".

<h3>Endless possibilities</h3>

Remember the problems we dreamt up earlier?

 - What happens when you need to unit test functionality specific to the MVC app? 
 - What happens when you decide to switch to another 3rd party music service?

Both of those are quite easily solved now.  We can now create "mock" services that simply just return some dummy data for the purpose of testing our MVC app.  If we decide to switch from Spotify to another 3rd party service, we simply create a new project with the new service's specific code.  All thats needed is to change our bindings in the IoC container:

{% highlight csharp %}

public void ConfigureServices(IServiceCollection services)
{
    //We want to test our MVC app, so we switch to a fake music service (i.e dummy data).
    services.AddTransient<IMusicSearchService, MockMusicSearchService>();
}

{% endhighlight %}

{% highlight csharp %}

public void ConfigureServices(IServiceCollection services)
{
    //We want to test out potentially switching to Pandora, so we switch to our newly created Pandora service.
    services.AddTransient<IMusicSearchService, PandoraMusicSearchService>();
}

{% endhighlight %}

Can you see the benfits of writing modular code?  As long as we always _code against abstracts_, we make our code <strong>better</strong>.

