# Draw.io Direct Routing Algorithm

Use this routing pass for every editable `.drawio` diagram that contains connectors.
The goal is direct orthogonal routing: clear straight segments, at most a small number of intentional elbows, and no connector passing through unrelated shapes or labels.

## 1. Place Shapes Before Routing

Route quality is mostly decided before edges are written.

- Put related shapes into rows or columns on a grid.
- Keep at least `120 px` between unrelated shape bounding boxes; use `160-220 px` near busy hubs.
- Keep a visible corridor between groups or bounded contexts.
- Put highly connected shapes near the center of their cluster and near the shapes they connect to.
- If a canvas has no clear corridors after spacing, split the diagram into context/detail files instead of compressing it.

### Overview Topology Mode

Use this mode for any `00_overview*.drawio` file or any diagram that summarizes multiple bounded contexts.
The overview should explain the system topology, not prove every relationship in detail.

- Partition the canvas into visible context bands or clusters before adding relationships.
- Keep each relationship local to one cluster whenever possible.
- Use only the backbone relationships needed to understand cross-context ownership and dependency.
- Omit detail-level cross-context relationships from the overview when a numbered context diagram shows them.
- When a cross-context relationship is required, add a small boundary portal/stub near the destination cluster and route to that portal instead of dragging a line through the whole canvas.
- Do not draw a connector that crosses behind another context's shapes. Reposition clusters or replace the long connector with a short note or portal.
- Treat any connector longer than about `600 px` as a design smell. Prefer moving the clusters closer, duplicating a lightweight external-reference stub, or leaving the detail to the context diagram.
- Put hub concepts in the visual center of their local cluster; do not let a hub send long spokes to every context.

## 2. Build Obstacle And Lane Data

Before writing connector XML, reason about the diagram as simple rectangles and line segments.

- Treat every vertex, card, table, diamond, note, and group boundary as an obstacle.
- Inflate each obstacle by `30 px`; inflate dense tables or text-heavy cards by `40 px`.
- Do not route through an inflated obstacle unless it is the source or target of that connector.
- Reserve horizontal lanes between rows and vertical lanes between columns.
- Track every routed segment as occupied so later connectors do not reuse the same corridor.

## 3. Choose Anchors Deterministically

Choose the side that faces the other shape before adding waypoints.

- If the target is mostly to the right or left, route from right-to-left or left-to-right.
- If the target is mostly above or below, route from bottom-to-top or top-to-bottom.
- Use side lanes in this order: `0.5`, `0.25`, `0.75`, `0.15`, `0.85`.
- Do not reuse the same `(exitX, exitY)` on one source side or the same `(entryX, entryY)` on one target side until every clearer lane is exhausted.
- For rhombus/diamond relationship nodes, use only cardinal vertices:
  - top: `exitX=0.5;exitY=0` or `entryX=0.5;entryY=0`
  - right: `exitX=1;exitY=0.5` or `entryX=1;entryY=0.5`
  - bottom: `exitX=0.5;exitY=1` or `entryX=0.5;entryY=1`
  - left: `exitX=0;exitY=0.5` or `entryX=0;entryY=0.5`

## 4. Try Route Candidates In Order

Use the simplest clear candidate. Do not add bends because the renderer will "probably sort it out".

1. **Straight segment**: use when source and target anchors align horizontally or vertically and the segment crosses no inflated obstacle.
2. **Single elbow**: use one waypoint when a clean horizontal-then-vertical or vertical-then-horizontal route fits in an empty lane.
3. **Two-elbow dogleg**: use two waypoints only when needed to avoid an obstacle or separate parallel routes.
4. **Reposition or split**: if a route needs more than two elbows, move shapes apart, move a hub to a clearer row/column, or split the diagram.

Prefer moving a shape over adding another bend. A connector that winds around the canvas is a layout failure, not a routing success.

## 5. Write Edge XML For Direct Orthogonal Lines

Use explicit orthogonal, non-curved styles:

```xml
style="edgeStyle=orthogonalEdgeStyle;rounded=0;curved=0;orthogonalLoop=1;jettySize=auto;html=1;..."
```

For a straight edge, omit `<Array as="points">`.

For a one-elbow edge, use exactly one waypoint in the clear corridor:

```xml
<mxGeometry relative="1" as="geometry">
  <Array as="points">
    <mxPoint x="520" y="180" />
  </Array>
</mxGeometry>
```

For a two-elbow dogleg, use exactly two waypoints. The two waypoints should create one shared horizontal or vertical middle corridor, not a zig-zag.

## 6. Validate Before Finishing

Perform these checks on every generated `.drawio` file:

- **Curves disabled**: connector styles use `edgeStyle=orthogonalEdgeStyle;rounded=0;curved=0`.
- **Obstacle intersection**: no horizontal or vertical segment intersects an unrelated inflated obstacle.
- **Colinear overlap**: no two segments share the same corridor for an overlapping interval unless the diagram intentionally uses a labeled bus.
- **Crossing minimization**: if two connectors cross, first try moving a shape or choosing the opposite side before adding bends.
- **Anchor uniqueness**: busy sides use separate lanes, not repeated anchors stacked on top of each other.
- **Label clearance**: connector segments do not pass through relationship labels, table text, card headers, notes, or aggregate boundaries.
- **Bend budget**: most connectors are straight; exceptions use one elbow; two elbows are rare and justified by obstacles.

If the diagram still looks tangled after these checks, split it into smaller context-first diagrams.
