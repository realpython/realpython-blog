# Python Programming Challenge - first to five

We've partnered with our friends at [Interview Cake](https://www.interviewcake.com/) to bring you a programming challenge to test your logic skills and abilities.

<br>

**Updated 10/16/2015:** Added additional challenges - cheers!

## The Challenge

1. There are two players.
1. Each player writes a number, hidden from the other player. It can be any integer 1 or greater.
1. The players reveal their numbers.
1. Whoever chose the lower number gets 1 point, unless the lower number is lower by only 1, then the player with the higher number gets 2 points.
1. If they both chose the same number, neither player gets a point.
1. This repeats, and the game ends when one player has 5 points.

The challenge is to write a script to play this game. Knowing the rules and all your opponent's previous numbers, can you program a strategy? (And, no - `return random.randint(1, 3)` is not a strategy.) You should really try playing this first with your friends - you'll see there's a deep human element to predicting your opponent's choice.

Is it possible to program a strong strategy?

> Want to make the strategy a bit more interesting? Add an additional constraint to the challenge so that players can only use each number once. *The first ten working submissions get a free copy of Real Python! (still open to submissions)*

## The Prizes

Need some motivation? We'll be giving out prizes to the strategies that perform the best:

- 1st Place: [3D Printing Pen](http://the3doodler.com/store/)
- 2nd and 3rd places: [RC Quadcopter with Camera](http://www.amazon.com/UDI-U818A-2-4GHz-RC-Quadcopter/dp/B00D3IN11Q/ref=sr_1_2)
- The top 5 submissions will receive a free account on Interview Cake as well as a free copy of the Real Python courses!

> Although the challenge is officially over you can still partake! First, see if you can beat the [current](https://gist.github.com/mjhea0/d7fc846ea8ab2b03e819) winner to receive $20 off of Real Python. Second, create a web application with Flask that (a) makes it easy to add a new strategy and then (b) runs a given strategy against all the other strategies.

## Grading

The grading is simple: We'll run each strategy through the random number generator 100 times as a first screen - `return random.randrange(1, 10)`. The strategies that beat the generator we'll then be ran against one other in a round robin format to determine the overall winners. *Be sure to test your code out in the [game runner](https://gist.github.com/mjhea0/0a6b0bb6cc7557776ab8) before submitting.*

To submit your script, just email us a link to a secret Gist - *info (at) realpython (dot) com*. Good luck!
