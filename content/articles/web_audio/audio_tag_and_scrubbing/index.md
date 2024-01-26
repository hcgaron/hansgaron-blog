---
title: "Synchronize HTML5 Audio With Animations"
date: 2024-01-20
draft: false
description: "Part Three: The simplicity of the HTML5 <audio> tag and synchonizing animations"
tags: ["Web Audio", "animation", "JavaScript", "frontend"]
series: ["Web Audio and Animation"]
series_order: 3
---

## Review

In [part two](/articles/web_audio/enabling_pause_and_resume/) of this series, we worked with the web audio api and implemented pause and resume.

In this article, we will go simplify things a bit for our use case. If you want to do any kind of complex audio tasks (for example DSP, audio routing, mixing, multiple sources, etc...), you'll need to work directly with the web audio api. But, if we just need to play a single audio file and synchronize animation, we can work with the HTML5 `<audio>` tag.

## The Code

The completed code can be found in [this codepen](https://codepen.io/hanzymusic/pen/PoLjGoK?editors=1010).

Just like in the first two parts of this series, I'll provide code snippets here but will strip out the extra stuff down to the core functionality. The codepen extra code to handle animations etc..., but once you're familiar with the code in this article the extra parts should be obvious.

## Updating To The HTML5 `audio` Tag

One nice feature of HTML5 is the [`HTMLMediaElement`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement), which is an abstract interface inherited by the HTML5 `<audio>` tag. This provides an easy way to load audio (the browser will handle it for us), events to which we can assign various listeners, and the ability to embed it with __controls__ into our page. This gives us a rudimentary scrubber for free:

```html
  <audio src="https://cdn.freesound.org/previews/571/571367_2282212-lq.mp3" controls id="audio-element" />
```

Now that we are using the browser to fetch our audio and play / pause, we can get rid of a lot of code. Also, will have a much easier time tracking the percentage complete of our audio file.

## Tracking Animation State

{{< figure
    src="tracking_animation_state.png"
    alt="Tracking animation state"
>}}

To track the state of our audio and animations, we will need three variables:

```JavaScript
const audioElement = document.getElementById('audio-element');
let isPlaying = false;
let isAnimating = false;
```

Even though we can use the `audioElement.paused` property to see if it's actually playing, to allow us to scrub through the animations with the audio controls we will need some variable, and abusing the `isPlaying` seems like a reasonable choice.

We can use the `loadedmetadata` event to trigger a callback that will set up our animation (adds a bunch of lines based on the duration of the audio that has been loaded):

```JavaScript
function onLoadedMetadata() {
  // audioElement.duration is in seconds, and we wanted it in milliseconds
  const audioClipLengthInMilliseconds = audioElement.duration * 1000; 
  // This is just some silly thing we do to create enough lines to scroll through 1 line per second during playback
  addManyLinesToScrollThroughAtOneLinePerSecond(audioClipLengthInMilliseconds)
}

audioElement.addEventListener('loadedmetadata', onLoadedMetadata);
```

We want to update `isPlaying` when play / pause happens because `isPlaying` controls whether or not we will request another animation frame inside `updateAnimation`. If `isPlaying` is false, we stop the animation loop. 

Also notice how simpler out `percentage` calculation has become to see how far along our animation (and our audio) we are on this animation frame.

We also set `isAnimating` to `true` on each animation frame (we could find a way to do this only once, but this is a cheap operation) and only set it back to `false` if we break out of the animation loop in `updateAnimation`.

```JavaScript
// Function to update animation
function updateAnimation(currentTime) {
    if (!isPlaying) {
      isAnimating = false;
      return;
    }
    isAnimating = true;
    // Calculate animation state based on currentTime
    // Assuming animation duration matches audio duration for simplicity
    const percentage = audioElement.currentTime / audioElement.duration;

    const newTranslateY = calculateTranslateYAmount(percentage);
    updateTranslateY(animatedElement, newTranslateY)

    // Request the next frame
    requestAnimationFrame(updateAnimation);
}
```

But, one thing that's not obvious is the early returns from these callbacks below:

- onAudioPlay will call `requestAnimationFrame` to kick off the animation loop. But we are already in an animation loop (like perhaps because the user hit play, scrubbed through the audio, then let go and playback started again) we don't want to call `requestAnimationFrame` again or we will suffer a performance hit because we will be doing twice the compute on every animation frame.

So we are tracking if we are animating with the `isAnimating` variable and, if we are already animating when the audio begins to play, we just set `isPlaying` to true and skip the call to the animation loop (saving us precious compute).

{{< figure
    src="precious_compute.png"
    alt="Precious compute"
    caption="Precious, precious compute"
>}}

```JavaScript
function onAudioPlay() {
  isPlaying = true;
  if (isAnimating) {
    return;
  }
  requestAnimationFrame(updateAnimation);
}

audioElement.addEventListener('play', onAudioPlay);
```

We do a similar thing with `onAudioPause`; we want to allow the animation to visually move along with the scrubber, but as we scrub the `onAudioPause` callback will be called. Here we check if the audioElement is currently `seeking` and if it is, we just return. Otherwise we can set `isPlaying` to false.

```JavaScript
function onAudioPause() {
  if (audioElement.seeking) {
    return;
  }
    isPlaying = false;
}

audioElement.addEventListener('pause', onAudioPause);
```

## Scrubbing

{{< figure
    src="scrubbing.png"
    alt="Scrubbing"
    caption="Scrubbing"
>}}

To allow scrubbing through the animation visually, we have to add listeners to the `seeking` and `seeked` events on the audio tag (see below). 

Also, what is odd is that the `seeking` and `seeked` events don't happen exactly at intuitive times. I would have thought `seeking` starts when the user clicks the mouse button on the thumb of the scrubber and stays true until the user releases the mouse, but in practice I've found that when the user stops actually scrubbing (moving left / right), `seeking` goes back to false and `seeked` event fires. For this reason, we do a little debounce of the `onSeeked` event, since it seems to happen much too often during scrubbing.

In `onSeeking` we want to make sure to set `isPlaying` to true (if it isn't already) so our animation loop will continue. Then, if we are already animating we don't need to kick off the animation loop again, but if we aren't already animating we will start the animation loop again.

In `onSeeked`, if the audio element is paused we can just set `isPlaying` to false. This is useful because if the user played audio, scrubbed some, then let go, it'll go back to playing right away. But if they were paused and started scrubbing, we want to animate the scrubbing but then the audio remains paused after the scrubbing, so we want to end the animation loop and save us compute cycles.

```JavaScript
function onSeeking() {
  if (isPlaying) {
    return;
  }
  isPlaying = true;
  if (isAnimating) {
    return;
  }
  requestAnimationFrame(updateAnimation);
}

function onSeeked() {
    if (audioElement.paused) {
      isPlaying = false;
    }
}

const deouncedOnSeeked = debounce(onSeeked, 200);

audioElement.addEventListener('seeking', onSeeking);
audioElement.addEventListener('seeked', deouncedOnSeeked);
```

If you're confused about these optimizations, just put a `console.log` at the beginning of `updateAnimation` and comment out some of these optimizations and you'll see an ever-increasing log in your console even when there's no audio or animation playing!

And that's it! Now we can play, pause, seek, scrub, use browser to fetch, etc... Pretty cool!

{{< figure
    src="making_scrubbing_cool.png"
    alt="Making scrubbing cool"
    caption="We made scrubbing cool"
>}}

## Review

We simplified things for our use case:

- We need only a single audio file, no DSP, and one output destination, so we can use the `<audio>` tag for our use case
- We updatd to the `<audio>` tag and don't need to work with web audio api
- We simplified our code a lot with fewer things to track
- We implemented scrubbing through animation when user controls the audio slider
- We optimized our animation loop to save us compute and not double-process each animation frame

## Next Steps

We still have one thing that might be useful. We want to be able to `cue` animations along a timeline, and we might not want our animations to be the full length of the audio file! (That second point is sort of an implementation detail... but from the user's perspective, perhaps something will only happen visually during a small part of the audio file).

We also might want some kind of DSL or api to define animations to be added to our timeline. We can use this to abstract away the implementation detail of all this logic from any UI we build. 

Then, inevitably, we will have to think about performance. We will explore a couple of ideas to improve the performance of our current approach, then we can look at some off the shelf solutions. To achieve this we will pull in a dependency to make our lives much easier; because drawing all these animation frames in real time is a lot on the wrist!

Let's do it!