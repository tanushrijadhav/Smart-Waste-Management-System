# Smart Waste Management Algorithms Documentation
> **Note:** This document describes the algorithms and data structures used in the Smart Waste Management System, including their implementation, workflow and time and space complexity.It serves as a technical reference for understanding the system's logic, maintaining the codebase and extending its functionality.This document describes the algorithms and data structures used in the Smart Waste Management System. 

## Contents

1. [Overall System Flow](#overall-system-flow)
2. [Dijkstra's Algorithm](#dijkstras-algorithm)
3. [Insertion Sort](#insertion-sort)
4. [Bin Priority System](#bin-priority-system)
5. [Priority Queue](#priority-queue)
6. [Normal Queue](#normal-queue)
7. [Time Simulation](#time-simulation)

## Implementation Details

- Language: C
- GUI: GTK dashboard (`gui.c`, `gui_callbacks.c`, `gui_helpers.c`)
- Graph representation: Adjacency matrix (`AreaGraph`), with the depot as vertex 0
- Shortest-path algorithm: Dijkstra's algorithm, O(V^2)
- Area sorting algorithm: Insertion sort
- Urgent-bin queue: Sorted linked list, highest priority score first
- Normal-bin queue: FIFO linked list
- Dustbin storage: Singly linked list


---

## Overall System Flow

### Aim
To describe the end-to-end operation of the Smart Waste Management System, from bin monitoring to a truck dispatch.

### Input
- Dustbin records and fill levels.
- City areas and road distances.
- Depot location.
- Collection requirements.

### Algorithm
1. Initialize the dustbin linked list and the road network (`AreaGraph`, an adjacency matrix with the DEPOT as vertex 0).
2. Load the road connections and create the dustbin records. Each bin gets its priority score when it is created (`createBin`) and is placed in a queue by `classify`.
3. Display bin information and urgency status on the GTK dashboard.
4. When collection routing is requested, clear both queues.
5. Run Dijkstra's algorithm from the depot to get the shortest road distance to each reachable area.
6. Identify the areas that contain at least one bin.
7. Sort those areas by shortest road distance using insertion sort.
8. Visit the areas in sorted order and enqueue each bin: fill level >= 90% to the priority queue (kept sorted by score, highest first), otherwise to the normal FIFO queue.
9. Select the next target: the front of the priority queue first or the front of the normal queue if no urgent bin exists. If both are empty or the target bin is already at 0% then stop.
10. Run Dijkstra's algorithm again to get the road distance to the target's area (falls back to the bin's straight distance if the area is unreachable).
11. Collect every bin in the target's area whose fill level is above 0%. Each collected bin is set to 0% and re-enqueued in the normal queue.
12. Calculate the trip time: outward travel + return travel + loading time (30 km/h, 3 minutes per bin).
13. Save the dispatch summary and refresh the dashboard and event log.
14. Each dispatch request sends one truck to one area. Another dispatch request repeats steps 4 to 13.
15. When time simulation is triggered, bins gain fill level, are moved between queues as their urgency changes, and the queues are rebuilt ([Time Simulation](#time-simulation)).

### Output
- Updated bin information and urgency status.
- Collection queues ordered by urgency and area distance.
- Shortest road distances.
- Truck dispatch and trip-time information.
- Dashboard and event-log updates.

### Complexity
Let V be the number of areas, E the number of road connections, B the number of bins.
- Dijkstra with an adjacency matrix: O(V^2) per run (two runs per dispatch).
- Insertion sort of areas: O(V^2) worst case.
- Queue rebuild: O(V x B) for matching bins to areas, plus the priority-queue insertions.
- Full dustbin traversal: O(B).

---

## Dijkstra's Algorithm

### Aim
To find the shortest road distance from the depot to every area in the road network.

### Input
- `g_areaGraph`: `count` vertices and a weight matrix `weight[i][j]` in km (0 means no road).
- `src`: source vertex index (always 0, the `DEPOT`).
- A weight of 0 or less means there is no road between two areas.

### Data Structure
`AreaGraph` is an adjacency matrix. Areas are vertices, roads are undirected weighted edges. `graph_addEdge` writes `weight[a][b]` and `weight[b][a]`.

### Algorithm (`graph_dijkstra`)

1. Set `dist[i] = INF_DIST` and `prev[i] = -1` for every vertex and `dist[src] = 0`.
2. Repeat up to `n` times:
   1. Scan all unvisited vertices and pick `u` with the smallest `dist`.
   2. If no vertex is left or `dist[u]` is `INF_DIST`, stop.
   3. Mark `u` as visited.
   4. For every vertex `v` with a road (`weight[u][v] > 0`) that is not visited: if `dist[u] + weight[u][v] < dist[v]`, set `dist[v]` and `prev[v] = u`.
3. Return `dist[]` (shortest distances) and `prev[]` (predecessors for rebuilding the path).

### Output
- `dist[i]`: shortest road km from the depot to area `i`.
- `prev[i]`: previous vertex on that path.

### Example
On the Pune graph, most areas are reached directly from the depot. Kharadi is the exception:

| Area | Direct road | Dijkstra distance | Path |
|---|---|---|---|
| Shivajinagar | 3 | 3 | DEPOT -> Shivajinagar |
| Swargate | 4 | 4 | DEPOT -> Swargate |
| Koregaon Park | 5 | 5 | DEPOT -> Koregaon Park |
| Kharadi | 15 | **14** | DEPOT -> Koregaon Park -> Kharadi (5 + 9) |

### Where it is used
- `queueBinsByDistance`: road distance for sorting areas.
- `simulateTruckCollection`: road distance for travel time. Falls back to the area's `depotDistance` (sorting) or the bin's `distance` (dispatch) if the area is unreachable.

### Complexity
- Time: O(V^2) per run (linear scan for the minimum, then a row scan).
- Space: O(V).
- With V = 11 (depot + 10 areas) the cost is negligible.

### Notes
- No heap is used. A min-heap with an adjacency list would give O((V + E) log V), but that does not pay off at this graph size.
- A weight of 0 or less is treated as "no road".

---

## Insertion Sort

### Aim
To order the areas that contain bins by shortest road distance from the depot, so the queues are filled nearest area first.

### Input
`entries[]`, an array of `AreaEntry { int areaIdx; float roadDist; }`, one per area that has at least one bin. `roadDist` comes from Dijkstra.

### Algorithm (inside `queueBinsByDistance`)

1. Build `entries[]` from areas 1..n-1 (skipping the depot) that have at least one bin.
2. For `i` from 1 to `entryCount - 1`:
   1. `key = entries[i]`, `j = i - 1`.
   2. While `j >= 0` and `entries[j].roadDist > key.roadDist`, shift `entries[j]` right by one and decrement `j`.
   3. Place `key` at `entries[j + 1]`.
3. Visit the sorted entries and enqueue the bins of each area.

### Output
Areas in ascending order of road distance.

### Example
| Before | After |
|---|---|
| Kothrud 6, Swargate 4, Kharadi 14, Shivajinagar 3 | Shivajinagar 3, Swargate 4, Kothrud 6, Kharadi 14 |

### Complexity
- Time: O(k^2) worst case, O(k) if already sorted, where k is the number of areas with bins (at most the number of areas).
- Space: O(1) extra.

### Notes
- The comparison is strict (`>`), so areas with equal distance keep their graph order. The sort is stable.
- The sort only affects the order bins enter the normal queue. The priority queue orders itself by score.

---

## Bin Priority System

### Aim
To classify dustbins by fill level and calculate a priority score for collection planning.

### Input
- Current fill level of the bin (0-100%).
- The bin's `distance` in km. This is its straight distance from the depot (the area's depot distance, varied by up to 0.5 km when the bin is created). It is **not** the Dijkstra road distance.

### Priority Score

`priority = (fillLevel * 2) - (int)(distance * 5)`

- A fuller bin gets a higher score.
- A bin farther from the depot gets a lower score.
- `(int)` truncates the distance term toward zero.
- The score is calculated in `createBin` and recalculated in `updateFillLevel`. It is not calculated in `classify`.
- After a collection the bin is at 0%, so its score becomes `-(int)(distance * 5)`.

### Algorithm

#### Scoring (`createBin`, `updateFillLevel`)
1. Read the bin's fill level.
2. Read the bin's `distance`.
3. Calculate the priority score using the formula above and store it in the bin.

#### Queue classification (`classify`)
4. If the fill level is >= 90%, the bin is URGENT: add it to the priority queue (`priorityenqueue`).
5. Otherwise add it to the normal queue (`enqueue`).

#### Status labels (display only)
6. `displayBins` and `displaySystemStatus` label bins for the dashboard:
   - URGENT: 90% and above.
   - HIGH: 70% to 89%.
   - MEDIUM: 50% to 69%.
   - LOW: below 50%.

Only the 90% threshold decides which queue a bin enters. HIGH, MEDIUM and LOW are labels, and all three go to the normal queue.

### Classification Table

| Status | Fill Level | Queue |
|---|---|---|
| URGENT | >= 90% | Priority queue |
| HIGH | 70-89% | Normal queue |
| MEDIUM | 50-69% | Normal queue |
| LOW | < 50% | Normal queue |

### Example

| Bin | Fill | Distance (km) | Score |
|---|---|---|---|
| A | 95 | 3.0 | 190 - 15 = 175 |
| B | 92 | 3.2 | 184 - 16 = 168 |
| C | 100 | 14.0 | 200 - 70 = 130 |

A fuller but distant bin (C) ranks below nearer bins that are almost as full.

### Complexity
- Score calculation and classification: O(1) per bin.
- Insertion into the normal queue: O(1).
- Insertion into the priority queue: O(n) in the worst case, because it is a sorted linked list ([Priority Queue](#priority-queue)).
- Auxiliary space: O(1) per operation.

---

## Priority Queue

### Aim
To hold urgent bins (fill >= 90%) so the highest-scoring bin is dispatched first.

### Data Structure
A singly linked list of `priorityqueue` nodes (`priorityfront`, `priorityrear`), **sorted in descending priority score**. The front node has the highest score. It is not a heap.

Each node stores a copy of the bin's `binID`, `area`, `distance`, `fillLevel` and `priority`.

### Algorithm

#### Insert (`priorityenqueue`)
1. Create a node from the bin.
2. If the queue is empty, it becomes front and rear.
3. If `n->priority >= priorityfront->priority`, insert at the front.
4. Else if `n->priority < priorityrear->priority`, append at the rear.
5. Otherwise walk the list while `n->priority < cur->priority` and insert before the first node whose score is less than or equal to the new score.

#### Remove arbitrary bin (`deletefrompriorityqueue`)
Search by `binID`, unlink the node, fix `priorityrear` if needed, free it.

#### Pop next target (`popPriorityTarget`)
Take the front node, read the live bin from the bin list (area, distance, fill), unlink and free the node, return the `binID`.

#### Clear (`clearPriorityQueue`)
Free all nodes and reset `priorityrear`.

### Output
Urgent bins ordered from highest to lowest score.

### Example
Scores 175, 168, 130 are served in that order. A new bin scoring 168 is placed **before** the existing 168, because ties go in front.

### Complexity
- Insert: O(n) worst case (O(1) at either end).
- Pop front: O(1).
- Delete by ID: O(n).

### Notes
- Because the queue sorts itself by score, the area order from [Insertion Sort](#insertion-sort) does not change its order.
- Among equal scores the most recently inserted bin comes first (LIFO for ties).

---

## Normal Queue

### Aim
To hold non-urgent bins (fill < 90%) in first-in, first-out order.

### Data Structure
A singly linked list of `queue` nodes with `front` and `rear` pointers. Each node stores a copy of `binID`, `area`, `distance`, `fillLevel` and `priority`.

### Algorithm

#### Enqueue (`enqueue`)
1. Create a node from the bin.
2. If the queue is empty, it becomes front and rear.
3. Otherwise link it after `rear` and move `rear`.

#### Dequeue (`popNormalTarget`)
Take the front node, read the live bin from the bin list, unlink and free the node, return the `binID`.

#### Remove arbitrary bin (`deletefromqueue`)
Used when a bin's fill level changes or the bin is deleted.
Search by `binID`, unlink, fix `rear` if the last node was removed, free the node.

#### Clear (`clearQueue`)
Free all nodes and set `rear = NULL`.

### Output
Bins in the order they were enqueued. After `queueBinsByDistance`, that is nearest area first.

### Example
Areas sorted as Shivajinagar, Swargate, Kothrud. Normal bins are queued Shivajinagar bins, then Swargate bins, then Kothrud bins.

### Complexity
- Enqueue: O(1).
- Dequeue: O(1).
- Delete by ID: O(n).

### Notes
- The normal queue is used only when the priority queue is empty.
- Collected bins (0%) are re-enqueued here by `markBinCollectedAndRequeue` and are included again on the next rebuild.

---

## Time Simulation

### Aim
To simulate waste accumulation over time and keep every bin in the correct queue as its urgency changes.

### Input
- Existing dustbin records and their current fill levels.
- The simulation rule: each bin gains a random 5 to 24 percentage points (`rand() % 20 + 5`).

### Algorithm (`simulateFillLevelIncrease`)
1. The simulation is triggered from the dashboard.
2. Traverse the dustbin linked list.
3. For each bin, calculate `newLevel = fillLevel + (rand() % 20) + 5`.
4. Cap `newLevel` at 100.
5. If `newLevel > fillLevel`, call `updateFillLevel(binID, newLevel)`. Bins already at 100% are skipped.
6. `updateFillLevel` does the following:
   1. Validates that the level is 0-100 and finds the bin.
   2. Removes the bin from the normal queue and from the priority queue.
   3. Stores the new fill level and recalculates the priority score.
   4. Calls `classify`, which re-inserts the bin into the priority queue (>= 90%) or the normal queue (< 90%).
   5. Prints a warning if the bin just became urgent or a notice if it is no longer urgent.
7. After the traversal, call `queueBinsByDistance()`, which clears both queues and rebuilds them in road-distance order.
8. The GUI then refreshes the dashboard with the updated bin information.

### Automatic queue movement
Bins move between queues automatically. A bin that crosses 90% leaves the normal queue and enters the priority queue during `updateFillLevel`, and the final `queueBinsByDistance()` call rebuilds both queues from the bin list.

### Output
- Updated fill levels and priority scores.
- Queues that match the new urgency levels.
- A count of the bins updated.

### Example
Bin 4 is at 85% and gains 12 points. Its new level is 97%. It is removed from the normal queue, classified as urgent, inserted into the priority queue by score and a warning is printed.

### Complexity
Let B be the number of bins and Q the length of a queue.
- Each `updateFillLevel`: O(Q) for the queue removals, plus the insertion cost.
- Whole simulation: about O(B x Q), plus one queue rebuild (see [Overall Flow](#overall-system-flow)).
- Space: O(1) auxiliary for the updates.

### Notes
- The increase is random, so results differ between runs.
- This function only raises fill levels. Emptying bins happens during truck dispatch.
