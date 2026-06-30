# Zombie Ducks

Zombie Ducks is a small browser game inspired by maze-chase games like Pac-Man.
You play as a normal yellow duck in a ruined, 8-bit pond. Your job is to collect
all the glowing snacks before the zombie ducks catch you.

Play it here:

https://bobitheduck.github.io/zombie-ducks/

## How To Play

- Use the arrow keys or WASD to move.
- On a phone or tablet, use the direction buttons.
- Collect every glowing snack to win.
- Avoid the zombie ducks.
- Use the side portals to escape. Your duck can teleport through them, but the
  zombie ducks cannot.
- Open Settings to choose an unlocked level and a difficulty.
- Level 1 is the original Wrecked Pond.
- Level 2 is Toxic Tunnels, which is bigger and harder.
- You must clear Level 1 before Level 2 unlocks.
- In Level 2, collect sword power points to get one sword attack.
- If the duck has a sword, touching a zombie duck defeats that zombie instead
  of ending the game.
- Difficulty choices:
  - Really Easy
  - Easy
  - Medium
  - Hard
  - Horribly Hard

## Project Shape

This game is made from one file:

```text
index.html
```

That one file contains three main parts:

```html
<style>
  /* CSS: colors, layout, ducks, zombies, buttons, and the 8-bit look */
</style>

<body>
  <!-- HTML: the score bar, game board, settings page, and controls -->
</body>

<script>
  // JavaScript: movement, scoring, zombies, portals, music, and game rules
</script>
```

This is a nice way to learn because you can see the whole game in one place.

## Architecture

The architecture is the way the game is organized.

Zombie Ducks has four big layers:

| Layer | What It Does |
| --- | --- |
| HTML | Creates the buttons, score display, board, message box, and settings page. |
| CSS | Makes the game look 8-bit, spooky, toxic, and apocalyptic. |
| JavaScript state | Stores things that change, like score, unlocked levels, selected level, duck position, zombie positions, and difficulty. |
| JavaScript functions | Run the game: draw the board, move characters, check collisions, and restart. |

## The Maze

The game has a list of mazes. Each maze has a name, a portal row, a player start
position, zombie spawn points, and a map.

```js
const mazes = [
  {
    name: "Wrecked Pond",
    portalRow: 8,
    duckStart: { x: 1, y: 1 },
    map: [
      "#####################",
      "#.........#.........#",
      "#.###.###.#.###.###.#"
    ]
  },
  {
    name: "Toxic Tunnels",
    portalRow: 9,
    duckStart: { x: 1, y: 1 },
    map: [
      "#########################",
      "#...........#...........#",
      "#.###.#####.#.#####.###.#"
    ]
  }
];
```

Inside each maze, the map is stored as an array of strings. Each string is one
row of the board.

```js
const map = [
  "#####################",
  "#.........#.........#",
  "#.###.###.#.###.###.#",
  "#.#.....#...#.....#.#",
  "#.#.###.#####.###.#.#",
  "#...#.........#.....#",
  "###.#.###.#.###.#.###",
  "#.....#...#...#.....#",
  "..###.#.#####.#.###..",
  "#...#...#...#...#...#",
  "###.#.###.#.###.#.###",
  "#.....#.......#.....#",
  "#.###.#.#####.#.###.#",
  "#.#.....#...#.....#.#",
  "#.###.###.#.###.###.#",
  "#.........#.........#",
  "#####################"
];
```

In this map:

- `#` means wall.
- `.` means open space.
- The two open dots at the far left and far right of row 8 are the portals.

The code works out the size of the selected board like this:

```js
let map = currentMaze.map;
let width = map[0].length;
let height = map.length;
```

That means if the map changes size, the game can still understand its width and
height.

## Levels And Unlocking

The current level number is saved in `localStorage`, so the browser can remember
which level was selected.

```js
let mazeIndex = Number(localStorage.getItem("zombieDucksMaze") || 0);
```

The game also remembers the highest unlocked level.

```js
let maxUnlockedMaze = Number(localStorage.getItem("zombieDucksUnlockedMaze") || 0);
```

