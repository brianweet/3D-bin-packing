# Chapter 4: Packing Algorithm & Placement Logic

Welcome back! In the previous chapters, we learned how to describe our items using [Item Representation](02_item_representation_.md) and the containers they go into using [Bin Representation](03_bin_representation_.md). We know *what* we're packing and *where* we're packing it. But how does the [Packer Engine](01_packer_engine_.md) actually figure out the *exact* spot and orientation for each item inside the bin?

That's the magic we'll explore in this chapter: the **Packing Algorithm & Placement Logic**.

## The Warehouse Worker's Strategy

Imagine you're a warehouse worker tasked with filling a large empty container (`Bin`) with various boxes (`Item`s). You wouldn't just randomly toss items in! You'd have a strategy, a set of rules you follow to pack efficiently. The `3D-bin-packing` library's algorithm works similarly.

Think of the **Packing Algorithm & Placement Logic** as the specific step-by-step instructions the virtual warehouse worker follows:

1.  **Pick an Item:** Choose the next item to pack (often the largest one first, as decided by the `Packer`).
2.  **Find Potential Spots:** Look for promising places inside the bin where the item *might* fit. You wouldn't check every single millimeter of space, but rather corners and edges created by the bin walls or items already placed. These are like starting points for placement attempts.
3.  **Try Fitting:** At a potential spot:
    *   **Check Orientation:** Can the item fit if you turn it? (See [Item Rotation](06_item_rotation_.md)).
    *   **Check for Overlaps:** Does placing it here crash into items already in the bin? (The `intersect` check).
    *   **Check Bin Limits:** Does adding this item exceed the bin's maximum weight?
    *   **Check Stability:** Will the item be stable? Is it properly supported by what's underneath? Is it respecting gravity (`fix_point`)?
4.  **Place or Skip:**
    *   If a spot and orientation work, place the item there! Record its position and move to the next item.
    *   If it doesn't fit in that spot/orientation, try another rotation or another potential spot.
    *   If it can't fit anywhere in the current bin after trying all reasonable options, mark it as unfitted for this bin.

This careful process ensures items are placed logically, respecting physical constraints.

## Key Concepts in Placement Logic

Let's break down the core ideas the algorithm uses when deciding *how* and *where* to place an item:

### 1. Finding Potential Spots (Pivots)

Instead of checking infinite possible positions, the algorithm is smarter. It focuses on strategic points called **pivots**. Think of these as the most likely corners where a new item could start. Initially, the only pivot is the bin's origin `[0, 0, 0]`.

Once an item is placed, it creates new potential corners. For example, if you place a box, its top corners, front corners, and side corners become new candidate pivots for the *next* item.

```
Imagine a bin with one item already placed:

      +-------+
     /|      /|
    / |     / |  <-- New potential pivots created
   +-------+  |      on the top face of the item
   |  |    |  |
   |  +----|--+
   | /     | /
   |/      |/   <-- Original item placed at [0,0,0]
   +-------+
```

The algorithm iterates through these available pivot points when trying to place the next item.

### 2. Checking for Overlaps (`intersect`)

This is crucial! Before placing an item, the algorithm must ensure it doesn't occupy the same space as an item already in the bin. This is done using an `intersect` function (found in `py3dbp/auxiliary_methods.py`).

Analogy: You can't put two physical boxes in the exact same spot. The `intersect` check mathematically verifies that the 3D bounding boxes of the new item (in its potential position and rotation) and all existing items do *not* overlap.

### 3. Considering Rotations

As discussed in [Item Rotation](06_item_rotation_.md), items can often be placed in multiple orientations (like standing a book up or laying it flat). For each `pivot` point, the algorithm will try the allowed rotations for the item (based on its `updown` property and type) to see if any orientation fits.

### 4. Applying Gravity (`fix_point`)

Imagine placing an item based *only* on overlaps. It might find a valid spot floating in the middle of the bin! That's not realistic.

The `fix_point=True` option (which is often recommended) addresses this. When `fix_point` is enabled:

*   After finding a potential spot that avoids overlaps, the algorithm checks what's *directly below* the item.
*   It effectively lets the item "drop" along the Z-axis (downwards) until its bottom face rests flush against either the bin floor (Z=0) or the top face of another item already placed.
*   This ensures items don't float unrealistically.

Example (Conceptual):
*   Algorithm finds space for Item B at `[10, 10, 20]`.
*   With `fix_point=True`, it checks below `[10, 10, 20]`.
*   It finds the bin floor is at Z=0 and Item A's top is at Z=15 in that XY area.
*   It adjusts Item B's position to `[10, 10, 15]` so it rests on Item A.

```python
# In packer.pack() call:
packer.pack(
    # ... other options ...
    fix_point=True # <<< Enable gravity simulation
)
```

### 5. Stability Checks (`check_stable`, `support_surface_ratio`)

