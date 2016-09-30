---
layout: post
title: Customising your application's favicon
tags:
- Play!
- Scala
- Cake
- Stupidity
---

This blog post would be better titled *Customising your application's favicon the hard way* and was originally a lightning talk I gave as I was beginning to use Scala and the Play framework.

Imagine you are working in a department that is producing Play apps at a rate of knots. Without careful planning you can end up with a whole row of identical looking tabs in which you can't distinguish one app from another. 

![Play apps all look identical]({{ site.baseurl }}assets/favicon-lots-of-play-apps.png)

Using the Guardian favicon doesn't really help that much either...

![Play apps all look identical]({{ site.baseurl }}assets/favicon-lots-of-guardian-apps.png)

One approach to create an easily identifiable favicon would be to modify an existing favicon in Photoshop or GIMP. However I am neither rich enough to own Photoshop nor have a mind that is twisted enough to use GIMP.  Clearly the answer is to write some scala code:

{% highlight scala %}
val favicon = ImageIO.read(new File("src/favicon-blue.png"))

val inputPixels = for {
  x <- 0 until favicon.getWidth
  y <- 0 until favicon.getHeight
} yield Pixel(favicon.getRGB(x,y))

val sortedPixels = inputPixels.distinct.sorted

val paletteSize = 4
val darkest = sortedPixels.head.luminance - 20 // fudge factor
val lightest = sortedPixels.last.luminance
val range = darkest to lightest by ((lightest - darkest) / (paletteSize - 1))

val colourPalette = range.map{ Pixel.luminance(_) }

val remappedPixels = inputPixels.map { pixel =>
  colourPalette.reduceLeft { (result, elem) =>
    if (result.compare(pixel) < elem.compare(pixel)) {
      result
    } else {
      elem
    }
  }
}

...
{% endhighlight %}

This code reads an existing favicon and converts it into a greyscale image with a smaller palette size. It's not particularly pretty code but this what happens when we pump in the Guardian favicon:
 
![How the scala code transforms the Guardian favicon]({{ site.baseurl }}assets/favicon-transformation.png)

Why stop there?! Let's write some increasingly ridiculous Scala code:

{% highlight scala %}
import sihil.baking._

val ediblePixels = remappedPixel.map { pixel =>
  new EdiblePixel(pixel.luminance)
}

val ingredients = List(flour, chocolate, egg, sugar, 
                       moreSugar, moreChocolate, bakingSoda)

val cake = new ChocolateCake(ingredients).baked()

cake.decorate(outputImage, ediblePixels)
{% endhighlight %}

So what does that look like when you run it? Well let's start with the edible pixels. Using dark, milk and white chocolate (plus a mix) we can create four different tones of chocolate. When spread thinly and cut into one centimeter squares with a pizza cutter you get something that looks a little like this:
 
![Edible chocolate pixels]({{ site.baseurl }}assets/favicon-pixels.png)

Next we need to bake a cake and make sure we've got enough pixels of each type of chocolate.

![Piles of chocolate pixels and a chocolate cake]({{ site.baseurl }}assets/favicon-pixels-and-cake.jpg)

And, having not built a robot for the task, we'll also need a grid to work from.

![Favicon grid]({{ site.baseurl }}assets/favicon-grid.jpg)

In order to start laying down the pixels we go from the middle outwards. First step is to mark the centre and then mark the center lines in both the x and y direction. Next pixels are individually laid onto the still warm chocolate icing, partially melting as they go on.

![Laying the first pixels]({{ site.baseurl }}assets/favicon-first-pixels.jpg)

Some time later (really quite a long time later, it's dark now) it is beginning to take shape and the technique of placing pixels has been mastered.

![Taking shape]({{ site.baseurl }}assets/favicon-recognisable.jpg)

The final cake is carefully photographed directly from above. and then turned into a lower resolution icon suitable for being a favicon.

![The finished cake viewed from above]({{ site.baseurl }}assets/favicon-finished.jpg)

It is carefully masked.

![Masked version of the finished cake]({{ site.baseurl }}assets/favicon-masked.png)

Finally the resolution is reduced so it is suitable for being a favicon.

![Final favicon]({{ site.baseurl }}assets/favicon-final.png)

As you can now see we have now created a recognisable favicon for our new application. The hard way.

![Our favicon amoungst many others]({{ site.baseurl }}assets/favicon-riff-raff-icon-tab.png)