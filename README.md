# bfs: what level-by-level traversal really does, the Python template, the three bugs that break it, and how to tell BFS from DFS in 30 seconds

Most people who type "bfs" into a search box fall into one of two camps. Either they're 25 minutes into a graph problem, they can feel that a queue is involved, and they need the template before their brain melts — or they have a quiz on Monday and want the definition that actually sticks. This covers both, plus the parts most explanations skip: which bugs actually cost people offers, and when BFS is the wrong tool entirely.

## What BFS is, in one paragraph

Breadth-first search explores a graph in layers. You start at a source node, visit every neighbor of that node before touching anything two hops away, then every node two hops away, and so on. The data structure that enforces this order is a plain FIFO queue: nodes come out in the same order they went in, so you can never reach a distance-3 node before finishing distance 2.

A concrete run. Take a graph where node 0 connects to 1 and 4, node 1 connects to 2 and 3, and node 4 has no new neighbors. Queue starts as `[0]`, visited as `{0}`.

1. Dequeue 0, enqueue 1 and 4 → queue `[1, 4]`
2. Dequeue 1, enqueue 2 and 3 → queue `[4, 2, 3]`
3. Dequeue 4, nothing new → queue `[2, 3]`
4. Dequeue 2, dequeue 3, queue empty

Exploration order is `0 → 1 → 4 → 2 → 3`, and the distances are `{0: 0, 1: 1, 4: 1, 2: 2, 3: 2}`. Notice that node 4 — which sits at distance 1 — gets processed before node 2, which sits at distance 2, even though 2 was discovered later in the adjacency list. That ordering guarantee is the entire point. It's what makes the first time you reach a node the shortest path to it, in any graph where every edge costs the same.

## The template you'll actually type in an interview

Python, node-based, single source:

python
from collections import deque

def bfs(start, neighbors):
    queue = deque([start])
    visited = {start}          # mark on enqueue, not on dequeue
    distance = {start: 0}
    while queue:
        node = queue.popleft()
        for nb in neighbors(node):
            if nb not in visited:
                visited.add(nb)
                distance[nb] = distance[node] + 1
                queue.append(nb)
    return distance


When the problem asks for an answer "per level" — tree level order, right-side view, minimum number of moves — use the level-tracking variant instead:

python
level = 0
while queue:
    for _ in range(len(queue)):     # snapshot the current level's size
        node = queue.popleft()
        # ... process, enqueue unvisited neighbors
    level += 1


Two implementation details that aren't optional:

- `deque`, not `list`. `list.pop(0)` is O(n) because every remaining element shifts. Inside a BFS loop that turns an O(V + E) algorithm into O(V²) on a bad input, which is a real failure mode on the larger test cases.
- `len(queue)` snapshot inside the loop, not `while queue` with a level counter incremented per node. Incrementing per node counts individual pops, not levels, and quietly returns the wrong number for anything distance-related.

## Complexity, and why it's that complexity

Time is **O(V + E)**: every vertex enters the queue at most once and leaves at most once, and you look at each adjacency list entry once per endpoint. Space is **O(V)** for the visited set and the queue — the queue itself can hold up to O(V) nodes in a wide graph, which is why BFS on a graph with 10⁶ nodes is a memory decision, not just a correctness one.

The visited set is an invariant worth stating out loud when you're writing on a whiteboard: it guarantees each node is enqueued at most once, and therefore the queue size is bounded by V. Interviewers ask about it because it's the thing that separates a working BFS from one that loops forever.

## Three bugs that break BFS

**1. Marking visited on dequeue instead of enqueue.** This is the classic. If you only add to `visited` when you pop a node, the same node can be pushed multiple times from different predecessors before it's ever processed. On a tree you might get away with it. On a graph with a cycle you get duplicated work at best and a non-terminating loop at worst. Fix: `visited.add(nb)` on the line next to `queue.append(nb)`.

**2. Hand-rolling grid neighbors wrong.** For most grid problems the offsets are:

python
dirs = [(0,1), (0,-1), (1,0), (-1,0)]


For 8-directional movement — Shortest Path in Binary Matrix, Minimum Knight Moves variants — add the diagonals. Bugs here are usually bounds checks (`0 <= nr < rows`) or a missing direction, not the offsets themselves. Worth memorizing the pattern so you're not rebuilding it under time pressure.

