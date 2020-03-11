---
title: "Snakes and ladders"
description: A simulation of a classic children's game
date: '2020-03-27'
---

In this exercise, we will create a simulation of the classic
children's game of Snakes and Ladders. 

## Find yourself a board

Run a Google search to find a suitable board;
e.g. [this one](https://www.shutterstock.com/image-vector/snakes-ladders-board-game-start-finish-163384724).

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

We use `board[99]` etc. (rather than 99) to take any snakes that might
appear on those positions into account.

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
	  std::cout << "Red won!" << std::end;
	  return;
    }

	blue = board[blue + roll()];
	if (blue == 100) {
	  std::cout << "Blue won!" << std::end;
	  return;
    }
  }
}
```

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

## Rolling a dice

C++ provides a high-quality library of random number sources and
distribution generators. It is worth studying, as most languages
provide more simplistic support.

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

std::default_random_engine generator(0);
std::uniform_int_distribution<int> distribution(1, 6);
int roll() {
  return distribution(generator);
}
```

The `0` passed to `generator` is a "seed". This determines the initial
state of the generator. For a given seed, a pseudo-random number
generator will always return the same sequence. If you want a
different sequence every time you run the program, you can use a true
random number source for the seed:

```cpp
std::default_random_engine generator(std::random_device{}());
```

Using a random source as the generator would also be possible, but (a)
it's slower, and (b) you won't be able to reproduce your results exactly.
Here's an example:
```cpp
auto seed = std::random_device{}();
std::default_random_engine generator(seed);
std::uniform_int_distribution<int> roll(1, 6);

int main() {
  std::cout << "To reproduce this run, set seed to " << seed << "\n";
  std::cout << roll(generator) << "\n";
}
```