Just resting on *something* isn't always enough. An item might be placed precariously, like balancing a large box on a very small one. The `check_stable=True` option adds further realism.

When `check_stable` is enabled (it also requires `fix_point=True`):

1.  **Support Surface Ratio:** The algorithm checks how much of the item's bottom surface area is actually supported by the surfaces underneath it. You define a required ratio (e.g., `support_surface_ratio=0.75` means 75% of the bottom must be supported). If the support is less than this ratio, the placement might be rejected.
2.  **Vertex Check:** As an additional check (especially if the ratio test is borderline), it might verify if all four bottom corners (vertices) of the item have *some* support underneath them. If a corner is hanging completely in empty space, the placement is considered unstable and rejected.

Analogy: Making sure the box you're placing isn't likely to tip over because it's mostly hanging off the edge of the box below it.

```python
# In packer.pack() call:
packer.pack(
    # ... other options ...
    fix_point=True,         # Required for stability checks
    check_stable=True,      # Enable stability checks
    support_surface_ratio=0.75 # Require 75% support
)
```

### 6. Respecting Weight Limits

This is a simpler check. Before finalizing placement, the algorithm ensures that adding the current item's weight to the bin's current total weight doesn't exceed the `bin.max_weight` defined in [Bin Representation](03_bin_representation_.md).

## How it Comes Together: Inside the Packer

The main `packer.pack()` method orchestrates this. While it handles sorting items and looping through bins (as seen in [Chapter 1: Packer Engine](01_packer_engine_.md)), the core placement logic for a single item within a single bin happens in helper methods, primarily `Packer.pack2Bin` and `Bin.putItem`.

Here's a simplified view of the process flow for placing ONE item:

```mermaid
sequenceDiagram
    participant Packer
    participant Bin
    participant ItemToPlace as Item (Trying to Place)
    participant PlacedItem as Item (Already in Bin)

    Packer->>Bin: Try placing ItemToPlace in this Bin (calls pack2Bin)
    Note over Bin: pack2Bin finds potential pivot points
    Bin->>Bin: Identify pivot points (e.g., [0,0,0], corners of PlacedItem)
    loop For each Pivot Point
        Bin->>Bin: Try placing ItemToPlace at Pivot (calls putItem)
        Note over Bin: putItem(ItemToPlace, Pivot) starts checks
        loop For each allowed Rotation of ItemToPlace
            Bin->>ItemToPlace: Get dimensions for this rotation
            Bin->>Bin: Check: Fits within bin boundaries?
            Bin->>PlacedItem: Check: Intersects with PlacedItem?
            Bin->>Bin: Check: Bin weight limit okay?
            opt fix_point enabled
                Bin->>Bin: Adjust position downwards (gravity)
            end
            opt check_stable enabled
                Bin->>Bin: Check stability (surface ratio, vertices)
            end
            alt All Checks Passed
                Bin->>ItemToPlace: Update item.position & item.rotation_type
                Bin->>Bin: Add ItemToPlace to bin.items list
                Bin->>Bin: Update internal occupied space map (fit_items)
                Bin->>Packer: Report SUCCESS (Item placed!)
                Note over Packer: Move to next item
                break
            else Checks Failed
                Note over Bin: Try next rotation
            end
        end
        Note over Bin: If no rotation worked for this pivot, try next pivot
    end
    alt Item Could Not Be Placed in Any Way
        Bin->>Bin: Add ItemToPlace to bin.unfitted_items
        Bin->>Packer: Report FAILURE (Item did not fit)
    end
```

## Under the Hood: Code Glimpse

The detailed implementation is in `py3dbp/main.py`.

1.  **`Packer.pack()`:** Sets up sorting and loops through bins and items, calling `pack2Bin`.

    ```python
    # File: py3dbp/main.py (Simplified view inside Packer.pack)
    class Packer:
        # ... (init, addBin, addItem) ...
        def pack(self, ..., fix_point=True, check_stable=True, ...):
            # ... (sorting bins and items) ...

            for bin in self.bins:
                items_to_pack = self.items # Or remaining items
                for item in items_to_pack:
                    # Call the helper to try and fit this item in this bin
                    self.pack2Bin(bin, item, fix_point, check_stable, ...) 
                
                # ... (update item list if distributing) ...
            
            # ... (collect final unfitted items) ...
    ```

