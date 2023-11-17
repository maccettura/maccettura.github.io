---
layout: post
title:  "Generating FPO Images with ImageSharp in .NET 6"
tags: [.NET 6, ImageSharp] 
comments: true
excerpt_separator: <!--more-->
---

Throughout the course of my work day, I probably think of a dozen or so applications, libraries or tools I could/should write to make my job easier. Once in a while I will go off on an all night adventure, cram learning some new library, banging my head against a desk until 3 AM. After seemingly endless build fails, syntax errors and a severe lack of water or food, I finally finish the task I set out to do. More often than not, this all-night coding adventure does not result in the genius time-save I thought it would be, but I always learn something, and I never regret it.

 <!--more-->

A little while back, I created <a href="https://fllr.art/" target="_blank">fllr.art</a>. The idea was simple: build an API with a simple URL structure that spits out FPO images. I imagined that this would be a useful tool for anyone who ever needed a quick image, with _specific_ dimensions and didn't want to Google it or pull up an image editor. Originally this was built as a .NET Core project but I recently updated it to utilize some new features of the recently released .NET 6. See the source code <a href="https://github.com/maccettura/Fllr" target="_blank">on Github</a>.

### How to use

Fllr is super simple to use, for the most basic image all you need is a width and height:

`https://i.fllr.art/400x400.jpg`

If you want to use specific colors for the text and background, simply add the hex colors to the url:

`https://i.fllr.art/555555/ffffff/400x400.jpg`

You can even use simple 3 digit hex codes: 

`https://i.fllr.art/555/fff/400x400.jpg`

Fllr supports two Google Fonts: Montserrat & PT Serif. Montserrat is the default font, but if you would like to use PT Serif, simply pass it as the `f=` query string parameter:

`https://i.fllr.art/555/fff/400x400.jpg?f=pt%20serif`

Fonts are easy enough to add, so feel free to submit a PR for review.

### Under the hood

The API solution contains 3 projects:

`Fllr`: The Web API project.

`Fllr.Infrastructure`: Shared classes / extensions.

`Fllr.Generator.Default`: The meat and potatoes of the whole thing. This generates the two color images that are requested from the API.


`Fllr.Generator.Default.DefaultImageService` is where most of the heavy lifting gets done, let me walk you through how ImageSharp is being used and how it makes it very simple to generate the image. 

For example, to generate a very simple image with a specified width, height and background color. All you need is:

{% highlight csharp %}
using var image = new Image<Rgba32>(
                Configuration.Default,
                request.Width,
                request.Height,
                request.Color.HexStringToColor()); // .HexStringToColor() is an extension for converting hex strings into useable type
{% endhighlight %}

Now it gets slightly more complicated since Fllr also needs to write text on the generated image:


{% highlight csharp %}
// width of image indent left & right by padding
float textMaxWidth = request.Width - _padding * 2; 

Color fontColor = request.TextColor.HexStringToColor();

// PlaceholdFontCollection is part of Fllr and just loads from an embedded resource by name
FontFamily family = PlaceholdFontCollection.Instance.Families.FirstOrDefault(x => string.Equals(x.Name, request.Font, StringComparison.CurrentCultureIgnoreCase));

// Create a new font object, with requested family and size
var font = new Font(family, request.FontSize);
// Then we get a measurement of the space our intended text and font will take up
var size = TextMeasurer.Measure(request.Text, new TextOptions(font));

// Here is where we determine the center-point of our image based on 
// the image and font's dimensions.
int xPos = (int)((request.Width - size.Width) / 2);
if(xPos < 0)
{
    xPos = (int)_padding;
}
int yPos = (int)((request.Height - size.Height) / 2);

// Mutate is part of `SixLabors.ImageSharp.Processing` and does the actual "writing"
image.Mutate(i => i.DrawText(request.Text, font, fontColor, new PointF(xPos, yPos)));
{% endhighlight %}

Now our `image` object is all loaded up with the information we need, and all that's left is to turn it into something we can display to the user:


{% highlight csharp %}
using var outputStream = new MemoryStream();

// ImageSharp supports saving as various file types, for now I just use Jpg
image.SaveAsJpeg(outputStream, new JpegEncoder { Quality = 100 });
outputStream.Position = 0;

// this is the response object that the Fllr Web API project receives to display the image
return new PlaceholdImage 
{
    Payload = outputStream.ToArray(),
    MimeType = JpegFormat.Instance.DefaultMimeType,
    Name = $"{request.Width}x{request.Height}.{ext}"
};
{% endhighlight %}

...and that's basically it! It did take quite a bit of trial and error to get the image in an output that I wanted, but after some time everything worked perfectly.

Thinking ahead to future need and/or extensibility, any number of new projects can be added (i.e `Fllr.Generator.*`). Each one having its own image generation logic. Perhaps a library to generate images with a nice gradient? Maybe a project that uses AI generation? I'm not quite that ambitious yet so for now it will remain a plain ol' FPO image.

### Prettying it up

I realized that an API hidden among the clouds that only I knew how to use, wasn't super exciting. This is when I decided to finally dive a bit deeper into learning Vue and make a simple app to demo how the API works.

Since Vue.js is reactive, I wanted to <a href="https://github.com/maccettura/Fllr/blob/main/www/src/components/LinkGenerator.vue" target="_blank">make a component</a> that allowed you to play with the various parameters and see the URL change in real time. 

I don't want to detract too much from the API side for this post, so I will keep this part brief. I really like the ecosystem that is expanding around Vue.js and I prefer working with it over React. 


### In Conclusion

This was a fun little project that exposed me to a couple new libraries/frameworks. I am thankful to James and the team at <a href="https://github.com/SixLabors/ImageSharp" target="_blank">ImageSharp</a> for creating a library that makes this task so easy and enjoyable. Please <a href="https://github.com/sponsors/SixLabors" target="_blank">sponsor them</a> if you can, or perhaps convince your team to use their various libraries with a paid <a href="https://sixlabors.com/pricing/" target="_blank">commercial license</a>. 
