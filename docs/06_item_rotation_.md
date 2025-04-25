# Chapter 6: Item Rotation

Welcome back! In [Chapter 5: Packing Constraints & Features](05_packing_constraints___features_.md), we learned how to add rules like stability checks (`check_stable`), item priorities (`level`), and binding items together (`binding`) to make our packing more realistic and controlled. Now, let's look at another fundamental way the packer optimizes space: **Item Rotation**.

## Why Turn Things Around?

Imagine you have a tall, thin book and a shoebox. If you try to put the book into the shoebox standing straight up, it might be too tall to fit. But if you lay the book flat, it probably fits perfectly!

This simple idea is exactly what Item Rotation is about in 3D bin packing. Sometimes, an item won't fit into a spot in its "standard" orientation (how you first defined its Width, Height, and Depth). However, if you rotate the item—like laying the book flat, or putting it on its spine—it might slide right in!

**Item Rotation** defines the different possible ways an item can be oriented when placed inside the [Bin](03_bin_representation_.md). The [Packer Engine](01_packer_engine_.md) automatically tries these different orientations to find the best fit.

## The Six Basic Rotations (for Cubes)

Think about a standard rectangular box (a 'cube' type item in our library). You can place it on any of its 6 faces. For each face on the bottom, you can usually orient it in a couple of ways (though some might result in the same dimensions). The library defines 6 standard rotations that cover all possible unique dimension combinations for a cuboid:

Let's say your item's original dimensions are Width (W), Height (H), and Depth (D). The 6 rotations correspond to these dimension orders being used as the effective width, height, and depth in the bin:

1.  **RT_WHD (Rotation Type 0):** Standard orientation. Width=W, Height=H, Depth=D.
2.  **RT_HWD (Rotation Type 1):** Width=H, Height=W, Depth=D. (Like standing the item on its 'width' side).
3.  **RT_HDW (Rotation Type 2):** Width=H, Height=D, Depth=W.
4.  **RT_DHW (Rotation Type 3):** Width=D, Height=H, Depth=W. (Like standing the item on its 'depth' side).
5.  **RT_DWH (Rotation Type 4):** Width=D, Height=W, Depth=H.
6.  **RT_WDH (Rotation Type 5):** Width=W, Height=D, Depth=H. (Like laying the item flat on its 'height' side).

**Analogy:** Think of a cereal box.
*   RT_WHD: Standing normally on the shelf (Width across, Height up, Depth back).
*   RT_WDH: Lying flat on its back (Width across, Height is now the thickness, Depth is the original height).
*   RT_HWD: Lying on its narrow side panel (Original Height is now the width across, original Width is now the height up, Depth stays the same).
...and so on for the other rotations.

The packer will try these different `rotation_type` values when checking if an item fits at a specific `pivot` point.

## Constraints on Rotation

Can *every* item be rotated in *all* 6 ways? Not always!

1.  **The `updown` Parameter:** Remember the `updown` parameter when creating an [Item](02_item_representation_.md)?
    *   `updown=True`: Allows the packer to try *all 6* rotations. This means the item's original "top" might end up facing sideways or downwards.
    *   `updown=False`: Restricts the packer to only try rotations where the item's original "bottom" face stays oriented downwards (relative to the item's original definition). This typically means only **RT_WHD** (0) and **RT_HWD** (1) are allowed, effectively just allowing rotation around the Z-axis (height axis). This is crucial for items that must remain upright, like a box marked "This Side Up".

2.  **Cylinders:** For items defined with `typeof='cylinder'`, the `updown` flag is always treated as `False`. Cylinders can typically only be rotated around their height axis, maintaining their circular base orientation relative to the bin floor.

## Using Rotation (It's Mostly Automatic!)

The good news is that you don't need to write complex code to *tell* the packer to try rotations. It does this automatically as part of the [Packing Algorithm & Placement Logic](04_packing_algorithm___placement_logic_.md).

The main way you *control* rotation is by setting the `updown` parameter when you define your items.

Let's see an example: Imagine a small bin (5x5x5) and a tall, thin item (2x4x2).

