---
layout: post
title: Introducing Badly Drawn Workouts
---

I've built a little workout app because I see these things advertised all the time on Instagram
and they are expensive, intrusive, or force you to watch an ad before you start your workout.

The requirements from my point of view are very simple. A workout consists of a sequence of
exercises, with a rest period between them, then a countdown into the next one. And it must be fun!

The first workout I picked was based on the one I've done for years and know it off by heart but I still
like having the automatic timer and a voice to read out the exercises. And to make it fun, I wanted to
include a simple animated figure for each exercise. And so, with cultural inspiration from [xkcd](https://xkcd.com/),
[Poorly Drawn Lines](https://poorlydrawnlines.com/), and [Badly Drawn Footballers](https://www.instagram.com/badlydrawnfootballers),
"Badly Drawn Workouts" was born.

It's built using plain HTML, CSS and JavaScript. Initially I used the Web Speech API to say the cues
but on Firefox on macOS it resulted in an unpleasant static noise after each utterance so at build time,
the cues are compiled into pre-recorded clips and uploaded to Cloudflare with the rest of the site.

On the site as I write, there are two workouts: 7 Minute Abs and the 5 Minute Desk Break. I've had a
lot of toil and fun creating the animations and have learned a few things that were new to me about publishing
on Cloudflare so I might write more about that in the coming weeks if I can tear myself away from building
new animations.

Give them a try at [badlydrawnworkout.co.uk](https://badlydrawnworkout.co.uk). There's no sign-up, no ads, nothing to get in the way
of your first workout.
