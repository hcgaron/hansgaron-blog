---
title: "Synchronize Animation To An Audio File With Web Audio"
date: 2024-01-18
draft: false
description: "Part One: Play an audio file and synchronize animation using the web audio api and requestAnimationFrame."
tags: ["Web Audio", "animation", "JavaScript", "frontend"]
series: ["Web Audio and Animation"]
series_order: 1
---

## Synchronize Audio and Animation: Part One

The title says it all; so let's cut right to it: how do we synchronize audio and animation on the web? When I say animation, I mean one of two things:
- updating layout properties at the refresh rate of the screen by manually changing some CSS properties
- defining a css animation with keyframes and controlling the current playback position through that animation programatically

In both cases, we want this to be tied tightly to an audio clock, so that the animation can start at some specified time (like a "cue" for the animation to start), and never get out of sync with the audio. We also want to be able to pause and resume.

This will be a 4 part series in which we explore some of how to do this, starting from the simplest case and going to more complex. The 4 parts will be:

1) Simple playback of audio with simple animation starting when audio starts
  - This simple animation will take the full duration of the audio clip to complete
2) Adding a pause / resume feature to what we built in part 1
3) Adding a scrubbing feature to what we built in part 2
4) Defining a more interesting animation with CSS keyframes and having the features from parts 1-3

Let's GOOOOO.

{{< figure
    src="animating_1.png"
    alt="Animating"
>}}

## The Code

All of the code for a finished product from this part of the series can be found at [this codepen](https://codepen.io/hanzymusic/pen/zYbwvRQ).

There's a lot of extra fluff in there, so this article will focus on the core functionality that drives it all. I'll add code snippets which will be similar to the ones in the codepen, but might take out some of the extra fluff to make the core functionality more clear. The extra bits will be obvious to you when you look at the finished product.

## Playing Audio With Web Audio

If you want to do anything with Web Audio, you need to begin with an [`AudioContext`](https://developer.mozilla.org/en-US/docs/Web/API/AudioContext). The `AudioContext` represents an audio processing graph, which can do many amazing things - but we don't need very much from it to play audio and sync an animation so we will use only some very basic features.

### AudioContext

So to begin, create a `new AudioContext()`.

```JavaScript
const audioContext = new AudioContext();
```

If we were using `React` we might store this in a `ref`, but this tutorial is STRAIGHT JAVASCRIPT (maybe some React at the end if I'm not burned out). So you'll notice we just store this in a global variable in our js script.

### Get Your Audio File

Now that you have an `AudioContext`, how do you play some audio with it? Well, you __can__ use an HTML5 `audio` tag to play some audio and hook it up to your audio context; we will actually do that in part 3 of this series. But an even simpler idea is *just fetch an audio file and play it directly*.

Here we use the `fetch` api to fetch some audio (you can pick any open source audio you want, but if it's hosted on some other website you might get a CORS error. I'm using sounds from freesound.org).

Basic steps are:
- Fetch the audio file
- Turn it into an `arrayBuffer`
- use `AudioContext.decodeAudioData(arrayBuffer)` to get it into a playable audio buffer
- create a new `audioBufferSource`, which is part of your audio graph, and assign the `audioBuffer` to the `buffer` property of your new `AudioBufferSourceNode`.
  - Your audio graph can have many sources (ie. you can play many things at once!), you need a source for each audio you intend to play
- __CONNECT YOUR SOURCE NODE TO YOUR AUDIO CONTEXT OUTPUT__
  - This basically routes it to your speakers (although, if we were making something fancy we could send this somewhere OTHER than speakers -- like to some other audio node for some cool effects processing)
- Calling `start` on the source takes the first param as `when` to start the playback
  - passing `0` means play NOW!
- Then, we called `requestAnimationFrame` and pass it our `updateAnimation` function (more on that in a moment) which kicks off the animation process

```JavaScript
async function playAudio(url) {
  const response = await fetch(url);
  const arrayBuffer = await response.arrayBuffer();
  
  const audioBuffer = await audioContext.decodeAudioData(arrayBuffer);

  const source = audioContext.createBufferSource();
  source.buffer = audioBuffer;
  source.connect(audioContext.destination);
  source.start(0);
  // We will define `updateAnimation` function in the next code snippet
  requestAnimationFrame(updateAnimation)
  }
  catch (err) {
    // Hey, do what you want here, I'm just logging this! You can do better!
    console.error(JSON.stringify(err));
  }
```

This is enough to get our animation playing directly in sync with our audio (it will begin when the audio begins). 

{{< figure
    src="rock_guitar_1.png"
    alt="Rock Guitar"
>}}

In this simple example, we don't allow pause or resume, so all we've done is kicked off an animation at the start of an audio file. But that's plenty good enough for part 1; now let's explore the animation in the `updateAnimation` function.

## Updating The Animation

The Web Audio api is the right choice when we need precise timings related to audio because it uses a very high precision clock (and its own thread to do all its work). In the same way, we use `requestAnimationFrame` because it is synced to the refresh rate of the screen, so it will typically be called once per screen refresh and give the smoothest possible animation.

We pass a custom `updateAnimation` function as a callback to `requestAnimationFrame`. This is passed a timestamp that is used to coordinate animation properties. It's not quite as high precision as the audio clock, but it's precise enough for smooth animation.

In `updateAnimation` we set the `animationStartTime` so we can use it in calculations later, then calculate the `animationProgress` and, since we want our animation to run for the full length of the audio clip, we calculate the current percentage of the animation. We use the percentage to update the animation values we need (here it's `translateY`, but you could update anything). 

Note that we are updating CSS manually, but later when we use CSS keyframes, we'll do it differently.

```JavaScript
let animationStartTime;

function updateAnimation(currentTime) {
    if (!animationStartTime) {
      animationStartTime = currentTime;
    }
    // Calculate animation state based on currentTime
    // Assuming animation duration matches audio duration for simplicity
    const animationProgress = (currentTime - animationStartTime) % audioClipLengthInMilliseconds;
    const percentage = animationProgress / audioClipLengthInMilliseconds;

    // We are just vertically scrolling through some content in our animation -- but
    // you can update any properties you want here. Since we are scrolling, we just update
    // the translateY.
    // Later when we use CSS keyframes, we will do it differently.
    const newTranslateY = calculateTranslateYAmount(percentage);
    updateTranslateY(animatedElement, newTranslateY)

    // Request the next frame
    requestAnimationFrame(updateAnimation);
}
```

Check out all those animation frames.

{{< figure
    src="animation_frames_2.png"
    alt="Animation Frames"
    caption="Animation Frames"
>}}

Our animation is simple, just a scrolling text in a little viewport. The rest of the code in the code pen is in service of just creating enough lines of text so that we can scroll 1 per second. We can change any audio file we want and it'll only add enough lines so that it will scroll 1 per second until the audio ends.

## Next Steps

What did we accomplish?

- Play audio with Web Audio api
- Start an animation synchronized with the audio playback
- Make the animation last the full length of the audio clip
- Synchronize the amount of progress through the animation with the amount of progress of the audio

In the next chapter we will add pause and resume functionality to the audio and animation, ensuring that they both stay in sync together!