2.  **`Packer.pack2Bin()`:** Finds potential pivots based on existing items in the bin and calls `Bin.putItem` for each pivot.

    ```python
    # File: py3dbp/main.py (Simplified view of Packer.pack2Bin)
    class Packer:
        # ...
        def pack2Bin(self, bin, item, fix_point, check_stable, support_surface_ratio):
            # ... (Set bin options like fix_point) ...

            # Special handling for the very first item or empty bin
            if not bin.items:
                response = bin.putItem(item, START_POSITION) # Try at [0,0,0]
                if not response:
                    bin.unfitted_items.append(item)
                return

            fitted = False
            # Loop through 3 axes (W, H, D) to generate pivots
            for axis in range(0, 3): 
                items_in_bin = bin.items
                # Loop through items already in the bin
                for ib_item in items_in_bin:
                    # Calculate a pivot point based on the corner of ib_item
                    pivot = calculate_pivot(ib_item, axis) # Simplified concept

                    # Try putting the item at this pivot
                    if bin.putItem(item, pivot, axis): 
                        fitted = True
                        break # Stop trying pivots if fitted
                if fitted:
                    break # Stop trying axes if fitted
            
            if not fitted:
                bin.unfitted_items.append(item)
    ```
    *Explanation:* `pack2Bin` iterates through items already in the bin (`ib_item`). For each, it calculates potential `pivot` points (like the top-front-right corner). It then calls `bin.putItem` to see if the `item` can fit starting at that `pivot`.

3.  **`Bin.putItem()`:** This is the core logic described in the sequence diagram. It checks rotations, intersections, weight, gravity (`fix_point` logic using `checkHeight`, `checkWidth`, `checkDepth`), and stability (`check_stable` logic).

    ```python
    # File: py3dbp/main.py (Simplified structure of Bin.putItem)
    class Bin:
        # ... (init, string, getVolume, etc.) ...
        def putItem(self, item, pivot, axis=None):
            fit = False
            original_position = item.position # Remember original position
            item.position = pivot # Tentatively place at pivot

            # Determine which rotations to try (all 6 or fewer if updown=False)
            rotations_to_try = RotationType.ALL if item.updown else RotationType.Notupdown

            for rotation_index in range(len(rotations_to_try)):
                item.rotation_type = rotation_index
                dimension = item.getDimension() # W, H, D for this rotation

                # 1. Check basic boundary fit
                if not self.check_bin_boundaries(pivot, dimension):
                    continue # Skip this rotation

                # 2. Check for overlaps with existing items
                 Falsedoes_intersect = False
                for existing_item in self.items:
                    if intersect(existing_item, item): # Check collision
                        does_intersect = True
                        break
                if does_intersect:
                    continue # Skip this rotation

                # 3. Check weight constraint
                if self.getTotalWeight() + item.weight > self.max_weight:
                    # (Note: Weight check might happen earlier/later in real code)
                    # For simplicity, assume it doesn't fit if overweight
                    fit = False 
                    # We can probably stop trying rotations if weight is the issue
                    # but for now, let's just mark this attempt as fail and continue loop
                    continue 


                # 4. Apply Gravity / Fix Point (if enabled)
                final_position = list(pivot) # Start with pivot
                if self.fix_point:
                    # This part calls checkHeight, checkWidth, checkDepth
                    # to find the lowest stable resting point below the pivot
                    final_position = self.apply_gravity_fix(item, dimension, pivot)
                    item.position = final_position # Update position after gravity

                # 5. Check Stability (if enabled)
                is_stable = True
                if self.check_stable:
                     # Check surface support ratio and vertex support
                    is_stable = self.verify_stability(item, dimension, final_position)

                if is_stable: # Only if all checks pass...
                    # SUCCESS!
                    fit = True
                    # Finalize position and add item (deep copy)
                    item.position = [set2Decimal(p) for p in final_position]
                    self.items.append(copy.deepcopy(item))
                    # Update internal tracking of occupied space (self.fit_items)
                    self.update_fit_map(item, dimension) 
                    break # Found a fit, stop trying rotations

            # If loop finished without finding a fit
            if not fit:
                item.position = original_position # Reset item position
                return False # Signal failure
            else:
                return True # Signal success
        
        # ... (Helper methods like check_bin_boundaries, apply_gravity_fix, verify_stability, update_fit_map) ...
        # ... (checkHeight, checkWidth, checkDepth used by apply_gravity_fix) ...
    ```
    *Explanation:* This highly simplified `putItem` shows the sequence: try rotation, check boundaries, check intersections, check weight, apply gravity (`fix_point`), check stability (`check_stable`). If all pass, the item is added, and `True` is returned. Otherwise, it tries the next rotation or eventually returns `False`. The actual code involves detailed geometric calculations within the helper methods.

## Conclusion

The Packing Algorithm & Placement Logic is the heart of the `3D-bin-packing` library's decision-making process. It's the detailed strategy used to find a suitable spot for each item, considering its dimensions, allowed rotations, potential collisions with other items, bin weight limits, and physical constraints like gravity and stability. By understanding these steps—finding pivots, checking intersections, applying `fix_point` gravity, and verifying stability—you get a clearer picture of how the packer achieves an efficient and realistic packing solution.

In the next chapter, we'll look more closely at some of the specific constraints and features you can control, like load-bearing limits and item grouping.

Next: [Chapter 5: Packing Constraints & Features](05_packing_constraints___features_.md)

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)