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
- [Making a move](#making-a-move)
- [Rolling a dice](#rolling-a-dice)
- [Does the first player have an advantage?](#does-the-first-player-have-an-advantage)
- [That forgettable `init` function](#that-forgettable-init-function)
- [A `Dice` class](#a-dice-class)
- [Running simulations in parallel](#running-simulations-in-parallel)
- [Let's not repeat ourselves](#lets-not-repeat-ourselves)
- [Collecting more game statistics](#collecting-more-game-statistics)

<div id="find-yourself-a-board" />

## Find yourself a board

Run a Google search to find a suitable board;
e.g. [this one](https://www.shutterstock.com/image-vector/snakes-ladders-board-game-start-finish-163384724).

### Modelling the board

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

- Cell 0 is the start, positioned off the main board
- Cells 1 to 100 are the main board
- Cells 101 to 105 are positions past the end cell (100). These are
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

### House rule #1: knock off

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

### House rule #2: sixes

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
<http://xkcd.com/221>).

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

### Tidying up the notation

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

### Working solution

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

### References summary

1. The `&` after a type means the address of the value is passed to
   the function. Without an `&` the value is copied.
2. An address either 32 or 64 bits (depending on the computer you are
   using). Every address is the same size, regardless of the type of
   thing it is addressing.
3. Passing a reference means the code can change the contents of a
   variable declared by the caller. This is one way of passing
   information back from a function (the other way is to use the
   function return value).
4. References also get used when copying is expensive. In this case,
   we declare the reference `const`, to make it clear that the value
   is not changed by the function. This is a common pattern in C++
   code.

<div id="collecting-more-game-statistics" />

## Collecting more game statistics

We currently collect just one game statistic - the number of times the
player who moves first wins. Two more statistics:

1. A heatmap of the board, showing which squares are used most.
2. The number of rolls games take. This will be a frequency
   distribution (i.e. the number of games that are completed in a
   given number of rolls).

Collecting these statistics will motivate using some of the C++
"collections" classes.

### The heatmap

For the heatmap we need a count for every square on the board. The
counts will start out all zero, and every time a player is moved we
increment the count for that square. The data structure for this is an
array:

```cpp
int heatmap[106];
```

While this looks identical to the definition of the `board`, it has a
very different meaning. The contents of the `board` array are all
numbers between 0 and 105. In the heat map, the contents are counts of
how many times a player has landed on that square.

### Duration frequency

Storing the duration frequency using an array would be awkward for
several reasons. First, we have no prior knowledge of the maximum game
duration. Our choices would be to either determine a reliable upper
bound, or to not record games that take longer than a pre-determined
maximum. Further, the duration distribution is "sparse", in the sense
that there cannot be any games shorter than 6 or 7 rolls. There are
also likely to be gaps at the upper end of the distibution, when a
rare game goes on for much longer than any other. An array is a dense
structure - every element in the index range is stored.

C++ provides several library classes that are much better suited to
storing sparce mappings. The `std::map` class is ideal for our
purposes:

```cpp
#include <map>

std::map<int, int> distribution;
```

The `<int, int>` specifies the types of the key and the value. Maps
can use any ordered type for the key (unlike arrays, which can only
use integers). For example, maps of `string`s are common. The value
can be any type (just like arrays). Here, the value is the count of
the number of games of the given duration (the key).

Internally, maps store a collection of (key, value) pairs. However,
they are designed in a way that makes looking up a value for a given
key fast.

### A `Statistic` class

As usual, we will want to put our statistics into a class. When we
are running our simulation in parallel, we will need each thread
collect its own statistics. We can merge the separate statistics
as each thread competes.

```cpp
class Statistics {
  std::array<int, 106> heatmap { 0 };
  std::map<int, int> duration;
  int firstPlayerWins = 0;
```

I have changed the heatmap from using a built-in C++ array to the library `std::array` class. The `std::array` class has several advantages over the built-in arrays and really no disadvantages, so we will use it exclusively from now on. In particular, a `std::array` knows its size, how to make a copy of itself, and can do index checking if we want. Otherwise, it behaves exactly like a built-in array. To enable it, put the line `#include <array>` at the start of the file.

We now need some function to update the statistics. For when a player lands on a square, we have either:

```cpp
void landing(int position) {
  heatmap[position]++;
}
```

or

```cpp
void landing(int position) {
  heatmap.at(position)++;
}
```

The latter will check that position is between 0 and 105, and throw a standard exception if it is not. This will protect our program in the event that a bad `position` is passed in.

At the end of the game we know how many rolls were made and whether the first player won:

```cpp
void firstPlayerWon() {
  firstPlayerWins++;
}

void gameMoves(int numberOfMoves) {
  duration[numberOfMoves]++;
}
```

Two more functions are needed. The first will be used when each thread finishes, and needs to merge its own statistics into the grand total. In C++, the conventional name for a "merge" operation is `+=`. We "merge" one number into another using `total += n`, and so it will look natural to merging one set of `Statistics` into another by writing `grandTotal += stats`.

```cpp
void operator+=(const Statistics& other) {
  firstPlayerWins += other.firstPlayerWins;
  for (int i = 0; i < heatmap.size(); i++)
    heatmap[i] += other.heatmap[i];
  for (const auto& kv: other.duration)
    duration[kv.first] += kv.second;
}
```

Finally, we need some means of printing the results. For now, we can use a `print` function. We will re-design this to something more standard shortly:

```cpp
void print() {
  std::cout << "First player won " << firstPlayerWins << "\n";
  std::cout << "Durations: ";
  for (const auto& kv: duration)
    std::cout << kv.first << ":" << kv.second << ", ";
  std::cout << "\nHeatmap: ";
  for (auto h: heatmap)
    std::cout << h << ", ";
  std::cout << "\n";
}
```

The `for` loops use the modern C++ syntax for iterating over all elements in a collection. For `duration`, each element is a pair with two fields - one called `first` and one `second`. In our case, `first` is the game length and `second` is the number of games that took that many rolls. The purpose of the `auto` keyword is apparent here. Without it, we would need to write `for (const std::pair<const int, int>& kv: duration)`.

### Updating `Game`

We now need to "instrument" the `Game` class to collect the appropriate `Statistics`. This will involve changes to the `move` function (to record a landing event) and the `firstPlayerWins` function (to record a `gameEnd`). Our `Game` class becomes:

```cpp
class Game {
  Dice dice;
  Statistics stats;

  bool move(int& player, int& otherPlayer, int& rollCount) {
    int r;
    do {
      r = roll();
      rollCount++;
      player = board[player + r];
      stats.landing(player);
      if (player == 100) return true;
      if (player == otherPlayer) // House rule #1: landing on another player bumps them off
        otherPlayer = 0;
    } while (r == 6); // House rule #2: rolling 6 gives you another turn
    return false;
  }

  bool firstPlayerWins() {
    int red = 0;
    int blue = 0;
    int rollCount = 0;
    while (true) {
      if (move(red, blue, rollCount)) {
        stats.firstPlayerWon();
        stats.gameMoves(rollCount);
        return true;
      }

      if (move(blue, red, rollCount)) {
        stats.gameMoves(rollCount);
        return false;
      }
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

There is a little bit of tidying up left to do. Our original `countWins` function is duplicating some of the statistics. Now, it just need to play `nTrials` games. Similarly, `firstPlayerWins` can be renamed to `playOneGame`, and it doesn't need to return anything.

```cpp
  void playOneGame() {
    int red = 0;
    int blue = 0;
    int rollCount = 0;
    while (true) {
      if (move(red, blue, rollCount)) {
        stats.firstPlayerWon();
        break;
      }

      if (move(blue, red, rollCount))
        break;
    }

    stats.gameMoves(rollCount);
  }

public:
  operator const Statistics&() const { return stats; }

  void runTrials(int nTrials) {
    for (int trials = 0; trials < nTrials; trials++)
      playOneGame();
  }
```

### Running the trials in parallel

Our `main` function can coordinate the available processor cores in running independent trials, where each parallel thread collects statistics using its own `Game` and then (serially) merges its results with into a grand total.

```cpp
int main() {
  Statistics grandTotals;
#pragma omp parallel
  {
    Game game;
    game.runTrials(10000);
#pragma omp critical
    grandTotals += game;
  }

  std::cout << grandTotals << "\n";
  return 0;
}
```

This code glosses over two details:

1. How the statistics for a `Game` are returned (the `stats` field is nowhere mentioned)
2. What happened to the `print` method? We are writing out `grandTotal` directly.

The first trick is achieved by defining an "implicit conversion". A "conversion" is a function that C++ will insert for us in order to obtain a value of the required type. Various standard implicit conversions are provided automatically to convert, say, `int` values to `double`s. But its possible to define conversions of our own. The conversion we want here is to turn a `Game` into a `Statistics` value, so it can be used by the `+=` "merge" function. We could write an ad-hoc conversion, such as

```cpp
Statistics getStats() { return stats; }
```

This has several problems. First, it will return a copy of the `stats`, which is a costly operation. Second, the name `getStats` is rather awkward. Defining an implicit conversion means we don't need to name the function at all (as C++ will call the function for us). We avoid the copy by returning a `const Statistics&`. The function doesn't change the `Game` class, so it needs to be declared `const`:

```cpp
operator const Statistics&() const { return stats; }
```

If the "magic" of a `Game` turning into `Statistics` disturbs you, you can declare the conversion as `explicit`. The calling code will then need to use a `static_cast` to perform the conversion.

```cpp
  // In class Game:
  explicit operator const Statistics&() const { return stats; }

  // In main:
  grandTotals += static_cast<Statistics>(game);
```

The choice of implicit or explicit conversion is up to you. The `static_cast` is somewhat cumbersome, but highlights that a conversion is being performed. Writing an ad-hoc `getStats` function is discouraged, and would obscure the fact that a simple conversion is being performed.

The `print` method has been replaced by the standard `<<` function:

```cpp
std::ostream& operator<<(std::ostream& out, const Statistics& stats) {
  out << "First player won " << stats.firstPlayerWins << "\n";
  out << "Durations: ";
  for (const auto& kv: stats.duration)
    out << kv.first << ":" << kv.second << ", ";
  out << "\nHeatmap: ";
  for (auto h: stats.heatmap)
    out << h << ", ";
  out << "\n";
  return out;
}
```

For technical reasons, this function can't be written in the `Statistics` class. Instead, the `Statistics` to print is passed as the _second_ argument to the `<<` function. The first argument is the output stream. We used `std::cout` in the original `print` function. This version will write to whichever output stream it is called on, making it much more versatile.

As it stands, this `<<` function won't compile as it refers to private fields of `stats`. Usually, we wish to restrict access to the internals of a class, but this is a special case. To circumvent the access restriction for just this function, we declare it a `friend` of `Statistics`:

```cpp
  friend std::ostream& operator<<(std::ostream&, const Statistics&);
```

Here is the complete code:

```cpp
#include <array>
#include <iostream>
#include <map>
#include <random>
#include <vector>

class Board {
  std::array<int, 106> board;
public:
  Board() {
    // From https://www.shutterstock.com/image-vector/snakes-ladders-board-game-start-finish-163384724
    int p = 0;
    for (auto& i: board)
      i = p++;
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
    return board.at(i);
  }
};

const Board board;

class Dice {
  std::default_random_engine rng { std::random_device{}() };
  std::uniform_int_distribution<int> dice { 1, 6 };
public:
  int operator()() { return dice(rng); }
};

class Statistics {
  std::array<int, 106> heatmap { 0 };
  std::map<int, int> duration;
  int firstPlayerWins = 0;
public:
  void landing(int position) {
    heatmap.at(position)++;
  }

  void firstPlayerWon() {
    firstPlayerWins++;
  }

  void gameMoves(int numberOfMoves) {
    duration[numberOfMoves]++;
  }

  Statistics& operator+=(const Statistics& other) {
    for (int i = 0; i < heatmap.size(); i++)
      heatmap[i] += other.heatmap[i];
    firstPlayerWins += other.firstPlayerWins;
    for (const auto& kv: other.duration)
      duration[kv.first] += kv.second;
    return *this;
   }

  friend std::ostream& operator<<(std::ostream&, const Statistics&);
};

std::ostream& operator<<(std::ostream& out, const Statistics& stats) {
  out << "First player won " << stats.firstPlayerWins << "\n";
  out << "Durations: ";
  for (const auto& kv: stats.duration)
    out << kv.first << ":" << kv.second << ", ";
  out << "\nHeatmap: ";
  for (auto h: stats.heatmap)
    out << h << ", ";
  out << "\n";
  return out;
}

class Game {
  Dice roll;
  Statistics stats;

  bool move(int& player, int& otherPlayer, int& rollCount) {
    int r;
    do {
      r = roll();
      rollCount++;
      player = board[player + r];
      stats.landing(player);
      if (player == 100) return true;
      if (player == otherPlayer) // House rule #1: landing on another player bumps them off
        otherPlayer = 0;
    } while (r == 6); // House rule #2: rolling 6 gives you another turn
    return false;
  }

  void playOneGame() {
    int red = 0;
    int blue = 0;
    int rollCount = 0;
    while (true) {
      if (move(red, blue, rollCount)) {
        stats.firstPlayerWon();
        break;
      }

      if (move(blue, red, rollCount))
        break;
    }

    stats.gameMoves(rollCount);
  }

public:
  operator const Statistics&() const { return stats; }

  void runTrials(int nTrials) {
    for (int trials = 0; trials < nTrials; trials++)
      playOneGame();
  }
};

int main() {
  Statistics grandTotals;
#pragma omp parallel
  {
    Game game;
    game.runTrials(10000);
#pragma omp critical
    grandTotals += game;
  }

  std::cout << grandTotals << "\n";
  return 0;
}
```
