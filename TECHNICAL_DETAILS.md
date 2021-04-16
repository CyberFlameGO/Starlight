Starlight Technical Details (Draft)
==
First and foremost, Starlight is a Vanilla-like light engine. I've seen
some people say it's not, however just because Starlight is fast enough
to defeat typical "light suppressors" does not mean I break Vanilla lighting.
The end result of lighting with Starlight and Vanilla will be the same, and
I have always and will always intend it to be that way. Any lighting difference
between Starlight and Vanilla is considered a bug, unless of course Vanilla
isn't lighting things properly.

This document is intended to clearly outline to programmers
interested why there is such an unbelievable difference in performance
between Starlight and the Vanilla light engine and the existing modifications
(Paper's and Phosphor's specifically) to said light engine. It's also
to highlight the differences between Starlight and Vanilla, and how
they affect Minecraft.


## Light propagation algorithm
In order to discuss light propagation I first need to give a definition
for it. Light propagation is how a light engine takes
a light level, a position, and maybe other parameters, and then
propagates one of two things: An increase in light value to neighbours,
or a decrease in light value to neighbours. Changes to lighting values
in neighbours will then cause said neighbours to queue changes
for their neighbours. Eventually no more changes are queued, which
means the original light change(s) have been "propagated."

While at first glance Vanilla and Starlight seem to propagate
light exactly the same, especially given there are no differences
in the end result, we take entirely different approaches. I'm going
to first outline how Starlight propagates, and then a simplified version
for Vanilla. Why a simplified version for Vanilla? Because Vanilla
has quite a few complexities about propagating light decreases that in some
cases will cause it to eliminate needless updates, however I don't
fully understand how it works so it is not appropriate for me to explain it.

This section will be the longest since it's the most
important factor in Starlight's performance uplift.

### Starlight propagation algorithm
Starlight's propagation algorithm is very basic. I designed the entirety
of Starlight around one goal: An extremely fast propagator. Typically
the light engine in Minecraft (even the current light engine) has had _two_ similar
light propagation algorithms: One for block lighting, and one for
sky lighting. Typically the light propagator is responsible for setting up
skylight sources, and typically the block propagator is responsible for
recognizing and propagating block sources.

Starlight only has one light propagation algorithm however. It does
recognize block sources and propagates them, but only when a special
parameter is set (otherwise it would propagate block sources for skylighting,
not good). So how are skylight sources detected and propagated? That is left
to the code using the propagator. The code using the propagator is entirely
responsible for telling the propagator what positions are supposed to be
skylight sources.

Starlight's light propagator algorithm was designed with two very
strict principles: Propagate light changes correctly and have an extremely
low cost per positional update.

Starlight achieved the first goal by using an extremely basic algorithm,
one that's even simpler than the Vanilla algorithm, and even more basic
than even 1.12's light engine.

So how does it work?

I'm going to use the case of propagating block light increases to explain
the fundamental algorithm. Light decreases are basically the same except
the algorithm is modified a bit. However, the general method is exactly the same,
so I am not going to show it.

Basically, the propagator takes a light value and a position, and for
all of its neighbours will calculate what that light value would be for them
given the neighbour opacity. If the new light value is greater, then
it will update the neighbours light value and then queue that neighbour
to propagate its new light value. The light will propagate in a
BFS manner.

