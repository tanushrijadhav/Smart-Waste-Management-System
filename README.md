# ♻️ Smart Waste Management System

> A GTK-based desktop application for real-time urban waste bin monitoring, priority-based alerting and graph-routed truck dispatch optimization.

---

## Overview

The **Smart Waste Management System** is a C/GTK+ desktop application that helps municipal administrators monitor dustbin fill levels across city areas, receive urgency alerts and dispatch collection trucks along the most efficient road routes.

The system models the city's road network as a **weighted graph** and uses **Dijkstra's shortest path algorithm** to determine the optimal truck route from the depot to each collection area, replacing a naive crow-fly distance sort with true road-network awareness.

---

## Problem Statement

Traditional waste management relies on fixed schedules and manual inspection, leading to:

- Overflowing bins and deteriorating public hygiene
- Fuel waste from suboptimal collection routes
- No real-time visibility into which bins are critical

---

## Solution

The system provides a live dashboard where administrators can:

- Monitor all bins by fill level, area and urgency status
- Receive automatic URGENT alerts for bins at or above 90% capacity
- Sort and queue bins by **shortest road distance from the depot** using Dijkstra's algorithm
- Dispatch trucks along the road-optimal route, collecting all bins in a target area in a single trip
- Simulate time passage to watch fill levels rise and urgency change dynamically

---

## Architecture

The application is split into a GTK+ frontend and a pure-C core logic layer.

**Frontend (GTK+)**
- `gui.c` - Window creation, tabs, CSS theming, analytics chart
- `gui_callbacks.c` - Button and event handlers
- `gui_helpers.c` - Tree view refresh functions

**Core Logic (`main.c`)**
- Dustbin linked list
- AreaGraph (adjacency matrix, $V \le 20$)
- Dijkstra shortest path engine & tracking arrays
- Priority queue for urgent bins (fill $\ge$ 90%)
- Normal queue for all other bins
- Truck dispatch simulation

The frontend calls into the core exclusively through the API declared in `core.h`. No core file imports any GTK header.

---

## Data Structures & Pointer Reference

| Structure / Pointer | Type | Role & Memory Reference |
|---------------------|------|-------------------------|
| `Dustbin` | Linked List Node | Master store of all bins (`id`, `area`, `fill_level`, `distance`, `priority_score`, `next` pointer) |
| `AreaGraph` | Adjacency Matrix | City road network graph storing area nodes and weighted edge distances in road km |
| `Dijkstra State` | Distance & Predecessor Arrays | `dist[]` and `parent[]` arrays tracking shortest-path distances from DEPOT and node predecessors for $O(V^2)$ routing |
| `priorityqueue` | Sorted Linked List | Urgent bins (fill $\ge$ 90%), ordered by priority score |
| `queue` | FIFO Linked List | Normal bins, enqueued in Dijkstra road-distance order |

### Why a Graph instead of a BST?

The previous version built a **Binary Search Tree** on bin distance values, traversed it in-order, copied the result into queues, then freed the BST - every single time sorting was needed. This had two problems:

1. It sorted individual bins by **straight-line distance**, not by which area the truck should visit first on the actual road network.
2. The BST was a **temporary throwaway structure** - rebuilt and discarded every call, with no lasting value.

The graph replaces both the BST and the old `AreaDistance` linked list with a single persistent `AreaGraph` paired with Dijkstra's algorithm state arrays. Dijkstra runs once per dispatch to produce a road-optimal area visit order, which then fills the queues. The result is a more realistic and more efficient routing engine.

---

## Routing Algorithm

**Step 1: Dijkstra from DEPOT**
Finds the shortest road path from the depot to every area node in the graph using distance/visited/parent lookup tracking.

**Step 2: Sort areas by road distance**
Areas that contain bins are sorted by their Dijkstra road distance (insertion sort, $V \le 20$).

**Step 3: Fill queues**
Bins are enqueued area-by-area in road-optimal order. Bins with fill $\ge$ 90% go to the priority queue; all others go to the normal queue.

**Step 4: Dispatch**
The truck pops the highest-priority target (priority queue first, then normal queue), re-runs Dijkstra to get the exact road distance, collects all bins in that area in one trip, then computes total route time as: travel out + loading + return.

---

## Pune Road Network

The system ships with an approximate Pune area road graph. Key connections:

| From | To | Road km |
|------|----|---------|
| DEPOT | Shivajinagar | 3 |
| DEPOT | Swargate | 4 |
| DEPOT | Kothrud | 6 |
| DEPOT | Camp | 6 |
| DEPOT | Koregaon Park | 5 |
| Shivajinagar | Koregaon Park | 4 |
| Shivajinagar | Swargate | 3 |
| Shivajinagar | Kothrud | 5 |
| Koregaon Park | Viman Nagar | 6 |
| Koregaon Park | Kharadi | 9 |
| Viman Nagar | Kharadi | 5 |
| Kharadi | Hadapsar | 7 |
| Hadapsar | Swargate | 8 |
| Swargate | Camp | 4 |
| Kothrud | Baner | 6 |
| Kothrud | Hinjewadi | 8 |
| Baner | Hinjewadi | 5 |

Dijkstra finds the true shortest path even when a direct depot edge is longer than a multi-hop route through intermediate areas.

---

## Tech Stack

| Category | Technology |
|----------|------------|
| Language | C (C11) |
| GUI Framework | GTK+ 3 |
| Build Tool | GCC / MinGW |
| Graph Algorithm | Dijkstra ($O(V^2)$, $V \le 20$) |
| Platform | Linux / Windows |

---

## Project Structure

| Path | Description |
|------|-------------|
| `build/smartwaste.exe` | Compiled binary |
| `include/core.h` | All structs, graph API, extern declarations |
| `include/gui.h` | GTK widget externs and callback prototypes |
| `src/main.c` | All core logic: graph, queues, simulation, entry point |
| `src/gui.c` | GTK window, tabs, CSS theming, analytics chart |
| `src/gui_callbacks.c` | Button and event handlers — calls core API |
| `src/gui_helpers.c` | Tree view refresh functions |

---

## Build & Run

### Prerequisites

- GCC or MinGW
- GTK+ 3 development libraries

**Linux:**
```bash
sudo apt install libgtk-3-dev
