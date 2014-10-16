---
layout: post
title: "Functional playground: solving the Rush Hour game in Haskell"
date: 2014-10-15 17:01:36 -0700
comments: true
sharing: true
categories: haskell functional tutorials puzzles
---
{% img right /images/rushhour.png 250 250 %}
In this blog post, I will demonstrate the functional way of solving problems using the popular [Rush Hour](http://www.thinkfun.com/rushhour) board game as an example. I will employ simple yet powerful language features that Haskell provides to produce expressive and modular programs, which are: 

- Types and abstractions
- Pattern matching and recursions
- Higher-order functions and combinators
- Function composition and currying


## Rush Hour: the game 

The Rush Hour game consists of a traffic grid in which some cars and trucks are laid out. The goal is to find a path for the Red Car to exit by moving the other blocking vehicles out of its way. Created by Nob Yoshigahara in the 70's, this game has kept people entertained and puzzled with its several levels of difficulty. 

In Rush Hour, each vehicle can move only in either horizontally or vertically. At the start, the Red Car is placed horizontally on the 3rd line from the top. The other cars and trucks are blocking Red Car's straight path to the Exit. 

Now, let's see how functional programming can be of help to solve the Rush Hour puzzle.



## Game abstraction
Thinking about the game, we can capture the locations and orientations of all vehicles at any instant as a state. Our objective is to find a state in which the Red Car is placed just in front of the Exit. Let's call such state a _goal state_. To that, we transition from the initial state to a goal state through a sequence of moves. In each move, we move a car in a specific direction. The move is valid if the new location of the car does not land off the grid and does not overlap with another car. Ok, that's enough English we can translate to Haskell. So let's create some data types and type synonyms to abstract the cars, the states and the moves. 

{% codeblock lang:haskell %}
data Orientation = H | V deriving (Show, Eq)
data Direction = L | R | U | D deriving (Show)
data Car = Car {
				x :: Int, 				-- x-coordinate: from top-left corner of grid 
				y :: Int, 				-- y-coordinate
				o :: Orientation, 		-- Vertical or Horizontal
				sz :: Int 				-- Car vs Truck
			} deriving (Show, Eq)		
type Line = String 	
type CarLetter = Char					-- Representation of vehicle 
type State = Map CarLetter Car 			-- State of grid: Map with CarLetter/Car pairs
type Move = (CarLetter, Direction)	
{% endcodeblock %}

Now, let's create some functions for moving cars. The first function `moveCar` takes a car and a direction to move and returns another car with the same orientation and size but at a new location. 

{% codeblock lang:haskell %}
moveCar :: Car -> Direction -> Car 
moveCar (Car xpos ypos H size) L = Car (xpos-1) ypos H size
moveCar (Car xpos ypos H size) R = Car (xpos+1) ypos H size
moveCar (Car xpos ypos V size) U = Car xpos (ypos-1) V size
moveCar (Car xpos ypos V size) D = Car xpos (ypos+1) V size
moveCar car _ = car
{% endcodeblock %}

The next function `move` takes a State and a Move then returns a new State. 
{% codeblock lang:haskell %}
move :: State -> Move -> State 
move state (key, dir) = Map.insert key newCar state where 
	newCar :: Car 
	newCar = moveCar (state ! key) dir
{% endcodeblock %}

In `move`, we assume that the Move is a valid one. To make sure we respect the rules of the game, we need two functions that tell us whether two cars are overlapping in the grid and whether a Car is off the grid.

{% codeblock lang:haskell %}
isOverlapping :: Car -> Car -> Bool 
isOverlapping (Car x y H sz) (Car xnew ynew H sznew) 
	| y /= ynew = False
	| y < ynew && y+sz-1 < ynew = False
	| ynew < y && ynew+sznew-1 < y = False
	| otherwise = True
isOverlapping (Car x y V sz) (Car xnew ynew V sznew) 
	| x /= xnew = False
	| x < xnew && x+sz-1 < xnew = False
	| xnew < x && xnew+sznew-1 < x = False
	| otherwise = True
isOverlapping (Car x y H sz) (Car xnew ynew V sznew) 
	| xnew < x || xnew > x+sz-1 = False 
	| ynew > y = False   
	| ynew < y && ynew+sznew-1 < y = False 
	| otherwise = True 
isOverlapping (Car x y V sz) (Car xnew ynew H sznew) 
	| x < xnew || x > xnew+sznew-1 = False 
	| y > ynew = False   
	| y < ynew && y+sz-1 < ynew = False 
	| otherwise = True 

isOffgrid :: Car -> Bool 
isOffgrid (Car x y H size) 
	| x+size-1 > gridSize = True 
	| x < 1  = True
	| y < 1 || y > gridSize = True
isOffgrid (Car x y V size) 
	| y+size-1 > gridSize = True
	| y < 1 = True
	| x < 1 || x > gridSize = True
isOffgrid _ = False
{% endcodeblock %}

## State-Space search
There are two ways to measure our ability to play the game. The first one is the number of moves that you make to exit the Red Car. The fewest moves you make, the more impressed your friends will be. The second one is the amount of time it takes to actually find the path to the exit. No matter how short the path you find, your friends will not be very impressed if you ask them to wait for hours. Since, we are going to cheat twice by using a computer and also Haskell, we will not be as much concerned about the amount of time to find a path. Instead, we will focus on finding the shortest path. So how are we going to tell Haskell to do that for us? 

The mechanism to solve Rush Hour and other similar puzzles is [State Space search](http://en.wikipedia.org/wiki/State_space_search). At any instant of the game, we can actually imagine ourselves lost in an "invisible" directed graph where each node is a State and each edge is a Move. Two States `fromState` and `toState` are linked by a Move `move` whenever we can legally move a car in `fromState` and obtain the new state `toState`. Since we are interested to find the shortest path to exit, we'll use a depth-first approach. 

Beginning with the end in mind, let's declare the function `statesearch`. It takes the following three inputs: 

1. An initial State
2. A list of States that have already been explored
3. A list of Paths to be explored

Then, `statesearch` returns a Path to the exit as output. A path is simply a tuple composed of the current state and a list of moves that led to that state from the initial one. 
{% codeblock lang:haskell %}
type Path = (State, [Move])				
{% endcodeblock %}

For clarity, let's give some meaningul names to the types of the last two inputs of `statesearch` as follows:

{% codeblock lang:haskell %}
type ExploredStates = [State]			 
type UnexploredPaths = [Path] 			
{% endcodeblock %}

In our state-space search, we also need a function `solved` that checks  if for a given state we have reached our goal. We assume that the key for the Red Car is the character `'X'`. So here, we just check if it is already in front of the exit.
{% codeblock lang:haskell %}
gridSize = 6
solved :: State -> Bool 
solved state = (x xcar + sz xcar - 1 == gridSize) where 
	xcar = state ! 'X'
{% endcodeblock %}

Another helper function that will be useful in the state-space is `generateNewMoves`, which produces a list of possible move for each car in a given state and then concatenates all possible moves. 

{% codeblock lang:haskell %}
generateNewMoves :: State -> [Move]
generateNewMoves state = concat $ map generateNew (Map.keys state) where 
	generateNew :: CarLetter -> [Move] 
	generateNew key 
		| o (state ! key) == H 	= [(key, L), (key,R)] 
		| otherwise 			= [(key, U), (key,D)] 
{% endcodeblock %}

The generated moves do not have to be valid yet. We just make sure to generate moves that correspond to the orientation of each moved car. In other words, a horizontally oriented car cannot be moved up or down and vice versa. However, we will still need later to filter the valid moves. 

### Our depth-first state-space search
Now, let's present the state-space search with depth-first traversal strategy: 

{% codeblock lang:haskell %}
statesearch :: ExploredStates -> UnexploredPaths -> Maybe Path 
statesearch _ [] = Nothing 
statesearch explored (p@(state, mvs) : paths)
	| solved state = Just p 
	| state `elem` explored = statesearch explored paths 
	| otherwise = statesearch 
					(state:explored) 
					(paths ++ nextPaths) where 
						nextPaths :: [Path]
						nextPaths = [(move state m, mvs ++ [m]) | m <- generateNewMoves state, isValid state m]
{% endcodeblock %}

To find a solution form a state `start`, we call the above function as follows: 

{% codeblock lang:haskell %}
solve :: State -> Maybe Path 
solve start = statesearch [] [(start,[])]
{% endcodeblock %}

In `solve`, we initialize the list of unexplored paths to `[(start,[])`. On the other hand, the list of explored states is initially empty. 

`statesearch` is implemented using pattern matching and recursion. The first thing it does is looking at the list of unexplored paths. If there is no more path to explore, the function returns `Nothing` which means there is no solution to the puzzle. Otherwise, we continue the search by evaluating the next path named `p@(state, mvs)` which is the head of the unexplored paths using the pattern matching: 

{% codeblock lang:haskell %}
statesearch explored (p@(state, mvs) : paths)
{% endcodeblock %}

If the final `state` in the currently evaluated path is a _goal state_, then the function declares success by returning just that path `p`. 

Next, let's look at what happens if we have not yet reached our goal. `statesearch` then checks whether the current `state` has already been explored before. In that case, the path or list of moves that led us to `state` is no good. So, we simply discard that path and move on to the next one otherwise we will fall into a cycle. This is done through the recursive call `statesearch explored paths`. 

Now, suppose that the current `state` is not a `goal state` and it has not yet been explored. Then, we continue our search by generating some valid moves. For each valid move `m`, we produce an updated state `move state m` and an updated list of moves `mvs ++ [m]` and combine these into the current path `p`. We do that for each valid move and then we concatenate the generated paths to the list of path to be explored. Note that because we adopt a breadth-first search, we must append these paths at the end of the list. If we did otherwise, it is very unlikely that we would have found the shortest path to the exit. 
After evaluating the previous state, we add it to the list of explored states and make the recursive call: 

{% codeblock lang:haskell %}
statesearch (state:explored) (paths ++ nextPaths) where 
	nextPaths :: [Path]
	nextPaths = [(move state m, mvs ++ [m]) | m <- generateNewMoves state, isValid state m]
{% endcodeblock %}
So that's how elegant a state-space search can be implemented in Haskell. `statesearch` is the core function to solve the puzzle. The rest is a representation of the problem and helper functions.

## Time to play 
Well, before we can show some descent user-interface to this puzzle. We'll need helpers functions that converts a string representation of the grid to a State and vice versa. So, we define the following functions: 
{% codeblock lang:haskell %}
type Line = String
stateToLines :: State -> [Line]
linesToState :: [Line] -> State
{% endcodeblock %}

Let's illustrate how these would work with an example without showing their implementations. Suppose our initial state prints like this: 
{% codeblock%}
-A----
-A---D
XXXC-D
---C-D
-BBB--
------
{% endcodeblock %}

Then, the function `linesToState` would return the following State when applied to the above `[Line]` representation: 
{% codeblock lang:haskell %}
Map.fromList [('A',Car {x = 2, y = 1, o = V, sz = 2}),
				('B',Car {x = 2, y = 5, o = H, sz = 3}),
				('C',Car {x = 4, y = 3, o = V, sz = 2}),
				('D',Car {x = 6, y = 2, o = V, sz = 3}),
				('X',Car {x = 1, y = 3, o = H, sz = 3})]
{% endcodeblock %}

Solving this _beginner level_ puzzle with Haskell and your supercomputer would be a real overkill. But for the sake of fun, here is the output:

{% codeblock%}
-A----
-A---D
XXXC-D
---C-D
-BBB--
------

-A----
-A---D
XXXC-D
---C-D
BBB---
------

-A----
-A---D
XXX--D
---C-D
BBBC--
------

-A----
-A----
XXX--D
---C-D
BBBC-D
------

-A----
-A----
XXX---
---C-D
BBBC-D
-----D

-A----
-A----
-XXX--
---C-D
BBBC-D
-----D

-A----
-A----
--XXX-
---C-D
BBBC-D
-----D

-A----
-A----
---XXX
---C-D
BBBC-D
-----D 
{% endcodeblock %}

Yay! Now, try it with a super tough challenge.