Below is an example of the above algorithm used to propagate increases
in a `World`
```java
class Light {
    queue = ...; // Simple FIFO queue, like ArrayDeque
    // This queue will have hold an object that contains a position and a light value.
    
    
    public void increaseBlockLight(World world, BlockPos pos, int value) {
        if (value < 0 || value > 15) {
            throw new IllegalArgumentException();
        }
        // assume pos is immutable
        int existingLevel = world.getLightLevel(pos);
        if (existingLevel < value) {
            queue.add(new QueueEntry(pos, value));
            // this is very important: we need to set the light value
            // for positions we add into the queue. the propagator WILL NOT
            // do it for us! Remember, it only sets NEIGHBOUR light values
            world.setLight(pos, value);
            
            // now we can increase
            this.propagateIncrease(world);
        }
    }
    // This is the propagation algorithm
    public void propagateIncrease(World world) {
        while (!queue.isEmpty()) {
            QueueEntry entry = queue.poll();
            BlockPos pos = entry.pos;
            int lightValue = entry.value;
            // iterate through all of the cardinal directions: -x, +x, -y, +y, -z, +z
            for (Direction direction : AXIS_DIRECTIONS) {
                BlockPos neighbourPos = pos.offset(direction);
                // we use max because AIR and maybe others can have opacity 0, which is only useful
                // for the sky light engine. but we don't care here, since we are propagating the sources
                // we were told to.
                int currentLevel = world.getLightLevel(neighbourPos);
                if (currentLevel >= (lightValue - 1)) {
                    // quick short circuit for when the light value is already greater-than where we could set it
                    // this might seem minor but it actually reduces our block get count by 6 times!
                    // this is because it ensures we only can read block state for positions where we 
                    // _could_ set light level, i.e ones we _have not already set_
                    // Since vanilla just recalulates light, this information is lost and it cannot determine
                    // what blocks it has calculated for already...
                    continue;
                }
                BlockState neighbourState = world.getState(neighbourPos);
                int targetLevel = lightValue - Math.max(1, neighbourState.getOpacity(world, neighbourPos));
                if (targetLevel > currentLevel) {
                    // sometimes the neighbour is brighter, maybe it's a source we're propagating.
                    world.setLight(neighbourPos, targetLevel);
                    // now light has been propagated to this neighbour, so
                    // we need to queue this neighbour to propagate to its neighbours
                    queue.add(new QueueEntry(neighbourPos, targetLevel));
                }
            }
        }
    }
}
```

In order to understand why this algorithm is superior to Vanilla's,
I need to explain how Vanilla's even works.

### Vanilla propagation algorithm

Vanilla's still uses a simple FIFO queue (for explanation purposes),
however instead of trying to propagate light levels to neighbours, it
instead calculates the light level for a queued position FROM its neighbours.
So now propagateIncreases would look something more like this:
```java
class Light {
    queue = ...; // Simple FIFO queue, like ArrayDeque
    // This queue will have hold an object that contains a position. No light
    // value is stored.


    public void increaseBlockLight(World world, BlockPos pos, int value) {
        if (value < 0 || value > 15) {
            throw new IllegalArgumentException();
        }
        // assume pos is immutable
        int existingLevel = world.getLightLevel(pos);
        if (existingLevel < value) {
            // this is very important: we need to set the light value
            // in the world, and then add the queued values for the NEIGHBOURS.
            // this is because the propagator now uses queued values to 
            // calculate NEW lighting for the position.
            world.setLight(pos, value);

            // queue recalculation for neighbours
            for (Direction direction : AXIS_DIRECTIONS) {
                queue.add(new QueueEntry(pos.offset(direction)));
            }

            // now we can increase
            this.propagateIncrease(world);
        }
    }
    // This is the propagation algorithm
    public void propagateIncrease(World world) {
        while (!queue.isEmpty()) {
            QueueEntry entry = queue.poll();
            BlockPos pos = entry.pos;
            BlockState state = world.getState(pos);
            int lightValue = world.getLightLevel(pos);
            int calculatedLevel = 0;
            // iterate through all of the cardinal directions: -x, +x, -y, +y, -z, +z
            for (Direction direction : AXIS_DIRECTIONS) {
                BlockPos neighbourPos = pos.offset(direction);
                int neighbourLight = world.getLightLevel(neighbourPos);
                // get light from propagating from neighbour into our pos
                int lightFromNeighbour = neighbourLight - Math.max(1, state.getOpacity(world, pos));
                if (lightFromNeighbour > calculatedLevel) {
                    calculatedLevel = lightFromNeighbour;
                }
            }
            // now the new light value is calculated
            if (lightValue < calculatedLevel) {
                // update our light
                world.setLight(pos, calculatedLevel);
                // queue neighbours for recalculation
                for (Direction direction : AXIS_DIRECTIONS) {
                    queue.add(new QueueEntry(pos.offset(direction)));
                }
            }
        }
    }
}
 ```

