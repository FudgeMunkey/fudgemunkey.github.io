---
layout: post
title:  "How we became Kings of the Grid at BSides Canberra 2026"
date:   2026-09-27 18:00:00 +1000
categories: ctf
---

## BSides Canberra 2026

BSides Canberra is my favourite conference to attend each year because competing in the Capture The Flag (CTF) competitions is always such a highlight. This year was no different as we represented the **h4-x_X_x-0rs: Sniper Elite Squad** and placed 25th with 1781 points out of 210 teams.

An exciting spin on this years CTF was the introduction of King of the Hill (KITH) challenges where the best solution achieves the most points. This was an exciting change because it meant that the difficulty of the challenge wasn't decided by Skateboarding Dog (the creators), but instead the solutions of all the other teams. If you really enjoyed a KITH challenge it meant that you could spend the entire competition improving your solution in the hopes of achieving that Top Dog spot. Naturally, this is exaclty what ended up happening...

## King of the Grid

King of the Grid (KITG) was one of the new KITH style challenges this year and it totally absorbed all of my attention by combining two of my favourite things - coding and video games. For KITG you had to create a script to play Tron against 9 other bots every two minutes on a central server and gain the most points by outlasting or eliminating them. The game screen looked like this.

![Screenshot of King of the Grid challenge game](/assets/images/2026-09-27-kitg/kitg-screenshot.png)

Creating a bot was simple. At every game tick, you needed to return a dictionary with a move (up, down, left, right) and an optional boost flag (true, false). To be able to make your choice, you were provided with a `decide` function and the game's current state. Below is the `example_bot.py` that was provided to teams.

```python
"""
Example bot that walks clockwise in a square loop.
"""

_CLOCKWISE = {"up": "right", "right": "down", "down": "left", "left": "up"}
_steps_since_turn = 0

def decide(state: dict) -> dict:
    global _steps_since_turn
    facing = state["you"]["direction"]
    _steps_since_turn += 1
    if _steps_since_turn >= 4:
        _steps_since_turn = 0
        return {"move": _CLOCKWISE[facing]}
    return {"move": facing}
```

The example bot was simple but it worked quite well. It was impossible for it to run into its own trail and unless it spawned right next to another bot, it was unlikely to collide with another bot's trail. Its biggest weakness was that it could not move towards the centre of the grid as the safe zone shrunk.

![Bots are allowed to move in the safe zone but it shrinks by one square every 5 game ticks. Players in the danger zone are eliminated.](/assets/images/2026-09-27-kitg/kitg-zones.png)

## Controlling the Centre

After watching some of the bots compete on the server it was clear that being the last bot alive gained you a lot of points. You would get 1000 points for being the last player alive as a win bonus and you would get 0 - 800 points depending on how long you were alive. You would also get a lot of points for eliminating other players but trying to survive seemed like the most reliable strategy.

As the safe zone shrinks every 5 ticks I thought that getting to the centre of the grid and preventing other bots from staying in the safe zone was the easiest way to survive until the end of the game. In chess, this strategy is known as "controlling the centre" and I thought I could apply the same idea here. To achieve this, I set a target of `x: 16, y: 16` (on a 32x32 grid) and moved the bot directly to the centre.

```python
def move_to_centre(state: dict):
    x = state.get("you").get("x")
    y = state.get("you").get("y")

    target_x_distance = target_x - x
    target_y_distance = target_y - y

    # Go X or Y first
    move = None
    if abs(target_x_distance) >= abs(target_y_distance):
        if target_x_distance >= 0:
            move = "right"
        else:
            move = "left"
    else:
        if target_y_distance >= 0:
            move = "down"
        else:
            move = "up"

    return move
```

This script got the bot to the centre of the grid quickly but it did a poor job at "controlling the centre". To increase it's footprint, I followed the example bot's example and circled around the centre of the map. After reaching the centre of the map, the bot swapped to "centre mode" which looked like this.