This helper checks whether a level can be played.

```js
function isMazeUnlocked(index) {
  return index <= maxUnlockedMaze;
}
```

When Level 1 is cleared, the next level unlocks.

```js
function unlockMaze(index) {
  if (!mazes[index] || index <= maxUnlockedMaze) return false;
  maxUnlockedMaze = index;
  localStorage.setItem("zombieDucksUnlockedMaze", maxUnlockedMaze);
  return true;
}
```

The `loadMaze()` function updates the important maze variables.

```js
function loadMaze(index) {
  mazeIndex = mazes[index] ? index : 0;
  currentMaze = mazes[mazeIndex];
  map = currentMaze.map;
  width = map[0].length;
  height = map.length;
  portalRow = currentMaze.portalRow;
  zombieSpawns = currentMaze.zombieSpawns;
}
```

This is why the bigger second level can work without rewriting the whole game.

## Game State

Game state means "the facts about the game right now."

Examples:

```js
let pellets = new Set();
let powerPoints = new Set();
let score = 0;
let swordCharges = 0;
let running = false;
let direction = dirs.right;
let queuedDirection = dirs.right;
let duck = { x: 1, y: 1 };
let zombies = [];
```

These values change while you play.

For example:

- `score` goes up when the duck eats a snack.
- `powerPoints` stores the special sword pickups.
- `swordCharges` says whether the duck has a sword attack ready.
- `duck` stores the duck's current position.
- `zombies` stores all the zombie ducks.
- `running` says whether the game is currently playing.
- `queuedDirection` remembers the direction you want to turn next.

## Coordinates

The board uses `x` and `y` coordinates.

```js
let duck = { x: 1, y: 1 };
```

Think of the board like graph paper:

- `x` means left and right.
- `y` means up and down.
- `{ x: 1, y: 1 }` means the duck starts near the top-left corner.

## Directions

Directions are stored as small objects.

```js
const dirs = {
  up: { name: "up", x: 0, y: -1 },
  down: { name: "down", x: 0, y: 1 },
  left: { name: "left", x: -1, y: 0 },
  right: { name: "right", x: 1, y: 0 }
};
```

For example, moving right adds `1` to `x`. Moving up subtracts `1` from `y`.

## Drawing The Board

The `drawBoard()` function creates the visible maze.

```js
function drawBoard() {
  stage.querySelectorAll(".cell, .actor").forEach((node) => node.remove());
  pellets = new Set();

  for (let y = 0; y < height; y += 1) {
    for (let x = 0; x < width; x += 1) {
      if (map[y][x] === "#") {
        const wall = document.createElement("div");
        wall.className = "cell wall";
        setCellPosition(wall, x, y);
        stage.append(wall);
      }
    }
  }
}
```

This uses nested loops:

- The outer loop goes through every row.
- The inner loop goes through every square in that row.
- If the map has `#`, the code creates a wall.

The real function also creates portals, snacks, the player duck, and zombie ducks.

## Placing Things On The Grid

This helper function puts a game object on the board:

```js
function setCellPosition(node, x, y) {
  node.style.setProperty("--x", x);
  node.style.setProperty("--y", y);
}
```

It sets CSS variables called `--x` and `--y`. Then CSS uses those variables to
move the object to the right place.

This is a good example of JavaScript and CSS working together.

## Making Ducks And Zombie Ducks

The `createDuck()` function creates both the player duck and the zombie ducks.

```js
function createDuck(className, name = "") {
  const wrap = document.createElement("div");
  wrap.className = "actor";
  const body = document.createElement("div");
  body.className = className;
  body.innerHTML = '<span class="eye"></span><span class="wing"></span>';
  if (className === "duck") {
    body.innerHTML += '<span class="lower-bill"></span>';
  }
  if (className === "zombie") {
    body.innerHTML += '<span class="stitch"></span>';
    const tag = document.createElement("span");
    tag.className = "name-tag";
    tag.textContent = name;
    wrap.append(tag);
  }
  wrap.append(body);
  return wrap;
}
```