In practice Vanilla has ordered queues by light level, so it always
processes the highest value queued before any else, but for this
simple example it doesn't matter. That stuff only comes into play when you
start queueing up multiple changes initially at _differing_ initial light values.


Like with Starlight, decreases are propagated by modifying the algorithm a bit.
It doesn't change the method, so I'm not going to show it. The skylight
propagator for Vanilla is also the same but has additional checks for determining
if the recalculated block should be a skylight source.

### Propagation comparison

Ok, so why is Starlight's better? It looks like they're both doing
the same thing...

Except they're not. Not even close. Vanilla is doing WAY more getLight calls
than Starlight. Why? Because for each block it updates, it is checking ALL 6
of its neighbours, but for starlight it only checks JUST ONE (the block it
propagated from). To prove this, I wrote a simple piece of code that
just counted how many calls each light propagator did:
https://gist.github.com/Spottedleaf/583b606a217ed0bdcdd7f9739f8f45b3

Output:
```
Starlight did 24535 getLightLevel calls
Starlight did 4089 setLight calls
Starlight did 4088 getState calls
Vanilla did 171739 getLightLevel calls
Vanilla did 4089 setLight calls
Vanilla did 24534 getState calls
```

Wow! `171739` getLight calls vs just `24535`! That's almost `7` times more
calls (each neighbour must fetch the light value from the world, but
there are optimisations that can be done to eliminate it, however
you only eliminate 1 call, so it would be 6 times more calls still...)!
On top of that, we reduced our block get calls from `24534`
to just `4088`, a factor of `6`!

This gets even worse when you realise Vanilla must fetch the block state for
each neighbour as well, since it needs that to do additional checks against
collision shape. Starlight avoids this by adding another field into its queued
entry indicating whether the collision shape needs to be checked at all, and
the vast majority of blocks in this game do NOT need conditional shape checks,
so in the vast majority of cases Starlight never does more block reads
than light sets. So adjusting for how Vanilla truly reads block states, it would
be something like this:
```
Vanilla did 171739 getLightLevel calls
Vanilla did 4089 setLight calls
Vanilla did 171738 getState calls
```
Yikes. So Starlight's algorithm reduces light gets by 7 times and
block reads by 6 to 42 times. At least in theory.

While this looks awful for Vanilla in theory, does this really
happen with Vanilla in practice? Well I wrote a test, and here's
the output:
```
Starlight block place
[08:13:41 INFO]: Light gets: 16523
[08:13:41 INFO]: Light sets: 4089
[08:13:41 INFO]: Block gets: 4089

Starlight block remove
[08:14:48 INFO]: Light gets: 20447
[08:14:48 INFO]: Light sets: 4089
[08:14:48 INFO]: Block gets: 4089



Paper light place:
[09:18:13 INFO]: Light gets: 28623
[09:18:13 INFO]: Light sets: 4089
[09:18:13 INFO]: Block gets: 49062

Paper light remove:
[09:18:06 INFO]: Light gets: 152079
[09:18:06 INFO]: Light sets: 4089
[09:18:06 INFO]: Block gets: 181452
```
Here's the diff I used to get the output:
https://gist.github.com/Spottedleaf/b6c366a3314e89f7c36375e554b736b7
Note that diff was applied to Tuinity, because it was far
easier to write it that way (and I can disable/enable the light
engine via config and the same diff will apply).
So technically I am comparing Starlight and Paper's changes, but
Paper doesn't make really any regressions and tests do show
Paper does perform about the same as Vanilla, at least when compared
to Starlight. So I'll be assuming that these numbers would be
exact or similar on Vanilla.

Both light engines unsurprisingly beat the theory, as there are
more optimisations made. For example, Starlight's propagation
algorithm does not check the light level of the neighbour it propagated
from. That alone wipes out 1/6th of the light get checks. Additionally, for
processing light level increases, Starlight will not add
light levels greater-than 1 to the queue. Why? Because propagating zero
to neighbours is never going to work - light values are always >= 0.
You can modify the example algorithm I wrote to see how significant that change
really is: from `24535` calls to `19819` calls. A whole 1.2x reduction, just
by adding one if statement. Further, reducing by 1/6th gives basically
exactly what we got in our real test. So Starlight's numbers definitely
check out and apply in the real world.

