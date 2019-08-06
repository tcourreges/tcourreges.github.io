---
layout: post
title: Coders Strike Back
thumbnail: "/thumbnails/codersstrikeback.jpg"
description: "CodinGame challenge<br>Detailed report on how to reach Legend League"
repo: "https://github.com/tcourreges/HOPE"
permalink: "coders-strike-back"
---


## Introduction

[<img src="/images/codersstrikeback/codingamelogo.png" width="50%" style="float: right; padding-left: 2em;">](https://www.codingame.com/)
If you're not already familiar with the website [CodinGame](https://www.codingame.com/), I highly recommend you check it out. Its concept is pretty simple, and could be summed up as two concepts:
- [Competitive Programming](https://en.wikipedia.org/wiki/Competitive_programming): you have to write programs in order to solve problems (the input is a general test case, the output is your solution)
- [Gamification](https://en.wikipedia.org/wiki/Gamification): those problems are presented in a fun and interactive way, like a game (sometimes quite literally, as you're writing the code of a game)

The purpose of this document is to give a step by step walkthrough of how I reached Legend League in the problem [Coders Strike Back](https://www.codingame.com/multiplayer/bot-programming/coders-strike-back).
<br>My source code (C++) is available as well, in my [GitHub repository]({{ page.repo }}). Feel free to check out how I actually implemented things, should my explanations not be enough. However, I would advice against submitting my code as your own -- it would spoil the fun of actually understanding/solving the problem.

---
## Problem overview

[Coders Strike Back](https://www.codingame.com/multiplayer/bot-programming/coders-strike-back) is about a [pod race](https://en.wikipedia.org/wiki/List_of_Star_Wars_air,_aquatic,_and_ground_vehicles#Podracer), similar to the one in [Star Wars: Episode I](https://en.wikipedia.org/wiki/Star_Wars:_Episode_I_%E2%80%93_The_Phantom_Menace).

![screenshot](/images/codersstrikeback/csb.jpg)

- Every frame:
  - we receive a description of the current state of the simulation
  - we must output how we want our pod to move

- Our pod must reach the last checkpoint before the opponent.
- Additional input parameters/rules are added as we progress through leagues.
- To break out of a league, we must beat its boss (AI). The AI becomes stronger every league.

---
## Up to Bronze

Each of those steps is relatively simple, and aims at making us familiar with the problem.

### First question

- We are given basic input/output code.
- However it doesn't work, and we need to correct it (it outputs `y y` instead of `x y` for the checkpoint coordinates).

### First boss

- The boss goes faster than us, we have to manage the `thrust` value somehow.
- Always setting it to the maximum possible value does the trick.

### Wood 2

- We now have access to the **distance** and **angle** of our pod, relative to the checkpoint.
- There's a hint in pseudo-code: `if (angle<-90 or angle>90) thrust=0 else thrust=100`, ie don't accelerate while we are backwards. This implementation is enough to progress.

### Wood 1

- We can now replace our `thrust` with `BOOST` to gain a speed boost, once per race.
- Using it randomly isn't enough to get promoted -- the following implementation is:
  - is the boost available?
  - are we in the right direction (aligned with the target)?
  - is the checkpoint (arbitrarily) far enough, so we don't waste the boost?

---
## Bronze to Gold

This solution consists of several features, which individually weren't enough to break out of Bronze League. Their combination, however, got directly promoted to Gold.

### CheckpointManager (when best to use `BOOST`)

The idea:
  - I wasn't satisfied with the boost implementation submitted in Wood 1.
  - It is not explicitly said in the description, but the UI mentions "laps": even if we only discover one checkpoint at a time, they are repeated every lap.
  - This means we can **remember every checkpoint**, then have full knowledge of the track after the first lap.
  - I used it to `BOOST` during the longest step between two checkpoints. I thought I might later use it to predict trajectories, but didn't end up doing that.

Implementation:
  - To avoid bloating the main function, I implemented that logic in a separate `CheckpointManager` class.
  - I simply feed the `CheckpointManager` the current checkpoint, every turn.
  - It uses it to compute the current checkpoint index, the lap, and stores the checkpoint if it is new (during the first lap).
  - At the end of the first lap, we know every checkpoint, so we can compute the best one to `BOOST` for.

### Thrust management/angle

The pseudo-code from earlier makes sense for a small angle (< &epsilon;), and one above 90°.

However, the transition from &epsilon; to 90° feels primitive, so I tried to improve it:
  - If angle < &epsilon;, use `maxThrust` and even try to `BOOST`.
  - Otherwise, return `maxThrust` multiplied by:
    - `(1 - angle/90)` clamped in [0,1]: the more misaligned we are, the more we slow down.
    <br>(Note: it preserves the old behavior above 90°)
    - `(distanceToCheckpoint / (k*checkPointRadius))` clamped in [0,1]: we slow down as we get closer to the checkpoint.
    <br>I empirically selected the factor `k=2`.
    <br>(Note: we don't need actual distances (sqrt), squared distance are enough and better for performance)

### Rotation

There is no information on how the pod turns at this point (is there a turnrate or something?).

It is rather frustrating, as we would often like the pod to **steer more effectively** (turn faster, in advance) to avoid drifting.

I tried to achieve that by adding an **offset** to the target of the pod: `-h*podSpeed`
- `podSpeed = (position of the pod this turn) - (position the turn before)`
- I came up with it by drawing a simple sketch:
  - the direction adapts to how misaligned the pod is, and compensates that
  - the norm adapts to how much the ship is drifting away already
- I didn't know the exact norm to use, so I empirically selected `h=3`.
- I am unsure whether there is a physical explanation to it, it just happened to work well.

---
## Gold (Heuristic approach)

- In Gold league, we control a second pod as well.
- The output remains the same, but must be performed once for each pod during a turn.
- The input, however, has changed a lot, and requires consequent rewriting.
- We also have access to "Expert rules" which explain in more details how the simulation works.
- There are also 2 features introduced respectively in Bronze and Silver, that I ignored until now (the first one because I didn't know how to use it, the second one because I got promoted from Silver to Gold directly): collisions, and shield.

### Input changes

- `CheckpointManager` became obsolete:
  - we already receive every checkpoint at the start
  - each pod knows its current checkpoint id
  - we can still compute the best checkpoint in a free function
- There is no need to manually keep track of the pods' previous positions to compute their speed, as we receive their speed directly.
- The `angle` changed:
  - before, it was the checkpoint angle relative to the pod orientation
  - now it is the rotation of the pod (absolute)
  - we can compute the angle of the checkpoint direction, thanks to `acos`, then substract the new `angle` to get the old angle value.
- For the sake of readability, some refactoring seemed mandatory at this point, which led me to:
  - create a `pod` class to:
    - hold all the values of a pod
    - manage the state of the boost (we only `RequestBoost` from the outside)
    - be updated from the input, each turn
    - provide a consistent output, at the end of a turn
  - move the thrust computation to a separate function `ComputeThrust`
- I also adapted the existing algorithms to account for all the changes.


### More features

#### Collisions/Shield

- Since Silver, we can output `SHIELD` instead of the thrust. This makes us gain more weight during a collision, therefore get pushed/deviated less). However it prevents us from accelerating for the next 3 turns.
- It is easy to manage:
  - make the client call `pod.RequestShield()` (so we output `SHIELD` at the end of the turn)
  - prevent the pod from using its boost/outputting a thrust for the next 3 turns
- When to use it?
  - There is no point for pod A to use it, unless it is about to collide with a pod B.
  <br> This happens when `|A(t+1) - B(t+1)| < 2*podRadius` (where `P(t)` is the position of pod P at turn t)
  - Even then, we only really care if the collision hurts our trajectory a lot (if A is pushed from behind, it's probably beneficial).
  <br>We can check that with: `AB.AT > k` (dot product of the vectors, T is A's current target)

#### Racer/Interceptor behaviour

- Since we control a second pod, it might be worth it to come up with a strategy to make use of it.
- The idea is to split the pods between a **racer** (the pod ahead) and an **interceptor** (the one behind):
  - the **racer** just tries to win the race, as usual
  - the **interceptor** tries to intercept the best pod (ahead) from the opponent
- To know if a pod is **ahead** of another, we compare their laps / their current checkpoint / their distance to that checkpoint (in that order).
- The **interceptor** simply heads towards its target (enemy pod) when it can. When it is not possible, it heads for the target's next checkpoint to block it later.

Adding this behaviour allowed me to go from rank 1258 to 482 (out of 4386 in Gold league at the time), but wasn't enough to beat the boss.

---
## Gold to Legend (Simulation approach)

At this point, I was unsure how much I could still improve my solution, with the same heuristic approach.

However, the "Expert rules" we are given describe precisely how the pods move:

  > On each turn the pods movements are computed this way:
  >
  >- Rotation: the pod rotates to face the target point, with a maximum of 18 degrees (except for the 1rst round).
  >- Acceleration: the pod's facing vector is multiplied by the given thrust value. The result is added to the current speed vector.
  >- Movement: The speed vector is added to the position of the pod. If a collision would occur at this point, the pods rebound off each other.
  >- Friction: the current speed vector of each pod is multiplied by 0.85
  >- The speed's values are truncated and the position's values are rounded to the nearest integer.
  >
  >Collisions are elastic. The minimum impulse of a collision is 120.
  >
  >A boost is in fact an acceleration of 650. The number of boost available is common between pods. If no boost is available, the maximum thrust is used.
  >
  >A shield multiplies the Pod mass by 10.
  The provided angle is absolute. 0° means facing EAST while 90° means facing SOUTH.

  What we get from that is twofold:
  - A "move" (the action of one pod during a given turn, ie one line of the output) can be described in a much simpler (finite) way:
    - a rotation in [`-maxRotation`, `maxRotation`] (`maxRotation=18`)
    - a thrust in [`0`, `maxThrust`] (`maxThrust=100`)
    - either `BOOST`, or don't
    - either `SHIELD`, or don't
  - Given the initial state of a turn, and the moves performed, we should be able to accurately predict the outcome, ie the initial state of the next turn.

### Building the simulation

Outside of the collisions, implementing those rules in order to update the values of the pods, by simulating a turn, isn't particularly hard, although it requires some debug/fine-tuning, and my implemention seemed to be perfectly accurate.

**Collisions**, however, are the tricky part, as they are described in a rather obscure way:
  > the pods rebound off each other.

  > Collisions are elastic. The minimum impulse of a collision is 120.

- I implemented the rebound between pods A and B thanks this [wikipedia article](https://en.wikipedia.org/wiki/Elastic_collision#Two-dimensional_collision_with_two_moving_objects), describing the force applied to them.

  We can write:
  - `u = (b.position - a.position).normalized`: that's always the direction of the force
  - `m = (mA*mB) / (mA+mB)` where `mA`, `mB` are the masses of A, B (10 times more than their default value while shielding)
  - `k = (b.speed - a.speed) . u` (dot product)
  - `impulse = -2*m*k`

  This turns the speed variation into:
    ```
    a.speed += (-1/mA) * impulse * u
    b.speed +=  (1/mB) * impulse * u
    ```
  I assumed what the rules call "minimum impulse" (`minImpulse=120`) is the minimum value of what I call `impulse`. I therefore clamp it in [`-minImpulse`, `minImpulse`].

- To apply the rebound, we have to detect collisions. Not only that, we actually have to move the pods until the date `t` during which the collision happens (otherwise the previous computation won't work).
<br>This is done by computing `t` for a pair of pods p1,p2:
  - during this turn, the trajectory of a pod p is described by: `p(t) = p.position + t * p.speed`
  - a collision between p1 and p2 happens when `|p2(t) - p1(t)| < 2*podRadius`
  - we can replace `p1(t)` and `p2(t)` by their definition, then square each member.
  - this gives us a [quadratic equation](https://en.wikipedia.org/wiki/Quadratic_equation): `a*t^2 + b*t + c = 0` which we know how to solve.
    <br>Note: we want `a>0` (the distance between pods becomes shorter, ie collision), `t>0` (for obvious reasons) and we only care about the first root.
- We can then simulate a turn:
  - find the smallest duration `dt` until a pair of pods p1, p2 collide (or the turn ends)
  - move all the pods during `dt` (`p.position = p.position + dt * p.speed`), and apply the rebound between p1 and p2
  - repeat until the end of the turn (sum of the dt = 1)

I'm still not sure how accurate that prediction is (compared to how the collisions are actually computed) but it looked good enough.

### Comparing solutions

Now that we can simulate a solution (a sequence of moves for both pods), how can we tell how good it actually is?

We can build a [fitness function](https://en.wikipedia.org/wiki/Fitness_function) to quantify that, following a similar reasoning to what we did in the heuristic approach. We just need to rate the state of the pods, once we are done simulating a solution.
  - First, we can rate individual pods, in a way that always promotes a pod which is ahead:
    - `score(pod) = C * checkpointsPassed - distanceToNextCheckpoint`
    - Passing the current checkpoint should always be better than getting closer to it. Therefore we need at least `C > maxCheckpointDistance`. I selected a value about 1.5 times as long as the map diagonal.
  
    Note: Thanks to this score, we can easily tell which pod should be the racer, and which one should be the interceptor (see racer/interceptor section from earlier).
  - We want to be ahead of our opponent, the more, the better:
    ```
    aheadScore = score(myRacer) - score(opponentRacer)
    ```
  - We want our interceptor to be in the path of our opponent, by blocking its next checkpoint, just like we did earlier. We can quantify that with:
    ```
    interceptorScore = -distance(myInterceptor, opponentRacer.nextCheckpoint)
    ```
  - The final score is therefore:
    ```
    solutionRating = aheadScore * K + interceptorScore
    ```
    I selected `K=2`, to give more bias to being ahead.

### Building a solution

We do have **75ms** to execute costly computations every turn, but that still won't be enough to build a good solution through brute force. We need a smarter way to explore the solution space.
<br>A way to proceed, which was teased in the tags of the problem description, is [genetic algorithms](https://en.wikipedia.org/wiki/Genetic_algorithm). That's the approach I chose.

#### Genetic evolution
- We start with a population of `N` solutions, built randomly. Each one describes the moves for `T` turns.
- Every turn:
  - We assume our population of `N` solutions was relatively "good" for the next `T-1` turns already: we use it as a start (by shifting every solution by 1 turn).
  - Until the end of the timer, we compute as many steps of the evolution as possible:
    - duplicate, mutate, simulate and rate the `N` solutions
    - sort the `2*N` solutions we now have, by score
    - only keep the best `N`
    - repeat
  - When the timer is over, we just return the best solution we found.

#### Mutations
When a mutation occurs, we simply select one move to modify, somewhere in the solution (ie we select a turn and a pod).
<br>We modify exactly one of the following values in that move:
  - **rotation**: the new value is picked at random, by adding much more weight to `-maxRotation`, `0`, and `maxRotation`, as they intuitively seem more important
  - **thrust**: the new value is picked at random, by adding more weight to `0`, and even more to `maxThrust`
  - **shield**: opposite value
  - **boost**: opposite value

Which of the values to modify is also selected at random, the rotation and thrust having more weight.

I realized managing the boost was a waste of resource, as it can only happen once during the entire race. I fixed that by setting it on/off arbitrarily the first turn, and never worrying about it again. We could instead try to manage it in a clever way, but I didn't.

### Result

> You have been promoted to Legend League!

This implementation got promoted to Legend league, where it got the rank of #495 (out of 725).

---
## How to go further

My goal being to simply reach Legend league, I stopped there. However, my solution is far from perfect, and there are a couple ways to improve it further.

### Simulation

From my testing, the simulation is 100% accurate when it comes to anything that isn't collision/rebound.

Since the description of that last part is relatively obscure, and my predictions looked good enough, I didn't bother looking into it further, but maybe there's something to fix here.

### Performance optimization

Simply making the code more efficient allows to test more solutions every turn. This can in turn lead to a better solution, without changing anything else.

I already replaced `std::rand` with [this method described by Intel](https://software.intel.com/en-us/articles/fast-random-number-generator-on-the-intel-pentiumr-4-processor), in order to compute random numbers much faster, but profiling the code and optimizing intensive parts will probably lead to better results.

### Fitness function

It seems pretty good already, but it might be possible to consider additional things I didn't think of.

### Genetic evolution

This implementation is pretty basic:
- It starts with completely **random solutions**.
- It doesn't apply the concept of [crossover mutations](https://en.wikipedia.org/wiki/Crossover_(genetic_algorithm)) (which might be a good thing, moves being dependent on the previous ones).
- Mutations are **biased**, but maybe not in the best way.
- When a mutation occurs, it completely ignores the previous value of what it modifies. We may want the mutations to have a **wide range at the beginning of the evolution** (to explore as much as possible), then **gradually reduce that range**, to perfect the solutions already found.

Focusing on these points could improve our results.

### Not ignoring the opponent's input

I completely ignored the moves of the opponent. The simulation just assumes his pods will never rotate/thrust/shield/boost during the following turns.

Adapting the `Simulation` class, to manage the moves from the opponent as well, shouldn't be too much trouble.
<br>We could then **make our population evolve against a solution attributed to the opponent**.
<br>That solution could just be generated via an _if-else_ heuristic like what we did at first. We could also perform a genetic evolution, from the opponent's point of view, during a smaller fraction of the turn.


### Magic numbers

There are a lot of magic numbers everywhere (how many turns do we simulate, how big is our population of solutions, bias for many values...). Finding and testing a better configuration might lead to better results.

---
## Closing thoughts

- While the first leagues help getting familiar with the problem, the **transition from Silver to Gold** feels rather frustrating:
  - It breaks our previous solutions by modifying the protocol.
  - We suddenly have access to more values and don't have to resort to "clever" strategies to get them (see `CheckpointManager`), which makes a lot of code obsolete.
  - Even worse, since we get the "Expert rules" at this point, we can just discard everything we've done so far, and restart from scratch, by abusing the fact we can simulate turns.

- The problem **incentivizes a genetic algorithm** (or similar) approach, because we can just execute whatever we want for a given duration, and a purely logical, heuristic approach, will probably always lose. I have mixed feelings about this:
  - On one hand, **building the simulation** is pretty satisfying (as long as you make the correct guess regarding the rather obscure collision part).
  - On the other hand, genetic algorithms **don't actually tell you _how_ to solve the problem**, they just use all the resource you give them to come up with a _good_ solution, and trivialize the thinking process (although you still need to come up with the fitness function).

- As usual with **competitive programming**, being limited to a single file to write our code in, and only having access to [_print debugging_](https://en.wikipedia.org/wiki/Debugging#Techniques) is not a fun coding experience. It also gives bad habits regarding error handling, because without a debugger, it just becomes a waste of time and effort.

- The **simulation UI** is extremely well made, and makes solving the problem very interactive/stimulating. It would be nice if we had the option to display custom messages there, instead of `cerr`.

- Overall, I had a lot of fun solving that problem, props to CodinGame for making it!
