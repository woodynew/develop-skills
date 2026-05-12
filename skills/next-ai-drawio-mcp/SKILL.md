---
name: next-ai-drawio-mcp
description: Use when the user has a drawing or diagramming intent and wants an AI agent to create, preview, edit, inspect, or export draw.io / diagrams.net diagrams through the Next AI Draw.io MCP server from github.com/DayuanJiang/next-ai-draw-io or the @next-ai-drawio/mcp-server package. Trigger for requests such as drawing an architecture diagram, flowchart, sequence diagram, system design diagram, network topology, cloud architecture, ER diagram, mind map, process diagram, data pipeline, deployment diagram, class/component diagram, or turning natural language into a visual diagram. Also applies to configuring the drawio MCP server, starting diagram sessions, generating mxGraphModel XML, preserving browser edits, using ID-based edit operations, exporting .drawio/.svg/.png files, or troubleshooting "No active session", stale browser preview, port, XML validation, or draw.io embed issues.
---

# Next AI Draw.io MCP

Use this skill when a user wants a diagram, not only when they mention MCP. The server provides a browser-backed draw.io preview and these core tools: `start_session`, `create_new_diagram`, `get_diagram`, `edit_diagram`, and `export_diagram`.

## Intent Routing

Treat these as diagramming requests and prefer this MCP when it is available:

- Architecture diagrams: system architecture, cloud architecture, deployment architecture, microservice architecture.
- Flow diagrams: flowcharts, business process diagrams, approval flows, user journeys, state flows.
- Technical diagrams: sequence diagrams, component diagrams, class diagrams, ER diagrams, data pipelines, network topology, Kubernetes or infrastructure topology.
- Planning diagrams: mind maps, dependency maps, roadmap/process visuals, decision trees.
- Conversion requests: turn this text/spec/code/README into a diagram, visualize this process, make this easier to understand as a diagram.

If the user asks for a diagram and has not specified a format, create a draw.io diagram with a clear layout and export only when requested. If the user explicitly asks for Mermaid, PlantUML, ASCII art, an image file, or another format, follow that requested format instead of forcing draw.io.

## Setup

Configure the MCP client with the published package:

```json
{
  "mcpServers": {
    "drawio": {
      "command": "npx",
      "args": ["@next-ai-drawio/mcp-server@latest"]
    }
  }
}
```

Optional environment:

- `PORT`: preferred embedded HTTP server port. The server defaults to `6002` and tries the next ports up to `6020` if occupied.
- `DRAWIO_BASE_URL`: draw.io embed base URL. Defaults to `https://embed.diagrams.net`; set this to a private draw.io deployment when the environment requires it.

Restart the MCP client after changing config. The package requires Node.js 18 or newer.

## Workflow

1. Start a preview session.
   - Call `start_session` before diagram tools that need active state.
   - Keep the browser tab opened by the server; it contains a `?mcp=<session-id>` URL that syncs the preview.

2. Create a new diagram only for a fresh canvas.
   - Use `create_new_diagram` when there is no current diagram or the user explicitly wants to start over.
   - Pass complete `mxGraphModel` XML with root cells `0` and `1`.
   - Do not use `create_new_diagram` for normal edits; it replaces the whole diagram and can discard manual browser changes.

3. Read before changing existing diagrams.
   - Call `get_diagram` before `edit_diagram` whenever updating or deleting existing cells.
   - Treat the returned XML as the source of truth because it includes manual edits made in the browser.
   - The server rejects `edit_diagram` when `get_diagram` was not called recently; perform edits soon after reading the diagram.

4. Edit with ID-based operations.
   - Use `edit_diagram` for `add`, `update`, and `delete`.
   - For `add` and `update`, provide a complete `mxCell` in `new_xml`, including `mxGeometry`.
   - Ensure `cell_id` exactly matches the `id` attribute inside `new_xml`.
   - Use unique IDs for added cells. Never use `0` or `1` for normal content.
   - Delete by `cell_id`; the server protects root cells and cascades dependent cells/edges.

5. Export when the diagram is ready.
   - Use `export_diagram` for `.drawio`, `.svg`, or `.png`.
   - For `.png` and `.svg`, keep the browser tab open so the iframe can produce export data.

## XML Rules

- Generate well-formed draw.io XML with escaped attribute text: `&amp;`, `&lt;`, `&gt;`, `&quot;`, and `&apos;` when needed.
- Keep cells as siblings under `<root>`; do not nest ordinary `mxCell` elements inside other `mxCell` elements.
- Avoid duplicate IDs and duplicate structural attributes such as `edge`, `parent`, `source`, `target`, `vertex`, and `connectable`.
- Use `parent="1"` for top-level shapes.
- Use explicit edge routing points in styles such as `exitX`, `exitY`, `entryX`, and `entryY` to reduce overlapping connectors.
- Keep new diagrams compact by default: start near `x=40`, `y=40`, keep related nodes close, and stay roughly within `x=0-800`, `y=0-600` unless the user asks for a larger canvas.

Minimal new diagram:

```xml
<mxGraphModel>
  <root>
    <mxCell id="0"/>
    <mxCell id="1" parent="0"/>
    <mxCell id="2" value="Start" style="rounded=1;whiteSpace=wrap;html=1;fillColor=#dae8fc;strokeColor=#6c8ebf;" vertex="1" parent="1">
      <mxGeometry x="40" y="40" width="120" height="60" as="geometry"/>
    </mxCell>
  </root>
</mxGraphModel>
```

Example add operation:

```json
{
  "operations": [
    {
      "operation": "add",
      "cell_id": "step-2",
      "new_xml": "<mxCell id=\"step-2\" value=\"Review\" style=\"rounded=1;whiteSpace=wrap;html=1;fillColor=#d5e8d4;strokeColor=#82b366;\" vertex=\"1\" parent=\"1\"><mxGeometry x=\"220\" y=\"40\" width=\"120\" height=\"60\" as=\"geometry\"/></mxCell>"
    }
  ]
}
```

## Tool Choice

- `start_session`: use once per active diagram session, or again after "No active session".
- `create_new_diagram`: use for a fresh diagram or explicit reset.
- `get_diagram`: use before targeted edits, debugging XML, or confirming current state.
- `edit_diagram`: use for preserving and modifying the current diagram.
- `export_diagram`: use to save a finished diagram to disk.

## Troubleshooting

- "No active session": call `start_session`, then retry the operation.
- Browser not updating: confirm the opened URL includes `?mcp=...`; if not, restart the session.
- Port conflict: set `PORT` or let the server try the `6002-6020` range.
- Export timeout for PNG/SVG: keep the browser tab open and wait until the diagram is visible, then retry export.
- XML validation error: check escaped characters, duplicate IDs, unclosed tags, nested cells, and malformed `mxGeometry`.
- Private/offline deployment: set `DRAWIO_BASE_URL` to a reachable self-hosted draw.io instance.

## Safety Defaults

- Preserve user browser edits by reading with `get_diagram` before editing.
- Prefer small `edit_diagram` operations over whole-diagram replacement after a diagram exists.
- Do not overwrite a populated diagram unless the user asked to start over.
- Export to paths requested by the user or clearly scoped project/output paths.
