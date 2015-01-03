---
layout: post
title: The Game of Life
redirect_from:
  - /2010/11/24/the-game-of-life-2/
  - /2010/12/07/a-simple-implementation-of-life-part-1-2/
  - /2010/12/10/a-simple-implementation-of-life-part-2/
---

![A board from Conway's Game of Life](/assets/images/2010-12-10-the-game-of-life-intro1.png)

### Making games

A lot of kids who take an interest in programming really just want to make games. But only a few of those kids who want to make games ever make it past the programming part. What begins as a dance of light and color and apes and mushrooms ends with tragic flair in a semicolonic nightmare of mathematical tedium and memory leaks. This is the insidious bait-and-switch built into the reality of making games.

There is (or at least *was*) a prominent movement in computer pedagogy, whose ranks consist primarily of what I imagine to be friendly, pale men with beards and PhDs, that recommends John Conway's [Game of Life](http://en.wikipedia.org/wiki/Conway%27s_Game_of_Life) as an ideal first exercise in the discipline of games. I have long believed this conceit to be well-intentioned but insane.

At a glance there is nothing game-like at all about Life. It is a grid-based simulator for cellular automata, which description is at once concise, accurate, and wholly uninspiring. There is no interaction or narrative. It is visually interesting only inasmuch as you take the time to understand what is actually going on. And it goes without saying that it doesn't even begin to touch on the kind of violence and vicarious world-beating that comprise a young person's experience of games.

![A board from Conway's Game of Life](/assets/images/2010-12-10-the-game-of-life-intro2.png)

One of the early chapters of the first game programming book I ever bought was about Life, and it proved an effective roadblock in my career as a game programmer. In all fairness, the whole enterprise of programming was over my 15-year-old head. I was a relatively late bloomer, and it wasn't until over a decade after I first learned to write programs that I felt competent enough to work seriously on games. Nevertheless, I'll submit to you that Life is a pretty cold bath into which to chuck the novice coder. If you're just starting to make games, you're far better off doing *Pong*.

As part of a recent effort to teach myself the Python language, I at last got around to attempting an implementation of Life. Having had the experience, I can partly understand why our well-meaning bearded forebears were so enthusiastic about it as an instructional exercise. You have a collection of similar objects participating in a step-by-step, discrete simulation. You compute the current state of those objects based on their previous state. After each state is computed, it gets drawn to screen. And then you do that over and over again, until the user quits the program. Indeed, Life bears many of the structural hallmarks of a game. All it lacks is the sex appeal--small wonder, then, that it represents the academic's ideal reduction of the game form.

![A board from Conway's Game of Life](/assets/images/2010-12-10-the-game-of-life-intro3.png)

The funny part is that while I still think Life is a terrible introduction to game programming, I do think it represents a pretty good capture of non-trivial problems that are relevant to anyone who's interested in programming in general. As such, it's better to see Life as a fun, game-like exercise for people interested in technical problems, rather than as an unfun, technically problematic exercise for people interested in games.

For the rest of this post, I'll attempt to document the interesting parts of a relatively simple implementation of Conway's Life using Python. I'm trying to structure the discussion as a continuous train of thought, starting with the basic ideas and moving step-by-step through the important decisions, like how to organize the main loop and which data structures to use. This hopefully will be useful for someone who understands the basics of programming, but is still learning how to unify those concepts into a non-trivial program.

You can find the source code for [my implementation on GitHub](https://github.com/jeffomatic/life).

### A brief aside on writing code

Before we dive into the nitty-gritty, let me start with a couple of tangential remarks:

First, when I first sat down to write this program, I went straight to work on all the abstract stuff--the data structures and algorithms that would model the grid, cells, and the spawning/killing process. And for the sake of this discussion, it's simpler for me to explore those concepts sooner rather than later. But I believe that laying out a ton of simulation logic right away is a bad habit, albeit one that I indulge in a lot. At the very least, it's a good idea to build a few rudimentary drawing routines ("debug draw" in game industry parlance) that let you start running visualizations of your code right away.

It is also a good idea to be testing little parts of your program as you write them; the lads in the fancy pants refer to this as "unit testing". This will hopefully reduce the amount of debugging you have to do when your program has reached its full complexity. I was not in the habit of doing this either, which meant I had a lot of really basic problems creep out of the woodwork while I was actually running simulations, e.g., I didn't notice that my linked lists were getting corrupted until I noticed some weird spawning patterns. Anyway, do as I say, not as I do.

Second, I'm reasonably happy with the approach I came up with, but I don't want to deliver the impression that it just sprung fully-formed from my head, like Athena from the cracked skull of Zeus. Arriving at my solution involved a lot of wrong turns and revisions.

### The rules of Life

I'm not sure if there's a canonical terminology for the Game of Life, so let me quickly go over the basics, using the terms that I use in the source code.

We have a two-dimensional **grid** of **cells**. It can be as big or as small as you want.

![An empty Life grid](/assets/images/2010-12-10-the-game-of-life-board1.png)

Each cell is either **alive** or **dead**. Over the course of our simulation, a cell can go from being alive to being dead, or vice-versa. The term I use for turning a live cell to a dead one is **kill**, and the term I use for turning a dead cell to a live one is **spawn**.

The grid is seeded with live cells, arranged in any pattern we choose.

![A seeded Life grid](/assets/images/2010-12-10-the-game-of-life-board2.png)

This is the way the grid looks *now*, in the **current generation**. In the **next generation**, some cells will be killed, and other cells will be spawned.

On a 2D grid, a cell has eight **neighbors**, located directly or diagonally adjacent to it. You can think of the neighbors as lying in each of the cardinal and intermediate directions: north, north-east, east, south-east, etc.

Whether a cell gets spawned or killed depends on how many live neighbors the cell currently has. Let's refer to the number of a cell's live neighbors as its **live neighbor count**.

We will use the following rules to determine what the grid will look in the next generation.

- If a *dead* cell has **three** live neighbors in the current generation, we will spawn it; i.e., it will be alive in the next generation.
- If a *live* cell has **two** or **three** live neighbors in the current generation, we will leave it alone. It will go on its happy way into the next generation. Otherwise, we will kill it; i.e., it will be dead in the next generation.

Here are a few successive generations of our example grid.

![First generation](/assets/images/2010-12-10-the-game-of-life-board3.png)
![Second generation](/assets/images/2010-12-10-the-game-of-life-board4.png)
![Third generation](/assets/images/2010-12-10-the-game-of-life-board5.png)

Fun fact #1: this seed pattern, known as "acorn", starts with only seven live cells, but continues to expand and evolve for 5206 generations before it reaches a "stable" state, in which the grid no longer changes.

### Modeling the simulation

There are plenty of ways to represent the Game of Life in code, but it might be good to start with the most direct possible translation of the rules. From there, we can see where we might be able to improve upon it.

Let's start with a grid of boolean values called `alive`. If `alive[i, j]` is true, it means that the cell in the `i`th row and `j`th column of the grid is alive. Of course, a false value means that the cell is dead.

For the first generation of the "acorn" pattern pictured above, `alive` would look like this:

```
  0 1 2 3 4 5 6 7 8 9 10
0 f f f f f f f f f f f
1 f f f f f f f f f f f
2 f f f T f f f f f f f
3 f f f f f T f f f f f
4 f f T T f f T T T f f
5 f f f f f f f f f f f
6 f f f f f f f f f f f
```

We'll also keep another grid called `liveNeighborCount`, which contains numerical values instead of booleans. `liveNeighborCount[i, j]` tells us how many neighbors of the cell at (`i`, `j`) are alive.

For the first generation of the sequence pictured above, `liveNeighborCount` would look like this:

```
  0 1 2 3 4 5 6 7 8 9 10
0 0 0 0 0 0 0 0 0 0 0 0
1 0 0 1 1 1 0 0 0 0 0 0
2 0 0 1 0 2 1 1 0 0 0 0
3 0 1 3 3 3 1 3 3 2 0 0
4 0 1 1 1 2 2 2 2 1 0 0
5 0 1 2 2 1 0 2 3 2 0 0
6 0 0 0 0 0 0 0 0 0 0 0
```

These two different grids, one full of booleans and the other full of numbers, will be in constant conversation with one another. We'll use the state of `alive` to help us figure out what values should be in `liveNeighborCount`. Then, based on what we calculated for `liveNeighborCount`, we can figure out what the state of `alive` should be in the next generation.

First, to calculate `liveNeighborCount` from `alive`, we step through the grid one cell at a time, and see how many of the neighbors of each cell are marked TRUE in `alive`.

In pseudocode, this would look something like the following:

```
for each row "i"
for each column "j"
    set liveNeighborCount[i, j] to 0
    for each neighbor of the cell at (i, j)
        if alive[neighbor's row, neighbor's column] is true
            add 1 to liveNeighborCount[i, j]
```

One thing that gets glossed over by the psuedocode is how we determine the row and column coordinates for all the neighbors of the cell at (`i`, `j`). Hopefully this is obvious: for every cell, we can add 1 to j to move one cell to the right, and subtract 1 to move to the left.

Similarly, we can add or subtract 1 from i to move one cell up or down in the grid. Of course, we have to be a little careful when we're at the corners or edges the grid, since we can't go any further in certain directions. (Fun fact #2: Life simulations can be run on a "toroidal" grid, meaning that the neighbor relationships wrap around to the opposite edges of the grid, as in *Asteroids*.)

Now that we have an accurate count of each cell's live neighbors, we can apply Life's spawning and killing rules to produce the new generation of the grid. Those rules translate fairly directly into pseudocode:

```
for each row "i"
for each column "j"
    if alive[i, j] is true
        if liveNeighborCount[i, j] is not 2 or 3
            set alive[i, j] to false (kill!)
    else
        if neighborCount[i, j] is 3
            set alive[i, j] to true (spawn!)
```

Finally, now that `alive` represents a new generation of cells, we should draw our results:

```
Draw a blank grid

for each row "i"
for each column "j"
    if alive[i, j] is true
        draw a square at (i, j)
```

From here, we can repeat the same process over and over to display successive generations of the simulation.

### What's wrong?

The above implementation is simple enough, but it has one general flaw. Each one of the primary operations--counting neighbors, applying the rules to spawn or kill cells, and drawing live cells--happens to be preceded with the following:

```
for each row "i"
for each column "j"
```

In other words, for each operation, we are iterating through each and every cell of the grid.

The concern for the next part of this discussion is how we can go about making this process a little bit more efficient. Instead of repeatedly stepping through the entire grid, we can maintain several small lists that indicate which operations need to be performed on which cells. We'll keep a list of which cells to draw, which cells to spawn, which cells to kill, and even which cells we'll do a live neighbor count for. This takes some extra bookkeeping, but ultimately reduces the busywork.

### Optimizations

In order to speed the simulation up a little bit, or at least to make it a bit more efficient in theory, I'm going to complicate the algorithm by keeping some extra data on the side that summarize some information about the current generation. Specifically, this extra data tell us which operations need to be performed on which cells.

Wait, what were our operations again? Let's review:

1. We have to count the live neighbors for each cell. Let's refer to this operation as **examining** a cell. When we examine a cell, we count how many of its neighbors are alive.
2. We have to bring some dead cells to life. This is called **spawning** a cell.
3. We have to **kill** some cells.
4. We have to **draw** all of our live cells.

We're going to end up maintaining a separate list of cells for each of these operations. I'll discuss these one by one, but it's easiest to do it in reverse order.

### `alive` as a list

In the original implementation, we used a grid of boolean values, `alive`, to keep track of which cells were alive and which cells were dead. Let's instead treat `alive` as a *list* containing the coordinate pairs for every cell that is currently alive. We just assume that if a coordinate pair is not inside `alive`, then it represents a dead cell.

When treating `alive` as a list of coordinate pairs, rather than as a grid of booleans, our procedure for examining cells doesn't change a whole lot:

```
for each row "i"
for each column "j"
    set liveNeighborCount[i, j] to 0
    for each neighbor of the cell at (i, j)
        if neighbor is in alive
            add 1 to liveNeighborCount[i, j]
```

Spawning and killing don't look all that much different, either:

```
for each row "i"
for each column "j"
    if (i, j) is in alive
        if liveNeighborCount[i, j] is not 2 or 3
            remove (i, j) from alive (kill!)
    else
        if neighborCount[i, j] is 3
            insert (i, j) to alive (spawn!)
```

But check out how much simpler the draw operation is, now that we have an exact list of the cells that need to be drawn:

```
Draw a blank grid

for each (i, j) in alive
    draw a square at (i, j)
```

Importantly, we're no longer examining every single cell in the grid in order to determine whether or not we should draw it.

### The `toSpawn` and `toKill` lists

In our original implementation, we used two exhuastive passes through the grid to handle 1) examining our cells (i.e., counting each cell's live neighbors) and 2) applying our rules for spawning and killing cells. These had to be distinct passes, since we had to determine the live neighbor count for every cell before we could start spawning or killing anything.

Now let's try something new: during the first pass, while we are examining each cell, we'll apply our spawning/killing rules as soon as we determine an individual cell's live neighbor count. But instead of directly changing the contents of `alive`, we'll put the cells we need to spawn into a list called `toSpawn`, and put the cells we need to kill into a list called `toKill`.

The combined examining-spawning-killing pass looks something like this:

```
for each row "i"
for each column "j"
    # First, count this cell's live neighbors
    set liveNeighborCount[i, j] to 0
    for each neighbor of the cell at (i, j)
        if neighbor is in alive
            add 1 to liveNeighborCount[i, j]
    # Next, apply the spawning and killing rules
    if (i, j) is in alive
        if liveNeighborCount[i, j] is not 2 or 3
            insert (i, j) into toKill
    else
        if neighborCount[i, j] is 3
            insert (i, j) into toSpawn
```

Once this is done, we can now update the contents of `alive` by iterating through our two new lists, `toSpawn` and `toKill`:

```
for each (i, j) in toSpawn
    insert (i, j) into alive

for each (i, j) in toKill
    remove (i, j) from alive

empty out toSpawn
empty out toKill
```

Note that we have to empty out `toSpawn` and `toKill` once we've finished, or else their contents will affect the processing of the next generation.

It doesn't look like this particular modification has actually bought us very much efficiency. Sure, we've gone from having two passes through the grid to having a single pass, but that one pass does twice as much stuff at each cell. On top of that, we've piled on some additional iterating thanks to our new lists.

The efficiency advantage of organizing our algorithm like this won't be clear until we get to our final improvement, the `toExamine` list. But having a list of all the cells that get spawned and killed does have at least one neat side-effect: we can use them to display a preview of which cells will change when going to the next generation. In other words, it lets us make pictures like this:

(blue dot: `toSpawn`, red dot: `toKill`)

![First generation](/assets/images/2010-12-10-the-game-of-life-board6.png)
![Second generation](/assets/images/2010-12-10-the-game-of-life-board7.png)
![Third generation](/assets/images/2010-12-10-the-game-of-life-board8.png)

### The `toExamine` list

By this point, we've modified the simulation so that drawing, killing, and spawning all occur using a quick iteration through a list. These operations now process only the cells they have to, and no more. But *examining* cells still requires a complete pass through every cell in the grid.

Following the previous pattern, we'll keep a list of cells called `toExamine`. This list will tell us exactly which cells need to be examined, and thus eliminate the need for *any* complete passes through the grid.

But how do we know which cells need to be examined?

To answer this question, it helps to observe a useful property that emerges from the rules of Life: if none of Cell X's neighbors were spawned or killed in the *previous* generation, Cell X itself will be neither spawned nor killed in the *current* generation. In this situation, Cell X's live neighbor count in the current generation is *exactly the same* as it was in the previous generation, which means its own state won't change going into the next generation.

This implies that we don't have to bother examining Cell X as long as none of its neighbors were spawned or killed in the the previous generation. Obversely, this means that the only cells we need to examine are the *neighbors* of cells that we've spawned or killed.

Another way to think about this is that when we spawn Cell X, we have just increased the live neighbor count of each of Cell X's neighbors by one. Similarly, when we kill Cell X, we have just decreased the live neighbor count of Cell X's neighbors by one. In either case, we've *changed* the live neighbor count for each of Cell X's neighbors, and so we're forced to consider whether any of those neighbors need to be spawned or killed as a result.

The upshot for our algorithm is that every time we spawn or a kill a cell, we should insert all of its neighbors into `toExamine`. With this in mind, we can change our spawning and killing process to the following:

```
for each (i, j) in toSpawn
    insert (i, j) into alive
    for each neighbor of the cell at (i, j)
        add 1 to liveNeighborCount[neighbor]
        if neighbor isn't already in toExamine
            insert neighbor into toExamine

for each (i, j) in toKill
    remove (i, j) from alive
    for each neighbor of the cell at (i, j)
        subtract 1 from liveNeighborCount[neighbor]
        if neighbor isn't already in toExamine
            insert neighbor into toExamine

empty out toSpawn
empty out toKill
```

It's important to point out two things here:

1. Notice that each time we process a spawn or kill, we are modifying the live neighbor count for every neighboring cell. The live neighbor count is no longer something we have to recalculate from scratch at the start of each generation--it is now a *cumulative* value that changes with every spawn or kill. Of course, this means that we'll have to set the live neighbor count to zero for every cell at the start of the simulation.
2. We have to make sure a cell only gets inserted into `toExamine` once, even if more than one of its neighbors gets spawned or killed.

Once we've populated `toExamine`, we can move on to processing it:

```
for each (i, j) in toExamine
    if (i, j) is in alive
        if liveNeighborCount[i, j] is not 2 or 3
            insert (i, j) into toKill
    else
        if neighborCount[i, j] is 3
            insert (i, j) into toSpawn

empty out toExamine
```

With this modification, we've eliminated the last remaining pass through the grid. All of our operations now work by iterating through small lists of cells.

### Seeding the grid

You may have noticed that there's a bit of a chicken-and-egg problem here. We've thus far assumed our operations will occur in the following order:

1. Iterate through `toExamine`. This will determine which cells to insert into `toSpawn` and `toKill`.
2. Iterate through `toSpawn` and `toKill`. This will both insert and remove cells from `alive`. It will also determine which cells to insert into `toExamine`.
3. Draw cells using `alive`.

Step 1 and Step 2 produce what the other requires. This puts us in a bit of a pickle, since it's not clear how or where this algorithm is supposed to *start*. It seems like we either have to re-organize our steps, or be very crafty in the way we seed our grid. Or both.

The first thing to notice is that our simulation doesn't change much if we flip the ordering of Step 1 and Step 2:

1. Iterate through `toSpawn` and `toKill`. This will both insert and remove cells from `alive`. It will also determine which cells to insert into `toExamine`.
2. Iterate through `toExamine`. This will determine which cells to insert into `toSpawn` and `toKill`.
3. Draw cells using `alive`.

Think of it this way: Step 3 doesn't change the state of our simulation in any way--it merely reports the current status of the grid to the outside world. So we are really just alternating between Step 1 and Step 2. Once the simulation gets going, it really doesn't matter which of the two we treat as occurring "first".

With our re-ordered algorithm, we can seed our grid by inserting our initial live cells into `toSpawn`, rather than directly into `alive`. This will not only cause the cells to get spawned in the first step of the first generation, but it will also ensure that the neighbors of our seed cells will get examined properly.

And if we only did this much, we would be *almost* correct. When I wrote my implementation of Life, it actually took me a couple of hours to figure out why this wasn't *completely* correct. The problem here is that when `toSpawn` gets processed, only the *neighbors* of the spawned cells get inserted into `toExamine`. But imagine if we seed the grid like this:

![A grid with a single seeded cell](/assets/images/2010-12-10-the-game-of-life-board9.png)

According to the rules of Life, this seed cell should die when entering the next generation. But if we only add this cell to `toSpawn`, it will never *itself* get examined--when we spawn the cell, only its neighbors will be inserted into `toExamine`.

Thus, we can't insert the seed cells into `toSpawn` *only*. We also have to add them to `toExamine` as well. That way, we can make sure that we examine any seed cell that isn't *itself* a neighbor of another seed cell.

### Visualizing the algorithm

We've gone from an algorithm that does three complete passes through each cell of the grid to a tidy little algorithm that only processes the cells it needs to. I've written a lot of words on this, so let's review with pictures.

We start by inserting our seed cells into `toSpawn`:

![A seeded grid](/assets/images/2010-12-10-the-game-of-life-board10.png)

The seeds will be spawned into live cells in the first generation.

![First generation](/assets/images/2010-12-10-the-game-of-life-board11.png)

When `toSpawn` is processed for the first time, all of the neighbors of the seed cells will be inserted into `toExamine`. In addition to this, each of the seed cells will also be added to `toExamine`.

(light blue: `toExamine`, dark blue: `alive` AND `toExamine`):

![Cells to be examined in the first generation](/assets/images/2010-12-10-the-game-of-life-board12.png)

We check the live neighbor counts of each cell in `toExamine` to determine which cells to insert into `toSpawn` and `toKill`:

(blue dot: `toSpawn`, red dot: `toKill`)

![Cells to be killed and spawned in the first generation](/assets/images/2010-12-10-the-game-of-life-board13.png)

We spawn and kill the cells in `toSpawn` and `toKill`, leaving us with a new generation:

![Second generation](/assets/images/2010-12-10-the-game-of-life-board14.png)

Here is how `alive`, `toExamine`, `toSpawn`, and `toKill` look for this generation:

![Cells to be examined, killed, and spawned in the second generation](/assets/images/2010-12-10-the-game-of-life-board15.png)

And so on, into a third generation:

![Cells to be examined, killed, and spawned in the third generation](/assets/images/2010-12-10-the-game-of-life-board16.png)

…and then into a fourth generation:

![Cells to be examined, killed, and spawned in the fourth generation](/assets/images/2010-12-10-the-game-of-life-board17.png)

Notice that in this last image, some of the live cells appear in pure black. They were *not* inserted into `toExamine`, since none of their neighbors changed between the third and fourth generations.

### The whole simulation in pseudocode

Seeding the grid:

```
for each row "i"
for each row "j"
    set liveNeighborCount[i, j] to 0

for each (i, j) in our initial seed pattern
    insert (i, j) into toSpawn
    insert (i, j) into toExamine
```

The simulation loop:

```
loopStart:

for each (i, j) in toSpawn
    insert (i, j) into alive
    for each neighbor of the cell at (i, j)
        add 1 to liveNeighborCount[neighbor]
        if neighbor isn't already in toExamine
            insert neighbor into toExamine

for each (i, j) in toKill
    remove (i, j) from alive
    for each neighbor of the cell at (i, j)
        subtract 1 from liveNeighborCount[neighbor]
        if neighbor isn't already in toExamine
            insert neighbor into toExamine

empty out toSpawn
empty out toKill

for each (i, j) in toExamine
    if (i, j) is in alive
        if liveNeighborCount[i, j] is not 2 or 3
            insert (i, j) into toKill
    else
        if neighborCount[i, j] is 3
            insert (i, j) into toSpawn

empty out toExamine

Draw a blank grid

for each (i, j) in alive
    draw a square at (i, j)

go to loopStart
```

You can check out the Python source of my implementation [on GitHub](https://github.com/jeffomatic/life).

### Appendix: When I say "list", I actually mean "set"

In using the word "list" in the plain-old-English sense of *an unordered collection of things whose internal structure is unspecified*, I have been playing it a little fast and loose with my word choice. But as most of you are probably aware, the term "list" actually has certain structural implications to computer nerds: it could refer somewhat generically to a [linked list](http://en.wikipedia.org/wiki/Linked_list), or to a specific primitive collection type in languages such as Python or Lisp.

If we look back at the way our lists are being used in the above pseudocode, we can glean some idea of how they should behave from a structural standpoint. For example:

- When **examining** cells, we do a lot of checking to see if `alive` includes a particular coordinate pair. It would be pretty nice to be able to do this in O(1), i.e. in constant time. In other words, we would rather not have to perform an exhaustive search through the contents of `alive` until we either a) found the coordinate pair we were looking for, or b) reached the end of the list.
- When **spawning** and **killing** cells, we are inserting and removing cells arbitrarily from `alive`. We'd want this insertion and removal to occur in constant time as well. It wouldn't be great if we had to reorganize the contents of our "list" just because we removed a coordinate pair from the middle somewhere.
- When **drawing** cells, we just need to be able to iterate through `alive` very quickly.

I am unaware of any basic data structure that covers all these requirements intrinsically. The [hash table](http://en.wikipedia.org/wiki/Hash_table) comes closest: inclusion checks (#1), insertion (#2a), and removal (#2b) all work more or less in constant time.

But it's not necessarily that easy to iterate (#3) through a hash table's contents. Under the hood of your garden-variety hash table, you have a big array of bins and buckets, and trying to iterate directly through it would be painful.

Of course, you could try maintaining a linked list alongside the hash table. Since iterating through linked lists is very simple, you could keep the linked list around for the sole purpose of fulfilling the fast iteration requirement. You'd then use the hash table for inclusion checks, and also perhaps to retrieve references to the linked list's buckets. And of course, insertion and removal are simple for both data structures.

In my own implementation, I used Python's [`set`](http://docs.python.org/library/stdtypes.html#set-types-set-frozenset) data structure, which since version 2.6 is built directly into the syntax of the language. Handily, it more or less does everything we need it to do. Internally, it uses some kind of hash-centric structure, as indicated by the requirement that all insertable elements be "hashable". Moreover, its interface is very convenient for inclusion checks and for ensuring that inserted elements are unique. As for the iteration issue, Python sets are treated implicitly as iterables by the syntax of the language. I'm not sure how this is implemented, but I've assumed it's implemented in an efficient manner.