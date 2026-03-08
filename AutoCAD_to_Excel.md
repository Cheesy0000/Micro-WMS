# AutoCAD to Excel: Creating a Scale Map

How the warehouse floor plan was drawn to scale in AutoCAD and transferred into Excel to enable accurate distance calculations during picking and putaway. Essentially a GPS map that tells you exactly where to go and what to get.

## The Original Drawing

![Warehouse floor plan](images/Drawing1-Model.png)

To access or modify the original AutoCAD drawing, open `Drawing1.dwg` from the [AutoCAD/](AutoCAD/) folder.

---

## Export Settings

![AutoCAD export settings](images/AutoCAD_settings.png)

### Paper Size

The goal was to export the drawing as a high-resolution image without stretching or squishing it. If the image is distorted, the walls and racks in the Excel grid won't line up with their real positions, and the distance calculations will be wrong. The export resolution (paper size in AutoCAD) needs to have the same aspect ratio as the drawing itself to avoid this.

The drawing is 620" wide x 500" tall, which gives a ratio of 620/500 = **1.24**.

The first attempt used a built-in preset (Sun Hi-Res: 1600 x 1280 px), but its ratio is 1600/1280 = **1.25**, slightly wider than the drawing. This would stretch the image horizontally, throwing off the scale.

The fix was to create a custom paper size of **1587W x 1280H**, because 1587/1280 = **1.24**, an exact match.

To create the custom paper size: Properties -> Custom Paper Sizes -> Add

### Plot Selection

What to plot? -> **Window** -> Click "Window" -> Select the entire drawing

### Lineweight Fix

When zooming out or on computers with lower-end graphics, the lines would become pixelated and some would disappear entirely. The fix was to increase the lineweight 5x (to 0.5mm):

Plot style table -> select `monochrome.ctb` -> click the printer icon next to it -> modify the lineweight

---

## Mapping the Drawing to Excel

### Making Cells Square

Before inserting the image, the Excel cells had to be converted to perfect squares: **15 height x 2.18 width**.

Why square? The A* pathfinding algorithm moves one cell at a time (up, down, left, right). If cells were rectangular, moving one cell sideways would cover a different real-world distance than moving one cell up or down. Square cells mean one step = the same distance in every direction, which keeps the distance calculations accurate.

Smaller cells would mean higher precision (more cells = more detail in the map), but A* has to check more cells to find a path, which slows it down significantly. For a larger or more detailed warehouse, you'd want to switch to a faster algorithm like **Jump Point Search** which skips over open floor instead of checking every cell one by one. The current 41x33 grid is a good balance between precision and speed for this warehouse.

### Grid Dimensions

To preserve the 1.24 aspect ratio in Excel, the grid was set to **41 columns (A to AO) x 33 rows**.

| Calculation | Value |
|-------------|-------|
| 500H / 33 rows | 15.15" per row |
| 620W / 15.15 | 40.92 columns -> rounded to **41 (column AO)** |
| 41 columns / 33 rows | **1.24 ratio** (matches the drawing) |

*33 rows was chosen to fit the screen well while keeping high enough precision (lots of cells).*

**Scaling to a bigger warehouse**: If your drawing is larger, you'd increase the number of rows and columns to keep the same level of detail. The grid range (A1:AO33) is referenced in the VBA code and in the Map_Helper formulas, so you'd need to update those references to match the new range. You'd also need to recalculate `TILE_SCALE` since each cell would represent a different real-world distance.

### Real-World Scale

| Metric | Value |
|--------|-------|
| Width per cell | 620" / 41 = 15.12" |
| Height per cell | 500" / 33 = 15.15" |
| Average | **15.14" = 0.3848m** |

Since the cells are perfectly square, moving one cell in any direction (up, down, left, right) equals walking **0.3848 meters** in the real warehouse. This value is stored as `TILE_SCALE` in the VBA macro and is used to convert grid steps into real distances.

### Tracing the Layout onto the Grid

![Grid with transparent AutoCAD image overlay](images/Grid_Tracing.png)

The exported image is placed over the grid and made **transparent** so you can see both the drawing and the cells at the same time. Then you manually fill in the cells by looking at what's underneath:

1. Insert the exported image (Insert -> Pictures), resize it to fit the grid, and set it to **transparent** so the cells show through
2. Cells that cover a wall or obstacle get a `1`
3. The cell where the dock/entry point is gets a `2` (this is where the worker starts and where pathfinding begins)
4. Cells where bin locations are get the bin name (e.g., `A1-R1-L1-A`)
5. Everything else stays empty (open floor the worker can walk through)

Once the grid is filled in, run `Analyze_Map_Locations`. This calculates the walking distance from the dock to every bin, assigns ABC ranks (A = closest, B = medium, C = farthest), and **color-codes the bin cells** on the map (green = A, yellow = B, red = C). It also populates the Map_Helper table with all bin coordinates, distances, and ranks.

The image is just a visual guide. The pathfinding algorithm only reads the cell values, not the image. This also makes the layout **very easy to modify**: you can rename bins, move locations to different cells, add or remove walls, or rearrange the entire layout just by editing cell values. After any change, re-run `Analyze_Map_Locations` to recalculate distances and ranks.

---

## Key AutoCAD Technique: m2p (Midpoint Between Two Points)

The most important function for making the drawing accurate was `m2p`. It finds the true midpoint between any two points, which is essential for centering objects precisely.

**How it works:**
1. Start moving an object (e.g., a rack)
2. Instead of clicking a random anchor point, type `m2p`
3. Select the two furthest points on opposite edges of the object. This snaps to its true center
4. For the destination, type `m2p` again and select the two opposite edges of the target area
5. The object drops perfectly into the center of the target

Without `m2p`, you'd have to manually eyeball the center point, which is slower and less accurate. This was critical for ensuring all racks and aisles were positioned correctly so that the distance calculations in VBA would be reliable.