The same function can make two different things:

- If `className` is `"duck"`, it adds the normal duck beak.
- If `className` is `"zombie"`, it adds a stitch and a name tag.

This is a useful coding idea: one function can be reused in more than one way.

## Difficulty Settings

The difficulty settings are stored in an object.

```js
const difficultySettings = {
  "Really Easy": {
    duckSpeed: 5.05,
    zombieSpeed: 1.8,
    zombieCount: 2,
    note: "Bobi and Mylo only. Slow chase."
  },
  "Horribly Hard": {
    duckSpeed: 4.5,
    zombieSpeed: 5.2,
    zombieCount: 6,
    note: "Six fast zombie ducks. Deep trouble."
  }
};
```

Each difficulty controls:

- how fast the player duck moves,
- how fast the zombie ducks move,
- how many zombie ducks appear,
- and the note shown in the settings menu.

This function returns the settings for the current difficulty:

```js
function activeDifficulty() {
  return difficultySettings[difficulty];
}
```

## Naming The Zombies

The first two zombie ducks are always Bobi and Mylo.

```js
const zombieBaseNames = ["Bobi", "Mylo"];
const randomZombieNames = ["Luna", "Kai", "Mina", "Nova", "Jax", "Ivy", "Rex", "Zara", "Ozzy", "Nico"];
```

Extra zombie ducks get random names:

```js
function shuffledNames() {
  return [...randomZombieNames].sort(() => Math.random() - 0.5);
}
```

This copies the names with `[...randomZombieNames]`, then shuffles them.

## Portals

The portals are at the left and right edges of one row. Each maze can choose a
different portal row.

```js
let portalRow = currentMaze.portalRow;
```

This function checks whether a square is a portal:

```js
function isPortal(x, y) {
  return y === portalRow && (x === 0 || x === width - 1);
}
```

This function teleports the duck:

```js
function portalExit(nextCell, vector) {
  if (nextCell.y !== portalRow || vector.x === 0) return null;
  if (nextCell.x < 0) return { x: width - 1, y: portalRow };
  if (nextCell.x >= width) return { x: 0, y: portalRow };
  return null;
}
```

If the duck leaves the left side, it appears on the right side. If it leaves the
right side, it appears on the left side.

Zombie ducks are blocked from portals here:

```js
function isZombieWall(x, y) {
  return isWall(x, y) || isPortal(x, y);
}
```

For the zombies, portals count as walls.

## Player Movement

The duck moves continuously, not one whole square at a time.

```js
function moveDuck(deltaSeconds) {
  let remaining = activeDifficulty().duckSpeed * deltaSeconds;
  let guard = 0;

  tryBufferedTurn(queuedDirection);

  while (remaining > 0.0001 && guard < 8 && running) {
    guard += 1;
    alignToLane(direction);

    const axis = direction.x !== 0 ? "x" : "y";
    const sign = direction.x || direction.y;
    const current = duck[axis];
    const nextCenter = sign > 0 ?
      Math.floor(current + 0.0001) + 1 :
      Math.ceil(current - 0.0001) - 1;

    const distanceToNext = Math.abs(nextCenter - current);
    const step = Math.min(remaining, distanceToNext);
    duck[axis] += sign * step;
    remaining -= step;
  }
}
```

Important ideas in this function:

- `deltaSeconds` means "how much time passed since the last frame."
- `remaining` is how far the duck can move this frame.
- `axis` chooses whether we are changing `x` or `y`.
- `step` moves only part of a square, which makes movement smooth.
- `guard` stops the loop from accidentally running forever.

## Buffered Turning

Buffered turning means the game remembers your next turn for a short time.

That way, if you press left just before the corner, the duck can turn when it
reaches the corner.

```js
function setDirection(name) {
  const nextDirection = dirs[name];
  if (!nextDirection) return;

  queuedDirection = nextDirection;

  if (running && canReverse(nextDirection) && canMove(duckCell(), nextDirection)) {
    direction = nextDirection;
    alignToLane(nextDirection);
  } else if (running) {
    tryBufferedTurn(nextDirection);
  }
}
```