```python
def move_around_centre(state: dict):
    x = state.get("you").get("x")
    y = state.get("you").get("y")

    coords_to_move_mapping = {
        (14, 14): "right",
        (15, 14): "right",
        (16, 14): "right",
        (17, 14): "down",
        (17, 15): "down",
        (17, 16): "down",
        (17, 17): "left",
        (16, 17): "left",
        (15, 17): "left",
        (14, 17): "up",
        (14, 16): "up",
        (14, 15): "up",
    }

    return coords_to_move_mapping[(x, y)]
```

This bot performed very well and managed to top the leaderboard for the first few hours of the competition. The path of the centre bot for a game looked like this.

![The bot moves from it's spawn point to the centre of the map. Once it is near the centre of the map, it runs a circuit around the centre to increase it's control of the centre of the grid.](/assets/images/2026-09-27-kitg/kitg-control-the-centre.png)

## Benchmarking

A big problem I faced when trying to create a new bot was knowing if it was actually better than the previous bot I made and knowing if it was better than the other team's bots. You could submit your bot to the server to see how it performed but games only played every 2 minutes. Although you see how it performs against other teams, this is an extremely long feedback loop. After 20 minutes of waiting you may have 10 games to review and those games may not be representative of that bot's strengths and weaknesses.

Skateboarding Dog did provide you with the game's engine code and the scripts to run the game locally which greatly decreased the feedback time. I often ran a command like this to see if my current bot was an improvement over the last bot `clear && python play.py -n 2 --replay replay.json bot_04.py bot_05.py` which worked relatively well. I did worry that I was optimising my bots performance against itself rather than against all possible strategies so I would run something like this before submission `clear && python play.py -n 2 --replay replay.json bot_01.py bot_02.py bot_03.py bot_04.py bot_05.py` but tracking a bot's performance over multiple games became very difficult.

To address this issue, @Isaac created a benchmarking script so we could easily compare the performance of all our bots in a local competition. The script would play thousands of games where each game had a random sample of 10 bots on a new seed. The results would be summarised in a table for us to review. @Isaac and @Eric both created many more bots with a variety of different strategies to reduce the risk of hyper optimising a bot's performance against its last version. The output for that tool looked like this.

![alt text](/assets/images/2026-09-27-kitg/benchmark-start.png)

Obviously making tools like this during a CTF take up a lot of time that could have been spent making a better solution for our bot, but tools like this help you make better solutions. Decreasing the feedback time for a new iteration of the bot and having the confidence to say "yes this bot is now definitely better than our previous ones" before submitting it to the server is just so valuable. This tool and the other bots didn't actually exist until the Friday night after Fudge #5 was made, but I will use it from now on to help quantify how much the bots were improving. For the centre bots (fudge_01 and fudge_04) mentioned previously, these were the benchmarking results. At their time, they were first on the server but compared to the bots we had when @Isaac created the tool and @Eric created the other bots they performed poorly.

![alt text](/assets/images/2026-09-27-kitg/benchmark-centre.png)

## Avoiding Immediate Collisions

The centre bot is able to move towards and control the centre of the grid but it fails to avoid immediate collisions with other bot's trails. This causes the bot to eliminate itself very early into the game and score very few points. The centre bot does not consider the game state and only calculates the best move based on the resulting distance to its goal - reaching the centre of the grid. In the decide function we are given access to the `state` object which is the entire game state at that game tick. Now when calculating the best move we can consider all of the trail coordinates from the `players.trail` object in the game state.

```json
{
  "tick": 42,
  "arena": {
    "width": 32,
    "height": 32
  },
  "safe_zone": {
    "x0": 5,
    "y0": 5,
    "x1": 26,
    "y1": 26,
    "next_shrink_tick": 45,
    "shrink_interval": 5,
    "shrink_step": 1
  },
  "trail_lifetime": 12,
  "you": {
    "id": 7,
    "alive": true,
    "x": 16,
    "y": 9,
    "direction": "up",
    "trail": [
      [
        16,
        10,
        53
      ],
      [
        16,
        11,
        52
      ]
    ],
    "boost": 100
  },
  "players": [
    {
      "id": 0,
      "name": "t8",
      "kind": "submission",
      "alive": true,
      "x": 11,
      "y": 40,
      "direction": "right",
      "trail": [
        [
          10,
          40,
          51
        ]
      ]
    }
  ]
}
```