**3. Forgetting multi-source BFS exists.** Rotting Oranges: every initially rotten orange goes into the queue at time 0 simultaneously, and each BFS level is one time step. If you instead run one BFS per rotten orange you get a correct-but-quadratic mess. The signal phrase in a problem statement is "starting from all of them at once" or "distance to the nearest X" — that's multi-source, and the fix is literally seeding the queue with more than one node.

> A useful habit for BFS problems: write down what one "level" means before you write code. One hop, one move, one time step, one rotation — the answer to that question determines whether you need level tracking, per-node distances, or both.

## BFS vs DFS: pick in 30 seconds

The decision is usually easier than it looks.

| Signal in the problem | Use |
| --- | --- |
| Shortest path, unweighted graph | BFS |
| "Minimum number of moves / steps / transformations" | BFS |
| Level order, level-by-level output, right side view | BFS |
| State-space search where everything costs the same (Open the Lock, Word Ladder) | BFS |
| Shortest path in a **weighted** graph | Dijkstra, not BFS |
| Does a path exist at all, cycle detection, permutations | DFS |
| Connected components (islands, provinces) | Either — DFS is shorter to write |
| Backtracking, enumeration, memoized recursion | DFS |

AlgoMonster's public decision flowchart asks the same two questions in sequence: "level-order or shortest-level answer?" routes to BFS, and "is the graph weighted?" routes away from BFS entirely to Dijkstra's algorithm. That ordering matters — a lot of candidates reach for BFS on a weighted graph and then wonder why the answer is wrong rather than slow.

One nuance worth knowing: BFS and DFS have the same worst-case complexity, so "BFS is faster" isn't the reason to choose it. You choose BFS for the *ordering guarantee*, not for speed. Writing about ratios of whiteboard time — for narrow, deep trees the two are effectively equivalent in expectation, and the real reason to prefer BFS there is that the queue depth stays bounded.

## The two shapes of BFS problems

Almost every BFS question you'll see in an interview is one of these.

**Grid BFS (implicit graph).** Cells are nodes, edges come from coordinate offsets. Number of Islands, Rotting Oranges, Shortest Path in Binary Matrix. There's no adjacency list — you generate neighbors with the offset array and bounds checks. A common shortcut is mutating the input grid (`grid[r][c] = '0'`) to mark visited, which is fine in an interview as long as you say out loud that you're mutating the caller's input and that you'd use a separate visited set if that weren't allowed.

**Node BFS (explicit or implicit graph).** Nodes are hashable objects — integers, strings, tuples. Either you have an adjacency list, or you have a `neighbors(node)` function that generates them. Word Ladder (words, one letter apart), Open the Lock (4-digit strings, one rotation apart), Clone Graph (node objects).

Both compile to the same four-line loop body. The only thing that changes is `neighbors_fn`. If you internalize that, you stop memorizing per-problem solutions and start recognizing one template wearing different costumes.

## Where BFS actually shows up, by the numbers

AlgoMonster publishes pattern-frequency stats scraped from interview problem datasets, and BFS lands in the highest-ROI tier alongside DFS and two pointers. Their company breakdown is more specific: at Amazon, two pointers, DFS and BFS make up roughly half the questions; at Meta the same three dominate; at unicorn-stage startups, BFS stays a favorite. Google's stated policy is not to reuse problems found online, which is why their rounds often combine two patterns — DFS plus prefix sum, for instance — rather than testing a textbook BFS in isolation.

The practical takeaway: BFS isn't a topic you can skip because you "know it." It's a topic you need to be able to write cold, from a blank editor, in under two minutes, including the visited-set invariant and the grid offsets.

## The canonical problems to practice, in order

If you're building fluency rather than checking a box, this order works because each one adds exactly one new idea:

1. **Number of Islands** — grid BFS, connected components. You can solve it with DFS too; do both.
2. **Rotting Oranges** — adds multi-source seeding and level-as-time.
3. **01 Matrix** — multi-source BFS to nearest zero.
4. **Word Ladder** — node BFS on an implicit graph, plus why bidirectional BFS is the optimization worth mentioning.
5. **Open the Lock** — state-space search with a pre-loaded visited set (deadends).
6. **Shortest Path in Binary Matrix** — 8-directional grid BFS with early exit.
7. **Minimum Knight Moves** — BFS on an infinite implicit graph; the classic "the state space isn't given to you" problem.

Seven problems is roughly a weekend if you're recognizing the template by problem four and writing it from memory by problem six.

## Practicing BFS as a pattern, not a problem list