```python
from py3dbp import Packer, Bin, Item, Painter

# --- Setup ---
packer = Packer()
small_bin = Bin('SmallBin', (5, 5, 5), 100)
packer.addBin(small_bin)

# Tall, thin item: Width=2, Height=4, Depth=2
# Volume = 16. Fits easily in Bin volume (125)

# Scenario 1: Item must stay upright
item_upright = Item(
    partno='TallItem_NoFlip', 
    name='Widget', 
    typeof='cube', 
    WHD=(2, 4, 2),  # W=2, H=4, D=2
    weight=1, 
    level=1, 
    loadbear=0, 
    updown=False, # << Cannot be flipped!
    color='red'
)

# Scenario 2: Identical item, but can be flipped
item_flippable = Item(
    partno='TallItem_CanFlip', 
    name='Widget', 
    typeof='cube', 
    WHD=(2, 4, 2),  # W=2, H=4, D=2
    weight=1, 
    level=1, 
    loadbear=0, 
    updown=True,  # << Can be flipped!
    color='blue'
)

# --- Packing ---
# Try packing ONLY the upright item
# packer.addItem(item_upright) 
# packer.pack() 
# Result: item_upright will likely NOT fit because its Height (4) fits, 
# but maybe the only available pivot point requires a different orientation.
# Let's try packing the flippable one instead.

packer.addItem(item_flippable)
packer.pack(fix_point=True) # Use fix_point for realistic placement

# --- Check Results ---
print(f"--- Results for Bin: {small_bin.partno} ---")
print("Fitted Items:")
for item in small_bin.items:
    print(f"- {item.partno} at {item.position} rotation type {item.rotation_type}") # Check the RT!

print("\nUnfitted Items:")
for item in packer.unfit_items:
    print(f"- {item.partno}")

# --- Visualize ---
# painter = Painter(small_bin)
# fig = painter.plotBoxAndItems(title=small_bin.partno, alpha=0.2, write_num=True)
# fig.show()

```

**Explanation:**

*   `item_upright` has `updown=False`. The packer will only try rotations RT_WHD (2x4x2) and RT_HWD (4x2x2). If neither orientation fits at an available pivot point in the 5x5x5 bin, it will be marked as unfitted.
*   `item_flippable` has `updown=True`. The packer can try all 6 rotations: (2x4x2), (4x2x2), (4x2x2), (2x4x2), (2x2x4), (2x2x4).
*   It's very likely that one of the rotations where the height becomes 2 (like RT_DWH: 2x2x4) *will* fit into the 5x5x5 bin.
*   When you check the results for `item_flippable`, you'll likely see it *is* fitted, and its `rotation_type` will be something other than 0 or 1 (e.g., 4 or 5), indicating the packer found a working orientation by "flipping" it.

This shows how allowing rotation (`updown=True`) gives the packer more options and increases the chance of fitting items, especially in tight spaces.

## Under the Hood: How the Packer Tries Rotations

The core logic happens inside the `Bin.putItem()` method, which we discussed in [Chapter 4](04_packing_algorithm___placement_logic_.md). When `putItem` is called to check if an item can be placed at a certain `pivot` point:

1.  **Determine Allowed Rotations:** It checks the item's `updown` property (and `typeof`). If `updown` is `True` (and it's a 'cube'), it prepares to loop through all 6 `RotationType` values. If `updown` is `False` or it's a 'cylinder', it prepares to loop only through the restricted set (e.g., RT_WHD, RT_HWD).
2.  **Loop Through Rotations:** It enters a loop, trying each allowed `rotation_type` one by one.
3.  **Get Dimensions:** Inside the loop, for the current `rotation_type`, it calls `item.getDimension()` to get the effective width, height, and depth for *that specific orientation*.
4.  **Perform Checks:** It then performs all the necessary checks (boundary checks, overlap/intersection checks, weight checks, stability checks if enabled) using these rotated dimensions.
5.  **Place or Continue:**
    *   If all checks pass for the current rotation, the item's `rotation_type` is set to the successful value, its `position` is finalized, and the item is placed. `putItem` returns `True`.
    *   If the checks fail, the loop continues to the *next* allowed rotation and tries again.
6.  **Fail if No Rotation Works:** If the loop finishes without finding *any* valid rotation that passes all checks at that `pivot` point, `putItem` returns `False`.

Here's a simplified sequence diagram showing this rotation loop within `putItem`:

```mermaid
sequenceDiagram
    participant Packer
    participant Bin
    participant Item

    Packer->>Bin: putItem(Item, Pivot)
    Note over Bin: Determine allowed rotations (e.g., All 6 if updown=True)
    loop For each Allowed RotationType (RT)
        Bin->>Item: Set item.rotation_type = RT
        Bin->>Item: Get dimensions for RT (W', H', D')
        Bin->>Bin: Check Boundaries(Pivot, W', H', D')
        alt Boundaries OK
            Bin->>Bin: Check Overlaps(Item@RT, Pivot)
            alt No Overlaps
                Bin->>Bin: Check Weight, Stability etc.
                alt All Checks OK
                    Bin->>Item: Finalize position & rotation_type
                    Bin->>Bin: Add Item to bin.items
                    Bin->>Packer: Return True (Success!)
                    break # Exit loop
                else Checks Failed
                    Note over Bin: Continue to next rotation...
                end
            else Overlaps Found
                Note over Bin: Continue to next rotation...
            end
        else Boundaries Not OK
            Note over Bin: Continue to next rotation...
        end
    end # End Rotation Loop
    alt Loop Finished, No Fit Found
        Bin->>Packer: Return False (Failure)
    end

```

Let's look at the relevant code parts (simplified):

