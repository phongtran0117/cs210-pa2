# PA2 Reflection

## 1. What each function does, and its time complexity

n = the number of scenes in the story. h = the height of the tree (the longest path), which is at most n.

### Scene.h

- **TODO 1, constructor: O(1).** If the scene has at least one choice, it allocates exactly `numChoices_` slots with `new Scene<T>*[numChoices_]` and sets every slot to `nullptr`, because new slots hold garbage addresses. An ending skips the allocation, so `choices_` stays `nullptr`. Since a scene can never have more than `MAX_CHOICES` (3) slots, the work is constant.
- **TODO 2, destructor: O(size of the subtree), O(n) for the root.** It deletes every child scene in the slots, then frees the slot array with `delete[]`, then deletes `data_`. Deleting a child runs that child's destructor too, so deleting one scene frees its whole branch. Deleting the root frees all n scenes.
- **TODO 3, isEnding: O(1).** Returns true when `numChoices_ == 0`.
- **TODO 4, getChoice: O(1).** Checks that the index is between 0 and `numChoices_ - 1` (using `||`, which I first got wrong with `&&`), returns `nullptr` for a bad index, and otherwise returns the scene in that slot. For an ending, every index fails the check, so it never touches the `nullptr` array.
- **TODO 5, setChoice: O(1).** Same range check, then stores the scene in the slot. A bad index does nothing.

### StoryTree.h

- **TODO 6, countScenes: O(n).** Recursive. An empty story has 0 scenes. Otherwise it counts 1 for this scene plus `countScenes` of every child. Every scene is visited once.
- **TODO 7, countEndings: O(n).** Recursive. Returns 1 for an ending. For any other scene it starts at 0 (the scene itself is not an ending) and adds up the endings below each child. Every scene is visited once.
- **TODO 8, height: O(n).** Recursive. An empty story has height -1. For a scene, it finds the biggest height among its children and adds 1. I start `best` at -1 so an ending comes out to -1 + 1 = 0 without a special case. Every scene is visited once.
- **TODO 9, clear: O(n).** `delete root_` then `root_ = nullptr`. The Scene destructor does the rest, so the one delete frees all n scenes.

### Adventure.cpp