When scoring the best move for the bot I considered collisions with trails to be worth `1_000_000` as they are the worst thing a bot can do (they immediately get eliminated). Other moves still keep the bot alive and work towards their goal, so they are worth the distance to the centre of the grid. I sorted each of the moves based on the score and chose the lowest scoring move. An example of the bot's choices may look like `{"right": 11, "up": 12, "down": 12, "left": 1000000}` which means it would choose to move `right`. If multiple moves are tied for the lowest score, then a random move is chosen. Adding this logic to the previous centre bots to avoid immediate collisions improved their performance drastically.

![alt text](/assets/images/2026-09-27-kitg/benchmark-centre-collision.png)

Now that there are so many bots in our local competition moving towards the same goal (centre of the grid) a common issue appeared. If two bots wanted to move to the same location they would collide with each other and eliminate each other from the game. These moves were valid because they were unnocupied spaces without bots or trails. This was a big problem for our local competition but not a big problem on the actual server. To fix this issue I considered coordinates that were `adjacent` to other players to be worth `500_000` as there was a chance to collide with the other players but it did not guarantee an elinimation. An example of a bot's choices may now look like `{"right": 11, "up": 12, "down": 500000, "left": 1000000}` which means the bot would choose to move `right`. Adding this logic to the previous centre circuit bot greatly improved its performance again.

![alt text](/assets/images/2026-09-27-kitg/benchmark-adjacent.png)

The scoring function for each move looked like this.

```python
if is_collision:
    move_scores[move_name] = 1_000_000
elif is_adjacent:
    move_scores[move_name] = 500_000
else:
    move_scores[move_name] = abs(next_x - target_x) + abs(next_y - target_y)
```

### The Oracle

At this stage the bot is able to move towards the centre, move in a square around the centre, avoid immediate collisions with all trails, and avoid moves into spaces adjacent to other players. However, the bot is only able to look ahead one move which works fine most of the time, but there are some scenarios where the bot gets stuck on its own trail. To avoid situations like the one shown below, the bot needs to be able to look at least two moves ahead.

```
# B is the current location of the bot and X is its trail

[ ][ ][ ][ ][ ]
[ ][ ][X][X][X]
[ ][ ][B][ ][X]
[X][X][X][X][X]
[ ][ ][ ][ ][ ]

# If the bot moves left it is okay, but if it moves right it gets stuck.
```

To fix this issue, I implemented a tree based search to look up to five moves ahead. I only simulated my bot's moves and not the entire game state for all bot moves because that was too complicated in the time I had. I was really only aiming to address the issue where the bot was getting stuck on its own trail. At the time I knew that a [Flood Fill Algorithm](https://en.wikipedia.org/wiki/Flood_fill) was probably more appropriate, but this was a good opportunity to learn more about trees. The code for this can be seen below.

```python
class Node:
    # If the score for a move was 500k or higher, it was considered a terminal node
    SCORE_THRESHOLD = 500_000

    # Each node stores the information it needs to calculate the scores for each move
    def __init__(self, x, y, depth, trails, target_x, target_y, move=None, score=0):
        self.x = x
        self.y = y
        self.move = move
        self.depth = depth # The search depth (starts at 5, ends at 0)
        self.trails = trails # The list of trails in the game
        self.target_x = target_x
        self.target_y = target_y
        self.score = score
        # Does the move on this node result in being eliminated
        self.is_bad = score and score >= self.SCORE_THRESHOLD 
        self.children = []

        # The children for a node are only generated if depth is available and it isn't a terminal node
        if depth > 0 and not self.is_bad:
            self.generate_children(x, y, move, depth)

    def generate_children(self, x, y, move, depth):
        # Generate the children nodes for each move
        for move_name, move_delta in MOVE_DELTA.items():
            next_x = x + move_delta[0]
            next_y = y + move_delta[1]

            # Generate a score for this move
            score = self.generate_move_score(next_x, next_y)

            # We need to add the bot's moves to the list of trails so collisions can be checked in children nodes
            trails_copy = self.trails.copy()
            trails_copy.append([x, y, 0])

            self.children.append(
                Node(
                    next_x,
                    next_y,
                    depth - 1, # Reduce the depth at each layer
                    trails_copy,
                    self.target_x,
                    self.target_y,
                    move_name,
                    score,
                )
            )

    # The score algorithm only checks for collisions against trails.
    def generate_move_score(self, next_x, next_y):
        is_collision = False
        if self.trails:
            for trail in self.trails:
                if next_x == trail[0] and next_y == trail[1]:
                    is_collision = True
                    break

        if is_collision:
            return 1_000_000
        else:
            return calculate_coordinate_distance(
                next_x, next_y, self.target_x, self.target_y
            )
```

