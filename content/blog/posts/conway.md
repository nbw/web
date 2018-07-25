---
title: "Conway's Webcam of Life: HTML5 Canvas, Webcam, and Video Effects"
date: 2018-07-23T22:30:43-07:00
Description: "A breakdown of Conway's Webcam of Life"
Tags: ["HTML", "Javascript", "Programming"]
Categories: ["code" ]
draft: false

---
<style>
  .screenshot {
    display: block;
    margin: auto;
  }
  .text-center {
    margin-top: 0;
    text-align: center;
  }
</style>
<link rel="stylesheet"
      href="//cdnjs.cloudflare.com/ajax/libs/highlight.js/9.12.0/styles/default.min.css">
<script src="//cdnjs.cloudflare.com/ajax/libs/highlight.js/9.12.0/highlight.min.js"></script>
<script>hljs.initHighlightingOnLoad();</script>


**Difficulty level:** Beginner to Intermediate</br>
**Languages:** HTML, Javascript


This post covers the inner workings of [Conway's Webcam of Life](https://conway.nathanwillson.com). Check it out if you haven't already.

The code is available [here](https:github.com/nbw/conway).

For the sake of simplicity, the entire project was written without any external libraries -- just good ol' Javascript (cringe).

# Introduction

[Conways Game of Life](https://en.wikipedia.org/wiki/Conway%27s_Game_of_Life) "is a cellular automaton devised by the British mathematician John Horton Conway in 1970." Yep, I'm quoting wikipedia.

There are a lot of implementations of Conway's Game of Life out there. I'd originally started by writing an implementation in Golang, and afterwards found a more graceful implementation [here](https://golang.org/doc/play/life.go).

To me, **what makes Conway's Game of Life interesting is the initial state** -- and so Conway's Webcam of Life was born.💡

---

# Content of this post

There are already so many implementations of Conway's Game of Life out there that I'm not going to spend any time explaining mine. The code is on [github](https://github.com/nbw/conway/blob/master/js/conway.js) if you're interested. Instead, I'm mostly going to dive into some of the video/canvas tricks that went into the project.

---

# How it works (aka the TLDR):

1. Get video feed from a webcam

2. Attach that video feed onto an HTML canvas object

3. Apply effects to the canvas context (like pixelation)

4. When the start button is clicked, pause the video.

5. Convert the current state of the canvas
into an array of booleans (black = true, white = false)

6. Feed that array into a conway's game of life implemention

7. Convert the output of the conway implementation (the next life cycle; an array of booleans) back into pixel values and render it on the canvas.

8. Repeat steps 6 & 7.

---

# Accessing the webcam with javascript

There are a lot of better tutorials/documentation out there so I'll mostly cover the most basic case. If you'd like to go straight to the final code, [link here](https://github.com/nbw/conway/blob/master/js/pixel_video.js)

The Web API has gone through a couple changes, so depending on what tutorial you're following the might be using [Navigator.getUserMedia()](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/getUserMedia), which has since been depricated in favor of [MediaDevices](https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getUserMedia). I'd recommend using [this polyfill](https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getUserMedia#Using_the_new_API_in_older_browsers) to accommodate older browsers.

```html
<!-- HTML -->
<video autoplay></video>
```
```javascript
// Javascript
navigator.mediaDevices.getUserMedia({
    video: {
      width:     ...,
      height:    ...,
      frameRate: ...
    }
  }
).then(function(stream) {
  let video = document.querySelector('video');
  video.srcObject = stream;
  video.onloadedmetadata = function(e) {
    video.play();
  };
}).catch(function(err) {
  // deal with an error (such as no webcam)
});


video.addEventListener('play', function() {
  // trigger business logic
}, false);
```

MediaDevices support list of properties that you can customize such as a _width_, _height_, _frame rate_, _aspect ratio_, etc.. Start with the official [w3c documention](https://w3c.github.io/mediacapture-main/getusermedia.html#constrainable-properties) for more details.

**In my experience, it's better to keep the video stream simple, do any complex image manipuation on a canvas object.** You'll find yourself hitting your head against the wall trying to manipulate the video directly.

---

# Drawing video onto a canvas

Capturing webcam stream and putting it on a canvas is actually quite straight forward. Assuming that you have an HTML canvas object that's been instantiated with width and height, then the `drawImage` method can be used on a canvas' context. A good place to capture the video is in that 'play' event listener from above.

```html
<!-- HTML -->
<video autoplay></video>
<canvas></canvas>
```

```javascript
  //javascript
  var canvas = document.querySelector('canvas');
  canvas.width  = ...;
  canvas.height = ...;

  var context = canvas.getContext('2d');

  context.drawImage(video, 0, 0, canvas.width, canvas.height);

  // video 'play' event listener
  video.addEventListener('play', function() {
    context.drawImage(this, 0, 0, canvas.width, canvas.height);
  }, false);

```

The above code works, but will only capture the video once -- like a snapshot. You'll want something recursive that'll recapture the video at a frame rate, such as:


```javascript
  function draw(video, canvas, context, frameRate) {
    context.drawImage(video, 0, 0, canvas.width, canvas.height);

    setTimeout(draw, 1/frameRate, video, canvas, context, frameRate);
  }
```

---

# Pixelating a canvas

Principle: the trick for pixelating a canvas is to scale the canvas to a smaller size, then scale it back up again.

Dig it:

```javascript
function.pixelate(image, canvas, context, pixelSize) {
  let wScaled, hScaled, scale;

  scale = 1/pixelSize;
  wScaled = canvas.width*scale;
  hScaled = canvas.height*scale;

  context.drawImage(image, 0, 0, wScaled, hScaled);
  context.mozImageSmoothingEnabled = false;
  context.imageSmoothingEnabled = false;
  context.drawImage(canvas, 0, 0, wScaled, hScaled, 0, 0, canvas.width, canvas.height);
}

```

In this case, pass `video` in as the _image_.

<img class='screenshot' src="/images/conway_pixel.png" alt="Pixelated Image">

---

# Threshold effect (black and white)

I mostly bring this one up as a plug for how easy it is to manipulate individual canvas pixels.

Principle: take an average of red, green, and blue pixel values and render them white if above a certain threshold and black if below.

```javascript
function(context, threshold, width, height) {
    let image, data, r, g, b, color;

    image = context.getImageData(0, 0, width, height);

    data = image.data;

    for (let i = 0; i< data.length; i = i+4) {
      r = data[i];
      g = data[i+1];
      b = data[i+2];

      if ((r + b + g) / 3 < threshold) {
        color = 0; // black
      } else {
        color = 255; // white
      }

      data[i] = data[i+1] = data[i+2] = color;
    }

    image.data = data;

    context.putImageData(image, 0, 0);
  }
```

Pixels are accessible through the `data` property. They can be treated in increments of 4: red, green, blue, and alpha. In this example I'm leaving alpha alone.

<img class='screenshot' src="/images/conway_threshold.png" alt="Threshold Image">

---

# Wireframe effect

This effect removes the fill from inside shapes creating a faux wireframe effect.

Principle: If a pixel has neighbours above, below, left, and right, then it can be considered to be inside a shape and can be rendered white.

The code for this one is actually quite long, so I'll just link to it [here](https://github.com/nbw/conway/blob/master/js/pixel_video.js#L204).


<img class='screenshot' src="/images/conway_wireframe.png" alt="Wireframe Image">

---

# Canvas to conway and back again

The missing piece to the puzzle is getting from an HTML canvas object to an implementation of Conway's Game of Life. Then doing the reverse once you've calculated the next life cycle.

To handle that bridge, I wrote a _canvas conway engine_, which I'll link to [here](https://github.com/nbw/conway/blob/master/js/canvas_conway_engine.js).

---

# Conclusion

This was a fun project I hope you enjoy it. Let me know if you have any questions.

---

# References

- [Streaming Your Webcam To An HTML5 Canvas Element (video)](https://www.youtube.com/watch?v=nCrQ1A2BEZ0)
- [HTML5 Video Basics](https://www.html5rocks.com/en/tutorials/video/basics/)
- [How to pixelate an image with canvas and javascript](https://stackoverflow.com/questions/19129644/how-to-pixelate-an-image-with-canvas-and-javascript)


