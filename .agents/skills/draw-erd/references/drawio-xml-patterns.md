# Draw.io ERD Patterns

Use these patterns when writing ERD files under `docs/technical/erd`.

## Minimal Skeleton

```xml
<mxfile host="app.diagrams.net" modified="2026-04-20T00:00:00.000Z" agent="Codex" version="26.0.11">
  <diagram id="sample-erd" name="Page-1">
    <mxGraphModel dx="1600" dy="900" grid="1" gridSize="10" guides="1" tooltips="1" connect="1" arrows="1" fold="1" page="1" pageScale="1" pageWidth="2200" pageHeight="1400" math="0" shadow="0">
      <root>
        <mxCell id="0" />
        <mxCell id="1" parent="0" />
      </root>
    </mxGraphModel>
  </diagram>
</mxfile>
```

## Entity Style

```xml
<mxCell id="users" value="users" style="rounded=0;whiteSpace=wrap;html=1;fillColor=#5DADE2;strokeColor=#2E86C1;fontStyle=1;fontSize=14;" vertex="1" parent="1">
  <mxGeometry x="120" y="120" width="160" height="50" as="geometry" />
</mxCell>
```

## Relationship Style

```xml
<mxCell id="rel_has_profile" value="Has" style="rhombus;whiteSpace=wrap;html=1;fillColor=#F8C471;strokeColor=#D68910;fontSize=11;" vertex="1" parent="1">
  <mxGeometry x="165" y="240" width="70" height="40" as="geometry" />
</mxCell>
```

Use one diamond per relationship.
Keep labels short.

## Relationship Segment Pattern

Model each relationship as two segments:

1. source entity -> relationship diamond
2. relationship diamond -> target entity

Example:

```xml
<mxCell id="edge_users_has_profile" value="" style="edgeStyle=orthogonalEdgeStyle;rounded=0;orthogonalLoop=1;jettySize=auto;html=1;startArrow=ERone;startFill=0;endArrow=ERone;endFill=0;strokeColor=#AAB7B8;exitX=0.5;exitY=1;entryX=0.5;entryY=0;" edge="1" parent="1" source="users" target="rel_has_profile">
  <mxGeometry relative="1" as="geometry" />
</mxCell>
<mxCell id="edge_has_profile_user_profiles" value="" style="edgeStyle=orthogonalEdgeStyle;rounded=0;orthogonalLoop=1;jettySize=auto;html=1;startArrow=ERone;startFill=0;endArrow=ERone;endFill=0;strokeColor=#AAB7B8;exitX=0.5;exitY=1;entryX=0.5;entryY=0;" edge="1" parent="1" source="rel_has_profile" target="user_profiles">
  <mxGeometry relative="1" as="geometry" />
</mxCell>
```

Marker names:

- `ERone`
- `ERzeroToOne`
- `ERzeroToMany`
- `ERoneToMany`

## Note Style

```xml
<mxCell id="note_tenants" value="Analysis requirements still mention &quot;workspace&quot;.&lt;br&gt;Phase 1 technical persistence maps that collaboration boundary onto &lt;code&gt;tenants&lt;/code&gt;." style="shape=note;whiteSpace=wrap;html=1;fillColor=#FFF2CC;strokeColor=#D6B656;" vertex="1" parent="1">
  <mxGeometry x="1260" y="70" width="280" height="100" as="geometry" />
</mxCell>
```

## Routing Rules

- Prefer straight corridors first
- On each entity side, assign unique anchor lanes like `0.2`, `0.5`, `0.8`
- When two edges leave the same side, do not reuse the same `(exitX, exitY)` pair
- On diamonds, use only the four cardinal vertices
- Separate parallel routes into distinct lanes with at least `30 px`
- If a line would cross another line, reposition the entity or diamond before adding a second elbow
