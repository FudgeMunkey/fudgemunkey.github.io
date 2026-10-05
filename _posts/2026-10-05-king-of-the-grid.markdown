---
layout: post
title:  "How we became Kings of the Grid at BSides Canberra 2026"
date:   2026-10-05 17:19:00 +1100
categories: ctf
---

## BSides Canberra 2026

BSides Canberra is my favourite conference to attend each year because competing in the Capture The Flag (CTF) competitions is always such a highlight. This year was no different as we represented the **h4-x_X_x-0rs: Sniper Elite Squad** and placed 25th with 1781 points out of 210 teams.

An exciting spin on this year's CTF was the introduction of King of the Hill (KOTH) challenges where the best solution achieved the most points. This was an exciting change because it meant that the difficulty of the challenge wasn't decided by Skateboarding Dog (the creators), but instead the solutions of all the other teams. If you really enjoyed a KOTH challenge it meant that you could spend the entire competition improving your solution in the hopes of achieving that Top Dog spot. Naturally, this is exactly what ended up happening...

## King of the Grid

King of the Grid (KOTG) was one of the new KOTH style challenges this year and it totally absorbed all my attention by combining two of my favourite things - coding and video games. To win the KOTG challenge, you had to create the best bot for playing an inspired version of the [Tron](https://en.wikipedia.org/wiki/Tron) game. In the original Tron game, players would ride a "light cycle" in a flat arena with the goal of eliminating their opponents. Their light cycles would leave a permenant wall behind them which their opponents could crash into. The KOTG challenge was similar, but there were a few key differences. The humans on light cycles were now dogs on skateboards, the walls left behind were up to ~12 units long and temporary, the dogs could boost to move more quickly, and the safe arena in the arena would shrink over time forcing players to the centre. The game screen looked like this.

![Screenshot of King of the Grid challenge game](/assets/images/2026-10-05-kotg/screenshot.png)

Creating a bot was simple. At every game tick, you needed to return a dictionary with a move (up, down, left, right) and an optional boost flag (true, false). To be able to make your choice, you were provided with a `decide()` function and the game's current state. Below is the `example_bot.py` that was provided to teams.

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

The example bot was simple, but it worked quite well. It was impossible for it to run into its own trail and unless it spawned right next to another bot, it was unlikely to collide with another bot's trail. The example bot's biggest weakness was that it would eventually be eliminated by the danger zone due to the safe zone shrinking over time.

![Bots are allowed to move in the safe zone but it shrinks by one square every 5 game ticks. Players in the danger zone are eliminated.](/assets/images/2026-10-05-kotg/zones.png)

## Controlling the Centre

After watching some of the bots compete on the server it was clear that being the last bot alive gained you a lot of points. You would get 1000 points for being the last player alive as a win bonus and you would get 0 - 800 points depending on how long you were alive. You could also get a lot of points for eliminating other players, but trying to survive seemed like the most reliable strategy.

As the safe zone shrunk every five game ticks, I thought that getting to the centre of the grid and preventing other bots getting there was the best way to win. In chess, this strategy is known as [Controlling the Centre](https://www.chess.com/blog/Gertsog/how-to-control-the-center-and-why-its-important) and I thought I could apply the same idea here. I achieved this by setting a target of `x: 16, y: 16` (on a 32x32 grid) and moved the bot directly to the centre.

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

`fudge_01` moved to the centre quickly, but it did a poor job at controlling the centre. To better control the centre, it needed to increase the space it was taking up so other bots could not easily enter it. I followed the example bot's strategy and moved around the centre of the map in a square. Once it was near the centre of the map, the bot swapped to "centre mode" which looked like this.

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

`fudge_04` performed very well and managed to top the leaderboard for the first few hours of the competition. The path of `fudge_04` during a game looked like this.

![Fudge 04 moves from its spawn point to the centre of the map. Once it is near the centre of the map, it runs a circuit around the centre to increase its control of the centre of the grid.](/assets/images/2026-10-05-kotg/control-the-centre.png)

## Benchmarking

A big problem I faced when trying to create a new bot was knowing if it was actually better than the previous bot I made and knowing if it was better than the other team's bots. The only way to know how well your bot did against the other teams was to submit it to the server where games were played every two minutes. Two minutes is a long time to wait for the results of a single match that may not be representative of your bot's strengths and weaknesses.

Skateboarding Dog did provide you with the game's engine code and the scripts to run the game locally which greatly decreased the feedback time, but you could only run scripts for the bots you made yourself. I ran this command to check if my new bot was an improvement over the previous bot `clear && python play.py -n 2 --replay replay.json bot_04.py new.py`. This worked well but only comparing your new bots against your previous best means you are only trying to beat your last strategy. Before submission I would run `clear && python play.py -n 5 --replay replay.json bot_01.py bot_02.py bot_03.py bot_04.py new.py` but tracking a bot's performance over multiple games became very difficult.

To address this issue, [@Isaac][isaac-github] created a benchmarking script so we could easily compare the performance of all our bots in a local competition. The script would play thousands of games where each game had a random sample of 10 bots on a new seed. The results would be summarised in a table for us to review. [@Isaac][isaac-github] and [@Eric][eric-github] both created many more bots with a variety of different strategies to reduce the risk of hyper optimising a bot's performance against its last version. The output for that tool looked like this.

![Benchmarking results on a variety of bots from the Friday night.](/assets/images/2026-10-05-kotg/benchmark-start.png)

Obviously making tools like this during a CTF take up a lot of time that could have been spent making a better solution for our bot, but tools like this help you make better solutions. Decreasing the feedback time for a new iteration of the bot and having the confidence to say "yes this bot is now definitely better than our previous ones" before submitting it to the server is just so valuable. This tool and the other bots didn't exist until the Friday night after `fudge_05` was made, but I will use it from now on to help quantify how much the bots were improving. These are the benchmarking results for the centre bots `fudge_01` and `fudge_04`. At the time, they were first on the server but compared to the bots we had when the benchmarking tool was created, they performed poorly.

![Benchmarking results for Fudge 01 and Fudge 04. Fudge 01 (straight to the centre) places eighth with an average win rate of 9.3% and an average score of 851. Fudge 04 (square around centre) places ninth with an average win rate of 5.2% and an average score of 714.](/assets/images/2026-10-05-kotg/benchmark-centre.png)

## Avoiding Immediate Collisions

`fudge_04` was able to move toward and control the centre of the grid, but failed to avoid immediate collisions with other bot's trails. This weakness caused the bot to eliminate itself very early into the game and score very few points. `fudge_04` did not consider the game `state` when deciding on a move and always chose the move that got it closer to the centre or moved it in a square around the centre of the grid. In the `decide()` function we were given access to the `state` object which represented the entire state of the game at that game tick. After parsing the trail coordinates from the `players.trail` object, `fudge_01_collision` and `fudge_04_collision` were able to consider player trails when calculating the best move.

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

When scoring the best move I considered collisions with trails to be worth `1_000_000` points as they are the worst thing a bot can do - they get eliminated. All other moves kept the bot alive and were scored based on their distance to their goals. I sorted each of the moves based on the score and chose the lowest scoring move. An example of the bot's choices may look like `{"right": 11, "up": 12, "down": 12, "left": 1000000}` which means it would choose to move `right`. If multiple moves are tied for the lowest score, then a random tied move is chosen. Adding this logic to `fudge_01_collision` and `fudge_04_collision` greatly increased their performance.

![Benchmarking results for Fudge 01 with collision avoidance and Fudge 04 with collision avoidance. Fudge 01 Collision places fourth with an average win rate of 19.1% and an average score of 1207. Fudge 04 Collision bot places first with an average win rate of 26.8% and an average score of 1403.](/assets/images/2026-10-05-kotg/benchmark-centre-collision.png)

At this point there were now four bots in our local competition moving towards the same goal (centre of the grid) which surfaced a common issue. If two bots wanted to move to the same location in the same game tick, they would collide with and eliminate each other. This was a big problem for our local competition but not such a big problem on the actual server. To fix this issue I considered coordinates that were `adjacent` to other players to be worth `500_000` points as there was a chance to collide with the other players but it did not guarantee an elimination. An example of a bot's choices looked like `{"right": 11, "up": 12, "down": 500000, "left": 1000000}` which means the bot would choose to move `right`. Adding this logic to `fudge_05` greatly improved its performance.

![Benchmarking results for Fudge 05 which avoids immediate and adjacent collisions. Fudge 05 places first with an average win rate of 38.2% and an average score of 1842.](/assets/images/2026-10-05-kotg/benchmark-adjacent.png)

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

`fudge_05` is able to move towards the centre, move in a square around the centre, avoid immediate collisions with all trails, and avoid possible collisions in squares adjacent to other players. It is however, only able to look ahead one move which works fine most of the time, but there are some scenarios where the bot gets stuck on its own trail. To avoid situations like the one shown below, the bot needs to be able to look at least two moves ahead.

```
# B is the current location of the bot and X is its trail

[ ][ ][ ][ ][ ]
[ ][ ][X][X][X]
[ ][ ][B][ ][X]
[X][X][X][X][X]
[ ][ ][ ][ ][ ]

# If the bot moves left it is okay, but if it moves right it gets stuck.
```

To fix this issue, I implemented a tree based search to look up to five moves ahead. I only simulated the bot's moves and not the entire state of the game for all bot moves. I was really only aiming to address the issue where the bot was getting stuck on its own trail and not simulate all possible moves by all players. At the time I knew that a [Flood Fill Algorithm](https://en.wikipedia.org/wiki/Flood_fill) was probably more appropriate, but this was a good opportunity to learn more about trees. The code for this can be seen below.

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

After calculating the best move, the bot checks if that move results in a number of possible moves below a certain threshold. For a given game state, the branch scores may look like `{"up": 2, "down": 43, "left": 0, "right": 85}`. If `up` was the best move in that state (i.e. it is in the bottom half of the grid) it would see that there are only two possible moves after moving up which suggest it is going to get stuck. In this scenario, the bot chooses the move with the most possible future moves to increase its chances of surviving. The scoring algorithm for a branch only counted the number of non terminal nodes, it disregarded the individual scores of the nodes.

```python
def calculate_branch_score(node):
    score = 0
    score += (1 if not node.is_bad else 0) + sum(
        [calculate_branch_score(child) for child in node.children]
    )
    return score
```

Applying this new tree based searching algorithm to `fudge_06` (centre) and `fudge_07` (centre circuit) improved their performance, but not by as much as you might expect.

![Benchmarking results for Fudge 06 and Fudge 07 who look five moves ahead. Fudge 06 moves to the centre of the grid and places third with an average win rate of 23.4% and an average score of 1572. Fudge 07 makes a circuit around the centre of the grid and places second with an average win rate of 25.6% and an average score of 1626. Fudge 05 which makes a circuit around the centre and only looks one move ahead places first with a win rate of 23.1% and an average score of 1708.](/assets/images/2026-10-05-kotg/benchmark-oracle.png)

The reason for their higher win rate but lower average score was because the oracle bots would move out of the safe zone and into the danger zone because they thought there were more possible moves. To nudge the best move calculation away from the edge of the safe zone I introduced a new scoring metric that capture the "distance to the edge of the safe zone". Moves towards the edge of the safe zone were punished and looked like this.

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

Adding this behaviour to `fudge_09` resulted in the best bot and final submission I made for the CTF. It was a huge improvement over all of our other bots.

![Benchmarking results for Fudge 09. Fudge 09 places first with an average win rate of 37.0% and an average score of 1944.](/assets/images/2026-10-05-kotg/benchmark-best.png)

Please enjoy my only recording of the `fudge_09` bot. Unfortunately I could not get more replays of the bots playing because the replay system was on the server only and it was shut down after the CTF ended.

![Gif of Fudge 09 playing out a match and winning.](/assets/images/2026-10-05-kotg/gameplay-fudge-09.gif)

Although I was able to add the edge scoring to the best move calculations, I ran out of time to incorporate this logic into the tree search properly. This meant that if the best move was invalid (due to getting stuck) the bot often chose to move away from the centre and into the danger zone because that's where the most possible moves were. I believe I got close with `fudge_11` and `fudge_12`, but when viewing replays and viewing the logs something was definitely wrong with their implementation. I just didn't have enough confidence to submit them to the server despite the remarkable benchmarking results. Here are those benchmarking results.

![Benchmarking results for Fudge 11 and Fudge 12 which aimed to fix the branch search of the edge calculations. Fudge 09 places third with an average win rate of 22.8% and an average score of 1668. Fudge 11 places second with an average win rate of 31.7% and an average score of 1704. Fudge 12 places first with an average win rate of 34.4% and an average score of 1807.](/assets/images/2026-10-05-kotg/benchmark-oracle-edge.png)

## Thank You

Despite spending almost all of my time this year on the King of the Grid challenge I am extremely proud of finishing the challenge in first place among 33 other teams. A huge shout out to [@Isaac][isaac-github] and [@Eric][eric-github] who helped me win the challenge and to `2g3` and `Emu Exploit` for putting up such a close fight. Thank you to Geoscape Australia for sending me out this year. Lastly, thank you to joseph for creating the challenge and Skateboarding Dog for hosting another great year of BSides Canberra CTFs. I can't wait for next year and I hope there will be another challenge like this!

![An image of the scoreboard where our team placed first in the King of the Grid challenge with 1518 points. 2g3 placed second with 1474 points and Emu Exploit placed third with 1414 points.](/assets/images/2026-10-05-kotg/scoreboard.png)

[isaac-github]: https://github.com/IsaacPushButton
[eric-github]: https://github.com/eb-h