The keyboard and touch buttons both call `setDirection()`.

## Zombie Movement

The zombie ducks also move continuously.

```js
function moveZombie(zombie, index, deltaSeconds) {
  let remaining = activeDifficulty().zombieSpeed * deltaSeconds;
  let guard = 0;

  while (remaining > 0.0001 && guard < 8) {
    guard += 1;

    if (!zombie.direction) {
      zombie.direction = chooseZombieDirection(zombie, index);
      if (!zombie.direction) break;
    }

    const axis = zombie.direction.x !== 0 ? "x" : "y";
    const sign = zombie.direction.x || zombie.direction.y;
    const current = zombie[axis];
    const nextCenter = sign > 0 ?
      Math.floor(current + 0.0001) + 1 :
      Math.ceil(current - 0.0001) - 1;

    const distanceToNext = Math.abs(nextCenter - current);
    const step = Math.min(remaining, distanceToNext);
    zombie[axis] += sign * step;
    remaining -= step;
  }

  return zombie;
}
```

The zombie uses the same smooth movement idea as the player duck, but it chooses
its own direction.

## Zombie Brains: Pathfinding

The zombie ducks need to decide where to go. They use a pathfinding method called
breadth-first search, often shortened to BFS.

BFS tries nearby squares first, then squares that are farther away. It is useful
for finding a short path through a maze.

```js
function nextToward(start, target) {
  const queue = [start];
  const seen = new Set([key(start.x, start.y)]);
  const parent = new Map();

  while (queue.length) {
    const current = queue.shift();

    for (const next of neighbors(current)) {
      const nextKey = key(next.x, next.y);
      if (!seen.has(nextKey)) {
        seen.add(nextKey);
        parent.set(nextKey, key(current.x, current.y));
        queue.push(next);
      }
    }
  }

  return start;
}
```

In the real game function, when BFS reaches the target, it walks backward through
the `parent` map to find the first step the zombie should take.

Useful BFS words:

- `queue`: a line of squares to check next.
- `seen`: squares already checked.
- `parent`: remembers which square led to which other square.
- `neighbors`: squares next to the current square.

## Different Zombie Personalities

Not every zombie chases in exactly the same way.

```js
function chooseZombieTarget(zombie, index) {
  let target = duckTargetForZombies();
  if (zombie.mood === "ambush") {
    target = {
      x: Math.max(1, Math.min(width - 2, Math.round(duck.x) + direction.x * 3)),
      y: Math.max(1, Math.min(height - 2, Math.round(duck.y) + direction.y * 3))
    };
  }
  if (zombie.mood === "wander" && score % 80 < 30) {
    const corners = [{ x: 1, y: 1 }, { x: 19, y: 1 }, { x: 1, y: 15 }, { x: 19, y: 15 }];
    target = corners[(Math.floor(score / 80) + index) % corners.length];
  }
  return target;
}
```

Zombie moods:

- `direct`: chases the duck.
- `ambush`: aims a few squares ahead of the duck.
- `wander`: sometimes heads toward corners.

This makes the game feel less predictable.

## Collecting Snacks

Snacks are stored in a `Set`.

```js
let pellets = new Set();
```

A `Set` is useful when you want to quickly ask, "Does this thing exist?"

```js
function collectPellet() {
  const cell = duckCell();
  const pelletKey = key(cell.x, cell.y);
  if (pellets.has(pelletKey)) {
    pellets.delete(pelletKey);
    score += 10;
    const pelletNode = stage.querySelector(`[data-key="${pelletKey}"]`);
    pelletNode?.remove();
    updateHud();
  }
}
```

When the duck reaches a snack square:

1. The snack is removed from the `Set`.
2. The score goes up by 10.
3. The snack disappears from the screen.
4. The score display updates.

## Sword Power Points

Level 2 has special power points. They are stored separately from normal snacks.

```js
let powerPoints = new Set();
let swordCharges = 0;
```

