# A Hyperparameter search for RL Agent for the 2048 Game

- This is a fork from [abachurin/2048](https://github.com/abachurin/2048).
- Hyperparameter search with Weights and Biases: [wandb.ai/duken72/2048](https://wandb.ai/duken72/2048-game2048?workspace=user-duken72)

-------

## Requirements

Almost none. Apart from Python you only need to install `numpy` and `pygame` libraries. Both can be installed with `pip install`.

Update: It seems that installing `pygame` is a bit of headache. The path, depending on the error messages one gets, is roughly this:

```bash
pip3 install homebrew
brew install gcc
brew install sdl sdl_image sdl_mixer sdl_ttf portmidi
pip3 install pygame
```

How to run it, for those as novice at this as me:

Push the green "Code" button. Clone this repository to a directory on your machine, using `git clone` command or GitHub Desktop. Then add the path to game2048 package like this (for Mac users, not familiar with other systems):

`export PYTHONPATH= [your directory here, for example /User/Documents/GitHub/2048/]`

Now go to the directory:

`cd [your directory here, for example /User/Documents/GitHub/2048/game2048/]`

Now you can run `python show.py` and it should fly. The colors and fonts are a bit different than if i run it from PyCharm for some reason, otherwise seems all fine.

### 2048 Game

2048 is a single-player sliding block puzzle game designed by Italian web developer Gabriele Cirulli. The game's objective is to slide numbered tiles on a grid to combine them to create a tile with the number 2048. Of course, one can keep playing and achieve bigger tiles, with theoretical but probably unachievable limit
of 131072 (2 to the power of 17). When I used to play the game as a time-killer and stress-releiver some years ago, the best i've sometimes achieved was 8192 tiles and my best score around 150,000.

For those who never played it but are nevertheless interested, here is a brief description:
2048 is played on a 4×4 grid, with tiles numbered by powers of 2: 2, 4, 8 etc. The board starts with two random 2 or 4 tiles. At each step the Player can try to shake the board in one of the four directions: left, up, right or down. Tiles slide as far as possible in the chosen direction until they are stopped by either another tile or the edge of the grid. If two tiles with the same number collide while moving, they merge into a tile with twice the value. This new number is added to the score. The resulting tile cannot merge with another tile again in the same move. If nothing on the board changed as a result of the Player's action, i.e. the move did not happen, the Player has to choose another move. If there are no valid moves - the game is over. Now, every turn after the Player's move, a new tile randomly appears in an empty spot on the board with a value of either 2 or 4, with 0.9 and 0.1 probabilities respectively.

You can play the game by choosing option 0 while running:

`python3 show.py`

### Reinforcement Learning with Approximating State Value Function

I won't describe RL in detail, as one can easily google it. Briefly: we'd like to program an Agent, supply it with the rules of the game and devise a learning strategy in such a way that it will gradually learn to play better and better from scratch without any further human input. Ideally, the Agent could learn what to do exactly in any given poisition of the board, i.e. learn the table that supplies the best action given the state of the game. Or, alternatively, some numeric valuation of the state, and then the action at each step is taken in the direction of the best valuation of the resulting state. (The latter approach is used in this work.) But in 2048 game the number of possible states is astronomical, hence the tabular approach is impossible and the Agent's valuation function has to be a true function, not a table.

### What I tried to do, how it didn't work, and my version of why it didn't

The Agent starts with the random moves, so the first milestone is to beat the random strategy. It is not easy!
The first thing that comes to the mind of somebody like me, who started to study Machine Learning recently, is to try Neural Networks. Atari games, Alpha-chess and Alpha-zero - the recent successes of Deep RL are plenty. Tensorflow and other frameworks make writing such an Agent pretty easy and Google Colab provides an opportunity to train it with GPU/TPU for free. And indeed a lot of people tried it before, google "2048 reinforcement learning github". Some even claim success ... although I tried and was not able to replicate any of those. Some admit that they were not able to beat random walk, which fits my experience.
I spent two weeks trying to achieve this with NNs of different architectures. Convolutions of different sizes, branching layers and adding them back for a final 1-2 dense layers etc. Nothing worked. I even tried to add some man-made features to supply my Agent with human heuristics, like have max tiles in the corners etc. (This is somewhat against the spirit of RL, but I was desperate). It could be because the training process with even a relatively small NN is very slow, and you need to play a lot of episoded to achieve anything in this particular game. Why is that? Consider Chess or Go. The games are way more complex than 2048, but how many random moves can one make before losing the game to even an amateur human player? Just 3 in Chess - google "checkmate in 3 moves". I don't play Go but as far as I understand one has to secure corners right at the start of the game, so random moves will get you killed very fast as well. Now, how many random moves can one make in 2048 without losing the opportunity to still get back on track to beat the records, if one plays carefully? 100 random steps are survivable in 60% of cases, 50 steps - in 99%. You can play with

```
from game2048.game_logic import Game, random_eval
Game.trial(estimator=random_eval, num=100)
```

to check that. This means that for an Agent to learn that some initial moves are statistically better than others, it has to play a lot of starts, during which it learns very little or none at all.

### What finally worked, and I don't really understand why it works so well

Firstly, one can notice that the numbers on the tiles are a distraction, they could as well be colors or some other tags. Tiles of the same color produce another color and an increase in score when joined. So, in effect, the values of tiles are categorical features and have to be treated as such, for Neural Networks or any other approach. Now, imagine we could make a huge table and assign to each combination of colors a valuation of the position. After a long time of training the Agent will encounter most positons many times, get the valuations right and thus learn to play an ideal game. Of course, we can't have such a table, plus it will be an extremely slow process. But what if we take some smaller pieces of the board, list all possible states there? We take pairs of adjacent tiles, there are 24 such pairs on the board and, assuming the highest tile we hope to see is 2 ** 15 = 32768, only 16 * 16 = 256 possibilities for those pairs to be. We can then one-hot them, i.e. make 24 * 256 = 6144 features that for any given state of the board are all 0 except 24 which are equal to 1. Now we can try to feed those as an input layer of a Neural Network ...
At this point I remember an observation made by a practicing data scientist in Medium article I've read recently: if you want to understand whether your features are any good at all, run a simple linear regression on them. If the result is the same as random noise - probably don't waste your time on more complictated models. So I run a linear model, which by the way is just a sum of weights of non-zero features in this case - and bingo! - the scores skyrocket and in the first 1000 episodes we already have 50% games with 1024 best tile and 7% with 2048. Run to see yourself, it takes just a couple of minutes.

```
from game2048.r_learning import Q_agent
agent = Q_agent(n=2)
episodes = 1000
Q_agent.train_run(episodes, agent=agent, saving=False)
```

After that I experimented with triples and quartets for a week, as well as with learning rates and reward functions. The best architecture I've come up with so far is having different variants of rows, columns and (2, 2) squares as features. Reward - a simple game score change. Learning rate - better start high at 0.1 - 0.25 range and gradually move towards 0.01 - 0.02. We update weights after every move as there seems to be no way/no bonus to vectorize this.

Now, if we load the best Agent and ask it to evaluate different board positions, we see a strange thing. It doesn't look smart at all. E.g. it would give much the same score to a dead position, where any remaining tile will leave no moves, as to a nice one. Neither there is any obvious preference for monotonic rows/columns. So it is quite strange to witness the Agent playing in a smart way.

### Conclusion

The Agent at `best_agent.npy` was trained for 100,000 episodes. Achieving 4096 more and more often. The rate of 2048 tile games is stuck around 84-85% since about 30k episodes.
The Agent still occasionally gets stuck at some ridiculously low number, like 256 or 128. I don't know how to deal with it. May be try something akin to boosting, backpropagating some heavy penalty for each state in the low-scoring episode.
Another thing to do is try 5 or 6-tile feautures. But the number of parameters goes up exponentially. Besides, i am not sure it's worth it. 4-tile features already capture an important heuristics - we prefer a row like 1024 256 8 2 to a row like 2 1024 8 256, and the Agent weights most probably reflect that. Same with (2, 2)-squares. You can see how it tries to keep good order if you choose option 3 while running `python3 show.py`.
Not sure bigger pieces will reveal much more (but i will think about implementing).

### Charts and results

* Model with combinations of 2 adjacent tiles, trained over 10,000 episodes.

```
average over last 1000 episodes = 16300.744
1024 reached in 73.1 %
2048 reached in 31.5 %
4096 reached in 0.3 %
8192 reached in 0.0 %
best score = 56776
1024    4096    8    2
128    64    32    8
8    256    16    4
4    64    32    2
```

<p align = "center">
<img src = https://github.com/abachurin/2048/blob/master/score_chart_2_tile.png?raw=true>
</p>

* Model with combinations of 3 adjacent tiles, trained over 10,000 episodes.

```
average over last 1000 episodes = 32519.164
1024 reached in 94.9 %
2048 reached in 81.7 %
4096 reached in 13.4 %
8192 reached in 0.0 %
best score = 80496
1024    256    512    2
32    128    2048    4096
8    16    64    16
2    4    8    4
```

<p align = "center">
<img src = https://github.com/abachurin/2048/blob/master/score_chart_3_tile.png?raw=true>
</p>

* Model with some combinations of 4 adjacent tiles, trained over 100,000 episodes.
/ I wrote in the comments in `rl_learning.py` how i tried all 4-combinations at the start but it didn't work. /

```bash
average over last 1000 episodes = 44422.796
1024 reached in 94.4 %
2048 reached in 83.4 %
4096 reached in 45.9 %
8192 reached in 0.4 %
best score = 130664
2    4    256    4
4    16    2048    8192
8    32    256    1024
2    16    128    32
```

<p align = "center">
<img src = https://github.com/abachurin/2048/blob/master/score_chart.png?raw=true>
</p>

* Model that combines my best RL Agent with my version of Expectimax.
In my code that would be:

```
from game2048.r_learning import *
agent = Q_agent.load_agent("best_agent.npy.bak")
eval = agent.evaluate

def mix_eval(game):
    if game.empty_count() > 6:
        return eval(game)
    else:
        return estimator_lf(depth=3, width=4, evaluator=eval)(game)

Game.trial(mix_eval, num=100)
```

The results:

```
Best game =
4    2    8    4
16    8    4    2
256    128    64    32
8192           512    4096            1024
 score = 157224 odometer = 6537
average score of 100 runs = 69743.04
8192 reached in 18.0%
4096 reached in 79.0%
2048 reached in 96.0%
1024 reached in 98.0%
3.5 minutes per game, 1 second per move
```

You can watch the best game unfold with this code:

```
from game2048.r_learning import Game
from game2048.show import Show
game = Game.load_game("mixed_with_expectimax.npy")
Show().replay(game, speed=20)
```
