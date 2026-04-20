# Draw.io RMD Patterns

Use these patterns when writing relational diagrams under `docs/technical/rmd`.

## Minimal Skeleton

```xml
<mxfile host="app.diagrams.net" modified="2026-04-20T00:00:00.000Z" agent="Codex" version="26.0.11">
  <diagram id="sample-rmd" name="Page-1">
    <mxGraphModel dx="1600" dy="900" grid="1" gridSize="10" guides="1" tooltips="1" connect="1" arrows="1" fold="1" page="1" pageScale="1" pageWidth="2600" pageHeight="1800" math="0" shadow="0">
      <root>
        <mxCell id="0" />
        <mxCell id="1" parent="0" />
      </root>
    </mxGraphModel>
  </diagram>
</mxfile>
```

## Table Box Pattern

Use one vertex with an HTML table label:

```xml
<mxCell id="users" value="&lt;table style=&quot;width:100%;border-collapse:collapse;font-size:11px;&quot;&gt;&lt;tr&gt;&lt;td colspan=&quot;4&quot; style=&quot;background:#E8A06A;color:#fff;font-weight:bold;text-align:center;border:1px solid #C97E46;padding:4px;&quot;&gt;users&lt;/td&gt;&lt;/tr&gt;&lt;tr&gt;&lt;td style=&quot;border:1px solid #D7A87A;padding:3px;font-weight:bold;&quot;&gt;Key&lt;/td&gt;&lt;td style=&quot;border:1px solid #D7A87A;padding:3px;font-weight:bold;&quot;&gt;Property Name&lt;/td&gt;&lt;td style=&quot;border:1px solid #D7A87A;padding:3px;font-weight:bold;&quot;&gt;Type&lt;/td&gt;&lt;td style=&quot;border:1px solid #D7A87A;padding:3px;font-weight:bold;&quot;&gt;Nullable&lt;/td&gt;&lt;/tr&gt;&lt;tr&gt;&lt;td style=&quot;border:1px solid #E3C3A3;padding:3px;&quot;&gt;PK&lt;/td&gt;&lt;td style=&quot;border:1px solid #E3C3A3;padding:3px;&quot;&gt;id&lt;/td&gt;&lt;td style=&quot;border:1px solid #E3C3A3;padding:3px;&quot;&gt;integer&lt;/td&gt;&lt;td style=&quot;border:1px solid #E3C3A3;padding:3px;&quot;&gt;no&lt;/td&gt;&lt;/tr&gt;&lt;/table&gt;" style="rounded=0;whiteSpace=wrap;html=1;align=left;verticalAlign=top;spacing=0;fillColor=#FFFDF8;strokeColor=#C97E46;overflow=fill;" vertex="1" parent="1">
  <mxGeometry x="120" y="120" width="300" height="150" as="geometry" />
</mxCell>
```

## Relationship Edge Style

```xml
<mxCell id="edge_users_profiles" value="" style="edgeStyle=orthogonalEdgeStyle;rounded=0;orthogonalLoop=1;jettySize=auto;html=1;startArrow=ERone;startFill=0;endArrow=ERone;endFill=0;strokeColor=#AAB7B8;exitX=1;exitY=0.3;entryX=0;entryY=0.3;" edge="1" parent="1" source="users" target="user_profiles">
  <mxGeometry relative="1" as="geometry" />
</mxCell>
```

Marker names:

- `ERone`
- `ERzeroToOne`
- `ERzeroToMany`
- `ERoneToMany`

## Routing Rules

- On each source table, every outgoing edge must have a unique `(exitX, exitY)`
- On each target table, every incoming edge must have a unique `(entryX, entryY)`
- When two edges terminate on the same side, keep their lanes separated, for example `0.25` and `0.75`
- Separate parallel corridors by at least `30 px`
- Move tables apart before adding dense extra bends
- Never let an edge segment overlap an unrelated table bounding box