When the duck collects one, the game removes that power point and gives the duck
one sword charge.

```js
if (powerPoints.has(pelletKey)) {
  powerPoints.delete(pelletKey);
  swordCharges = Math.max(swordCharges, 1);
  score += 25;
  const powerNode = stage.querySelector(`.power-point[data-key="${pelletKey}"]`);
  powerNode?.remove();
  updateHud();
}
```

The sword charge works like a shield and an attack at the same time. The next
zombie duck the player touches gets defeated for a few seconds.

Defeated zombie ducks are not deleted from the game forever. They are marked as
`defeated`, hidden, and given a respawn time.

```js
const zombieRespawnMs = 5000;
```

When the timer finishes, the zombie duck comes back at its home square.

```js
if (zombie.defeated && now >= zombie.respawnAt) {
  return {
    ...zombie,
    x: zombie.homeX,
    y: zombie.homeY,
    direction: null,
    defeated: false,
    respawnAt: 0
  };
}
```

## Collision Detection

Collision detection means checking if two things touched.

```js
function checkCollision() {
  const zombieIndex = zombies.findIndex((zombie) => !zombie.defeated && Math.hypot(zombie.x - duck.x, zombie.y - duck.y) < 0.55);
  if (zombieIndex === -1) return;

  if (swordCharges > 0) {
    swordCharges -= 1;
    score += 100;
    zombies[zombieIndex] = {
      ...zombies[zombieIndex],
      defeated: true,
      direction: null,
      respawnAt: performance.now() + zombieRespawnMs
    };
    updateHud();
    updateActors();
  } else {
    endGame(false);
  }
}
```

`Math.hypot()` measures the distance between the duck and a zombie duck. If that
distance is small enough, the game checks whether the duck has a sword. With a
sword, the zombie is defeated temporarily. Without a sword, the zombie catches
the player.

## The Game Loop

The game loop is the heartbeat of the game. It runs again and again while the
game is playing.

```js
function loop(now) {
  if (!running) return;

  const deltaSeconds = Math.min((now - lastFrame) / 1000, 0.05);
  lastFrame = now;
  moveDuck(deltaSeconds);
  checkCollision();
  moveZombies(deltaSeconds);
  checkCollision();

  updateActors();
  loopId = requestAnimationFrame(loop);
}
```

Each frame:

1. Work out how much time passed.
2. Move the player duck.
3. Check if a zombie caught the duck.
4. Move the zombie ducks.
5. Check for catching again.
6. Draw everyone in their new positions.
7. Ask the browser to run the loop again.

`requestAnimationFrame()` is the browser's smooth way to animate games.

## Keyboard And Touch Controls

The game listens for keyboard presses:

```js
document.addEventListener("keydown", (event) => {
  const keyMap = {
    ArrowUp: "up",
    w: "up",
    ArrowDown: "down",
    s: "down",
    ArrowLeft: "left",
    a: "left",
    ArrowRight: "right",
    d: "right"
  };

  if (keyMap[event.key]) {
    event.preventDefault();
    setDirection(keyMap[event.key]);
  }
});
```

It also listens for touch or mouse presses on the direction buttons:

```js
document.querySelectorAll("[data-dir]").forEach((button) => {
  button.addEventListener("pointerdown", () => setDirection(button.dataset.dir));
});
```

Both control systems use the same movement function, which keeps the code tidy.

## Saving Best Score And Settings

The game uses `localStorage` to remember things in the browser.

```js
let best = Number(localStorage.getItem("zombieDucksBest") || 0);
```

`localStorage` is like a tiny notebook for a website. It can remember your best
score, unlocked levels, selected level, difficulty, and music setting even after
you refresh the page.

## Music

The music is made with the Web Audio API. That means the browser creates simple
tones instead of loading an audio file.

```js
function playTone(frequency, duration, type = "square", volume = 0.05, delay = 0) {
  const context = ensureAudio();
  if (!context) return;

  const oscillator = context.createOscillator();
  const gain = context.createGain();
  oscillator.type = type;
  oscillator.frequency.setValueAtTime(frequency, context.currentTime + delay);
  oscillator.connect(gain);
  gain.connect(context.destination);
}
```