- **TODO 10, load: O(n).** Opens the file (printing the exact error if it can't), reads `START_HP` with `readField` and `std::stoi`, builds the whole story with `readScene(in, "START")`, and gives the root to `tree_.setRoot`. Returns false if anything failed. The work is the `readScene` call, which is O(n).
- **TODO 11, readScene: O(n) total.** Recursive. Reads the SCENE, HP, LOOT, ITEM, and CHOICES lines, converts the numbers, and rejects a CHOICES value below 0 or above 3 before creating anything. Then it creates the Moment and the Scene, and for each choice reads a LABEL line and calls itself to read that door's whole scene. If a LABEL or a child fails, it does `delete scene` and returns `nullptr`. That one delete frees everything built so far at that level, because filled slots hold real children and empty slots are still `nullptr`. Each scene in the file is read once, so the total is O(n).
- **TODO 12, askChoice: O(1) per attempt.** Prints `Your choice: `, reads a whole line with `getline`, and compares it to `"1"`, `"2"`, ... up to `numChoices` using `std::to_string`. A match returns that number. Anything else (banana, 7, an empty line) prints the reprompt and asks again. If input ends it returns 0. Each attempt checks at most 3 values, so it does not depend on n. How many attempts it takes depends on the player.
- **TODO 13, play: O(h) scenes, plus a bag search at each item.** Starts at the root with `startHp_`, 0 loot, and an empty `List<Item>` bag. On each scene it adds the HP and loot, prints the text, picks up the item (using `search` to skip duplicates and `addFront(new Item(...))` to add), and prints HP and loot. It stops on a game over or an ending, otherwise shows the doors and moves to the chosen child. It only walks one path, so at most h scenes. Each item check also searches the bag, which is O(k) for k items (see question 6).
- **TODO 14, printStats: O(n).** Prints the three numbers from `tree_.countScenes()`, `tree_.countEndings()`, and `tree_.height()`. Three O(n) walks is still O(n).
- **TODO 15, printSpoilers: O(n).** Calls `printEndings(tree_.getRoot(), "")` to start from the root with an empty path.
- **TODO 16, printEndings: O(n).** Recursive. At an ending it prints `Path <path>: <text>` (or `Path (start):` if the root is an ending). Otherwise it calls itself on each child with `extendPath(path, i + 1)`, in order 1, 2, 3. Every scene is visited once.
- **TODO 17, greedyRun: O(h).** A loop that enters scenes the same way `play` does, but picks the door itself: the child with the most loot right now, using `>` so ties keep the lower choice number. It never looks ahead and never checks HP before choosing. It stops when HP hits 0 or it reaches an ending, then prints the path, loot, and result. It follows one path, so at most h scenes, with up to 3 comparisons per scene.
- **TODO 18, bestLoot: O(n).** Recursive. Applies this scene's HP change and returns -1 if that kills you. At an ending it saves the path and returns the scene's loot. Otherwise it tries every child, each with its own `childPath` string so a losing child can't overwrite the winner's path, and keeps the biggest result with `>`. If every child returned -1 it returns -1, otherwise the best child's loot plus this scene's loot. Every scene is visited once (see question 5).
- **TODO 19, optimalRun: O(n).** Calls `bestLoot` once from the root with `startHp_` and an empty path, then prints the winning path and loot, or "no path survives this story" if it got -1.

## 2. Why exact-size allocation instead of `Scene<T>* choices[3]`? What does an ending hold in `choices_`?

A fixed `choices[3]` would give every scene 3 slots even when it uses fewer. In my Minecraft story, 6 of the 15 scenes are endings and most of the others have only 1 or 2 doors. With exact-size allocation the whole story uses 14 slots total (one for every scene except the root, since each scene sits behind exactly one door). With `choices[3]` it would be 15 x 3 = 45 slots, so 31 of them would be wasted. Exact-size also keeps the array and `numChoices_` in agreement, so the number of slots always equals the number of real choices.

An ending holds `nullptr` in `choices_`, because my constructor only allocates when `numChoices_ > 0`. That is safe everywhere else: the destructor's loop runs 0 times, `delete[] nullptr` does nothing, and `getChoice` rejects every index because `numChoices_` is 0, so nothing ever reads from the null array.

## 3. Walk through the Scene destructor. What gets deleted, in what order, and what goes wrong if `choices_` is deleted first?

My destructor does three things in this order:

1. A loop that runs `delete choices_[i]` for every slot. Each delete runs that child's destructor, which deletes its own children, so the whole branch below this scene is freed. Empty slots are `nullptr`, and deleting `nullptr` is safe.
2. `delete[] choices_` frees the slot array. It uses `delete[]` because the array was made with `new[]`.
3. `delete data_` frees this scene's Moment. At first I wrote `delete[] data_`, which compiled but was the wrong form, since the Moment was made with plain `new`. The memory checker caught it and I changed it to `delete`.

The children have to go before the array, because the loop reads the child addresses out of the array. If `delete[] choices_` ran first, the loop would read slots from memory that was already freed. That is undefined behavior: it can crash, or read garbage addresses and delete the wrong memory. And if you freed the array and skipped the loop, every child scene would leak, because nothing would point to them anymore. `data_` does not depend on the other two, so its position does not matter.

## 4. Greedy vs optimal on my story

| | Path | Loot | Result |
|---|---|---|---|
| Greedy | 1-1-1 | 10 | died |
| Optimal | 2-1-1-1-1 | 19 | reached an ending |

At the start, greedy compares the next scenes' loot: digging straight down (6), the creeper cave (3), and digging slowly (1). It takes the 6. At the next scene it compares digging deeper (4) against climbing back up (2) and takes the 4. That scene only has one door, the lava pool, so Steve goes from 10 HP to 8, then 4, then -6, and dies with 10 loot.

Optimal goes into the cave. That route starts with less loot and costs HP fighting creepers, zombies, and the mutant skeleton, but Steve heals once while eating, so he still has 4 HP when he finds the 10 diamonds and crafts the Diamond Pickaxe. That route is worth 19 loot.

Greedy lost because it only looks one step ahead at loot and never checks HP. The dig-down route puts its big loot first and its damage later, and the cave route puts its damage first and its big payoff at the end. I also made sure the "climb back up" ending has only 2 loot. When I tested it with 5, greedy picked climbing out instead of digging deeper and survived, which ruined the lava trap.

## 5. Why is bestLoot O(n) and not exponential?

`bestLoot` tries every path, but in a tree every scene has exactly one parent, so there is only one way to reach each scene from the root. The recursion calls `bestLoot` once on each child, and each child is only a child of one scene, so every scene is visited exactly once. Each call does a constant amount of its own work (at most 3 children to compare), so the total is O(n).

The number of paths it "tries" is the number of endings, because every path ends at a leaf. A tree with n scenes has at most n leaves, so there can't be exponentially many paths. On my story, `bestLoot` runs 15 times (once per scene) and compares 6 paths (one per ending). Exponential time would only happen if paths shared scenes, like in a graph where two doors lead to the same scene, so the same scene would get solved again and again.

## 6. Which List methods does my game use, and what does one search cost as the bag grows?

My `play` function uses:

- **`search(Item(name))`** to check for a duplicate before picking up an item. I pass a temporary `Item`, because search only looks at it and doesn't keep it.
- **`addFront(new Item(name))`** to put a new item in the bag. It needs `new` because the list owns the item and deletes it later.
- **`size()`** for the `Inventory (<count> items):` line.
- **`print()`** to list the items, one per line. They print newest first because `addFront` puts each new item at the front.
- The **destructor** (through `deleteFront`), which runs automatically when `play` ends, because the bag is a local variable. That frees every Item, so the bag never leaks.

One `search` walks the linked list from the head and compares each item's name until it finds a match or reaches the end. So it is O(k), where k is the number of items in the bag: twice as many items means up to twice as many comparisons. In my story every path picks up at most one item, so the bag stays tiny, but in a long story with an item on every scene the searches would add up to O(k) per scene.
