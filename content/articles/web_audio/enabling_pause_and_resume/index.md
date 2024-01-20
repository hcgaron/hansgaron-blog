---
title: "Enabling Pause And Resume Of Synchronized Audio And Animations"
date: 2024-01-19
draft: false
description: "Part Two: Extending our synchronized audio and animations to handle pause and resume."
tags: ["Web Audio", "animation", "JavaScript", "frontend"]
series: ["Web Audio and Animation"]
series_order: 2
---

## Review

In [part one](/articles/web_audio/animation_sync_with_audio/part_one/) of this series, we implemented a simple animation that was synchronized to the playing of an audio file. The complete code for part one can be found [in this codepen](https://codepen.io/hanzymusic/pen/zYbwvRQ).

One important point is that we have a very simple animation; the animation starts when our audio starts and it proceeds for the full length of the audio. These points won't change in this article, but we will explore more complex cases in a follow-up.

In this article, we will extend this approach further by implementing a pause and resume functionality. Both the animation and audio should be able to be paused and resumed from the location at which they were paused. They should remain in sync for any number of pause / resume invocations.

## The Code

The completed code for this feature can be found [in this codepen](https://codepen.io/hanzymusic/pen/RwdVoGv).

Just like in the first part of this series, I'll provide code snippets here but will strip out the extra stuff down to the core functionality. The codepen has a lot of extra code to handle animations and UX, but once you're familiar with the code in this article the extra parts should be obvious.

## Tracking Our Progress

To implement pause / resume, we need to track:

- the progress (elapsed time) we have played through the audio
- the progress (elapsed time) we have played through the animation

These values are the same, and we could do a little bit of conversion to track only one of these values (since our animation is the same length of our audio), but that makes an assumption that all of your animations will be the same length as your audio. We don't want to keep that assumption forever, so we will track each elapsed time separately.

In addition, the clocks used by our animation and our audio are slightly different (one is provided by the web audio api, the other directly by `requestAnimationFrame`), so it's helpful to track them separately.

Let's set up some variables (globally) to track our progress:

```JavaScript
let animationStartTime; // when on the requestAnimationFrame currentTime timestamp did animation begin on first playback?
let animationPauseOffset = 0; // how much elapsed time / progress we have played through the animation

let audioStartTime = 0; // When the current audio playback started
let elapsedAudioTime = 0; // Total accumulated audio play time across pauses

let isPlaying = false; // We will want to know this
```

Good, we can track animation with the first two variables and audio with the second two variables.

{{< figure
    src="two_clocks.png"
    alt="Two clocks"
    caption="Our two clocks"
>}}

## Caching Audio For Playback

In [part one](/articles/web_audio/animation_sync_with_audio/part_one/) we implemented a `playAudio` function which actually did too much -- it both fetched audio AND played it. That was fine for a simple example where we hit play and it just plays to completion, but if we are enabling pause and resume we only want to fetch the audio once. After fetching it, we can cache it in memory and just resume playback right away.

So first we update our `playAudio` to:
- Only fetch audio if we haven't already done so
- Set `source` to a global variable so we can call `stop` on it later
- Set `audioStartTime` to the web audio clock current time when playback begins 
- pass `elapsedAudioTime` to the `start()` function as a second parameter
  - This controls the offset of where to begin playing the file (how far into the audio source to begin playback)
- 

```JavaScript
async function playAudio(url) {
  // Note that audioBuffer is a global variable in this simple JS implementation.
  // If you were using React, you could put it in state or inside a `ref`.
  if (!audioBuffer) {
    await fetchAudioToPlay(url);
  }
  try {
  // Same with these variables, they are globals in this implementation
  // but in React put it in state or a `ref`.
  // Note that we need to be able to reference `source` again later to call
  // `stop()` on it, so it can't be local to this function (unless we returned a closure
  // we could use to stop it).
  source = audioContext.createBufferSource();
  source.buffer = audioBuffer;
  source.connect(audioContext.destination);
  audioStartTime = audioContext.currentTime;
  // The second parameter here is the `offset` into the audio source to begin playback
  source.start(0, elapsedAudioTime);
  
  // This too is a global, but good for a ref or state in React
  isPlaying = true;
  requestAnimationFrame(updateAnimation);
  }
  catch (err) {
    // Handle errors appropriately please :-)
    console.error(JSON.stringify(err));
  }
}
```

{{< figure
    src="turntable.png"
    alt="turntable"
    caption="We got our audio"
>}}

## Tracking Animation Progress

Now that we are tracking the audio start time and elapsed audio time (more on that in a bit), we also need to fix our `updateAnimation` function to handle tracking the state of our animation. If an animation frame is called and we see that we are meant to be paused, we update global variables to track:

- the `animationPauseOffset` (elapsed animation time since the first frame of the animation)
- resetting `animationStartTime`, which will be set again when we resume the animation

Every time we play the animation, we set `animationStartTime` to the `currentTime` minus the elapsed amount of animation, which is captured in `animationPauseOffset`. Imagine it like this:

- We started an animation
- We paused it, so we captured how much time has elapsed (stored in `animationPauseOffset`)
- When we resume the animation, we just tell it that the animation start time is __right now__ - __however much of the animation has elapsed__

```JavaScript
function updateAnimation(currentTime) {
    if (!isPlaying) {
      animationPauseOffset = currentTime - animationStartTime;
      animationStartTime = undefined;
      return;
    }
    if (!animationStartTime) {
      animationStartTime = currentTime - animationPauseOffset;
    }
      const elapsedAnimationTime = currentTime - animationStartTime;

    // Calculate animation state based on currentTime
    // Assuming animation duration matches audio duration for simplicity
    const animationProgress = elapsedAnimationTime % audioClipLengthInMilliseconds;
    const percentage = animationProgress / audioClipLengthInMilliseconds;

    const newTranslateY = calculateTranslateYAmount(percentage);
    updateTranslateY(animatedElement, newTranslateY)

    // Request the next frame
    requestAnimationFrame(updateAnimation);
}
```

{{< figure
    src="flip_book.png"
    alt="flip_book"
    caption="Animation"
>}}

## Implementing Pause

Now we just need a way to actually pause everything. `pauseAudio` is simple and does just that. A few notes:
- `source.stop()` is how we pause audio; it's not really a "pause" -- just "stop and track where we stopped" so we can start there later
- We increment the `elapsedAudioTime` every time we pause
  - Remember that `audioStartTime` gets a new value every time we resume playback, so `audioContext.currentTime - audioStartTime` is always equal to the amount of elapsed audio since the last time we hit play
- Of course, we set `isPlaying` so animations will pause on the next frame and our toggle switch for play / pause knows what to do the next time it is called

```JavaScript
function pauseAudio() {
  // This is why we needed to store `source` globally, so we can stop it here
  source.stop();
  // Update accumulatedTime with the amount of audio played in this session
  elapsedAudioTime += audioContext.currentTime - audioStartTime;
  isPlaying = false;
}
```

{{< figure
    src="pause.png"
    alt="pause"
    caption="pause"
>}}

## Adding A Toggle

```JavaScript
function togglePlayPause() {
  if (!isPlaying) {
    playAudio();    
  } else {
    pauseAudio();
  }
}
```

{{< figure
    src="toggle.png"
    alt="toggle"
    caption="Press it"
>}}

## Next Steps

What did we accomplish?

- Implement a pause / resume and track the animation and audio position
- Fetch our audio more intelligently

This is great because we need this ability. But questions arise: 
- what if the animation shouldn't start at the beginning of the audio playback, but at some specified point in the audio file?
- what if the animation isn't the same duration as the audio?
- what if we want to animate multiple things at once?

We will build up to this; but if we have different animation "cues" along points in our audio file, it'll make it easier if we are able to scrub through our audio and animations. We can do that in the next part of our series.