The game uses square and triangle waves because they sound old-school and 8-bit.

The duck does not make a fake quack sound because a simple 8-bit beep did not
sound much like a real duck. Sometimes the best design choice is to remove a
sound that feels wrong.

## Important Functions

Here is a quick guide to the main functions:

| Function | Job |
| --- | --- |
| `activeDifficulty()` | Gets the current difficulty settings. |
| `createZombies()` | Creates named zombie ducks for the chosen difficulty. |
| `loadMaze()` | Loads the selected maze and updates the board size. |
| `isMazeUnlocked()` | Checks whether a level is unlocked. |
| `unlockMaze()` | Unlocks the next level after a win. |
| `renderMazeOptions()` | Builds the settings page level buttons. |
| `renderDifficultyOptions()` | Builds the settings page difficulty buttons. |
| `ensureAudio()` | Starts the browser audio system if music is on. |
| `playTone()` | Plays one retro music note. |
| `startSoundtrack()` | Starts the background music loop. |
| `isWall()` | Checks whether a square is a wall. |
| `isPortal()` | Checks whether a square is a side portal. |
| `isZombieWall()` | Blocks zombies from walls and portals. |
| `setCellPosition()` | Moves a visible object to a grid position. |
| `createDuck()` | Builds a duck or zombie duck element. |
| `drawBoard()` | Draws walls, portals, snacks, the duck, and zombies. |
| `resetGame()` | Resets score, positions, zombies, and messages. |
| `updateHud()` | Updates score, snacks left, sword count, best score, and difficulty. |
| `moveDuck()` | Moves the player duck smoothly. |
| `tryBufferedTurn()` | Lets the player press a turn slightly early. |
| `collectPellet()` | Handles snack collection, sword pickups, and scoring. |
| `neighbors()` | Finds open squares around a zombie. |
| `nextToward()` | Uses BFS pathfinding to find a route through the maze. |
| `chooseZombieTarget()` | Decides what each zombie is chasing. |
| `moveZombie()` | Moves one zombie smoothly and respawns defeated zombies. |
| `checkCollision()` | Checks if a zombie caught the duck. |
| `loop()` | Runs the game every animation frame. |
| `setDirection()` | Handles keyboard and touch movement input. |

## Programming Concepts You Can Learn From This Game

### Variables

Variables store information that can change.

```js
let score = 0;
let running = false;
```

### Constants

Constants store information that should not be reassigned.

```js
const portalRow = 8;
```

### Arrays

Arrays hold lists.

```js
const randomZombieNames = ["Luna", "Kai", "Mina", "Nova"];
```

### Objects

Objects hold named pieces of information.

```js
let duck = { x: 1, y: 1 };
```

### Functions

Functions are reusable instructions.

```js
function key(x, y) {
  return `${x},${y}`;
}
```

### Loops

Loops repeat work.

```js
for (let y = 0; y < height; y += 1) {
  for (let x = 0; x < width; x += 1) {
    // Check each square.
  }
}
```

### Conditions

Conditions let code make decisions.

```js
if (isWall(nextCell.x, nextCell.y)) {
  break;
}
```

### Events

Events let the player control the game.

```js
startButton.addEventListener("click", startGame);
```

### The DOM

The DOM is the browser's tree of HTML elements. JavaScript can create, remove,
and change those elements.

```js
const wall = document.createElement("div");
stage.append(wall);
```

### Animation

Animation happens by changing positions many times per second.

```js
loopId = requestAnimationFrame(loop);
```

### Pathfinding

Pathfinding is how a character finds a route through a maze. The zombies use BFS
to chase the duck around walls.

## Ideas To Try Next

- Add a power-up that scares zombie ducks for a few seconds.
- Add a third maze.
- Give each named zombie a different color.
- Add a pause button.
- Add a high-score screen.
- Make the portals flash when the duck uses them.

Small changes are the best way to learn. Change one thing, refresh the game, and
see what happens.