Vanilla really did show an improvement over the theory, which was
expected - it's far more complicated in its implementation than the
example I wrote. There's also more to consider as well, since while it
made improvements to light gets in this test, that's not the full story.
The propagator algorithm stores propagated values inside a `Long2ByteMap`,
which is not counted in my test. It's certainly not the case that a read
from that map is going to be faster than a `NibbleArray` read, so there's
definitely hidden costs in this area. Speaking of hidden costs,
Vanilla does quite a few more additional hashtable lookups _per light update_,
which Starlight does NOT do. For example, it needs to a lookup to check
if the block it's about to update is in an initialised section, it needs
to add the updated section to a set of changed sets, and it needs
to do a hashtable lookup/remove per queued value process as the queue
isn't an array based FIFO queue, it's a `LongLinkedOpenHashSet` (this is
because Vanilla can cancel pending light updates).

In any case, Vanilla still fails to come close to Starlight. Comparing
the block reads in the real test for increases shows Starlight did 12x less, and
it did _at least_ 1.7 less light level reads (although see the above paragraph
for why Vanilla probably did more). For decreases, Starlight did _44_ times less
block reads and _at least_ 7.4 times less light gets. Again, there is
additional logic not covered by these numbers in the Vanilla light engine.

Ok, but do these numbers really matter for light propagation?

During development of Starlight 0.0.1 I was benchmarking the worst-case
lighting scenario: Block changes at max world height. You should read
the `A note about light data management` section to fully understand
what I'm about explain.

I was testing a world with grass on y = 254 and bedrock at y = 0, and
was just testing removing a grass block from y 254 and then adding
it back. In Starlight 0.0.1, there was no light data management -
so Starlight would update every position from the max world height down to
zero. Every. It took 10-11ms to do the removal of the grass block and 10-11ms
to place the grass block.

Vanilla on the other hand took ~30ms and ~150ms to do these updates (I don't
remember which number correspond to what operation). Remember, Vanilla
is only updating a FRACTION of the blocks (the math works out to ~4/16 of
the blocks, since it only touches 4/16 sections). It still lost by 3x-13.6x,
even though Starlight did ~4 times more updates. So while there might be
doubts about Starlight's changes to be incapable of cancelling light updates
while processing decreases, and thereby reducing total light updates, the math
shows Starlight is about ~12 to ~50 times faster per light update from this
test. This test is realistic, as it was literally showing Starlight beating
Vanilla while doing more way more updates.

Effectively, Starlight's propagation algorithm is extremely fast. Much
faster than Vanilla's, and the theory behind it certainly backs it up.
It is doing significantly less logic per light update, so while it might
be dumber and do more light updates total in some cases, it has more
than enough margin to do the additional wasteful updates and _still_
beat Vanilla.

## A note about light data management
In 1.14 the Vanilla light engine made very significant performance improvements
to light updates at extreme heights. They managed to do this by realising
that in a lot of cases, light data was actually redundant and identical to sections
below and above. Take for example a world with grass at y = 254 and bedrock
at y = 0. Only the light around the top and the bottom of the world
actually matter - the sections inbetween will actually always be the same.

The general rule imposed is that only light data (nibblearrays) that exists within 1 square
radius (`max(abs(x2 - x1), abs(y2 - y1), abs(z2 - z1))`) of a non-empty
chunk section is going to be initialised - no other light data will exist.
If a section turns empty, then the surrounding data is possibly removed to
comply with this rule. For skylighting, this is a pretty big gain. No longer
does lighting need to be propagated fully from y 255 down to y 0 - just only
in the sections that exist.

Starlight adheres to these rules and properly propagates skylight through
sections, even if they don't exist - just like Vanilla. I've also chosen
to document this behavior because it is by far the most complicated
aspect of the new light engine principles, and it is critical to understand
this principle to understand how lighting should work in modern Minecraft.

