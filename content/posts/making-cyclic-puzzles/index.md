---
title: Designing Cyclic Puzzles for Simon Says
date: 2024-08-08
draft: false
tags: ["game-design", "rust", "simon-says"]
cover:
  image: https://dylanj.xyz/posts/making-cyclic-puzzles/images/IMG_0054.jpeg
---

I recently participated in the [Bevy Jam 5](https://itch.io/jam/bevy-jam-5) Game Jam, where the theme was "Cycles". My take on this idea was [Simon Says](https://itch.io/jam/bevy-jam-5/rate/2853129), a puzzle game where the player character must follow a series of instructions in a loop or cycle to complete the puzzle. While designing puzzles for this game, I quickly discovered a set of interesting mathematical properties underlying the solution space that I'd like to show you.

![I think I can, I think I can](images/Aug-08-2024%2009-48-28.gif#center)

# Puzzle Components

I've played quite a few other "programming" style puzzle games like [7 Billion Humans](https://www.youtube.com/watch?v=Wo8gePOdv-k) (which is fantastic, check it out!), but they tend to incorporate many programming concepts such as conditional execution, goto statements, loops, sensors, etc. I wanted to keep my game simple to make it more approachable for non-programmers while staying true to the "Cycles" theme. This meant that whatever instructions were available must always execute in a loop so no branching / goto / etc. However, I also wanted to retain the interesting puzzle designs afforded by more complex constructs. So, I decided to try and "externalize" more complex programming concepts into environmental puzzle elements instead.

## Walls

Walls may not seem like much at first glance, but they're actually incredibly powerful as they act as a kind of conditional execution. When the character tries executes a move into a wall, it essentially becomes a no-op. This acts like an implicit `if` statement or `continue` statement:

```rs
if next_block == WALL {
  continue;
}
```

![Sometimes an obstacle is actually an opportunity](images/Aug-08-2024%2009-49-27.gif#center)

You can see in this example, the solutions is `↑↑→→` but we have a maximum of three commands, normally `↑→` wouldn't be enough to get around the corner, but thanks to the walls being placed where they are, we can conditionally execute the commands and make it around the corner. 

It also has the effect of shifting the phase of the cycle forward by one, which I'll discuss later when exploring solution symmetries.

## Ice

Ice acts as a nested loop. When a player steps onto ice, that move is repeated until they step onto a non-ice block or hit a wall:

```rs
while current_block == ICE && next_block != WALL {
  walk(direction)
}
```

![I left my skates at home](images/Aug-08-2024%2009-51-08.gif#center)

Here the solution if there was no ice would be `↑↑→→`, but because of the looping nature of the nice we end up with `↑↑↑↑→→` and overshoot. Ice also helps us break up the cyclic symmetry of plans like `↑→↓←`. The ice can allow one or more of the commands to loop ensuring we don't just end up back where we started. Without ice these kinds of solutions lead to very boring puzzles.

## Rotation

The rotation blocks are more challenging to map onto traditional programming concepts; the closest analogy would be something akin to runtime metaprogramming. It's like a program that rewrites your program by rotating all the instructions left or right. This allows the pre- and post-rotation parts of the puzzle to look very different while still having the same underlying solution. This also has an interesting interaction with walls: if your next move would push you into a wall, you stay on the rotation block and get rotated again, potentially rotating more than 90 degrees.

![I meant to go this way](images/Aug-08-2024%2009-52-15.gif#center)

## Multiple Players

The multiple player characters are the first mechanic that doesn't provide new tools but instead restricts the solution space. Since the puzzle is only completed when all players are on a finish block, despite looking more complicated, they actually drastically reduce the solution space to only solutions where their finishes coincide. You could argue this is somewhat analogous to synchronization points in multi-threaded programs.

![Like ships in the night](images/Aug-08-2024%2009-54-06.gif#center)

# Symmetries

It would be easy to bloat the game with many puzzles that are essentially the same, but I wanted each puzzle to either teach something new about the game's ruleset or explore a genuinely novel solution. For example, a puzzle solved with `→` is exactly the same as a puzzle solved by going `↑`, just rotated 90 degrees. This brings us to our first symmetry: rotation.

## Rotation

While it's immediately obvious that the plan `→` is rotationally isomorphic to `↑`, it becomes less obvious as the plans get more complex. For example, `↑→↑` is equivalent to `←↑←` and `→↓→`. To make it easier to identify puzzles as being the same, we can start by "canonicalizing" the rotation of the solution by always rotating it so the first move is `↑`.

## Reflection

Consider the solutions `↑→↑` and `↑←↑`. These are just mirror images of each other and would yield puzzles that are mirror images as well, which is not very interesting. So we should consider them the same. We can canonicalize the "parity" of a solution by finding the first `←` or `→`, and if it's a `←`, mirroring the puzzle so the first left or right is a right.

## Phase

Because walls can shift the phase of a solution, we should also consider that often phase-shifted versions of the same plans are isomorphic for levels that use walls. For example, starting with the plan `↑→↑`, if we shift the phase, it becomes `→↑↑`. However, under our previously established rules, we canonicalize its rotation to `↑←←` and then its parity to `↑→→`. It may seem counterintuitive that solutions `↑→↑` and `↑→→` are isomorphic under rotation, reflection, and phase shift, but if you spend some time thinking about it and rotating it in your head, it'll become apparent.

## Cycles

Lastly, any plan with a cycle in it is isomorphic to its smaller plan, as all plans must execute in a loop. So `↑↑↑↑` is isomorphic to `↑`.

# All Novel Solutions

With this understanding, we can now write a program that iterates over all possible solutions and maps them to their "canonical" version. I'll list each of the phase-symmetric solutions together because not all puzzles involve walls, and so they sometimes are indeed novel solutions.

## One Command Plans

As we discussed earlier, there is only a single novel one-command solution: `↑`. All other plans are just rotations of this plan.

## Two Command Plans

Initially, you might expect there to be quite a few of these as we now start with 4x4 possible plans, but there are actually only two: `↑→` and `↑↓`. `↑↑` is just `↑`, `↑←` is the reflection of `↑→`, and `↓↑` is the rotation of `↑↓`.

## Three Command Plans

You might think the solution space really opens up with three commands, but if you consider phase symmetry, there are actually only 3:

* ↑ ↑ →
  * ↑ → ↑
  * ↑ → →
* ↑ ↑ ↓
  * ↑ ↓ ↑
  * ↑ ↓ ↓
* ↑ → ↓
  * ↑ → ←
  * ↑ ↓ →

## Four Command Plans

With 4^4 (i.e., 256) possible solutions, you might expect a decent number of novel solutions. However, omitting phase-symmetric solutions for brevity, there are only 11. All of the other 245 possible solutions are symmetries of these 11 or repeated cycles of smaller solutions:

* ↑ ↑ ↑ →
* ↑ ↑ ↑ ↓
* ↑ ↑ → →
* ↑ ↑ → ↓
* ↑ ↑ → ←
* ↑ ↑ ↓ →
* ↑ ↑ ↓ ↓
* ↑ → ↑ ↓
* ↑ → ↑ ←
* ↑ → ↓ ←
* ↑ → ← ↓

# Summary

I hope you learnt something interesting today, unless you're a group theorist in which case you were probably rolling your eyes about how obvious this all was. The Jam is currently still under voting but keep an eye out once its over as I have loads more ideas for interesting environmental puzzle mechanics.