After calculating the best move, the bot checks if that move results in a number of possible moves below a certain threshold. For a given game state, the branch scores may look like `{"up": 2, "down": 43, "left": 0, "right": 85}`. If `up` was the best move in that state (i.e. it is in the bottom half of the grid) it would see that there are only two possible moves after moving up which suggest it is going to get stuck. If this occurs, the bot instead chooses the move with the most possible future moves to increase its changes of surviving. The scoring algorithm for a branch only counted the number of non terminal nodes, it disregarded the individual scores of the nodes.

```python
def calculate_branch_score(node):
    score = 0
    score += (1 if not node.is_bad else 0) + sum(
        [calculate_branch_score(child) for child in node.children]
    )
    return score
```

Applying this new tree based searching algorithm to a bot that tried to stay as close as it could to the centre and a bot that did a circuit around the centre improved their performance, but not by as much as you might expect.

![alt text](/assets/images/2026-09-27-kitg/benchmark-oracle.png)

The reason for their minior jump in win rate but lower average score is because these oracle bots would often move out of the safe zone because they thought moves towards the edge of the safe zone had more possible moves. To help nudge the best move calculation away from the edge of the safe zone I introduced a new scoring metric that capture the "distance to the edge of the safe zone". A move that moved the bot closer to the edge of the safe zone was punished and looked like this.

```python
if is_collision:
    move_scores[move_name] = 1_000_000
elif is_adjacent:
    move_scores[move_name] = 500_000
elif closest_edge_distance <= EDGE_DISTANCE:
    move_scores[move_name] = (
        EDGE_DISTANCE + 1 - closest_edge_distance
    ) * EDGE_COST
else:
    # Calculate the move heuristic
    move_scores[move_name] = abs(next_x - target_x) + abs(next_y - target_y)
```

Adding this behaviour to the best move calculation resulted in the best bot I was able to make during the CTF and it was a huge improvement over all of our other bots.

![alt text](/assets/images/2026-09-27-kitg/benchmark-best.png)

Unfortunately I could not get more replays of the bots playing because the replay system was server side and it was shut down after the CTF ended. Please enjoy my only recording of `fudge_09_oracle_circuit_edge_fixed.py`.

![alt text](/assets/images/2026-09-27-kitg/gameplay-fudge-09.gif)

Although I was able to add the edge scoring to the best move calculations, I ran out of time to incorporate this into the tree search. This meant that if the best move was invalid (due to getting stuck) the bot often chose to move away from the centre and into the danger zone because that's where the most possible moves were. I believe I got close, but I just didn't have the confidence to submit these bots to the server before time ran out because I definitely cooked something in their implementation.

![alt text](/assets/images/2026-09-27-kitg/benchmark-oracle-edge.png)

## Thank You

Despite spending almost all of my time this year on the King of the Grid challenge I am extremely proud of finishing the challenge in first place. A huge shout out to @Isaac and @Eric who helped me win the challenge and to 2g3 and Emu Exploit for putting up such a close fight. Thank you to Geoscape Australia for sending me out this year. Lastly, thank you to joseph for creating the challenge and Skateboarding Dog for hosting another great year of BSides Canberra CTFs.

![alt text](/assets/images/2026-09-27-kitg/scoreboard.png)