## Chunk lighting algorithms
The next important case to consider is chunk lighting. Starlight and Vanilla
do block chunk lighting basically the same; we iterate over the light sources in
the chunk and shove them into the light propagator algorithm. Then,
an edge check is performed to bring in light from neighbours. Edge checks
are rather simple: does the light value for the position match up with what
its neighbours say it should be? If not, they need to be checked as if
the block was changed. Given the logic we do is basically the same,
I don't expect any special improvements to come from block chunk lighting
other than the faster propagation algorithm. So I'm not going to go any
further on it.

Skylight is a different story. Like I referenced earlier, the light propagator
algorithm in Starlight does not set up light sources, unlike Vanilla's.

### Skylighting in Starlight
Starlight will simply start from one block above the highest non-empty
chunk section and try to "propagate" skylight downwards. The logic is
fairly simple, skylight source can propagate through a block if its
opacity is 0 (like air or glass) and the block is not conditionally
opaque (does it need to check its shape to see if lighting can pass) in
the +y direction on the block. From there it will just shove all the
lighting sources into the light propagator algorithm and run it.

Starlight sets up light sources very efficiently on Fabric though. Instead of
actually iterating from top down and reading the blocks, it creates and
uses a bitset stored on the chunk sections themselves to note what blocks
are guaranteed opacity 0. So it can determine and use a heightmap for setting
up sources. Starlight's light propagator can also be told for a given
queued entry what neighbours it should check, and of course you can
use the heightmap to figure out what neighbours are going to be level
15 so you can exclude checking them, and in most cases exclude
even queueing the position. For example, for lighting a desert
(just flat terrain, mostly) I noticed the queued levels to propagate
were only about ~300, whereas on the Tuinity implementation they could
be ~2000 or more. There was real benchmarking decisions behind making
these changes. Tux did some async profiling for me, and showed me these results:

