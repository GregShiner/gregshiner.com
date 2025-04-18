+++
title = "Let's make a nonagram solver!"
date = "2025-02-23T17:05:00-05:00"
author = "Greg Shiner"
authorTwitter = "" #do not include @
cover = ""
# tags = ["", ""]
# keywords = ["", ""]
description = "An exploration of puzzle solving and optimization"
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++
# What's a Nonagram?
Nonagram is one of my favorite puzzles. It's a picture-based logic puzzle where you have a grid of squares that must be filled or left unfilled to create a picture according to some rules.
Each puzzle has a set of numbers to be used as "hints" that lead you to the solution to the puzzle.

{{<figure
  src="nonagram_example.png"
  alt="Nonagram Example"
  caption="Source: <a href='https://en.wikipedia.org/wiki/Nonogram#Example' style='color:Blue'>Wikipedia</a>"
>}}

The hints tell you the length and the order of the line segments that make up the corresponding row or column.
So if a hint on a row is "2 4", that means that somewhere on that row there must be a span of 2 consecutive filled squares, then 4 consecutive filled squares, in that order.
The segments must be separated by at least 1 empty square.
All squares start as blank and as the game progresses, you fill squares in as either filled or with an "x" to mark that it must be empty.
I've been solving these puzzles for many years now, and I've always wondered how I would make an algorithm to solve them, and I finally decided that I was going to give it a solid stab as a bit of a challenge.

# Goals
This projects main technical goal is of course to build an algorithm that is able to solve these puzzles, but let's add some to that.
1. Solve the puzzle
2. Solve puzzles of arbitrary size
3. Solve really, really big puzzles
4. Solve really, really big puzzles in a reasonable amount of time

Aside from the technical goals, the actual main goal of this project is to just learn. Learning by doing is absolutely the best way to hone any skill, and programming is by far no exception. This post will serve as a sort of dev log as I build and improve this project.

# Terminology
Before we get into the weeds of solving a problem, it's important to define the relevant terms.
1. Square/Cell: A single box on the puzzle grid
2. Line: Either an entire row or column of squares on the grid
3. Hint/Clue: The given values on either the side or top
4. Segment: A single number in a hint
<!-- TODO: add graphics -->

# A super basic solution
The basic concept behind solving these puzzles is to repeatedly apply some rules to figure out what squares must or must not be filled in based on the hints and the current state of the game.
One observation we can make about this game is that *almost* (We will discuss the exceptions later) all puzzles can be solved just by repeatedly applying some logic to each individual row and column.
The simplest logic that can be used for this is to first determine all the possible valid states of a line.
These must be valid according to the clue and fit any constraints from the initial state of the line.
For example, if we have a clue of just 4, and an empty line of 6 squares, there are 3 possible solutions.
![Possibilities from empty line with a hint of 4](./4line.svg)
From here, we can see that the middle 2 squares of the line *must* be filled in because in all cases, those 2 squares are filled in. But in the outer squares, there are cases where they can be both empty and filled, so we have no definite information about them. In cases where we have no new information, we just keep the square blank.
![Solution from empty line with a hint of 4](./4line_sol.svg)
So, what happens when our initial line isn't empty and has some squares already solved? In this scenario there may be fewer cases that are valid.
![Solution with a hint of 4 and some initial square](./4line_sol_w_initial.svg)
Two things of interest happen now in this case.
The first is that there is only 2 possible placements of the line of 4.
If we try to place the segment further to the right, the solution will no longer be valid, because there will be 5 squares in a row instead of 4.
We also see though that in all solutions, the right-most square is empty.
This means that there is no possible way for that square to be filled in, so we can mark it as empty with an X.

Now to solve a full puzzle, you simply apply this process to each row and column and repeat until it's (most likely) solved.
There are a lot more rules you can apply, but this general notion of finding the overlap in every possible configuration of the line is enough to solve basically every puzzle. I highly encourage you to go try some of these puzzles on your own. Keep an eye out and see if you can come up with some more specific rules. If you want, the Wikipedia page has some great demonstrations of some more [Solution Techniques](https://en.wikipedia.org/wiki/Nonogram#Solution_techniques).
For now though, let's start writing down observations that we make along the way. This is a crucial step in the problem-solving process.
1. Information can be extracted for a given line only using the current state of the line and its hint. It does not depend on the states of other lines.
  a. This process will be called "refining a line"
2. A puzzle can be solved by refining each row and column repeatedly.

# We can do better
As you can imagine, the process of finding every possible solution of a line is quite slow. So how can we do better?
Even with a simple line with the clue (2, 3) and an empty line with a length of 10, we would have to compute 15 different permutations.
![All hint permutations](./AllPermutations_cropped_shrunk.png)
If what we are after is the overlap between every solution, maybe there is some way to get the overlap without actually calculating every solution.
Let's for a moment change our goal. Instead of looking for the overlap between *every* solution, let's look at just the overlap between *each segment*.
When we restrict our line refinement to this new condition, we can simplify how we calculate the overlap.
Now instead of calculating every possible solution of the line, we only have to find the possible positions of each segment.
![All segment positions](./AllPositions_cropped_shrunk.png)
In this case specifically, we went from 15 positions to calculate, down to 10.
In reality, the original method had the worst case growth of the product of the numbers of positions of each segment, (in reality, it would never hit this number because a lot of these permutations will be invalid) whereas now, we have reduced it to the sum.

Let's take a look at what it would look like to refine a line with this new strategy. We will start with an empty line and place all the segments as far left as possible. Then we will duplicate this line below it. We're going to move around the segments on this line, so we will also track the overlap between the individual segments between the lines. As the segments slide to the right across their possible positions, watch how the overlap shrinks to its refined form.
{{< video src="OverlapExplain.mp4" >}}
Note that we can only look at the overlap between the individual segments instead of the general overlap between the lines.
In this case here, the point in the middle where the red segment on the left solution overlaps with the blue segment on the right solution, we leave the cell blank.
This is despite the fact that both the left and right solutions have that square filled in.
This is because there are still many permutations of the line that have that space empty.
Looking only at the overlap between the individual segments prevents this issue.

Now notice that the line on the top is the left-most solution, meaning every segment is placed as far left as possible, hence why we label it as such.
As we slide the segments to the right as far as they can go, we end up with the right-most solution.
This means that we don't even need to calculate each possible position of each segment.
We only need to find the left and right-most solutions to refine our line.
Let's see what an algorithm might look like that just looks at these extremes.

{{< video src="OverlapAlg.mp4" >}}

This is an algorithm based on the [Simple Boxes](https://en.wikipedia.org/wiki/Nonogram#Simple_boxes) and [Simple Spaces](https://en.wikipedia.org/wiki/Nonogram#Simple_spaces) methods.
It takes the concepts from those algorithms and generalizes them to all possible states of a line, not just the beginning of the puzzle.
At a high level, the algorithm looks like this:
1. Find the left-most solution to a line (all segments placed as far left as possible)
2. Find the right-most solution to a line (all segments placed as far right as possible)
3. Any overlap between the same segment on both solutions must be filled.
4. Any gap in between the same 2 segments on both solutions must be empty.

Let's take a look at another example showing some x's by having a non-empty initial state.

{{< video src="OverlapX.mp4" >}}

Now we can see 
