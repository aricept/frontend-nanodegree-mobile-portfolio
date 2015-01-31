# Website Optimization
####Rendering and Performance Optimizations

###Table of Contents
1.[Objectives](#objectives)
1. [How to Test](#test)
2. [Tools Used](#tools)
3. [Resources Used](#resources)
4. [Process & Changes](#process)
    * [index.html](#index)
    * [pizza.html](#pizza)
    * [main.js](#main)
        * [Global Changes](#mainglobal)
        * [resizePizzas](#mainresize)
        * [updatePositions](#mainupdate)
5. [Failed Attempts](#failed)

<a name="objectives"></a>
##Objectives
###PageSpeed Insights
Target: 90+ for Mobile and Desktop

Achievement:

![Mobile PageSpeed Score](http://aricept.github.io/img/psi-mobile.png)
![Desktop PageSpeed Score](http://aricept.github.io/img/psi-desktop)

###Pizza Resizes
Target: < 5ms

Achievement:

![Pizza Resize](http://aricept.github.io/img/resize.png)

###Frames Per Second
Target: 60 FPS

Achievement:

![FPS](http://aricept.github.io/img/FPS.png)

<a name="test"></a>
## How to Test
[The portfolio](http://aricept.github.io/frontend-nanodegree-mobile-portfolio) is hosted on GitHub Pages.

Jump straight to [analyze it on PageSpeed Insights](https://developers.google.com/speed/pagespeed/insights/?url=http%3A%2F%2Faricept.github.io%2Ffrontend-nanodegree-mobile-portfolio&tab=mobile).

Scroll to your heart's content at [Cam's Pizza Parlor](http://aricept.github.io/frontend-nanodegree-mobile-portfolio/views/pizza.html).

<a name="tools"></a>
## Tools Used
**[Gulp.js](http://gulpjs.com)** -- used to automate processes and tasks. Plugins used:

* [gulp-uglify](https://github.com/terinjokes/gulp-uglify): minifies JavaScript. Primarily used for the pizza project.
* [gulp-imagemin](https://github.com/sindresorhus/gulp-imagemin): minifies images.
* [gulp-html-minifier](https://github.com/origin1tech/gulp-html-minifier): minifies HTML.
* [gulp-inline-source](https://github.com/fmal/gulp-inline-source): minifies and inlines specified CSS and JavaScript into an HTML document.  Used on index.html.

**[ImageMagick](http://www.imagemagick.com)** -- used to resize images and convert formats.

<a name="resources"><a>
## Resources Used
HTML5 Rocks - [Leaner, Meaner, Faster Animations with requestAnimationFrame](http://www.html5rocks.com/en/tutorials/speed/animations/)

* I had already decided I was going to utilize rAF, but this article helped solidify *how* I would use it.

Wilson Page - [Preventing 'layout' thrashing](http://wilsonpage.co.uk/preventing-layout-thrashing/)

* I had already begun moving variable declaration out of the two render loops, but this article really helped shape what further organizational improvements I could make.

StackOverflow - [Recommendation for compressing JPG files with ImageMagick](http://stackoverflow.com/questions/7261855/recommendation-for-compressing-jpg-files-with-imagemagick)

* I ran into issues with pizzeria.jpg and imagemin, so sought out some suggestions on using ImageMagick for manual conversion.

jsPerf.com

* I review many, many performance benchmarks here, especially surrounding querySelector and querySelectorAll, two methods I'd never seen previously; and various types of loops.

<a name="process"></a>
##Process & Changes
I was actually almost done with the project when I discovered Gulp, and wanted to see how different that would make my workflow, so I started all over.  Love it!

Then I got so frustrated with the pizzas I started all over AGAIN.

And this time I feel like I nailed it.

<a name="index"></a>
###index.html
With the amazing gulp-inline-source plugin (seriously, just adding the attribute `inline` in the HTML and the plugin inlines that resource, CSS or JS) , I inlined all of the CSS.  None of the JS was essential for layout, so I `async`ed all of it.  I moved the inlined JS that was already there into a separate file to decrease payload, and moved all CSS and JS references to the end of the page.

The pizzeria.jpg being used was a massive multi-megapixel image, so I resized and compressed it with ImageMagick, and moved the new pizzeria-thumb.jpg into the root /img folder.  (I tried using a Gulp plugin called gulp-inline-resize, which allowed you to add a width or height modifier to the HTML and it would dynamically create a resized/minified image with a new filename, and change the file reference in the output file, but that was a bust.)  I used gulp-imagemin to minify the rest.

I looked at the GoogleFonts stylesheet, and saw that it triggered yet another file request, so I grabbed just the section for the Latin character set and added it to style.css.

I changed all of the `tag .class` CSS selectors to just `.class` selectors, and just collapsed the tag and class into the name, ie `.tagclass`.  This minimizes DOM traversal for DOM+CSSOM creation.

During these changes, I worked in the /src folder, and had a gulp.watch task running. Every time I saved style.css or index.html, it would run a task that minified then inlined the CSS, and then piped it to another plugin than minified the HTML, and saved it to the primary folder.  Gulp is awesome.

<a name="pizza"></a>
###pizza.html
I modified my Gulp setup, since I wouldn't be inlining anything in this page.  HTML and JS were minified on save.

I started by resizing and compressing pizzeria.jpg in ImageMagick again for the new page.  I based its new dimensions off of the size of the image in a maximized Chrome on my laptop.  Also using IM, I converted pizza.png to pizza.webp, drastically reducing the filesize, and thus performance overhead for redraws.  I created a resized pizza-bg.webp for the background movers.

<a name="main"></a>
### main.js

<a name="mainglobal"></a>
####Global changes
 I changed all uses of `querySelector` to `getElementById`, and all uses of `querySelectorAll` to `getElementsByClassName`.  With large numbers of elements, like we have, the `getElement(s)` methods are more performant.

<a name="mainresize"></a>
####resizePizzas()
Optimized loop in three major ways:

* Collapsed `changeSliderLabel()` and `sizeSwitcher()` to avoid running through 2 switch statements
* Pulled dx and newwidth declaration out of the loop, and pulled the `offsetWidth` checks out of `determineDx()` and into `changePizzaSizes()` to avoid multiple layout operations.
* Wrapped the loop in a `requestAnimationFrame()`, allowing the browser to draw when ready.

This:

```javascript
function changePizzaSizes(size) {
    for (var i = 0; i < document.querySelectorAll(".randomPizzaContainer").length; i++) {
      var dx = determineDx(document.querySelectorAll(".randomPizzaContainer")[i], size);
      var newwidth = (document.querySelectorAll(".randomPizzaContainer")[i].offsetWidth + dx) + 'px';
      document.querySelectorAll(".randomPizzaContainer")[i].style.width = newwidth;
    }
  }
```

Became this:

```javascript
function changePizzaSizes(size) {
    var randPizzas = document.getElementsByClassName('randomPizzaContainer');
    var rLen = randPizzas.length;
    var currWidth = randPizzas[0].offsetWidth;
    var windowwidth = document.getElementById('randomPizzas').offsetWidth;
    var dx = determineDx(currWidth, windowwidth, size);
    var newwidth = currWidth + dx + 'px';
    requestAnimationFrame(function() {
      for (var i = 0; i < rLen; i++) {
          randPizzas[i].style.width = newwidth;
      }
    });
  }
```
<a name="mainupdate"></a>
####updatePositions()

Many of the same changes as resizing the pizzas:

* Moved variable declarations out of the loop whenever possible.
* Moved the scrollPos declaration to a separate function, and tied that to the event listener.
* `requestAnimationFrame` to allow the browser to time frames
* Moved the animation technique from `style.left` to `style.transform = translateX()`, which does not affect layout, removing a costly step from the render process.
* Limited pizza creation to the visible screen only, minimizing the number of elements we need to animate - this isn't done in this function, but has a huge impact in how it runs.

<a name="failed"></a>
##Failed Attempts :thumbsdown:

Things I tried that were not successful:

* `style.transform = scale()` for pizza resizes.  Since CSS transforms don't affect layout, this didn't have the nice collapsing effect the `style.width` changes did.
* Continuous `requestAnimationFrame()` on the moving pizzas.  Seemed to have a high overhead for no payoff.
* Appending a new `<link>` with a new class definition to resize all the pizzas without looping through them.
* Making scrollPos not a global variable by passing it to `animate()`, and then to `requestAnimationFrame(updatePositions)`.  This made the pizzas FLY.  No increase in FPS, though.
* Reverse `while` loops.  I seemed to get better response out of a regular `for` loop.  So long, and thanks for all the fish, jsPerf.

At one point, I amazingly had hit 60 FPS... and then the next day could NOT repeat it.  I was so utterly defeated.  I finally noticed I had checked "Paint" in the Timeline view in Chrome, and in mouseover it said "Has Performance overhead."  Unchecked, bam, back to 60 FPS.