![Profiling Results 1](https://cdn.discordapp.com/attachments/712294045312876586/772876777189802014/Screenshot_from_2020-11-02_12-35-57.png)
![Profiling Results 2](https://cdn.discordapp.com/attachments/712294045312876586/772876777189802014/Screenshot_from_2020-11-02_12-35-57.png)

You'll notice significant time (about ~12%) is spent setting reading blocks
from the chunks setting up skylight sources. However, only 4% of the time is
actually spent reading blocks while propagating light, and a whooping 27% of
time was spent reading light levels for propagating light. This
very significant difference made me think that most of the queued values
were simply not propagating light at all, since their neighbours were most
likely skylight as well.

Effectively all the changes come down to the fact that Starlight needs to
manually setup light sources, and of course by manually setting them up
optimisations can be made. It's also the case manually setting them is much
faster than using the propagation algorithm to do it, since even in the
Tuinity implementation by manually setting them up we eliminate light level
reads entirely, whereas Vanilla will have to do them because it's using
the propagator.

### Skylighting in Vanilla
Skylighting in Vanilla is all done via the light propagator algorithm. So
where does it start? It starts by edge checking all the light data
in the chunk (initially its all level 0. It's also guaranteed there is a light
data section above the highest non-empty section in the chunk due to the
light data management).

To explain how that procedure even lights a chunk:
It might be hard to think about, but the edge checks on the light data
above the highest non-empty section will eventually turn that light data
section into full 15 which will then propagate into the sections below. So
while it's not as straightforward to see as Starlight, it gets the job done.

### Chunk lighting algorithm comparison

Why might this approach be worse than Starlight? This is because
by iterating downwards from above explicitly I will always do the minimum
number of block gets (no light value reads) and light value set calls to
initialise the skylight sources in the chunk. Meanwhile, Vanilla has to
use the inefficient propagator to do it. So while it might not seem
like a lot, typically chunk sections are only going to ever need two or three
sections worth of light propagations to become lit - but adding on
an extra section for isn't very efficient.

Vanilla will also propagate skylight into chunks that are not lit. So it's also
going to do light propagations that will be later overwritten. Starlight will
not propagate into unlit chunks. This is a concern only for writing data to
disk, but this is covered later.

Starlight in this case doesn't really make insane improvements on chunk lighting
compared to Vanilla, but it does make improvements and does do it differently
than Vanilla. However, the end result in terms of light is going to be the same.


This about concludes the major improvements Starlight does to the light engine.
I make very small improvements everywhere else in the light engine (i.e the light
propagator algorithm is very optimised), but this document isn't supposed
to go line-by-line of Starlight, just the major points. Now it is time to
discuss the major implications of the changes I've made in Starlight.

## Other Benchmarks

For reference, here are a bunch of comparisons I made back around January 2021.
Note that the explanations and summary of comparisons is included in each
video's description.

Comparing standard world gen:

Vanilla:
https://www.youtube.com/watch?v=5ygQEuGMyDc

Phosphor:
https://www.youtube.com/watch?v=RWpv7AOMfNo

Starlight:
https://www.youtube.com/watch?v=UMuSegBIBuo

[MC-162253](https://bugs.mojang.com/browse/MC-162253):

Vanilla:
https://www.youtube.com/watch?v=ECXk07XvFt4

Phosphor:
https://www.youtube.com/watch?v=6sTua6QaXSI

Starlight:
https://www.youtube.com/watch?v=57Y5wKLX7_w

Amplified world gen:

Vanilla:
https://www.youtube.com/watch?v=o5-WFpoQK_o

Phosphor:
https://www.youtube.com/watch?v=4jhzOTVpC1Y

Starlight:
https://www.youtube.com/watch?v=WczW8KmcReg

Block changes at maximum world height:

Vanilla:
https://www.youtube.com/watch?v=eDfn0Mb1ad4

Phosphor:
https://www.youtube.com/watch?v=twBL2DkJWM4

Starlight:
https://www.youtube.com/watch?v=5nVYjedJz-U

# Important Details to Note

## Chunk Save Format

You might have noticed Starlight modifies the format of light on disk.
It does this for two reasons: It needs to store whether a skylight data
section is uninitialised or absent (Vanilla conflates the two) and
Starlight does not propagate light into chunks not marked as lit. However,
in modifying the data stored, it also marks the chunk as "unlit." and add
its own special tag for whether the chunk is it. To Vanilla the data will look
like the chunk needs lighting, and to Starlight it will look "lit." So
the world save format is compatible if saved in Starlight, as it will force
Vanilla to relight the chunk. If the world is saved in Vanilla then
to Starlight the chunks will look unlit, so it will relight them.
Therefore, lighting will not break going from Starlight to Vanilla or
from Vanilla to Starlight.

## FPS Impact

### Block changes
The amount of time to propagate lighting is pretty minimal except for
high altitude scenarios. However, most people aren't up that high, so
I don't expect any changes Starlight does to actually make FPS improve
on the average - it just helps with those edge cases at maximum world
heights.

### Faster Chunk Loading and Generation
While it might seem that faster chunk loading/generation is only a good thing,
it really isn't in all cases. For example, the amplified world gen test showed
this. Chunk gen was considerably faster, however: The video clearly shows the
client is stuttering a lot more! As expected, when you throw more chunks
at the client, and then combine it with Mojang's top tier rendering code,
you get a bottleneck. Stuttering is no fun, probably much better on the
user end to just have chunks load slower if it means their FPS stays
smooth.

There's no reason the client cannot be simply improved to
simply limit the amount of chunks it tries to render per frame. As it stands,
it's not really doing that, so this problem will happen more on Starlight. So
while it's not technically my fault, it will happen more with MY changes, so
it should be noted.

However, if the client sided problems are resolved, it would mean having
larger world heights in terrain generation can be achieved without sacrificing
significant performance. Especially now with Minecraft looking to
change world height limits.

### Conclusion of changes to FPS

So overall I do not expect Starlight to improve FPS at all. If anything, it
will harm it due to the stuttering caused by generating and loading chunks faster.

## Mod Compatibility
Unsurprisingly a cutting edge change to Minecraft initially designed for
Bukkit-based servers has mod compatibility problems on platforms like
Forge and Fabric. Any mod that relies on hooking directly into the
light engine will be broken by Starlight, since Starlight is a complete
rewrite of the engine. You can find an active list of broken mods here:
https://github.com/Spottedleaf/Starlight/issues

The above issue tracker of course is not 100% complete, as it relies on
people reporting those issues there - and given it's unlikely I will
ever fix the incompatibilities, I wouldn't expect many people to even bother
reporting.