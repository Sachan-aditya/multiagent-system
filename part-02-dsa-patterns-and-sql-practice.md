# Part 2 of 13: DSA Patterns (Section 19) and SQL Practice (Section 23)

> **Assumptions** (you didn't specify, tell me to change them): language = **Python 3**, target = **both SDE1 and SDE2**. Items tagged `[SDE2]` are optional for SDE1.
> Problem names are from memory and may differ slightly from the platform's titles; verify on the platform.
> Every "Your turn" question has a model answer in the **Answer Key** at the end.

---

## Part A: Data Structures and Algorithms

### A.0 How to Solve Any Problem in an Interview

1. **Clarify:** input size, value ranges, duplicates, negatives, empty input, sorted or not, memory limits.
2. **Brute force first:** say it out loud with its complexity. It proves you understand the problem.
3. **Find the bottleneck:** what work is repeated? What information is being thrown away?
4. **Pick a pattern** (below) and state *why* it fits.
5. **Dry-run** on a small example, including an edge case.
6. **State time and space complexity**, then test edge cases.

**Constraint-to-complexity cheat sheet** *(rule of thumb for ~1 second of Python/Java work)*:

| n | Acceptable complexity | Typical technique |
|---|---|---|
| n <= 10-12 | O(n!) | permutations, brute force |
| n <= 20-25 | O(2^n) | subsets, bitmask DP, backtracking |
| n <= 500 | O(n^3) | Floyd-Warshall, interval DP |
| n <= 5,000 | O(n^2) | 2D DP, pairs |
| n <= 10^5-10^6 | O(n log n) | sorting, heap, binary search |
| n <= 10^7+ | O(n) | hash map, two pointers, prefix sums |

**Level targets** *(illustrative, adjust to your timeline)*:
- **SDE1:** arrays, strings, hash maps, linked lists, stacks, trees, BFS/DFS, binary search, basic DP. Mostly easy/medium.
- **SDE2:** all of that plus graphs, heaps, advanced DP, backtracking, tries, union-find, and medium-hard problems with clean optimization discussion.

---

### A.1 Two Pointers `[Both]` P0

**Recognize:** sorted array or string; "find a pair/triplet"; palindromes; in-place removal or partition; merging two sorted sequences.

```python
def two_sum_sorted(a, target):
    l, r = 0, len(a) - 1
    while l < r:
        s = a[l] + a[r]
        if s == target:
            return [l, r]
        if s < target:
            l += 1          # need a bigger sum: only moving l can help
        else:
            r -= 1          # need a smaller sum: only moving r can help
    return []
```

**Why it works:** in a sorted array, if `a[l] + a[r] < target`, then `a[l]` paired with *any* index <= r is also too small, so `l` can be discarded safely. Each step discards at least one candidate, so total work is O(n).

**Alternatives:** hash map O(n) time, O(n) space (works unsorted); brute force O(n^2). For 3Sum: sort, fix one element, two pointers on the rest = O(n^2).

**Edge cases and mistakes:** empty/one element; duplicates in 3Sum (skip equal values after sorting); using `l <= r` when you need distinct elements; forgetting the array must be sorted.

**Practice:** Two Sum II; Valid Palindrome; Remove Duplicates from Sorted Array; 3Sum; Container With Most Water.

**Production relevance:** merging sorted streams, deduplicating sorted logs, comparing two ordered result sets.

**Your turn (A1).** In Container With Most Water, why is it safe to always move the pointer at the *shorter* line?

---

### A.2 Sliding Window `[Both]` P0

**Recognize:** contiguous subarray/substring; "longest/shortest/at most K/exactly K"; running count or sum over a window.

```python
# Variable window: longest substring without repeating characters
def longest_unique_substring(s):
    last = {}                       # char -> last index seen
    left = best = 0
    for right, ch in enumerate(s):
        if ch in last and last[ch] >= left:
            left = last[ch] + 1     # jump left past the duplicate
        last[ch] = right
        best = max(best, right - left + 1)
    return best

# Shrinkable window (positive numbers only): minimum subarray with sum >= target
def min_subarray_len(target, nums):
    left = total = 0
    best = float("inf")
    for right, x in enumerate(nums):
        total += x
        while total >= target:
            best = min(best, right - left + 1)
            total -= nums[left]
            left += 1
    return 0 if best == float("inf") else best
```

**Why it works:** each element enters the window once and leaves once, so O(n) total, even with the inner `while`. The window invariant (e.g., "no duplicates inside") is restored by moving `left`.

**Alternatives:** brute force O(n^2). If the array has **negative numbers**, the "shrink while sum >= target" logic breaks; use prefix sums plus a monotonic deque or hash map.

**Edge cases and mistakes:** window shrink condition wrong; not updating the answer at the right time; "exactly K" = `atMost(K) - atMost(K-1)`.

**Practice:** Maximum Average Subarray I; Longest Substring Without Repeating Characters; Minimum Size Subarray Sum; Longest Repeating Character Replacement; Minimum Window Substring.

**Production relevance:** rate limiters (sliding window counters), rolling metrics (last 5 min error rate), anomaly detection.

**Your turn (A2).** Write "longest substring with at most K distinct characters" and state its complexity.

---

### A.3 Prefix Sums and Kadane `[Both]` P0

**Recognize:** many range-sum queries; "subarray sum equals K"; "maximum subarray"; counting subarrays with a property.

```python
def subarray_sum_equals_k(nums, k):          # works with negatives too
    count = prefix = 0
    seen = {0: 1}                            # prefix value -> times seen
    for x in nums:
        prefix += x
        count += seen.get(prefix - k, 0)     # earlier prefix = prefix - k => subarray sums to k
        seen[prefix] = seen.get(prefix, 0) + 1
    return count

def max_subarray(nums):                      # Kadane
    cur = best = nums[0]
    for x in nums[1:]:
        cur = max(x, cur + x)                # extend or restart
        best = max(best, cur)
    return best
```

**Why it works:** subarray sum(i..j) = prefix[j] - prefix[i-1]. So "sum = k" means finding an earlier prefix equal to `prefix[j] - k`. Kadane: the best subarray ending here either extends the previous best or starts fresh.

**Alternatives:** brute force O(n^2) or O(n^3); divide and conquer for max subarray O(n log n).

**Edge cases and mistakes:** forgetting `seen[0] = 1` (misses subarrays starting at index 0); all-negative arrays in Kadane (initialize with `nums[0]`, not 0).

**Practice:** Running Sum of 1D Array; Range Sum Query; Maximum Subarray; Subarray Sum Equals K; Continuous Subarray Sum.

**Production relevance:** cumulative metrics, time-series windowed aggregates, cumulative distribution in analytics.

**Your turn (A3).** Count subarrays whose sum is divisible by K. What do you store in the map?

---

### A.4 Hash Maps and Sets (including LRU Cache) `[Both]` P0

**Recognize:** need O(1) lookup, counting, grouping, deduplication, "have I seen this before?"

```python
from collections import OrderedDict, defaultdict

def group_anagrams(words):
    groups = defaultdict(list)
    for w in words:
        groups[tuple(sorted(w))].append(w)   # or a 26-count tuple for O(L) key
    return list(groups.values())

class LRUCache:                               # O(1) get/put
    def __init__(self, capacity):
        self.cap, self.d = capacity, OrderedDict()
    def get(self, key):
        if key not in self.d:
            return -1
        self.d.move_to_end(key)               # mark most-recent
        return self.d[key]
    def put(self, key, value):
        if key in self.d:
            self.d.move_to_end(key)
        self.d[key] = value
        if len(self.d) > self.cap:
            self.d.popitem(last=False)        # evict least-recent
```

**Why it works:** a hash table gives expected O(1) insert/lookup. LRU needs both fast lookup *and* fast reordering/eviction: hash map + doubly linked list (or `OrderedDict`, which does exactly that). In an interview, be ready to implement the doubly linked list by hand.

**Alternatives:** sorting then scanning (O(n log n), no extra memory for some problems); arrays as counters when the key range is small; LFU (frequency buckets), TTL caches.

**Edge cases and mistakes:** unhashable keys (lists), mutating keys, hash-collision worst case O(n), forgetting to update recency on `put` of an existing key.

**Practice:** Two Sum; Valid Anagram; Group Anagrams; Longest Consecutive Sequence; LRU Cache.

**Production relevance:** caches, deduplication, session stores, frequency counting in analytics. Real LRU caches in multi-threaded code need locking (see Part 5, concurrency).

**Your turn (A4).** Group Anagrams: what is the time complexity with the sorted-key approach vs the count-tuple approach?

---

### A.5 Linked Lists `[Both]` P0

**Recognize:** pointer manipulation; reverse; detect cycle; middle node; merge; remove nth from end.

```python
class ListNode:
    def __init__(self, val=0, next=None):
        self.val, self.next = val, next

def reverse_list(head):
    prev = None
    while head:
        nxt = head.next
        head.next = prev
        prev, head = head, nxt
    return prev

def has_cycle(head):                          # Floyd's tortoise and hare
    slow = fast = head
    while fast and fast.next:
        slow, fast = slow.next, fast.next.next
        if slow is fast:
            return True
    return False

def merge_two_lists(a, b):
    dummy = tail = ListNode()
    while a and b:
        if a.val <= b.val:
            tail.next, a = a, a.next
        else:
            tail.next, b = b, b.next
        tail = tail.next
    tail.next = a or b
    return dummy.next
```

**Why it works:** reversal re-points each node while keeping the rest reachable via `nxt`. Floyd: if there's a cycle the fast pointer gains one step per iteration on the slow one, so they must meet. A **dummy node** removes special cases for the head.

**Alternatives:** hash set of visited nodes (O(n) space) for cycles; recursion for reversal (O(n) stack).

**Edge cases and mistakes:** empty list, single node, losing the reference to `next` before re-pointing, off-by-one for "nth from end" (use a gap of n between two pointers).

**Practice:** Reverse Linked List; Merge Two Sorted Lists; Linked List Cycle; Remove Nth Node From End; Reorder List.

**Production relevance:** the structure behind LRU lists, free lists in allocators, and queue implementations.

**Your turn (A5).** Remove the nth node from the end in one pass.

---

### A.6 Stacks, Queues, and Monotonic Stack `[Both]` P0

**Recognize:** matching brackets; "next greater/smaller element"; histogram/stock span; expression evaluation; undo.

```python
def next_greater(nums):
    res, st = [-1] * len(nums), []            # st holds indices with decreasing values
    for i, x in enumerate(nums):
        while st and nums[st[-1]] < x:
            res[st.pop()] = x                 # x is the next greater for that index
        st.append(i)
    return res

def valid_parentheses(s):
    pairs, st = {")": "(", "]": "[", "}": "{"}, []
    for ch in s:
        if ch in pairs:
            if not st or st.pop() != pairs[ch]:
                return False
        else:
            st.append(ch)
    return not st
```

**Why it works:** the stack keeps only candidates still waiting for an answer, in monotonic order. Each index is pushed and popped once, so O(n) despite the nested `while`.

**Alternatives:** brute force O(n^2); segment tree or sparse table for range-query variants.

**Edge cases and mistakes:** equal elements (strict vs non-strict comparison), storing values instead of indices when you need distances, circular arrays (iterate 2n).

**Practice:** Valid Parentheses; Min Stack; Daily Temperatures; Next Greater Element II; Largest Rectangle in Histogram.

**Production relevance:** parsers and expression evaluators, undo stacks, stack traces, BFS queues (work queues).

**Your turn (A6).** Outline the monotonic-stack solution for Largest Rectangle in Histogram.

---

### A.7 Binary Search (including Binary Search on the Answer) `[Both]` P0

**Recognize:** sorted or rotated sorted data; "minimum X such that condition holds" where the condition is monotonic (false...false true...true).

```python
from bisect import bisect_left

def lower_bound(a, x):                        # first index with a[i] >= x
    lo, hi = 0, len(a)
    while lo < hi:
        mid = (lo + hi) // 2
        if a[mid] < x:
            lo = mid + 1
        else:
            hi = mid
    return lo                                 # same as bisect_left(a, x)

def min_eating_speed(piles, h):               # binary search on the answer
    lo, hi = 1, max(piles)
    while lo < hi:
        mid = (lo + hi) // 2
        hours = sum((p + mid - 1) // mid for p in piles)
        if hours <= h:
            hi = mid                          # feasible: try smaller
        else:
            lo = mid + 1                      # infeasible: need bigger
    return lo
```

**Why it works:** if the feasibility predicate is monotonic, the boundary can be found by repeatedly halving the search space: O(log range x cost of check).

**Alternatives:** linear scan O(n); interpolation search for uniform data; ternary search for unimodal functions.

**Edge cases and mistakes:** infinite loops from `mid` rounding (use `lo < hi` with `hi = mid`, or `mid = lo + (hi - lo) // 2`); mixing inclusive/exclusive bounds; searching when the predicate isn't monotonic.

**Practice:** Binary Search; Search Insert Position; Find Minimum in Rotated Sorted Array; Koko Eating Bananas; Split Array Largest Sum.

**Production relevance:** index lookups, `git bisect`, capacity planning ("smallest instance count that meets p99 under load").

**Your turn (A7).** Explain the invariant for "Search in Rotated Sorted Array".

---

### A.8 Sorting and Intervals `[Both]` P0

**Recognize:** overlapping ranges; scheduling; "merge/insert intervals"; "minimum rooms/platforms".

```python
import heapq

def merge_intervals(intervals):
    if not intervals:
        return []
    intervals.sort(key=lambda x: x[0])
    out = [intervals[0][:]]
    for s, e in intervals[1:]:
        if s <= out[-1][1]:
            out[-1][1] = max(out[-1][1], e)   # overlap: extend
        else:
            out.append([s, e])
    return out

def min_meeting_rooms(intervals):
    intervals.sort(key=lambda x: x[0])
    ends = []                                 # min-heap of end times of rooms in use
    for s, e in intervals:
        if ends and ends[0] <= s:
            heapq.heapreplace(ends, e)        # reuse the earliest-free room
        else:
            heapq.heappush(ends, e)
    return len(ends)
```

**Why it works:** after sorting by start, only the *latest* merged interval can overlap the next one. For rooms, the heap root is the room that frees up first; if even that one is still busy, a new room is needed.

**Alternatives:** sweep line over +1/-1 events (sort start/end events, track running count) O(n log n); counting array if the time range is small.

**Edge cases and mistakes:** touching intervals (`[1,2],[2,3]`: merge or not? clarify); not sorting; mutating input.

**Practice:** Merge Intervals; Insert Interval; Non-overlapping Intervals; Meeting Rooms II; Employee Free Time.

**Production relevance:** calendar systems, booking and scheduling engines, merging log time ranges, resource allocation.

**Your turn (A8).** Solve Meeting Rooms II with the sweep-line approach.

---

### A.9 Trees `[Both]` P0

**Recognize:** hierarchical data; recursion on left/right; BFS by levels; BST properties; "path", "ancestor", "depth".

```python
from collections import deque

class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val, self.left, self.right = val, left, right

def level_order(root):
    if not root:
        return []
    out, q = [], deque([root])
    while q:
        level = []
        for _ in range(len(q)):
            n = q.popleft()
            level.append(n.val)
            if n.left:  q.append(n.left)
            if n.right: q.append(n.right)
        out.append(level)
    return out

def diameter(root):                           # longest path in edges
    best = 0
    def depth(n):
        nonlocal best
        if not n:
            return 0
        l, r = depth(n.left), depth(n.right)
        best = max(best, l + r)               # path through this node
        return 1 + max(l, r)                  # report height upward
    depth(root)
    return best

def lca(root, p, q):                          # lowest common ancestor (binary tree)
    if not root or root is p or root is q:
        return root
    l, r = lca(root.left, p, q), lca(root.right, p, q)
    return root if l and r else (l or r)

def is_valid_bst(root, lo=float("-inf"), hi=float("inf")):
    if not root:
        return True
    if not (lo < root.val < hi):
        return False
    return is_valid_bst(root.left, lo, root.val) and is_valid_bst(root.right, root.val, hi)
```

**Why it works:** most tree problems decompose into "solve left, solve right, combine at this node." The recursion returns one thing upward (height) while a side variable tracks the answer that spans both children (diameter).

**Alternatives:** iterative traversals with an explicit stack (avoid recursion limits); Morris traversal (O(1) space); BFS vs DFS depending on whether you need levels or paths.

**Edge cases and mistakes:** empty tree, skewed tree (recursion depth = n, which can overflow Python's default ~1000 limit); validating a BST by comparing only with the parent (must carry bounds); duplicate values policy.

**Practice:** Maximum Depth of Binary Tree; Invert Binary Tree; Level Order Traversal; Validate BST; Lowest Common Ancestor; Binary Tree Maximum Path Sum.

**Production relevance:** file systems, DOM, org charts, B-tree indexes, decision trees, parsing ASTs.

**Your turn (A9).** Why does "check left.val < root.val < right.val" fail to validate a BST? Give a counterexample.

---

### A.10 Heaps / Priority Queues `[Both]` P1

**Recognize:** top-K, K-th largest, merge K sorted streams, running median, scheduling by priority.

```python
import heapq
from collections import Counter

def top_k_frequent(nums, k):
    return [x for x, _ in Counter(nums).most_common(k)]   # uses a heap internally

def kth_largest(nums, k):
    h = nums[:k]
    heapq.heapify(h)                          # min-heap of size k
    for x in nums[k:]:
        if x > h[0]:
            heapq.heapreplace(h, x)
    return h[0]

class MedianFinder:                           # two heaps
    def __init__(self):
        self.lo = []                          # max-heap (store negatives): lower half
        self.hi = []                          # min-heap: upper half
    def add(self, x):
        heapq.heappush(self.lo, -x)
        heapq.heappush(self.hi, -heapq.heappop(self.lo))   # balance: move max of lo to hi
        if len(self.hi) > len(self.lo):
            heapq.heappush(self.lo, -heapq.heappop(self.hi))
    def median(self):
        if len(self.lo) > len(self.hi):
            return -self.lo[0]
        return (-self.lo[0] + self.hi[0]) / 2
```

**Why it works:** a heap exposes the min (or max) in O(1) and updates in O(log n). A size-K min-heap keeps the K largest seen so far: O(n log K) vs O(n log n) for sorting. Two heaps split data at the median.

**Alternatives:** sort then pick (simple, O(n log n)); **quickselect** average O(n) for K-th element; bucket sort for frequency problems; balanced BST / order-statistic tree.

**Edge cases and mistakes:** Python only has a min-heap (negate values for max); tie-breaking with tuples that contain non-comparable items; K larger than n.

**Practice:** Kth Largest Element in an Array; Top K Frequent Elements; Merge K Sorted Lists; Task Scheduler; Find Median from Data Stream.

**Production relevance:** job schedulers, top-K trending queries, merging sorted shards, rate-limited priority queues, event simulation.

**Your turn (A10).** Merge K sorted lists: complexity, and why a heap beats merging two at a time repeatedly.

---

### A.11 Graphs: BFS, DFS, Topological Sort, Union-Find, Dijkstra `[Both basics, SDE2 depth]` P0/P1

**Recognize:** grid or network; connectivity; shortest path; dependencies/ordering; "number of components"; cycles.

```python
from collections import deque, defaultdict
import heapq

def num_islands(grid):
    if not grid:
        return 0
    R, C, count = len(grid), len(grid[0]), 0
    def dfs(r, c):
        if r < 0 or c < 0 or r >= R or c >= C or grid[r][c] != "1":
            return
        grid[r][c] = "0"                      # mark visited
        for dr, dc in ((1,0),(-1,0),(0,1),(0,-1)):
            dfs(r + dr, c + dc)
    for r in range(R):
        for c in range(C):
            if grid[r][c] == "1":
                count += 1
                dfs(r, c)
    return count

def shortest_path_unweighted(graph, src, dst):    # BFS = shortest path by edge count
    q, seen = deque([(src, 0)]), {src}
    while q:
        node, d = q.popleft()
        if node == dst:
            return d
        for nb in graph[node]:
            if nb not in seen:
                seen.add(nb)
                q.append((nb, d + 1))
    return -1

def topo_sort(n, edges):                       # Kahn's algorithm; [] means cycle
    g, indeg = defaultdict(list), [0] * n
    for u, v in edges:
        g[u].append(v)
        indeg[v] += 1
    q, order = deque(i for i in range(n) if indeg[i] == 0), []
    while q:
        u = q.popleft()
        order.append(u)
        for v in g[u]:
            indeg[v] -= 1
            if indeg[v] == 0:
                q.append(v)
    return order if len(order) == n else []

class DSU:                                     # union-find with path compression + union by size
    def __init__(self, n):
        self.p, self.sz = list(range(n)), [1] * n
    def find(self, x):
        while self.p[x] != x:
            self.p[x] = self.p[self.p[x]]
            x = self.p[x]
        return x
    def union(self, a, b):
        ra, rb = self.find(a), self.find(b)
        if ra == rb:
            return False                       # already connected (cycle if this is an edge)
        if self.sz[ra] < self.sz[rb]:
            ra, rb = rb, ra
        self.p[rb] = ra
        self.sz[ra] += self.sz[rb]
        return True

def dijkstra(graph, src):                      # graph[u] = [(v, weight)], weights >= 0
    dist, pq = {src: 0}, [(0, src)]
    while pq:
        d, u = heapq.heappop(pq)
        if d > dist.get(u, float("inf")):
            continue                           # stale entry
        for v, w in graph.get(u, []):
            nd = d + w
            if nd < dist.get(v, float("inf")):
                dist[v] = nd
                heapq.heappush(pq, (nd, v))
    return dist
```

**Why it works:**
- **BFS** explores in rings of increasing distance, so the first time it reaches a node is the shortest unweighted path.
- **Topological sort:** a node with in-degree 0 has no unmet prerequisites; removing it may free others. If nodes remain, there's a cycle.
- **Union-find:** near-O(1) amortized connectivity queries (inverse Ackermann).
- **Dijkstra:** greedy: the smallest tentative distance in the heap is final *because all weights are non-negative*.

**Alternatives:** DFS for connectivity/cycles/paths; Bellman-Ford O(VE) for negative weights; Floyd-Warshall for all pairs; A* with heuristics; 0-1 BFS with a deque; Kruskal (with DSU) or Prim for minimum spanning trees.

**Complexity:** BFS/DFS O(V+E); Dijkstra O((V+E) log V); topo sort O(V+E); DSU ~O(alpha(n)) per op.

**Edge cases and mistakes:** disconnected graphs; self-loops and duplicate edges; marking visited when *popping* vs when *pushing* (push-time is safer in BFS); Dijkstra with negative edges silently gives wrong answers; Python recursion limit on large grids (use iterative DFS).

**Practice:** Number of Islands; Clone Graph; Course Schedule (I and II); Number of Connected Components; Network Delay Time; Word Ladder; Cheapest Flights Within K Stops.

**Production relevance:** dependency resolution (package managers, build systems, Airflow DAGs), routing, social graphs, network reachability, deadlock detection (cycle in wait-for graph).

**Your turn (A11).** Implement "Course Schedule" (can all courses be finished?) and state how it detects an impossible schedule.

---

### A.12 Tries `[SDE2]` P2

**Recognize:** prefix search, autocomplete, dictionary of words with shared prefixes, word-search boards.

```python
class Trie:
    def __init__(self):
        self.root = {}
    def insert(self, word):
        node = self.root
        for ch in word:
            node = node.setdefault(ch, {})
        node["$"] = True                       # end-of-word marker
    def search(self, word):
        node = self.root
        for ch in word:
            if ch not in node:
                return False
            node = node[ch]
        return "$" in node
    def starts_with(self, prefix):
        node = self.root
        for ch in prefix:
            if ch not in node:
                return False
            node = node[ch]
        return True
```

**Why it works:** words sharing a prefix share a path, so lookup costs O(L) (word length) regardless of dictionary size, and one traversal prunes many candidates.

**Alternatives:** hash set (fast exact match, no prefix queries); sorted array + binary search (prefix via range); radix/compressed trie to save memory; inverted index (Elasticsearch) for full text.

**Edge cases and mistakes:** memory blowup (each node a dict); not marking end-of-word (confusing "app" with "apple"); case/unicode handling.

**Practice:** Implement Trie; Design Add and Search Words; Replace Words; Word Search II; Maximum XOR of Two Numbers.

**Production relevance:** search autocomplete, IP routing (longest prefix match), spell checkers.

**Your turn (A12).** Why is a trie better than a hash set for Word Search II?

---

### A.13 Recursion and Backtracking `[Both]` P1

**Recognize:** "generate all", subsets, permutations, combinations, constraint puzzles (N-Queens, Sudoku), paths in a grid with choices.

```python
def subsets(nums):
    res = []
    def bt(i, path):
        if i == len(nums):
            res.append(path[:])
            return
        bt(i + 1, path)                        # skip nums[i]
        path.append(nums[i])                   # choose
        bt(i + 1, path)                        # explore
        path.pop()                             # un-choose
    bt(0, [])
    return res

def permutations(nums):
    res, used = [], [False] * len(nums)
    def bt(path):
        if len(path) == len(nums):
            res.append(path[:])
            return
        for i in range(len(nums)):
            if used[i]:
                continue
            used[i] = True
            path.append(nums[i])
            bt(path)
            path.pop()
            used[i] = False
    bt([])
    return res

def combination_sum(cands, target):            # candidates reusable, no duplicates in output
    cands.sort()
    res = []
    def bt(start, path, remain):
        if remain == 0:
            res.append(path[:])
            return
        for i in range(start, len(cands)):
            if cands[i] > remain:
                break                          # prune: sorted, so the rest are too big
            path.append(cands[i])
            bt(i, path, remain - cands[i])     # i, not i+1: reuse allowed
            path.pop()
    bt(0, [], target)
    return res
```

**Why it works:** explore a decision tree depth-first; **choose, explore, un-choose** restores state. Pruning cuts branches that cannot succeed.

**Alternatives:** DP when subproblems overlap and you need only a count/optimum (not the actual lists); iterative bitmask for subsets; constraint propagation for puzzles.

**Complexity:** subsets O(n * 2^n); permutations O(n * n!); output size alone is exponential, which is why n is tiny.

**Edge cases and mistakes:** appending `path` instead of a copy `path[:]`; duplicates (sort and skip equal siblings: `if i > start and cands[i] == cands[i-1]: continue`); forgetting to undo state.

**Practice:** Subsets; Permutations; Combination Sum; Letter Combinations of a Phone Number; N-Queens.

**Production relevance:** configuration/test-case generation, constraint solvers, scheduling search, regex engines.

**Your turn (A13).** Combination Sum II (each candidate used once, input has duplicates): how do you avoid duplicate result lists?

---

### A.14 Dynamic Programming `[Both basics, SDE2 depth]` P0/P1

**Recognize:** "count the ways", "minimum/maximum cost", "is it possible"; choices at each step; **overlapping subproblems + optimal substructure**.

**Framework (use it every time):** (1) define the state in words, (2) write the transition, (3) base cases, (4) order of computation, (5) answer location, (6) optimize space.

```python
def rob(nums):                                 # state: best loot up to house i
    prev = cur = 0
    for x in nums:
        prev, cur = cur, max(cur, prev + x)    # skip house, or take it + best up to i-2
    return cur

def coin_change(coins, amount):                # state: fewest coins for amount a
    INF = amount + 1
    dp = [0] + [INF] * amount
    for a in range(1, amount + 1):
        for c in coins:
            if c <= a:
                dp[a] = min(dp[a], dp[a - c] + 1)
    return dp[amount] if dp[amount] < INF else -1

def lcs(a, b):                                 # 2D: state dp[i][j] = LCS of a[:i], b[:j]
    m, n = len(a), len(b)
    dp = [[0] * (n + 1) for _ in range(m + 1)]
    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if a[i-1] == b[j-1]:
                dp[i][j] = dp[i-1][j-1] + 1
            else:
                dp[i][j] = max(dp[i-1][j], dp[i][j-1])
    return dp[m][n]

from bisect import bisect_left
def lis(nums):                                 # O(n log n): tails[k] = smallest tail of an inc. subseq of length k+1
    tails = []
    for x in nums:
        i = bisect_left(tails, x)
        if i == len(tails):
            tails.append(x)
        else:
            tails[i] = x
    return len(tails)

def knapsack_01(weights, values, cap):         # iterate capacity BACKWARD so each item is used once
    dp = [0] * (cap + 1)
    for w, v in zip(weights, values):
        for c in range(cap, w - 1, -1):
            dp[c] = max(dp[c], dp[c - w] + v)
    return dp[cap]
```

Edit distance recurrence: `dp[i][j] = dp[i-1][j-1]` if chars match, else `1 + min(dp[i-1][j] (delete), dp[i][j-1] (insert), dp[i-1][j-1] (replace))`.

**Why it works:** brute-force recursion recomputes the same subproblems exponentially many times. Caching (memoization) or building bottom-up (tabulation) makes each subproblem cost O(1) amortized, so total work = number of states x transition cost.

**Alternatives:** greedy (faster, but only valid when the greedy-choice property holds); BFS for shortest "number of steps"; divide and conquer when subproblems don't overlap; memoized recursion (easier to write, uses recursion stack) vs tabulation (faster, easy space optimization).

**Complexity:** coin change O(amount x coins); LCS/edit distance O(mn) time, O(min(m,n)) space with rolling rows.

**Edge cases and mistakes:** wrong base cases; wrong iteration order (0/1 knapsack backward vs unbounded forward); **counting combinations vs permutations** (loop order: coins outer = combinations, amount outer = permutations); using DP when the state space is too big.

**Practice:** Climbing Stairs; House Robber; Coin Change (and Coin Change II); Longest Increasing Subsequence; Longest Common Subsequence; Edit Distance; Partition Equal Subset Sum; Unique Paths.

**Production relevance:** diff tools (LCS), spell correction (edit distance), resource allocation, pricing/knapsack-like optimization, sequence alignment, query planners (join ordering).

**Your turn (A14).** "Coin Change II": count the number of combinations that make an amount. Write it and explain the loop order.

---

### A.15 Greedy `[Both]` P1

**Recognize:** locally optimal choices that provably lead to a global optimum; scheduling, intervals, jump/reach problems, "minimum number of X".

```python
def can_jump(nums):
    far = 0
    for i, x in enumerate(nums):
        if i > far:
            return False                       # can't even reach index i
        far = max(far, i + x)
    return True

def erase_overlap_intervals(intervals):        # minimum removals so none overlap
    intervals.sort(key=lambda x: x[1])         # sort by END time
    end, removed = float("-inf"), 0
    for s, e in intervals:
        if s >= end:
            end = e                            # keep: finishes earliest
        else:
            removed += 1
    return removed
```

**Why it works (exchange argument):** in interval scheduling, picking the job that *finishes earliest* leaves the most room; any optimal solution can be transformed to include it without getting worse. Always justify greedy with an exchange argument or a counterexample search.

**Alternatives:** DP (always correct when greedy isn't). Classic counterexample: coins {1, 3, 4}, amount 6: greedy takes 4+1+1 = 3 coins, optimal is 3+3 = 2.

**Edge cases and mistakes:** choosing the wrong sort key (sorting by start time fails for interval scheduling); assuming greedy works without proof.

**Practice:** Jump Game; Jump Game II; Gas Station; Non-overlapping Intervals; Task Scheduler; Partition Labels.

**Production relevance:** schedulers (earliest-deadline-first), compression (Huffman), network routing heuristics, cache eviction heuristics.

**Your turn (A15).** Why does sorting by *end time* work for interval scheduling but sorting by *start time* doesn't?

---

### A.16 Bit Manipulation `[Both]` P2

**Recognize:** XOR tricks, parity, power-of-two checks, subsets as bitmasks, space-efficient flags.

```python
def single_number(nums):                       # every other number appears twice
    x = 0
    for n in nums:
        x ^= n                                 # a ^ a = 0, a ^ 0 = a
    return x

def is_power_of_two(n):
    return n > 0 and (n & (n - 1)) == 0        # n & (n-1) clears the lowest set bit

def count_set_bits(n):
    c = 0
    while n:
        n &= n - 1                             # drop lowest set bit each time
        c += 1
    return c
```

**Why it works:** XOR is commutative, associative, and self-inverse, so paired numbers cancel. `n & (n-1)` removes the lowest set bit, so a power of two (exactly one set bit) becomes 0.

**Alternatives:** hash map (O(n) space), sorting, `bin(n).count("1")`.

**Edge cases and mistakes:** negative numbers (Python ints are unbounded; mask with `0xFFFFFFFF` to emulate 32-bit); operator precedence (`&` binds weaker than `==`, so parenthesize).

**Practice:** Single Number; Number of 1 Bits; Power of Two; Counting Bits; Subsets via bitmask.

**Production relevance:** feature flags, permissions bitfields, Bloom filters, hashing, network masks.

**Your turn (A16).** Why does `n & (n - 1)` clear the lowest set bit?

---

### A.17 Production Problems in Algorithms

- **Quadratic code that passes tests, fails at scale.** Symptom: p99 grows with data size, CPU pegged. Cause: nested loops, repeated list `in` checks, string concatenation in loops. Detect: profiler, input-size scaling test. Fix: hash sets/maps, sorting, indexes. Prevention: load tests with production-sized data, complexity review.
- **Recursion depth.** Symptom: `RecursionError`/stack overflow on skewed input. Fix: iterative version or explicit stack; set limits carefully.
- **Hash collision attacks.** Symptom: CPU spikes from crafted keys. Fix: randomized hashing (modern languages do this), limits on input keys.
- **Regex catastrophic backtracking (ReDoS).** Symptom: one request pins a CPU. Fix: linear-time regex engines, input limits, timeouts.
- **Integer overflow/precision.** Money in floats; 32-bit overflow in midpoint `(lo+hi)//2` in fixed-width languages. Fix: decimals/integers (cents), `lo + (hi-lo)//2`.
- **Unbounded memory.** Loading entire datasets into lists. Fix: streaming/generators, external sort, bounded heaps.

---

## Part B: SQL Practice (Section 23)

### B.0 Schema Used

```sql
departments(id, name)
employees(id, name, dept_id, manager_id, salary, hire_date)
customers(id, name, email, country, created_at)
orders(id, customer_id, amount, status, created_at)     -- status: 'PAID','REFUNDED','PENDING'
logins(user_id, login_date)
```

Dialect: **PostgreSQL**. For MySQL/SQL Server, equivalents exist (`FILTER` becomes `SUM(CASE WHEN ...)`; `DISTINCT ON` becomes `ROW_NUMBER()`).

### B.1 Core Patterns, With Solutions

**1. Second-highest distinct salary**
```sql
SELECT MAX(salary) AS second_highest
FROM employees
WHERE salary < (SELECT MAX(salary) FROM employees);   -- returns NULL if no second value
```

**2. Top 3 salaries per department (ties share a rank)**
```sql
SELECT * FROM (
  SELECT e.*, DENSE_RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rnk
  FROM employees e
) t WHERE rnk <= 3;
```
`ROW_NUMBER` = unique 1,2,3; `RANK` = 1,1,3 (gaps after ties); `DENSE_RANK` = 1,1,2 (no gaps). Pick based on whether ties should count.

**3. Employees earning more than their manager (self-join)**
```sql
SELECT e.name
FROM employees e
JOIN employees m ON e.manager_id = m.id
WHERE e.salary > m.salary;
```

**4. Departments with no employees**
```sql
SELECT d.name
FROM departments d
WHERE NOT EXISTS (SELECT 1 FROM employees e WHERE e.dept_id = d.id);
```
Prefer `NOT EXISTS` over `NOT IN`: if the subquery returns any `NULL`, `NOT IN` returns no rows at all (three-valued logic).

**5. Duplicate emails**
```sql
SELECT email, COUNT(*) FROM customers GROUP BY email HAVING COUNT(*) > 1;
```

**6. Delete duplicates, keep the lowest id**
```sql
DELETE FROM customers c
USING customers d
WHERE c.email = d.email AND c.id > d.id;
```

**7. Running total per customer**
```sql
SELECT customer_id, created_at, amount,
       SUM(amount) OVER (PARTITION BY customer_id ORDER BY created_at, id) AS running_total
FROM orders;
```

**8. Month-over-month revenue growth**
```sql
WITH m AS (
  SELECT date_trunc('month', created_at) AS month, SUM(amount) AS revenue
  FROM orders WHERE status = 'PAID' GROUP BY 1
)
SELECT month, revenue,
       ROUND(100.0 * (revenue - LAG(revenue) OVER (ORDER BY month))
             / NULLIF(LAG(revenue) OVER (ORDER BY month), 0), 2) AS growth_pct
FROM m ORDER BY month;
```

**9. Customers who never ordered**
```sql
SELECT c.* FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```

**10. Consecutive-day login streaks (gaps and islands)**
```sql
WITH d AS (SELECT DISTINCT user_id, login_date FROM logins),
g AS (
  SELECT user_id, login_date,
         login_date - (ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date))::int AS grp
  FROM d
)
SELECT user_id, MIN(login_date) AS start_date, MAX(login_date) AS end_date, COUNT(*) AS days
FROM g GROUP BY user_id, grp
HAVING COUNT(*) >= 3;
```
Idea: for consecutive dates, `date - row_number` is constant, so it labels each streak.

**11. Latest order per customer**
```sql
SELECT DISTINCT ON (customer_id) *
FROM orders
ORDER BY customer_id, created_at DESC;
```
Portable alternative: `ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at DESC) = 1`.

**12. Median salary**
```sql
SELECT PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary) FROM employees;
```

**13. Pivot: paid vs refunded totals per customer**
```sql
SELECT customer_id,
       SUM(amount) FILTER (WHERE status = 'PAID')     AS paid,
       SUM(amount) FILTER (WHERE status = 'REFUNDED') AS refunded
FROM orders GROUP BY customer_id;
```

**14. Orders above that customer's own average**
```sql
SELECT * FROM (
  SELECT o.*, AVG(amount) OVER (PARTITION BY customer_id) AS avg_amt FROM orders o
) t WHERE amount > avg_amt;
```

**15. Revenue share by country**
```sql
SELECT c.country, SUM(o.amount) AS revenue,
       ROUND(100.0 * SUM(o.amount) / SUM(SUM(o.amount)) OVER (), 2) AS pct
FROM orders o JOIN customers c ON c.id = o.customer_id
GROUP BY c.country;
```

**16. Find gaps in a sequence of ids**
```sql
SELECT id + 1 AS gap_start, next_id - 1 AS gap_end
FROM (SELECT id, LEAD(id) OVER (ORDER BY id) AS next_id FROM orders) x
WHERE next_id - id > 1;
```

**17. All reports (direct and indirect) under a manager (recursive CTE)**
```sql
WITH RECURSIVE r AS (
  SELECT id, name, manager_id, 0 AS depth FROM employees WHERE id = 7      -- the manager
  UNION ALL
  SELECT e.id, e.name, e.manager_id, r.depth + 1
  FROM employees e JOIN r ON e.manager_id = r.id
)
SELECT * FROM r WHERE id <> 7;
```
Production caution: add a depth limit or cycle guard if data can contain loops.

**18. Day-1 retention by first-login cohort**
```sql
WITH first AS (SELECT user_id, MIN(login_date) AS d0 FROM logins GROUP BY user_id)
SELECT f.d0,
       COUNT(*) AS cohort_size,
       COUNT(DISTINCT l.user_id) AS returned_day1,
       ROUND(100.0 * COUNT(DISTINCT l.user_id) / COUNT(*), 1) AS retention_pct
FROM first f
LEFT JOIN logins l ON l.user_id = f.user_id AND l.login_date = f.d0 + 1
GROUP BY f.d0 ORDER BY f.d0;
```

### B.2 Extra Exercises (try first, answers in the Answer Key)

- **E1.** Employees earning more than their department's average salary.
- **E2.** Customers who placed at least two orders on the same calendar day.
- **E3.** Orders per hour for the last 24 hours, including hours with zero orders.
- **E4.** First and last order amount for each customer, in one row each.
- **E5.** 7-day moving average of daily paid revenue.

### B.3 Indexes, Plans, and Pagination in Practice

- **Index for a query:** equality columns first, then sort, then range. For `WHERE customer_id = ? AND status = 'PAID' ORDER BY created_at DESC LIMIT 20` use `(customer_id, status, created_at DESC)`.
- **Partial index** for a hot subset: `CREATE INDEX ON orders (created_at) WHERE status = 'PENDING';`
- **Keyset pagination** instead of deep `OFFSET`:
  ```sql
  SELECT * FROM orders
  WHERE (created_at, id) < (:last_created_at, :last_id)
  ORDER BY created_at DESC, id DESC
  LIMIT 50;
  ```
- **Avoid N+1:** one query with a join or `WHERE id = ANY(:ids)` instead of one query per row.
- **Atomic update instead of read-modify-write:**
  ```sql
  UPDATE inventory SET qty = qty - 1 WHERE sku = :sku AND qty >= 1;   -- check affected rows = 1
  ```
- **Row lock for a critical section:** `SELECT ... FROM accounts WHERE id = :id FOR UPDATE;` inside a short transaction.

### B.4 MongoDB Equivalents

```javascript
// Top 5 customers by paid revenue
db.orders.aggregate([
  { $match: { status: "PAID" } },
  { $group: { _id: "$customer_id", total: { $sum: "$amount" } } },
  { $sort: { total: -1 } },
  { $limit: 5 }
])

// Join orders to customers
db.orders.aggregate([
  { $lookup: { from: "customers", localField: "customer_id", foreignField: "_id", as: "customer" } },
  { $unwind: "$customer" }
])

// Supporting index (equality, then sort/range)
db.orders.createIndex({ status: 1, customer_id: 1, created_at: -1 })
// Verify: db.orders.find({...}).explain("executionStats") -> IXSCAN, totalDocsExamined close to nReturned
```
`$match` and `$sort` early in the pipeline can use indexes; `$lookup` is relatively expensive at scale, which is one reason document models favor embedding or denormalizing hot read paths.

### B.5 Production Problems in SQL

- **Slow query after data growth:** Symptom: p99 climbs gradually. Detect: slow query log, `EXPLAIN (ANALYZE, BUFFERS)`. Fix: right index, rewrite, stats refresh.
- **Lock waits and deadlocks:** long transactions or inconsistent update order. Detect: `pg_stat_activity`, `pg_locks`. Fix: short transactions, consistent ordering, retry.
- **`SELECT *` and unbounded result sets:** memory pressure and network cost. Fix: select needed columns, always `LIMIT`/paginate.
- **Deep `OFFSET` pagination:** latency grows per page. Fix: keyset pagination.
- **Non-sargable predicates** (`WHERE DATE(created_at) = ...`, `lower(email) = ...` without a functional index): index unused. Fix: rewrite to range predicates or create an expression index.
- **Implicit type casts** (comparing varchar to int) disable index use. Fix: match types.
- **Reporting queries on the primary:** analytic scans compete with OLTP. Fix: read replica or warehouse.

---

## Part C: Review

### Summary

Learn **patterns, not problems**: two pointers and sliding window (linear scans), prefix sums and hash maps (trade memory for time), monotonic stack and heaps (maintain just the candidates that matter), binary search (including on the answer), graphs (BFS/DFS/topo/DSU/Dijkstra), backtracking (explore with undo), DP (state + transition) and greedy (provable local choice). Always say the brute force first, then the bottleneck. In SQL, window functions, CTEs, `NOT EXISTS`, and gaps-and-islands cover most interview questions, and indexes and plans cover the production side.

### Five review questions

- **R1.** Why is the sliding window O(n) even with a nested `while` loop?
- **R2.** When does Dijkstra fail, and what do you use instead?
- **R3.** How do you decide between greedy and DP?
- **R4.** Why prefer `NOT EXISTS` over `NOT IN`?
- **R5.** Why is keyset pagination faster than `OFFSET`?

### Mock question

**M2.** "Design the algorithm for an API rate limiter that allows at most N requests per user in any rolling 60 seconds. Start with the data structure, give the time and space complexity per request, then explain what changes if the service runs on 20 servers." (Hint: sliding-window log vs counter, and where the state lives.)

---

## Answer Key

**A1.** Area = min(h[l], h[r]) x (r - l). If you move the *taller* pointer inward, width shrinks and the min height can't increase (it's still bounded by the shorter line), so area can only decrease. Moving the shorter pointer is the only move that might increase the limiting height, so everything discarded is provably not better.

**A2.**
```python
def longest_k_distinct(s, k):
    if k == 0: return 0
    counts, left, best = {}, 0, 0
    for right, ch in enumerate(s):
        counts[ch] = counts.get(ch, 0) + 1
        while len(counts) > k:
            counts[s[left]] -= 1
            if counts[s[left]] == 0:
                del counts[s[left]]
            left += 1
        best = max(best, right - left + 1)
    return best
```
O(n) time (each index added/removed once), O(k) space.

**A3.** Store counts of prefix sums modulo K. If `prefix[j] % K == prefix[i] % K`, the subarray sum between is divisible by K. Initialize `{0: 1}`. Python's `%` is non-negative for positive K, which handles negatives; in languages with negative remainders, normalize with `((x % K) + K) % K`.

**A4.** Sorted-key: O(N x L log L) for N words of length L. Count-tuple key (26 counts): O(N x L). Space O(N x L) for the groups.

**A5.**
```python
def remove_nth_from_end(head, n):
    dummy = ListNode(0, head)
    fast = slow = dummy
    for _ in range(n):
        fast = fast.next
    while fast.next:
        fast, slow = fast.next, slow.next
    slow.next = slow.next.next
    return dummy.next
```
The dummy node handles removing the head. The gap of n between pointers puts `slow` just before the target when `fast` reaches the last node.

**A6.** Keep a stack of indices with increasing heights. When a bar is shorter than the stack top, pop the top: its height is the limiting height, its right boundary is the current index, and its left boundary is the new stack top + 1. Area = height x (right - left - 1). Append a sentinel 0 height at the end to flush the stack. O(n).

**A7.** At each step, at least one half of `[lo, hi]` is sorted. Check whether the target lies within the sorted half's range; if so search that half, otherwise search the other. The invariant: the target, if present, is always within `[lo, hi]`.

**A8.** Create events `(start, +1)` and `(end, -1)`, sort (process ends before starts at equal times if touching intervals shouldn't conflict), sweep with a running count; the answer is the maximum running count. O(n log n) time, O(n) space.

**A9.** Local checks miss ancestors' constraints. Counterexample: root 5, left child 3, and 3's *right* child 6. Each node satisfies its parent comparison locally (6 > 3), but 6 is in the left subtree of 5 and must be < 5. Pass `(lo, hi)` bounds down, or validate that an inorder traversal is strictly increasing.

**A10.** Heap of K list heads: each of N total elements costs O(log K), so O(N log K) time, O(K) extra space. Merging two at a time sequentially costs O(N x K) in the worst case because early elements are re-copied repeatedly; divide-and-conquer pairwise merging also reaches O(N log K), as an alternative.

**A11.**
```python
from collections import deque, defaultdict
def can_finish(num_courses, prerequisites):
    g, indeg = defaultdict(list), [0] * num_courses
    for course, pre in prerequisites:
        g[pre].append(course)
        indeg[course] += 1
    q = deque(i for i in range(num_courses) if indeg[i] == 0)
    done = 0
    while q:
        u = q.popleft(); done += 1
        for v in g[u]:
            indeg[v] -= 1
            if indeg[v] == 0:
                q.append(v)
    return done == num_courses
```
If a cycle exists, the nodes in it never reach in-degree 0, so `done < num_courses`.

**A12.** Word Search II looks for many words on a board. With a trie you DFS the board once while walking the trie, pruning immediately when the current path isn't a prefix of any word, and finding all words in one pass. With a hash set you'd have to search each word separately (or check every path string), and could not prune by prefix.

**A13.** Sort candidates; in the loop, skip a candidate equal to the previous *at the same recursion level*: `if i > start and cands[i] == cands[i-1]: continue`. Recurse with `i + 1` (each element used once). Prune when `cands[i] > remain`.

**A14.**
```python
def change(amount, coins):
    dp = [1] + [0] * amount
    for c in coins:                  # coins OUTER => each combination counted once
        for a in range(c, amount + 1):
            dp[a] += dp[a - c]
    return dp[amount]
```
If you loop `amount` outside and coins inside, you count *ordered* sequences (permutations): 1+2 and 2+1 become different. Coins outer fixes the order in which coin types are considered.

**A15.** Earliest-finish-first leaves maximum time for remaining jobs; exchange argument: swap any optimal solution's first job for the earliest-finishing one, still feasible and no worse. Sorting by start fails: a job starting first may be very long and block many short jobs, e.g., `[0,100]`, `[1,2]`, `[3,4]`.

**A16.** Subtracting 1 flips the lowest set bit to 0 and all lower zeros to 1; ANDing with the original therefore zeroes that bit and everything below it (which were already 0), leaving higher bits unchanged.

**R1.** Each element is added once (right pointer moves forward) and removed at most once (left pointer only moves forward), so total pointer movement is at most 2n.

**R2.** Dijkstra fails with **negative edge weights**, since a finalized node could later be improved. Use Bellman-Ford (O(VE), detects negative cycles) or, for DAGs, topological-order relaxation.

**R3.** If a local choice provably never hurts (exchange argument) and a counterexample search finds none, use greedy. If choices interact or the greedy fails on small cases (e.g., coins {1,3,4}), use DP. When unsure: write brute force, find overlapping subproblems, then DP.

**R4.** `NOT IN` returns no rows if the subquery yields any NULL (comparison with NULL is unknown), a silent bug; `NOT EXISTS` doesn't have that problem and usually optimizes into an anti-join.

**R5.** `OFFSET n` makes the DB generate and discard n rows each request, so cost grows with page depth. Keyset pagination seeks directly via the index to the last-seen key and reads only the next page: constant cost per page.

**M2 (outline of a strong answer).** *Single server:* sliding-window log: per user keep a deque of timestamps; on each request pop timestamps older than 60 s, reject if `len >= N`, else append. O(1) amortized time, O(N) space per user. Cheaper alternatives: fixed-window counter (O(1) space, boundary bursts up to 2N), sliding-window counter (weighted blend of current and previous window, approximate, O(1)), token bucket (smooth refill, allows bursts, O(1) state). *20 servers:* state must be shared or sharded: Redis with atomic operations (sorted set per user, or a Lua script / `INCR` + `EXPIRE` for counters), or consistent-hash users to one limiter node. Trade-offs: central store adds latency and a failure mode (decide fail-open vs fail-closed), local limiters with periodic sync are cheaper but approximate (N x servers worst case). Mention clock source (use the store's clock), hot keys, per-IP vs per-user vs per-API-key, returning `429` with `Retry-After`, and observability (rejection rate by key).

---

### Next: Part 3 of 13
Databases, caching, and storage (Sections 1 and 7).