1.  **Rotation Type Constants (`constants.py`):** Defines the numerical values for each rotation type.

    ```python
    # File: py3dbp/constants.py
    class RotationType:
        RT_WHD = 0 # (W, H, D)
        RT_HWD = 1 # (H, W, D)
        RT_HDW = 2 # (H, D, W)
        RT_DHW = 3 # (D, H, W)
        RT_DWH = 4 # (D, W, H)
        RT_WDH = 5 # (W, D, H)

        # List of all possible rotations
        ALL = [RT_WHD, RT_HWD, RT_HDW, RT_DHW, RT_DWH, RT_WDH] 
        
        # List of rotations allowed if updown=False (no flipping)
        Notupdown = [RT_WHD, RT_HWD] 
    ```

2.  **Item's Dimension Getter (`main.py`):** Returns the dimensions based on the current `rotation_type`.

    ```python
    # File: py3dbp/main.py
    class Item:
        # ... (init, etc.) ...
        def getDimension(self):
            ''' Returns [width, height, depth] for the current rotation_type '''
            if self.rotation_type == RotationType.RT_WHD:
                dimension = [self.width, self.height, self.depth]
            elif self.rotation_type == RotationType.RT_HWD:
                dimension = [self.height, self.width, self.depth]
            elif self.rotation_type == RotationType.RT_HDW:
                dimension = [self.height, self.depth, self.width]
            elif self.rotation_type == RotationType.RT_DHW:
                dimension = [self.depth, self.height, self.width]
            elif self.rotation_type == RotationType.RT_DWH:
                dimension = [self.depth, self.width, self.height]
            elif self.rotation_type == RotationType.RT_WDH:
                dimension = [self.width, self.depth, self.height]
            else: # Default case or error
                dimension = [self.width, self.height, self.depth] 

            return dimension
    ```

3.  **Bin's Placement Logic (`main.py`):** Shows the loop trying rotations.

    ```python
    # File: py3dbp/main.py
    class Bin:
        # ... (init, etc.) ...
        def putItem(self, item, pivot, axis=None):
            ''' Attempts to place item at pivot, trying rotations '''
            fit = False
            original_position = item.position # Store original state
            item.position = pivot # Tentatively place at pivot

            # Determine which rotations to try based on item.updown
            rotations_to_try = RotationType.ALL if item.updown else RotationType.Notupdown

            # Loop through the allowed rotation types
            for i in range(0, len(rotations_to_try)):
                item.rotation_type = rotations_to_try[i] # Set current rotation
                dimension = item.getDimension() # Get W', H', D' for this rotation

                # === Start Checks for this rotation ===
                # 1. Check if dimensions exceed bin boundaries from pivot
                if (self.width < pivot[0] + dimension[0] or
                    self.height < pivot[1] + dimension[1] or
                    self.depth < pivot[2] + dimension[2]):
                    continue # This rotation doesn't fit spatially, try next

                # 2. Check for intersection with existing items
                does_intersect = False
                for current_item_in_bin in self.items:
                    if intersect(current_item_in_bin, item): # Collision check
                        does_intersect = True
                        break
                if does_intersect:
                    continue # This rotation collides, try next

                # 3. Check weight limit
                if self.getTotalWeight() + item.weight > self.max_weight:
                    # Note: If overweight, no rotation will help. Could potentially
                    # exit early, but simple approach continues checking rotations.
                    continue # Weight check fails for this attempt

                # 4. Apply fix_point (gravity) if enabled
                if self.fix_point:
                    # Adjust position downwards, find resting Z... (simplified)
                    # item.position = self.apply_gravity_fix(...)
                    pass # Actual logic involves checkHeight/Width/Depth

                # 5. Check stability if enabled
                if self.check_stable:
                    # Check support ratio, vertex support... (simplified)
                    # is_stable = self.verify_stability(...)
                    # if not is_stable: continue # Stability check fails, try next
                    pass # Actual stability check logic

                # === If ALL checks passed for this rotation ===
                fit = True 
                # (Finalize position, potentially copy item, add to self.items)
                # self.items.append(copy.deepcopy(item)) 
                # self.update_fit_map(...) # Update internal space tracking
                break # Found a working rotation, exit the loop!
            
            # === After the loop ===
            if not fit:
                item.position = original_position # Reset if no rotation worked
                # item.rotation_type = 0 # Optionally reset rotation too
                return False # Signal failure
            else:
                # If fit was True, item is already added conceptually
                return True # Signal success
    ```

## Conclusion

Item Rotation is a fundamental technique the `3D-bin-packing` library uses to maximize space utilization. By trying different orientations (the 6 rotation types for cubes), the packer can often find ways to fit items that wouldn't fit in their standard orientation. You control the extent of this rotation primarily through the `updown` parameter in the `Item` definition, allowing you to respect "This Side Up" constraints when needed. Understanding rotation helps you see how the packer explores different possibilities to solve the packing puzzle.

Now that we know how items are placed and rotated, how can we *see* the results? In the next chapter, we'll explore the visualization tools.

Next: [Chapter 7: Visualization (Painter)](07_visualization__painter__.md)

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)