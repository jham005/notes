---
title: "Snakes and ladders"
description: A simulation of a classic children's game
date: '2020-03-27'
---

In this exercise, we will create a simulation of the classic
children's game of Snakes and Ladders. 

[comment]: # (../../../../toc.sh)

## Contents
- [Find yourself a board](#find-yourself-a-board)
- [Modelling the board](#modelling-the-board)
- [Making a move](#making-a-move)
- [House rule #1: knock off](#house-rule--knock-off)
- [House rule #2: sixes](#house-rule--sixes)
- [Rolling a dice](#rolling-a-dice)
- [Does the first player have an advantage?](#does-the-first-player-have-an-advantage)
- [That forgettable `init` function](#that-forgettable-init-function)
- [Tidying up the notation](#tidying-up-the-notation)
- [A `Dice` class](#a-dice-class)
- [Running simulations in parallel](#running-simulations-in-parallel)
- [Let's not repeat ourselves](#lets-not-repeat-ourselves)
- [Working solution](#working-solution)


<div id="find-yourself-a-board" />

## Find yourself a board

Run a Google search to find a suitable board;
e.g. [this one](https://www.shutterstock.com/image-vector/snakes-ladders-board-game-start-finish-163384724).

<div id="modelling-the-board" />

## Modelling the board

First, lets consider an obvious but unhelpful model: the board as a
10x10 grid. The board certainly looks like a 10x10 grid, but looks can
be deceptive! Here are three reasons why this is not a good model:

1. Nothing special happens at the edges. When you get to the end of a
   row, you just continue on the next row.
2. There are positions outside the 10x10 grid. Players start off the
   board, and moves that run past 100 are bounced back (as if there
   are hidden snakes).
3. A better model exists!
   
The better model is: a single array of 106 cells. Why 106?

* Cell 0 is the start, positioned off the main board
* Cells 1 to 100 are the main board
* Cells 101 to 105 are positions past the end cell (100). These are
  not displayed in the game, but can still be landed on by rolling a
  total greater than 100 (e.g. from cell 99 a player can roll a 6 and
  land on cell 105).

Each cell contains the position you end up after you roll your counter
onto that cell. For most cells (those that don't contain the base of a
ladder or the head of a snake) you stay put. Snake and ladder cells
contain the position of the end of the snake or ladder.

```cpp
int board[106];

void init() {
  for (auto i = 0; i < 106; i++)
	board[i] = i
  board[1] = 38;
  board[4] = 14;
  board[9] = 31;
  // ...etc
}
```

In most versions of the game, a player must land exactly on cell 100
in order to win. Moving beyond cell 100 results in the player
"bouncing back". E.g. if you are on cell 99 and you roll a 2, you move
one forward to cell 100 and then one back to cell 99. If you roll a 3,
you end up on cell 98, etc. (if that cell contains a snake, you would
then slide down the snake).

We can implement this rule by making the 5 cells past 100 act as snakes:
```cpp
  board[101] = board[99];
  board[102] = board[98];
  board[103] = board[97];
  board[104] = board[96];
  board[105] = board[95];
```

We write `board[101] = board[99]` etc. (rather than 99) to take any
snakes that might appear on those positions into account.

<div id="making-a-move" />

## Making a move
To make a move, a player rolls a fair 6-sided dice, adds the value to
their current position, and then uses the `board` array to determine
their new position:
```cpp
player = board[player + roll()];
```

Provided `roll()` returns a number between 1 and 6, and `player` has a
value between 0 and 99 (and assuming the board array is set up
correctly) then the new position for the player will be between 1 and 100.
If the position is 100, then they have won! Otherwise, the next
player has a turn. Here's the code for a 2-player game, where one
player is `red` and the other is `blue`:

```cpp
void play() {
  int red = 0;
  int blue = 0;
  while (true) {
    red = board[red + roll()];
	if (red == 100) {
	  std::cout << "Red won!" << std::endl;
	  return;
    }

	blue = board[blue + roll()];
	if (blue == 100) {
	  std::cout << "Blue won!" << std::endl;
	  return;
    }
  }
}
```

<div id="house-rule-1-knock-off" />

## House rule #1: knock off

Implement the house rule that if a player lands on the same cell as an
opponent, the opponent is sent back to the start (cell 0).

<details>
<summary>Hint</summary>
<p>

```cpp
   red = board[red + roll()];
   if (red == 100) ...
   if (red == blue)
     blue = 0;
```

</p>
</details>

<div id="house-rule-2-sixes" />

## House rule #2: sixes

Implement the house rule that if a player rolls a 6 then they get
another roll.

<details>
<summary>Hint</summary>

<p>

```cpp
   int r;
   do {
     r = roll();
	 red = board[red + r];
	 if (red == 100) ...
   } while (r == 6);
```

</p>
</details>

<div id="rolling-a-dice" />

## Rolling a dice

C++ provides a high-quality library of random number sources and
distribution generators. It is worth studying, as most languages
provide more simplistic support (although not as poor as
http://xkcd.com/221).

A random number source can be either a "pseudo-random number
generator" or a true random source. Pseudo-random sources are entirely
deterministic (i.e. you can reproduce them perfectly if you wish), but
appear random for all intents and purposes. True random sources
generate unique sequences of numbers and cannot be reproduced. They
also tend to be significantly slower. If you are picking the winning
numbers for a raffle you will want a true random source. For most
purposes, pseudo-random numbers are quite sufficient.

A distribution generator takes your choice of random numbers and
"shapes" it into a particular statistical distribution. Distributions
include: uniform, normal, binomial, poisson, etc.

For our dice, we want a uniform integer distribution, so each of the
numbers between 1 and 6 have an equal chance of appearing. This is the
`uniform_int_distribution` generator.

```cpp
#include <random>

std::default_random_engine generator { 0 };
std::uniform_int_distribution<int> distribution { 1, 6 };
int roll() {
  return distribution(generator);
}
```

The `0` used to initialise `generator` is a "seed". This determines
the initial state of the generator. For a given seed, a pseudo-random
number generator will always return the same sequence. If you want a
different sequence every time you run the program, you can use a true
random number source for the seed:

```cpp
std::default_random_engine generator { std::random_device{}() };
```

Using a random source as the generator would also be possible, but (a)
it's slower, and (b) you won't be able to reproduce your results exactly.
Here's an example:
```cpp
auto seed = std::random_device{}();
std::default_random_engine generator { seed };
std::uniform_int_distribution<int> roll { 1, 6 };

int main() {
  std::cout << "To reproduce this run, set seed to " << seed << "\n";
  std::cout << roll(generator) << "\n";
}
```

<div id="does-the-first-player-have-an-advantage" />

## Does the first player have an advantage?

We can put the preceeding code fragments together to make a program
that counts how many times the first player wins out of some number of
trials.

```cpp
#include <iostream>
#include <random>

std::default_random_engine rng { std::random_device{}() };
std::uniform_int_distribution<int> dice { 1, 6 };
int roll() { return dice(rng); }

int board[106];

void init() {
  // From https://www.shutterstock.com/image-vector/snakes-ladders-board-game-start-finish-163384724
  for (int i = 0; i < 106; i++)
    board[i] = i;
  board[1] = 38;
  board[4] = 14;
  board[9] = 31;
  board[17] = 7;
  board[21] = 42;
  board[28] = 84;
  board[51] = 67;
  board[54] = 34;
  board[62] = 19;
  board[64] = 60;
  board[67] = 51;
  board[72] = 91;
  board[80] = 99;
  board[87] = 36;
  board[93] = 73;
  board[95] = 75;
  board[98] = 79;
  board[101] = board[99];
  board[102] = board[98];
  board[103] = board[97];
  board[104] = board[96];
  board[105] = board[95];
}

bool firstPlayerWins() {
  int red = 0;
  int blue = 0;
  while (true) {
    red = board[red + roll()];
    if (red == 100) return true;
    blue = board[blue + roll()];
    if (blue == 100) return false;
  }
}

int countWins(int nTrials) {   
  int wins = 0;
  for (int trials = 0; trials < nTrials; trials++)
    if (firstPlayerWins())
      wins++;
  return wins;
}

int main() {
  init();
  std::cout << "In 10,000 trials, the first player won " << countWins(10000) << "\n";
}
```

Our "best estimator" is first player wins / total games. If this is
close to 0.5 (i.e. if 0.5 falls within the maximum expected error
bounds) then the game can be considered fair.

[Wikipedia](https://en.wikipedia.org/wiki/Checking_whether_a_coin_is_fair)
gives the maximum error $E = \frac{Z}{2 \sqrt n}$, where $Z$ is the
confidence interval. If we want to be 95% confident of the result, we
can take $Z = 1.9599$.

For example, if the first player wins 5,078 out of 10,000 games then
the confidence interval for the actual probability of winning is $5078
/ 10000 \pm 1.9599 / 2\sqrt 10000 \ge 0.5000005$, so there is a faint
advantage to the first player.


<div id="that-forgettable-init-function" />

## That forgettable `init` function

The first time I ran this program it failed, running but never
producing any output. I omitted to call `init()`, and as a result the
board contained the value 0 in each entry and so no player ever won.

Forgetting to initialise data is one of the most common causes of
program failure. Fortunately, C++ provides some powerful features that
can be used to guarantee data is initialised. Essentially, we can wrap
up both the data and the code that initialises that data into a `class`:

```cpp
class Board {
  int board[106];
public:
  Board() {
    // From https://www.shutterstock.com/image-vector/snakes-ladders-board-game-start-finish-163384724
    for (int i = 0; i < 106; i++)
      board[i] = i;
    board[1] = 38;
    board[4] = 14;
    board[9] = 31;
    board[17] = 7;
    board[21] = 42;
    board[28] = 84;
    board[51] = 67;
    board[54] = 34;
    board[62] = 19;
    board[64] = 60;
    board[67] = 51;
    board[72] = 91;
    board[80] = 99;
    board[87] = 36;
    board[93] = 73;
    board[95] = 75;
    board[98] = 79;
    board[101] = board[99];
    board[102] = board[98];
    board[103] = board[97];
    board[104] = board[96];
    board[105] = board[95];
  }
```

This `Board` class holds the board array, together with a "constructor"
function that has the same name as the class. The constructor function
is somewhat unusual as it has no return type. It doesn't return
anything, it just initialises. We can use it like this:

```cpp
Board shutterStockBoard;
```

This both declares _and_ initialises a `Board` (which we have named
`shutterStockBoard`).

The `public:` separates functions and data that are internal to the
class from parts that can be used from elsewhere in the code. As it
stands, our `main` function can't actually do anything other than
initialise a `Board`. We could move the line `int board[106]` below
`public:`, which would allow our code to refer to this array as
`shutterStockBoard.board`:

```cpp
bool firstPlayerWins() {
  int red = 0;
  int blue = 0;
  while (true) {
    red = shutterStockBoard.board[red + roll()];
    if (red == 100) return true;
    blue = shutterStockBoard.board[blue + roll()];
    if (blue == 100) return false;
  }
}
```

However, there are some problems with doing
this:

1. Exposing `board` directly means `main` can change its contents. It
   seems odd not trusting our own code not to do this, but
   misunderstandings do happen.
2. Arrays are notorious in C++ by not having any checks made when they
   are indexed. So it is perfectly possible to refer to an element
   outside the array bounds, with undefined results.
   
We can protect against both of these problems by exposing a function
to return a single board item, rather than the entire board:

```cpp
class Board {
...
public:
  int get(int i) {
    if (i >= 0 && i < 106)
	  return board[i];
	return 0;
  }
```

Now our code can call, e.g. `shutterStockBoard.get(red + roll())` to
find the board item at position `red + roll()`.

<div id="tidying-up-the-notation" />

## Tidying up the notation

When `board` was just an array, we looked up an item using the square
bracket notation - e.g. `board[i]`. Now, with the `Board` class we
currently have to write `board.get(i)`. This is purely a notational
difference, but leaves us with two different ways of writing what is
conceptually the same thing.

C++ supports an "operator" notation that allows us to use the original
notation for some symbols. Instead of naming the function `get`, we
can define:

```cpp
public:
  int operator[](int i) {
    if (i >= 0 && i < 106)
	  return board[i];
	return 0;
  }
```

Now, our code can call, e.g. `shutterStockBoard[red + roll()]` and we can
think of `Board` as if it was an array (albeit one with
"super-powers", checking indexes are valid and preventing any
updating).

<div id="a-dice-class" />

## A `Dice` class

We can take this `class` idea further, and start wrapping together
more pieces of related code and data. Our initial motivation is to
hide details of the program that are not important elsewhere (but we
have a second agenda that will become clear later). The `roll()`
function is a good example: it makes use of a random number generator
and a distribution, but neither of these things are of any interest
elsewhere. We can use a class to hide them:

```cpp
class Dice {
  std::default_random_engine rng { std::random_device{}() };
  std::uniform_int_distribution<int> dice { 1, 6 };
public:
  int roll() { return dice(rng); }
};
```

The code now needs to declare a `Dice` and call its `roll()` function:

```cpp
Dice myDice;

bool firstPlayerWins() {
  int red = 0;
  int blue = 0;
  while (true) {
    red = shutterStockBoard.board[red + myDice.roll()];
    if (red == 100) return true;
    blue = shutterStockBoard.board[blue + myDice.roll()];
    if (blue == 100) return false;
  }
}
```

Like the `get` function earlier, the name `roll` is clumsy. I've named
the `Dice` as `myDice` to make clear that the object is different from
the function, but ideally we would like the code to remain as a call
to `roll()`. C++ lets allows us to do this, in this case by defining a
function with no name! (If we need to refer to it, it's called "apply").

```cpp
class Dice {
  std::default_random_engine rng { std::random_device{}() };
  std::uniform_int_distribution<int> dice { 1, 6 };
public:
  int operator()() { return dice(rng); }
};
```

With this definition, we can declare

```cpp
Board board;
Dice roll;
```

With this, the `firstPlayerWins` function returns to its original syntax:

```cpp
bool firstPlayerWins() {
  int red = 0;
  int blue = 0;
  while (true) {
    red = board[red + roll()];
    if (red == 100) return true;
    blue = board[blue + roll()];
    if (blue == 100) return false;
  }
}
```

<div id="running-simulations-in-parallel" />

## Running simulations in parallel

For an accurate analysis of Snakes and Ladders we will need to run a
large number of simulations. We can do that more quickly (or run a
larger number in the same time) if we use all of the processing power
available. Here's a first attempt:

```cpp
int main() {
#pragma omp parallel
  std::cout << "In 10,000 trials, the first player won " << countWins(10000) << "\n";
}
```

This attempt has a serious, invisible flaw: all of the parallel
threads are using the same random number generator, and there will be
conflicts when its state gets updated.

Do do it right, we need a separate `Dice` for each thread. We can
achieve this by wrapping the dice together with the code that uses the
dice into its own class:

```cpp
class Game {
  Dice roll;

  bool firstPlayerWins() {
    int red = 0;
    int blue = 0;
    while (true) {
      red = board[red + roll()];
      if (red == 100) return true;
      blue = board[blue + roll()];
      if (blue == 100) return false;
    }
  }

public:
  int countWins(int nTrials) {   
    int wins = 0;
    for (int trials = 0; trials < nTrials; trials++)
      if (firstPlayerWins())
	wins++;
    return wins;
  }
};
```

Now our parallel block can start by declaring a `Game` and calling `countWins`:

```cpp
int main() {
#pragma omp parallel
  {
    Game game;
    std::cout << "In 10,000 trials, the first player won " << game.countWins(10000) << "\n";
  }
}
```

It's still not quite right - the output will get mixed up as all the
threads try to write at the same time. We can fix this by requiring
only one write at a time:

```cpp
int main() {
#pragma omp parallel
  {
    Game game;
    int wins = game.countWins(10000);
#pragma omp critical
    std::cout << "In 10,000 trials, the first player won " << wins << "\n";
  }
}
```

Here is a working solution so far. Compile this using `g++ -fopenmp
snakes.cpp -o snakes`. If you are using MacOS you may need to use `g++
-Xpreprocessor -fopenmp -lomp snakes.cpp -o snakes`.

<details>
<summary>snakes.cpp</summary>
<p>

```cpp
#include <iostream>
#include <random>

class Dice {
  std::default_random_engine rng { std::random_device{}() };
  std::uniform_int_distribution<int> dice { 1, 6 };
public: 
  int operator()() { return dice(rng); }
};

class Board {
  int board[106];
public:
  Board() {
    // From https://www.shutterstock.com/image-vector/snakes-ladders-board-game-start-finish-163384724
    for (int i = 0; i < 106; i++)
      board[i] = i;
    board[1] = 38;
    board[4] = 14;
    board[9] = 31;
    board[17] = 7;
    board[21] = 42;
    board[28] = 84;
    board[51] = 67;
    board[54] = 34;
    board[62] = 19;
    board[64] = 60;
    board[67] = 51;
    board[72] = 91;
    board[80] = 99;
    board[87] = 36;
    board[93] = 73;
    board[95] = 75;
    board[98] = 79;
    board[101] = board[99];
    board[102] = board[98];
    board[103] = board[97];
    board[104] = board[96];
    board[105] = board[95];
  }

  int operator[](int i) {
    if (i >= 0 && i < 106)
      return board[i];
    return 0;
  }
};

Board board;

class Game {
  Dice roll;

  bool firstPlayerWins() {
    int red = 0;
    int blue = 0;
    while (true) {
      red = board[red + roll()];
      if (red == 100) return true;
      blue = board[blue + roll()];
      if (blue == 100) return false;
    }
  }

public:
  int countWins(int nTrials) {   
    int wins = 0;
    for (int trials = 0; trials < nTrials; trials++)
      if (firstPlayerWins())
        wins++;
    return wins;
  }
};

int main() {
#pragma omp parallel
  {
    Game game;
    int wins = game.countWins(10000);
#pragma omp critical
    std::cout << "In 10,000 trials, the first player won " << wins << "\n";
  }
}
```
</p>
</details>

<div id="lets-not-repeat-ourselves" />

## Let's not repeat ourselves

Time now to return to the game play. With the two house moves in
place, the code for playing a game is getting complicated, and this
complication is repeated for each player:

```cpp
  bool firstPlayerWins() {
    int red = 0;
    int blue = 0;
    while (true) {
      int r;
      do {
        r = roll();
        red = board[red + r];
        if (red == 100) return true;
        if (red == blue)
          blue = 0;
      } while (r == 6);

      do {
        r = roll();
        blue = board[blue + r];
        if (blue == 100) return false;
        if (blue == red)
          red = 0;
      } while (r == 6);
    }
  }
```

Our first attempt at writing a move function is wrong, but will
motivate introducing the "references" feature:

```cpp
  bool move(int player, int opponent) {
    int r;
    do {
      r = roll();
      player = board[player + r];
      if (player == 100) return true;
      if (player == opponent)
        opponent = 0;
    } while (r == 6);

    return false;
  }
```

The function returns `true` iff the `player` has won. Our
`firstPlayerWins` function becomes:

```cpp
  bool firstPlayerWins() {
    int red = 0;
    int blue = 0;
    while (true) {
      if (move(red, blue))
        return true;
      if (move(blue, red))
        return false;
    }
  }
```

The flaw with this code is that no player ever moves! The variables
`red` and `blue` never change. When the `move` function is called, a
_copy_ of the value is passed to the function. Changes `move` makes to
`player` or `opponent` are not reflected back to `red` or `blue`.

To fix this problem, we need to make `player` and `opponent`
_references_:

```cpp
  bool move(int& player, int& opponent) {
```

The `&` can be appended to any type to make it into a reference.
Referencing is implemented by passing the memory address of the
argument. Whenever the reference variable is used, this address is
followed to find the original value. If the reference variable is
assigned a new value, again the address is followed and the original
value is updated.

Note that a reference argument must have an address. It is not
possible to call, say, `move(red + 1, blue)` as there is no address
for `red + 1` - its simply a value.

<div id="working-solution" />

## Working solution

<details>
<summary>snakes.cpp</summary>
<p>

```cpp
#include <iostream>
#include <random>

class Dice {
  std::default_random_engine rng { std::random_device{}() };
  std::uniform_int_distribution<int> dice { 1, 6 };
public:
  Dice(unsigned int seed) : rng(seed) { }
  
  int operator()() { return dice(rng); }
};

class Board {
  int board[106];
public:
  Board() {
    // From https://www.shutterstock.com/image-vector/snakes-ladders-board-game-start-finish-163384724
    for (int i = 0; i < 106; i++)
      board[i] = i;
    board[1] = 38;
    board[4] = 14;
    board[9] = 31;
    board[17] = 7;
    board[21] = 42;
    board[28] = 84;
    board[51] = 67;
    board[54] = 34;
    board[62] = 19;
    board[64] = 60;
    board[67] = 51;
    board[72] = 91;
    board[80] = 99;
    board[87] = 36;
    board[93] = 73;
    board[95] = 75;
    board[98] = 79;
    board[101] = board[99];
    board[102] = board[98];
    board[103] = board[97];
    board[104] = board[96];
    board[105] = board[95];
  }

  int operator[](int i) const {
    return i >= 0 && i < 106 ? board[i] : 0;
  }
};

const Board board;

class Game {
  Dice roll;

  bool move(int& player, int& opponent) {
    int r;
    do {
      r = roll();
      player = board[player + r];
      if (player == 100) return true;
      if (player == opponent)
        opponent = 0;
    } while (r == 6);

    return false;
  }
  
  bool firstPlayerWins() {
    int red = 0;
    int blue = 0;
    while (true) {
      if (move(red, blue))
        return true;
      if (move(blue, red))
        return false;
    }
  }

public:
  int countWins(int nTrials) {   
    int wins = 0;
    for (int trials = 0; trials < nTrials; trials++)
      if (firstPlayerWins())
        wins++;
    return wins;
  }
};


int main() {
#pragma omp parallel
  {
    Game game;
    int wins = game.countWins(10000);
#pragma omp critical
    std::cout << "In 10,000 trials, the first player won " << wins << "\n";
  }
}
```
</p>
</details>
