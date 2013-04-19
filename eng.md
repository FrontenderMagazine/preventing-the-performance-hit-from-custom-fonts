# Preventing the Performance Hit from Custom Fonts

The issue is 1) custom fonts are awesome and we want to use them 2) custom fonts slow down our pages by being large additional resources. Dealing with this has been in the air recently so I thought I'd round up some of the ideas and add thoughts of my own.

## Only load on large screens

The first idea I saw was Dave Rupert's tests on only loading @font-face on large screens. Turns out if you use @font-face but don't ever apply that font-family, the font won't be downloaded. Pretty smart, browsers. [Dave's demo][1].

**CSS**

    @font-face {
      font-family: 'Dr Sugiyama';
      font-style: normal;
      font-weight: 400;
      src: local("Dr Sugiyama Regular"), local("DrSugiyama-Regular"), url(http://themes.googleusercontent.com/static/fonts/drsugiyama/v2/rq_8251Ifx6dE1Mq7bUM6brIa-7acMAeDBVuclsi6Gc.woff) format("woff");
    }

    body {
      font-family: sans-serif;
    }
    @media (min-width: 1000px) {
      body {
        font-family: 'Dr Sugiyama', sans-serif;
      }
    }

Jordan Moore has an article on the Typekit Blog ["Fallback Fonts on Mobile Devices"][2] that uses the same thinking.

> I applied this thinking to my own project by producing two separate font kits: a “full” font kit containing all the typographical styles I originally intended to use, and a “light” kit containing fewer fonts (and, consequently, weighing significantly less). I loaded the kits via JavaScript depending on the width of the user’s screen, with the value based on the smallest breakpoint.

With Dave's technique, you wouldn't have to worry about FOUT (Flash of Unstyled Text) since it's using native @font-face and most browsers have dealt with that. Jordan's technique would be more subject to FOUT I would think, having to load the fonts after a test, but you could fix that how you always fix that with Typekit: [using `visibility: hidden` while the fonts load][3].

## Ajax in the fonts

If you're biggest concern is slowing down the render time (not necessarily the fully loaded page time), you could Ajax in the stylesheet that contains the @font-face stuff after document ready. Omar Al Zabir [has a tutorial on this][4]. (thx [Kevin][5])

**JavaScript**

    $(document).ready(function(){
      $.ajax({
        url: fontFile,
        beforeSend: function ( xhr ) {
          xhr.overrideMimeType("application/octet-stream");
        },
        success: function(data) {
          $("<link />", {
            'rel': 'stylesheet'
            'href': 'URL/TO/fonts.css'
          }).appendTo('head');
        }
      });
    });

Making sure the font files have far expires headers is important too. In order to beat FOUT here, you would add a class to the `<html>` element (immediately with JavaScript) that you would use to `visibility: hidden` what you want to hide until the fonts load, and remove it in the Ajax success callback.

## Lazy load the fonts, load on subsequent page loads after cached

Extending that idea, perhaps we could only display custom fonts if we were pretty darn sure the font files were cached. On the back-end, we check for a cookie (that we'll set ourselves later) that indicateds the fonts are cached.

**PHP**

    // Check if cookie exists suggesting fonts are cached

    if (fonts_are_cached) {
      echo "<link rel='stylesheet' href='/URL/TO/fonts.css'>";
    }

On the front-end, we'll do the exact opposite. If the cookie isn't there, we'll lazy-load those fonts and then set the cookie.

**JavaScript**

    // Check if cookie exists suggesting fonts are cached

    if (!fonts_are_cached) {

      // Don't slow down rendering
      $(window).load(function() {

        // Load in custom fonts
        $.ajax({
          url: 'URL/TO/font.woff'
        });
        $.ajax({
          url: 'URL/TO/font.eot'
        });
        // Don't actually do anything with them, just request them so they are cached.

        // Set cookie indicating fonts are cached

      });
  
    }

Not foolproof, since that cookie isn't 100% proof that font is cached. But if you set it to expire in like one day it stands a decent chance. No FOUT here though, since it either doesn't load the fonts at all or does it natively with @font-face. If you don't mind FOUT (i.e. you want to show your custom font on that first page load no matter what), you could create the `<link>` and insert the fonts stylesheet instead of just requesting the fonts.

Another alternative would be to place a data URI version of the font into localStorage and yank it out when you need it. You would create a `<style>` element, put the @font-face code in that using the data URI version of the font, and inject that. Apparently The Guardian is trying that.

TWEET

Fair warning, localStorage can be slower than cache.

TWEET

A possible advantage to using JavaScript fanciness is knowing which font versions you need.

TWEET

And how:

TWEET

## The Future

This all gets better the more information we can get our hands on regarding the clients situation.

What kind of bandwidth and latency are they getting? This is pretty hard (and heavy) to test and not super reliable even when we can. Perhaps the [network information API][6] will help someday.

What is their screen size? What are the capabilities of the browser? We can test this stuff in JavaScript, but what if it would be better to know server side? Perhaps [Client-Hints][7] will help someday.

[1]: http://codepen.io/davatron5000/pen/nrfGA
[2]: http://blog.typekit.com/2013/04/17/fallback-fonts-on-mobile-devices/
[3]: http://blog.typekit.com/2010/10/29/font-events-controlling-the-fout/
[4]: http://www.codeproject.com/Articles/462209/Using-custom-font-without-slowing-down-page-load
[5]: https://twitter.com/ilikevests/status/324593491411873792
[6]: http://www.w3.org/TR/netinfo-api/#the-networkinformation-interface
[7]: https://github.com/igrigorik/http-client-hints