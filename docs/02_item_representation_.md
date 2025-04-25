# Chapter 2: Item Representation

Welcome back! In [Chapter 1: Packer Engine](01_packer_engine_.md), we learned about the `Packer`, the brain of our 3D packing operation. We saw that the `Packer` needs two main ingredients: the bins (boxes) and the items to pack. Before we can ask the `Packer` to do its job, we need a way to accurately describe *what* we are packing. That's where **Item Representation** comes in.

Imagine you're telling a friend how to pack a specific box. You wouldn't just say "pack this thing." You'd describe it: "It's a book, about this big (width, height, depth), it's kind of heavy, and make sure you don't put it upside down."

In `3D-bin-packing`, we need to give the computer the same kind of detailed description for every object we want to pack. This description is captured in an `Item` object.

## What is an Item?

An `Item` object represents a single, individual thing that needs to be placed into a [Bin](03_bin_representation_.md). Think of it as a digital blueprint for one of your packages. This blueprint holds all the important details the [Packer Engine](01_packer_engine_.md) needs to know about that specific package:

*   **Size:** How much space does it take up? (Width, Height, Depth)
*   **Weight:** How heavy is it? (Needed to check if the bin's weight limit is exceeded)
*   **Shape:** Is it a regular box (cube) or something like a can (cylinder)?
*   **Handling Rules:** Can it be flipped upside down? Can other heavy items be placed on top of it?

By creating an `Item` object for each thing you want to pack, you provide the `Packer` with the necessary information to figure out how and where it might fit.

## Creating an Item: The Blueprint

Let's create our first item blueprint using the `Item` class. Suppose we want to pack a small cardboard box containing books.

```python
# Make sure you import the Item class
from py3dbp import Item

# Create an Item object
# Item('part_number', 'name', 'type', (Width, Height, Depth), weight, 
#      packing_priority, load_bearing_capacity, allow_upside_down, 'color')

book_box = Item(
    partno='BB001',        # A unique ID for this specific item
    name='BookBox',        # A general name for this type of item
    typeof='cube',         # The basic shape ('cube' or 'cylinder')
    WHD=(12, 8, 5),        # Dimensions: (Width, Height, Depth) in your chosen units (e.g., inches, cm)
    weight=15,             # Weight in your chosen units (e.g., lbs, kg)
    level=1,               # Packing priority (lower number means pack earlier)
    loadbear=30,           # Max weight this item can support ON TOP of it
    updown=True,           # Can this item be placed upside down? (True/False)
    color='blue'           # Color for visualization (optional)
)

# Now you can print the item to see its details
print(book_box.string()) 
```

**Explanation of Parameters:**

*   `partno` (string): A unique identifier for *this specific instance* of an item. If you have 5 identical book boxes, each might have a unique `partno` like 'BB001', 'BB002', etc. (Important if you need to track individual physical items).
*   `name` (string): A general name or category for the item, like 'BookBox', 'Mug', 'Laptop'. Multiple items can share the same name.
*   `typeof` (string): The basic geometric shape. Currently supports `'cube'` (for rectangular prisms) and `'cylinder'`.
*   `WHD` (tuple of numbers): The dimensions **(Width, Height, Depth)**. This is crucial! Always provide them in this specific order. These dimensions are based on the item's "standard" orientation before any rotation.
    *   *Width:* Side-to-side dimension.
    *   *Height:* Bottom-to-top dimension.
    *   *Depth:* Front-to-back dimension.
*   `weight` (number): The item's weight. The units should be consistent with the `max_weight` you define for your [Bin](03_bin_representation_.md).
*   `level` (integer): Packing priority. Items with a lower `level` number (e.g., 0 or 1) are generally prioritized and packed earlier by the [Packer Engine](01_packer_engine_.md) (if sorting options are used). This is useful if some items *must* be packed.
*   `loadbear` (number): The maximum weight this item can withstand being placed *on top* of it. A fragile item might have a `loadbear` of 0, while a sturdy box might support 50 lbs. This helps ensure stability.
*   `updown` (boolean): `True` means the item can be rotated so its original top faces down (and other related rotations). `False` means it must keep its original up/down orientation. For cylinders, this is always `False`. This is a constraint related to [Item Rotation](06_item_rotation_.md).
*   `color` (string): A color name (like 'red', 'blue') or hex code ('#FF0000') used when drawing the item in the [Visualization (Painter)](07_visualization__painter__.md).

Running the `print(book_box.string())` line would output something like:

```
BB001(12x8x5, weight: 15) pos([0, 0, 0]) rt(0) vol(480)
```

This confirms the item's properties. `pos` is its current position (initially `[0,0,0]`), `rt` is its rotation type (initially `0` for standard WHD orientation), and `vol` is its calculated volume.

## Using Items with the Packer

Once you have defined your items, you add them to the `Packer` just like we saw in Chapter 1.

```python
# (Assuming packer and box are already created as in Chapter 1)
from py3dbp import Packer, Bin, Item

# --- Recreate Packer and Bin ---
packer = Packer()
box = Bin('MyBox', (20, 20, 10), 100) 
packer.addBin(box)

# --- Define Items ---
book_box1 = Item('BB001', 'BookBox', 'cube', (12, 8, 5), 15, 1, 30, True, 'blue')
book_box2 = Item('BB002', 'BookBox', 'cube', (12, 8, 5), 15, 1, 30, True, 'green')
fragile_vase = Item('FV001', 'Vase', 'cylinder', (6, 6, 10), 5, 1, 0, False, 'red') # Cylinder uses WHD for bounding box

# --- Add Items to Packer ---
packer.addItem(book_box1)
packer.addItem(book_box2)
packer.addItem(fragile_vase)

# --- Pack! ---
packer.pack(
    bigger_first=True, 
    distribute_items=False,
    check_stable=True # Let's consider stability based on loadbear
) 

# --- Check results (Simplified) ---
print(f"Items in {box.partno}:")
for item in box.items:
    print(f"- {item.partno} ({item.name}) at {item.position} rotation {item.rotation_type}")

print("\nUnfitted Items:")
for item in packer.unfit_items:
    print(f"- {item.partno} ({item.name})")
```

In this example:
1.  We create two identical `BookBox` items and one `fragile_vase`.
2.  Notice the vase is a `'cylinder'`. Its `WHD` (6, 6, 10) likely represents its diameter (6) used for both width and height of its bounding box, and its actual height (10) as depth in this initial orientation. The packing logic handles cylinders differently.
3.  The vase has `loadbear=0` (fragile, nothing heavy on top) and `updown=False` (can't be flipped).
4.  We add all items to the `packer`.
5.  When `packer.pack()` runs, it uses all the properties we defined for each `Item` (dimensions, weight, type, `loadbear`, `updown`, `level`) to find the best fit, respecting the constraints.

## Under the Hood: The `Item` Class

The `Item` class itself is relatively straightforward. It's mostly a container for the properties you provide. It's defined in the `py3dbp/main.py` file.

Here's a simplified look at its `__init__` method (the function called when you create an `Item` object):

```python
# File: py3dbp/main.py (Simplified view)

# Default position is the origin before packing
START_POSITION = [0, 0, 0] 
# Default number formatting
DEFAULT_NUMBER_OF_DECIMALS = 0 

class Item:
    # This function runs when you create an Item: Item(...)
    def __init__(self, partno, name, typeof, WHD, weight, level, loadbear, updown, color):
        ''' Store all the properties passed in '''
        self.partno = partno          # Unique ID
        self.name = name              # General name
        self.typeof = typeof          # 'cube' or 'cylinder'
        self.width = WHD[0]           # Get Width from the WHD tuple
        self.height = WHD[1]          # Get Height from the WHD tuple
        self.depth = WHD[2]           # Get Depth from the WHD tuple
        self.weight = weight          # Item weight
        self.level = level            # Packing priority level
        self.loadbear = loadbear      # Weight it can support on top
        # Only cubes can potentially be upside down
        self.updown = updown if typeof == 'cube' else False 
        self.color = color            # Visualization color
        
        # Internal state, changes during packing
        self.rotation_type = 0        # Initial rotation (0 = WHD)
        self.position = START_POSITION # Initial position (origin)
        self.number_of_decimals = DEFAULT_NUMBER_OF_DECIMALS # Formatting setting

    def getDimension(self):
        ''' Calculates current W, H, D based on rotation_type '''
        # (Logic depends on self.rotation_type)
        # Example: if self.rotation_type is 1 (HWD)
        # it would return [self.height, self.width, self.depth]
        # ... (implementation details for all 6 rotations) ...
        
        # For now, just shows the initial state:
        if self.rotation_type == 0: # RotationType.RT_WHD
            dimension = [self.width, self.height, self.depth]
        # ... other rotation types ...
        else:
             dimension = [self.width, self.height, self.depth] # Default if unknown

        return dimension

    def string(self):
        ''' Returns a readable string representation of the item '''
        return "%s(%sx%sx%s, weight: %s) pos(%s) rt(%s) vol(%s)" % (
            self.partno, self.width, self.height, self.depth, self.weight,
            self.position, self.rotation_type, self.getVolume()
        )

    def getVolume(self):
        ''' Calculates the item's volume '''
        # Note: For cylinders, this calculates the volume of the bounding box
        volume = self.width * self.height * self.depth
        # (Could add formatting here using self.number_of_decimals)
        return volume
        
    # ... other helper methods ...

```

**Key Takeaways:**

*   The `__init__` method simply takes the arguments you provide and stores them as attributes (like `self.width`, `self.weight`, etc.) inside the `Item` object.
*   It sets initial `rotation_type` to 0 (standard WHD orientation) and `position` to `[0, 0, 0]`. These will be updated by the [Packer Engine](01_packer_engine_.md) if the item is successfully packed.
*   Methods like `getDimension()` and `getVolume()` help the packer access the item's current size (considering rotation) and its volume during calculations.

## Conclusion

The `Item` class is your tool for describing the individual objects you need to pack. By carefully defining each item's dimensions, weight, type, and constraints (like `level`, `loadbear`, `updown`), you give the [Packer Engine](01_packer_engine_.md) the essential information it needs to solve the packing puzzle efficiently and correctly.

Now that we know how to describe the items, we need to describe the containers they'll go into. In the next chapter, we'll explore how to define the boxes or bins.

Next: [Chapter 3: Bin Representation](03_bin_representation_.md)

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)