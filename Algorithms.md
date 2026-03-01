# Algorithms Used in Warehouse.xlsm

This document explains the three algorithms implemented in the VBA macro (`Module1.bas`) that power the warehouse pathfinding and route optimization.

---

## 1. A* Algorithm

**VBA Functions**: `FindPath`, `ProcessNeighbor`, `ReconstructPath`, `IsWall`

Finds the shortest path to take to move from 1 point to another on the map without hitting walls.

### How It Works

```
F = G + H
```

- **G (Ground distance)**: How far we walked from start, cost = 1 per step
- **H (Heuristic)**: How far is it to the Target (Manhattan Distance = |RowDiff| + |ColDiff| = |NewRow - TargetRow| + |NewCol - TargetCol|)
- **F (Final Score)**: Lowest F score is the best candidate to look at next

### Example

![A* algorithm step-by-step example](images/A_star.png)

**Initial Situation**
A is at (2,1), B is at (2,3), 1 is a wall.

**Step 1**
A tries to move right, but blocked by wall (1).
A tries to move up: G = 1, H = |1-2| + |1-3| = 3, F = 1+3 = 4.
A tries to move down: G = 1, H = |3-2| + |1-3| = 3, F = 1+3 = 4.
A chooses to move top because it's the 1st one it found.

**Step 2**
A tries to move right: G = 2, H = |1-2| + |2-3| = 2, F = 4 >= 4.
A already visited (2,1) which is in a closed list. A cannot go back.
A chooses to move to the right.

**Step 3**
A tries to move right: G = 3, H = |1-2| + |3-3| = 1, F = 4 >= 4.
A already visited (1,1) which is in a closed list. A cannot go back.
A chooses to move to the right.

**Final Path**
A found B, the road taken is drawn.

**Backtracking**: If the road taken by the choice made at step 1 happened to be blocked by a wall or if the road happened to drift away from B which would make F increase greater than 4, the algorithm would go back to step 1 where the 2nd option had a lower F and try exploring it.

### How It's Used in the Macro

The A* algorithm is the foundation used by multiple macros:

- **`GenerateMapPath`** calls `FindPath` between each stop on the optimized route and draws the result as colored arrows on the Map_Grid sheet.
- **`CalculateBatchDistance`** calls `FindPath` to calculate total walking distance without drawing anything, used when logging transactions.
- **`Analyze_Map_Locations`** calls `FindPath` for every bin to compute round-trip distances from the dock, which determines ABC rankings.

**Grid rules**:
- Cell value `1` = wall (impassable)
- Cell value `2` = dock/start (walkable)
- Empty cells = open floor (walkable)
- Bin name cells = storage locations (walkable)

The algorithm uses 4 global `Scripting.Dictionary` objects (`OpenList`, `CameFrom`, `gScore`, `fScore`) for performance. Coordinates are passed as comma-separated strings (e.g., `"5,10"`).

### Pros and Cons

**Advantages**: Faster than checking every cell. We can add terrain cost, for example walking near this zone costs more. Easy to code.
**Disadvantages**: Has to remember every single open option in a list. If there is a lot of empty floor, wastes time checking every cell.
**Better alternative**: For bigger warehouses, use Jump Point Search.

---

## 2. Nearest Neighbor (Phase 1 of Route Optimization)

**Used in**: `GenerateMapPath`, `CalculateBatchDistance`

When a worker has multiple items to pick or put away, we need to decide what order to visit them. This is the "traveling salesperson problem" (finding the fastest route to pick all items). We solve it using a two-step strategy. Phase 1 is Nearest Neighbor.

### How It Works

Finds a quick order in which to visit locations.

**Step 1**: Calculates the distance from your starting point to every item on the list.
**Step 2**: It picks the location with the smallest distance.
**Step 3**: It repeats the scan for the next closest location.

### How It's Used in the Macro

In `GenerateMapPath`, after reading all bin coordinates from the Form, the macro runs Nearest Neighbor using Manhattan distance (`Abs(CurrX - Loc(0)) + Abs(CurrY - Loc(1))`). It builds arrays `OptSKU`, `OptX`, `OptY` in visit order. This gives a "good enough" initial route that Phase 2 then improves.

The same logic runs inside `CalculateBatchDistance` to compute distances without visualization.

### Pros and Cons

**Advantages**: Quickly sketches the rough outline (good enough path). Easy to code.
**Disadvantages**: Might grab an item close to start, but doing that puts you on the wrong side of the warehouse, forcing a long walk back later. Results in a path that crosses itself. That's why we have Phase 2.

---

## 3. 2-OPT (Phase 2 of Route Optimization)

**Used in**: `GenerateMapPath`, `CalculateBatchDistance`

Swaps order of locations to visit to see if total distance goes down. The swap is sequential (one at a time).

### How It Works

```
Initial: Start -> A -> B -> C -> D -> E -> End
Distance = 100m
```

**Step 1**: Start -> A -> C -> B -> D -> E -> End
We swap B-C. Distance = 95m. We lock in the change.

**Step 2**: Start -> A -> C -> B -> E -> D -> End
We swap D-E. Distance = 98m > 95m. We revert to step 1.

The algorithm keeps looping until no swap improves the total distance.

### How It's Used in the Macro

After Nearest Neighbor builds the initial visit order, the macro enters a `Do...Loop While Improved` loop. For every pair of stops (i, k), it calculates:

- **D1** = current cost of edges (A->B) + (C->D)
- **D2** = cost if we reversed the segment between i and k: (A->C) + (B->D)

If D2 < D1, it reverses the segment between positions i and k (swapping SKUs, coordinates, operations, and location collections in parallel). It sets `Improved = True` and restarts the loop. The loop exits only when a full pass finds no improvement.

The distance comparison uses Manhattan distance between the optimized coordinate arrays, not A* pathfinding. A* runs afterward to draw the actual wall-avoiding paths.

### Pros and Cons

**Advantages**: Solving the perfect route would take way too long mathematically. This makes it solvable within 1 second and gets you 95-99% of the way to the perfect route. Easy to code.
**Disadvantages**: Not the best route.

**Better alternative**: Lin-Kernighan (LKH). More swaps at the same time and accepts worse results in the hope of getting better ones later. Algorithm used by FedEx, UPS, etc.
**Why we don't use LKH**: 1 swap requires checking N^2 possibilities (for 50 items = 2500 checks). 2 swaps = N^4 = 6,000,000. Too slow for Excel.

---

## How They Work Together

```
Form data (list of items + bins)
        |
        v
  Nearest Neighbor
  Sorts items by closest-first from dock
        |
        v
     2-OPT
  Swaps pairs to uncross the route
        |
        v
      A*
  Finds wall-avoiding path between each stop
        |
        v
  Colored arrows drawn on Map_Grid
  Total distance = steps x 0.3848m (TILE_SCALE)
```