The trap with BFS is that you can solve a hundred problems and still freeze when a slightly different one shows up. What you actually want is the reflex: read the problem, name the pattern in 30 seconds, then apply a template you've written enough times that it comes out of your fingers.

That's the specific thing [AlgoMonster](https://algo.monster?fpr=li69) is built around — a pattern-first curriculum from ex-Google engineers, where BFS is taught as one node in a graph module rather than as a standalone trick. Instead of only watching solutions, you work through the concept, then a code template, then 3–5 variations of the same pattern so the differences between them stay visible. 👉 [See how the BFS pattern is structured inside the course](https://algo.monster?fpr=li69)

Three things about it are worth knowing before you pay, in both directions:

**What's there.** The Pro plan unlocks all patterns, lessons and problems, unlimited AI assistant usage for hints and concept explanations, unlimited Speedrun for timed practice, and interview questions from major tech companies refreshed monthly. The published content volume is 500+ lessons across 6 courses, 1,000+ illustrations and 3,000+ editorials. The Foundation Course and a Build-an-LLM course are bundled at no extra cost; a Quant Interview Prep course is sold separately.

**What's not.** There is no video content at all — the platform is text plus interactive diagrams, which is efficient if you learn by reading and coding, and dry if you learn by watching someone else type. Third-party reviews consistently flag two other gaps: the system design material is introductory rather than senior-level, and the problem bank is small compared to LeetCode's few thousand. Most people who succeed with it use AlgoMonster for the framework and LeetCode for volume, not one or the other.

**The pricing trade-off is genuinely simple.** There's a yearly option and a one-time lifetime option, and no monthly plan:

| Plan | What you get | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free access | Limited sample content — enough to judge whether the text-and-diagram format works for you | $0 | No card required | [Start with the free content](https://algo.monster?fpr=li69) |
| Pro — Yearly | All patterns, lessons and problems · unlimited AI assistant · unlimited Speedrun · company questions updated monthly · 500+ lessons across 6 courses · bonus Foundation and LLM courses · member discount on live bootcamps · 12 months of access | $8.25/mo, billed annually as $99/year (about $0.28/day) | Annual | [Get Pro yearly access](https://algo.monster?fpr=li69) |
| Pro — Lifetime | Everything in the yearly plan, plus permanent access and all future updates | $189 | One-time | [Get lifetime access](https://algo.monster?fpr=li69) |

The math on yearly versus lifetime: $189 is less than two years at $99. If you expect to interview once in the next six months and never think about BFS again, yearly is the cheaper bet. If you're early in your career and will re-prep before a promotion or a job hop, the lifetime plan covers the second and third rounds of prep without another payment. AlgoMonster runs promotional pricing on the subscribe page fairly often, so the numbers above are the ones currently displayed rather than a permanent list price — worth checking before you decide. If you only want the free sample, you can browse the flowchart and a good chunk of the written content without paying anything.

## Quick answers to what people actually ask

**Is BFS recursive?** Not naturally. The queue is the mechanism; recursion implies a stack, which is DFS. You can write a level-by-level recursive variant, but there's rarely a reason to.

**Can BFS find shortest paths in a weighted graph?** No. BFS assumes every edge costs the same. The moment edge weights differ, you need Dijkstra (non-negative weights) or Bellman-Ford (negative weights allowed).

**BFS or DFS for finding a connected component?** Either. Both are O(V + E) and both find the whole component. DFS is usually less code, which is why Number of Islands is so often written with DFS.

**Why does the same node sometimes appear twice in my queue?** Because you marked it visited on dequeue. Move the `visited.add` call up next to the `append` and it stops happening.

**Does BFS always explore the whole graph?** No — and that's a feature. If you're looking for one specific node or a condition like "reached the boundary," you return as soon as you dequeue a node that satisfies it. Full traversal is only needed when the answer requires distances to everything.

**How long does BFS take to learn properly?** The template, ten minutes. Knowing which of the two shapes a new problem is and writing it without notes, a few focused days of the seven problems above.

## The short version

BFS is a queue, a visited set, and a rule that you mark a node the moment you enqueue it. That gets you O(V + E) traversal and, in unweighted graphs, the shortest path to every reachable node as a side effect. The interview skill isn't the algorithm — it's the 30-second decision between BFS, DFS and Dijkstra, and the ability to write the template without hesitating over where the visited set goes.

Write it from memory three times this week. Then go practice it against problems that don't announce themselves as BFS problems, because that's the version that shows up. 👉 [Browse the full pattern library and start with the graph module](https://algo.monster?fpr=li69)
