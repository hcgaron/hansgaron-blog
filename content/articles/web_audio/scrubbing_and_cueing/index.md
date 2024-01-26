---
title: "Cueing and Scrubbing CSS Keyframe Animations Synced With Audio"
date: 2024-01-25
draft: false
description: "Part Four: Implementing a timeline for cueing and scrubbing css animations synchronized with audio"
tags: ["Web Audio", "animation", "JavaScript", "frontend"]
series: ["Web Audio and Animation"]
series_order: 4
---

## Review
In [part three](/articles/web_audio/audio_tag_and_scrubbing/) of this series, we updated our implementation to use the HTML5 `<audio>` tag which simplified the api of sychronizing animations with audio progress. We also implemented scrubbing which is pretty cool (if not entirely necessary for many projects).

## The Code
The completed code can be found in [this codepen](https://codepen.io/hanzymusic/pen/RwdZGNa?editors=1010).

Again, in this blog post I'll try to highlight the most important parts. Fair warning, there's a lot of extra stuff in the JavaScript portion of this codepen, mostly for illustrative purposes. If you follow this post, you'll know what it all does and it'll make sense. The core functionality is much less than the full JS portion.

## Cueing Animation Strategy
So here's the problem statement: I want to cue an animation to start at __some time__ during my audio file (not just immediately at the beginning). How do I make that happen?

{{< figure
    src="shouting_cues.png"
    alt="shouting cues"
    caption="Shouting out my cues"
>}}

Well, there are three main strategies:
- Create an animation that DOES start at the beginning of the audio file, but basically nothing happens until the place where you want to see something happen.
  - This animation could be quite a bit longer than the actual visibly animated part if you intend it to start toward the end of the audio file
- Create an animation that only lasts as long as you intend and keep it in the `paused` state until the __cue__ time, then switch the animation to a `playing` state
  - This is actually a good idea, but can make scrubbing and syncrhonizing a little harder
  - We would have to overhaul some of our core logic, but that's ok, because this is just fundamentals and we are expected to implement over top of this
- Create an animation that lasts only as long as you intend, consistent with our current approach, and compute what to update with javascript on each frame
  - This is kind of complicated, and could be JS intensive
  - As we get into performance in the next part of the series, we will want to reduce our JS burden and let the browser do more... so we shouldn't adopt a JS approach that has a higher JS burden

To be consistent with our current approach, we will choose option 1. Option 2 is best for performance, but requires more legwork. Option 3 just adds more JS burden we have to compute on the regular which can be bad for performance.

## Implementing Keyframes That Act Like Cues

Imagine I have a 60 second audio file and I want an animation to begin at the 6 second mark. I don't want an element to be visible until the 6 second mark and then I want to have it become visible and start moving.

The simplest way is to convert the "seconds" into percentages and define our keyframes to basically have time that does nothing until the correct percentage.  For example, we have this:

```css
@keyframes first-box {
  /*
  This animation will start at the beginning of the audio file.
  We don't want anything to happen until the 6 second mark of a 60 second audio file.
  So, for that first 10% of the keyframes, we just keep the opacity at 0 and do nothing.
  */
  0% {
    opacity: 0;
  }
  /* 
  Once we get to 10%, we can start implementing the rest of our keyframes.
  */
  10% {
    opacity: 0;
  }
  10.1% {
    opacity: 1;
    top: 0;
    left: 0;
  }
  55% {
    top: calc(100% - 50px);
    left: 50%;
  }
  100% {
    top: 0;
    left: calc(100% - 50px);
  }
}
```

As the file plays, we will be synchronized with these keyframes even when there is nothing happening. It'll appear to our user that the animation is cued and begins at 6 seconds (or 10% of our audio file), but really the animation started at the beginning of our audio file (without anything visible happening).

{{< figure
    src="waiting_for_my_animation_to_start.png"
    alt="Waiting for my animation to start"
    caption="Waiting for my animation to start?!"
>}}

This is the simplest approach; depending on the length of your audio file and the number of concurrent animations, this could be the easiest approach. With very long audio files and many concurrent animations, you might have some performance issues.

## Creating Keyframes With JavaScript

It would be nice to have some domain specific langues (basically some JSON structure) that describes our animation timeline (more on that in a minute). Then, we could just create these keyframes programatically using javascript.

Here's an example (don't read it too hard -- it's just some utility function):

```JavaScript
function createKeyframes(name, frames) {
    // Create a new style element
    const style = document.createElement('style');
    document.head.appendChild(style);

    // Construct keyframe rules
    let keyframeRules = `@keyframes ${name} {`;
    for (const key in frames) {
        keyframeRules += `${key}% { ${frames[key]} }`;
    }
    keyframeRules += '}';
    // Insert keyframes into the style element
    style.sheet.insertRule(keyframeRules, 0);
}
```

and you could use it like this:

```JavaScript
// First parameter is the animation name, 2nd parameter is the keyframe rules
createKeyframes('dynamicallyCreated', {
    0: 'opacity: 0;',
    20: 'opacity: 1; scale(1);',
    25: 'scale(2)',
    30: 'scale(1)'
});
```

Of course, this doesn't take advantage of a DSL really, it's almost exactly written like some css and JS hybrid. We could (and should) abstract that 2nd parameter into a nice DSL. But you get the idea of how this can be done.

Now, these utilities are great, but we don't just want to create keyframes -- we want to create animations __for some specific element__. We can define a `createAnimation` function that takes a domId (or any way to get a handle to a specific element) and the animation for that element. The we will utilize the `createKeyframes` function to create the keyframe animation, and then update the element to use it.

```JavaScript
function createAnimation(domId, animation) {
  var element = document.getElementById(domId);
  dynamicAnimationElements.push(element);
  // Names don't really need to be predictable, but you could associated it with the domId if needed
  var animationName = domId + "__" + Math.trunc((Math.random() * 1e10))
   createKeyframes(animationName, animation);
  
  element.style.animation = `${animationName} ${audioClipLengthInMilliseconds / 1000}s linear infinite`;
  element.style.animationDuration = `${audioClipLengthInMilliseconds}ms`;
  element.style.animationPlayState = 'paused';
}

// example usage of createAnimation that could be dynamically called
createAnimation('dynamic-animation-element', {
    0: 'opacity: 0; scale: 1;',
    20: 'opacity: 1; scale: 4;',
    25: 'scale: 2;',
    30: 'scale: 10;',
    35: 'scale: 2;',
    50: 'scale: 4;',
    70: 'scale: 0.5;',
    100: 'scale: 1'
})
```

Now we have something useful; and we can see how we could abstract those two parameters of `createAnimation` to some kind of domain specific language, or DSL.

{{< figure
    src="some_dsl.png"
    alt="Decoding another langues"
    caption="Some kind of new language..."
>}}

## Some Kind of DSL

Now, greater minds than mine have already done amazing work in this field, so I will point you to [this github](https://github.com/Rkaede/poopity-scoop/blob/c3c322dc331f9b004548a44c666c242e2ebe8a89/src/App.tsx#L10) and it's brilliant creator for a nice example. 

We will come back to this in our final article of the series, but notice here it makes use of `framer-motion` to define animations as a simple JavaScript data structure. Then, there's [a timeline](https://github.com/Rkaede/poopity-scoop/blob/c3c322dc331f9b004548a44c666c242e2ebe8a89/src/App.tsx#L16) that associated a dom `id` with a given animation and a time at which it should play. Brilliant!

This is just a small example, but you can see how creating a timeline of animations in a DSL can make it easy to sequence all this and, most importantly, __keeps all the animation logic outside of the actual UI__. That's great -- build some UI, then define animations separately; and have our animation engine unite the two! It's a nice 3 layer animation cake.

{{< figure
    src="animation_cake.png"
    alt="Animation cake"
    caption="Mmmmm, animation cake"
>}}

## Review

We discussed the options we have for implementing cueing of animations along a timeline. We also implemented some basic functionality that would allow us to create css keyframes programatically with JavaScript.

Those implementations can be extended / improved, but with those in place we can see how we can come up with a DSL to define what the animations are, __keeping it separate from our actual UI implementation__. Then we can see how the engine unites our UI elements and our DSL animation timeline and runs it all. Cool beans.

## Next Steps

In our final article, we will look at how a library like `framer-motion` might make a lot of our implementation simpler. Also, if we have many thing happening at once we might suffer from performance issues. We are losing a lot of performance by updating every frame with JS, and we are robbing some of the browser's CSS optimizations. We will see if doing less frequent updates in JS and leveraging the CSS `animation-play-state` might offload some of that work and give us a performance boost.