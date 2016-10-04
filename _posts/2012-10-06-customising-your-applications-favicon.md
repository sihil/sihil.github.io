---
layout: post
title: Customising your application's favicon (the hard way)
tags:
- Play!
- Scala
- Cake
- Stupidity
---

This blog post was originally a lightning talk I gave at the Guardian as I was beginning to use Scala and the Play framework.

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

val distinctPixelCounts = remappedPixels.distinct.sorted.map{ pixel => 
  pixel.luminance -> remappedPixels.count(pixel ==)
}

val outputImage = new BufferedImage(width, height, TYPE_INT_RGB)
for {
  x <- 0 until favicon.getWidth
  y <- 0 until favicon.getHeight
} {
  val pixel = remappedPixels.get(x * 16 + y)
  outputImage.setRGB(x, y, pixel.rgb)
}

ImageIO.write(outputImage, "png", new File("output-new.png"))

println(s"Distribution of remapped pixels (palette of $paletteSize):")
println(distinctPixelCounts.map{ case (lum, count) => s"Luminance: $lum  Count: $count" }.mkString("\n"))
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
 
![Edible chocolate pixels]({{ site.baseurl }}assets/favicon-pixels.jpg)

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

![Our favicon amongst many others]({{ site.baseurl }}assets/favicon-riff-raff-icon-tab.png)

The finished cake was shipped into the office and tasted pretty much as good as it looked...

<blockquote class="instagram-media" data-instgrm-captioned data-instgrm-version="7" style=" background:#FFF; border:0; border-radius:3px; box-shadow:0 0 1px 0 rgba(0,0,0,0.5),0 1px 10px 0 rgba(0,0,0,0.15); margin: 1px; max-width:658px; padding:0; width:99.375%; width:-webkit-calc(100% - 2px); width:calc(100% - 2px);"><div style="padding:8px;"> <div style=" background:#F8F8F8; line-height:0; margin-top:40px; padding:50% 0; text-align:center; width:100%;"> <div style=" background:url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAACwAAAAsCAMAAAApWqozAAAABGdBTUEAALGPC/xhBQAAAAFzUkdCAK7OHOkAAAAMUExURczMzPf399fX1+bm5mzY9AMAAADiSURBVDjLvZXbEsMgCES5/P8/t9FuRVCRmU73JWlzosgSIIZURCjo/ad+EQJJB4Hv8BFt+IDpQoCx1wjOSBFhh2XssxEIYn3ulI/6MNReE07UIWJEv8UEOWDS88LY97kqyTliJKKtuYBbruAyVh5wOHiXmpi5we58Ek028czwyuQdLKPG1Bkb4NnM+VeAnfHqn1k4+GPT6uGQcvu2h2OVuIf/gWUFyy8OWEpdyZSa3aVCqpVoVvzZZ2VTnn2wU8qzVjDDetO90GSy9mVLqtgYSy231MxrY6I2gGqjrTY0L8fxCxfCBbhWrsYYAAAAAElFTkSuQmCC); display:block; height:44px; margin:0 auto -44px; position:relative; top:-22px; width:44px;"></div></div> <p style=" margin:8px 0 0 0; padding:0 4px;"> <a href="https://www.instagram.com/p/KXIo4iubhu/" style=" color:#000; font-family:Arial,sans-serif; font-size:14px; font-style:normal; font-weight:normal; line-height:17px; text-decoration:none; word-wrap:break-word;" target="_blank">Guardian favicon as a cake (not made by me)</a></p> <p style=" color:#c9c8cd; font-family:Arial,sans-serif; font-size:14px; line-height:17px; margin-bottom:0; margin-top:8px; overflow:hidden; padding:8px 0 7px; text-align:center; text-overflow:ellipsis; white-space:nowrap;">A photo posted by Lynsey Smyth (@lynsey_s) on <time style=" font-family:Arial,sans-serif; font-size:14px; line-height:17px;" datetime="2012-05-08T09:58:59+00:00">May 8, 2012 at 2:58am PDT</time></p></div></blockquote>
<script async defer src="//platform.instagram.com/en_US/embeds.js"></script>