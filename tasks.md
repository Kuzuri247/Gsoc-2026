# GSoC 2026 — JSON Schema Studio: Enhanced Interaction and Navigation

## Qualification Task 1: `dependentSchemas` Keyword Handler

### What is `dependentSchemas`?

The `dependentSchemas` keyword conditionally applies a subschema to the **entire object** when a named property is present. For example:

```json
{
  "type": "object",
  "properties": {
    "name": { "type": "string" },
    "credit_card": { "type": "string" }
  },
  "dependentSchemas": {
    "credit_card": {
      "properties": {
        "billing_address": { "type": "string" }
      },
      "required": ["billing_address"]
    }
  }
}
```

If the `credit_card` property is present in an instance, the sub-schema (requiring `billing_address`) is applied to the **entire** object.

---

### How Hyperjump Compiles `dependentSchemas`

Looking at the Hyperjump source ([dependentSchemas.js](file:///d:/studio-json-schema/node_modules/@hyperjump/json-schema/lib/keywords/dependentSchemas.js)):

```js
const compile = (schema, ast) => pipe(
  Browser.entries(schema),
  asyncMap(async ([key, dependentSchema]) => [key, await Validation.compile(dependentSchema, ast)]),
  asyncCollectArray
);
```

The [compile](file:///d:/studio-json-schema/node_modules/@hyperjump/json-schema/lib/keywords/dependentSchemas.js#9-14) function:
1. Iterates over the `dependentSchemas` object entries (e.g., `"credit_card"` → sub-schema)
2. For each entry, it compiles the sub-schema into the AST using `Validation.compile`, which returns a **schema URI string** identifying the compiled sub-schema in the AST
3. Collects the results into an **array of `[propertyName, schemaUri]` tuples**

> [!IMPORTANT]
> Key difference from `properties`: The `properties` handler uses `asyncCollectObject` (producing `{ propertyName: schemaUri }`), while `dependentSchemas` uses `asyncCollectArray` (producing `[[propertyName, schemaUri], ...]`). This means the `keywordValue` in our handler will be an **array of tuples**, not an object.

---

### The Handler Implementation

The handler should be added to the `keywordHandlerMap` in [processAST.ts](file:///d:/studio-json-schema/src/utils/processAST.ts) at line 354 (where it is currently commented out):

```typescript
"https://json-schema.org/keyword/dependentSchemas": (ast, keywordValue, nodes, edges, parentId, nodeDepth, renderedNodes) => {
    const value = keywordValue as [string, string][];
    const propertyNames: string[] = [];
    for (const [propertyName, schemaUri] of value) {
        propertyNames.push(propertyName);
        processAST({
            ast,
            schemaUri,
            nodes,
            edges,
            parentId,
            renderedNodes,
            childId: propertyName,
            nodeTitle: `dependentSchemas["${propertyName}"]`,
            nodeDepth
        });
    }
    return { key: "dependentSchemas", data: { value: propertyNames } }
},
```

### Logic Explanation

**Step by step:**

1. **Cast the keyword value** — `keywordValue` arrives as the compiled output from Hyperjump: an array of `[propertyName, schemaUri]` tuples (thanks to `asyncCollectArray`).

2. **Iterate over each tuple** — For each `[propertyName, schemaUri]` pair:
   - Track the `propertyName` in the `propertyNames` array (used for source handle generation)
   - Call [processAST](file:///d:/studio-json-schema/src/utils/processAST.ts#97-185) recursively to render the dependent sub-schema as a child node:
     - `schemaUri`: the compiled URI pointing to the sub-schema in the AST
     - `childId: propertyName`: each child is identified by its triggering property name, producing unique source handle IDs like `parentId-credit_card`
     - `nodeTitle`: labeled as `dependentSchemas["credit_card"]` for clear visual identification
     - `nodeDepth`: same depth increment as the parent called with

3. **Return the result** — The handler returns:
   - `key: "dependentSchemas"`: the keyword name stored in node data (also used by [inferSchemaType](file:///d:/studio-json-schema/src/utils/inferSchemaType.ts#3-63) to identify object-type schemas)
   - `data: { value: propertyNames }`: the array of property names, which drives source handle generation (one handle per property name via the `Array.isArray(value)` branch in [generateSourceHandles](file:///d:/studio-json-schema/src/utils/processAST.ts#191-211))

**Why this pattern?**

- The structure mirrors `properties` and `patternProperties` — both keywords that map named keys to sub-schemas — but adapts to the **array-of-tuples** format that Hyperjump's `dependentSchemas` compile function produces
- Using `propertyName` as `childId` ensures that each source handle is uniquely identified, so edges connect correctly from the parent to each dependent schema node
- The [inferSchemaType.ts](file:///d:/studio-json-schema/src/utils/inferSchemaType.ts) ([line 12](file:///d:/studio-json-schema/src/utils/inferSchemaType.ts#L12)) already includes `"dependentSchemas"` in `objectKeywords`, so nodes with this keyword will be correctly inferred as object-type schemas with the cyan neon color

---

## Qualification Task 2: Topological Sorting of `$defs` Dependencies

### The Problem

Currently, [sortAST](file:///d:/studio-json-schema/src/utils/sortAST.ts#3-27) in [sortAST.ts](file:///d:/studio-json-schema/src/utils/sortAST.ts) only reorders nodes so that the `definitions` keyword handler runs first. This ensures `$defs` are processed before the main schema body. However, it doesn't handle **inter-definition dependencies** — when one `$defs` entry references another via `$ref`.

Consider:
```json
{
  "$defs": {
    "address": {
      "type": "object",
      "properties": {
        "country": { "$ref": "#/$defs/country" }
      }
    },
    "country": {
      "type": "string",
      "enum": ["US", "UK", "IN"]
    }
  },
  "type": "object",
  "properties": {
    "home": { "$ref": "#/$defs/address" }
  }
}
```

Here, `address` depends on `country`. If `address` is rendered first and `country` hasn't been created yet, the `$ref` handler calls [processAST](file:///d:/studio-json-schema/src/utils/processAST.ts#97-185) on the `country` URI. This works because [processAST](file:///d:/studio-json-schema/src/utils/processAST.ts#97-185) will create the `country` node at that point. **However**, the node gets created as a child of `address` in the rendering order rather than as a sibling at the `$defs` level, potentially causing:
- Inconsistent visual hierarchy (country appears nested under address instead of as a peer definition)
- Incorrect `nodeDepth` assignment, since it inherits the depth of the referencing context rather than the `$defs` context
- With recursive references, the `renderedNodes` guard prevents infinite loops, but the node may end up in the wrong visual position

### Proposed Approach: Kahn's Algorithm (BFS Topological Sort)

I propose implementing **Kahn's algorithm** for topological sorting of `$defs` dependencies.

#### Why Kahn's Algorithm?

1. **Explicit cycle detection** — Kahn's algorithm naturally detects cycles: if the sorted output has fewer nodes than the input, there is a cycle. This is critical for recursive schemas (e.g., `A → B → A`) where we need to detect and break cycles gracefully. DFS-based topological sort can also detect cycles, but requires additional back-edge tracking.

2. **Deterministic, breadth-first ordering** — Kahn's produces a level-by-level ordering. Definitions with no dependencies (leaf definitions) are rendered first, then definitions that depend only on those, and so on. This produces the most visually logical layout: foundational schemas appear first (leftmost in the LR dagre layout).

3. **Simpler to implement iteratively** — The BFS queue-based approach is straightforward to implement and debug, with no recursion stack to manage.

4. **Good fit for the existing pipeline** — The sort happens in [sortAST](file:///d:/studio-json-schema/src/utils/sortAST.ts#3-27) before [processAST](file:///d:/studio-json-schema/src/utils/processAST.ts#97-185) is called. We only need to reorder the `$defs` entries; the rest of the AST processing remains unchanged.

#### Implementation Outline

```
function topologicallySortDefs(ast, defsKeywordNode):
    1. EXTRACT defs entries from the keyword value (array of schema URIs)
    
    2. BUILD dependency graph:
       For each def URI:
         - Look up its AST node: ast[defUri]
         - Scan its keyword nodes for "$ref" handlers
         - If a $ref target points to another def in the same $defs block,
           add an edge: defUri → targetDefUri
    
    3. COMPUTE in-degrees for each def node
    
    4. INITIALIZE queue with all defs having in-degree 0 (no dependencies)
    
    5. BFS LOOP:
       - Dequeue a def
       - Add it to the sorted output
       - For each def that depends on it, decrement in-degree
       - If in-degree reaches 0, enqueue it
    
    6. HANDLE CYCLES:
       - If sorted output length < total defs count, cycles exist
       - Append remaining (cyclic) defs in their original order
       - The existing renderedNodes guard in processAST will handle
         the cyclic reference by creating an edge back to the 
         already-rendered node
    
    7. RETURN reordered defs array
```

#### Where It Fits in the Pipeline

The sorting would be integrated into the existing `$defs` keyword handler in [processAST.ts](file:///d:/studio-json-schema/src/utils/processAST.ts) (or in [sortAST.ts](file:///d:/studio-json-schema/src/utils/sortAST.ts) with the AST-level sorting):

```
compiledSchema → sortAST (with topological $defs sort) → processAST → dagre layout → resolveCollisions
```

The `$defs` handler at lines 290-296 currently iterates over `value` entries in their original order. After topological sorting, the entries would be reordered so dependencies come first, ensuring that when [processAST](file:///d:/studio-json-schema/src/utils/processAST.ts#97-185) is called for each def, its dependencies have already been rendered as sibling nodes at the correct depth.

#### Why Not DFS-Based Topological Sort?

DFS-based topological sort (using recursive post-order traversal) works too, but:
- It produces a reverse post-order which is less intuitive for visualization (deepest dependencies first can lead to less natural visual ordering)
- Cycle detection requires maintaining a separate "visiting" state set
- The recursive nature adds complexity when dealing with deeply nested schemas

#### Why Not Sort at the GraphView Level?

Sorting at the AST level (before nodes are created) is cleaner than rearranging nodes after layout because:
- The dagre layout respects insertion order for nodes at the same rank
- Node depth and parent-child relationships are established correctly during creation
- No need for a second layout pass

---
---

# Proposed Enhancements & Upgrades

## 1. Validation Error Navigation

### Current State

The editor in [MonacoEditor.tsx](file:///d:/studio-json-schema/src/components/MonacoEditor.tsx) already catches compilation errors (line 269) and displays them in a validation status bar (line 316-320). However, error messages are shown as flat text — clicking on them does nothing, and there is no way to navigate to the specific line in the schema that caused the error.

### Proposed Approach

**Goal:** When schema validation errors occur, clicking an error highlights and scrolls to the corresponding line number in the editor.

**Implementation:**

1. **Parse error locations from Hyperjump errors** — Hyperjump's compile/validation errors often include schema URIs with JSON Pointer fragments (e.g., `#/properties/age`). Use `jsonc-parser`'s `parseTree` + `findNodeAtLocation` (already imported in [MonacoEditor.tsx](file:///d:/studio-json-schema/src/components/MonacoEditor.tsx)) to resolve these pointers to line/column positions.

2. **Make the validation bar interactive** — Replace the static error text `div` (line 316-320) with a clickable list of errors. Each error item would:
   - Display the error message with a clickable target icon
   - On click, call `editorRef.current.revealPositionInCenter(errorPosition)` and apply a Monaco decoration to highlight the error line (similar to the existing `selectedNode` highlighting at lines 143-203)

3. **Add Monaco error markers** — Use `monaco.editor.setModelMarkers()` to show red squiggly underlines directly in the editor at error positions, matching standard IDE behavior. This gives immediate visual feedback without needing to click.

**Data flow:**
```
Hyperjump compile error → extract JSON Pointer from error
  → jsonc-parser resolves to line/column
  → Monaco markers (squiggly underlines) + clickable error list
  → onClick → revealPositionInCenter + highlight decoration
```

---

## 2. Edge Collision Resolution

### Current State

[resolveCollisions.ts](file:///d:/studio-json-schema/src/utils/resolveCollisions.ts) handles **node overlaps** only, using a iterative vertical push algorithm. **Edge overlaps** (edges crossing or overlapping each other) are not addressed, making large graphs hard to read.

### Proposed Approach

**Goal:** Reduce edge overlappings and crossings to improve readability in large graphs.

**Implementation:**

1. **Edge routing with orthogonal paths** — Replace the current `smoothstep` edge type with a custom edge component that uses orthogonal (right-angle) routing. This naturally reduces visual tangling since edges follow gridded paths rather than arbitrary curves.

2. **Edge bundling for parallel edges** — When multiple edges connect nodes in the same column, bundle them visually:
   - Detect edges that share a common source or target column
   - Offset bundled edges by a small margin so they fan out instead of overlapping
   - This can be done as a post-layout pass on the edges array

3. **Leverage dagre's `ranksep` and `nodesep`** — Increase spacing parameters in the dagre layout configuration to give edges more room. Currently `HORIZONTAL_GAP = 150` is applied manually (line 166 of GraphView.tsx). Tuning dagre's native spacing params would distribute nodes more evenly.

4. **Port ordering on handles** — Sort source handles on each node so that handles connecting to upper nodes are positioned above handles connecting to lower nodes. This reduces unnecessary crossings. The `handleOffsets` tracking in [CustomReactFlowNode.tsx](file:///d:/studio-json-schema/src/components/CustomReactFlowNode.tsx) already supports per-handle positioning.

---

## 3. Edge-Based Navigation Controls

### Current State

Edges currently support hover highlighting (edge color + animation, lines 193-212 of [GraphView.tsx](file:///d:/studio-json-schema/src/components/GraphView.tsx)) but no navigation actions.

### Proposed Approach

**Goal:** On hovering over an edge, display navigation buttons to focus/center the source or target node.

**Implementation:**

1. **Custom edge component** — Create a `NavigableEdge` React component that renders:
   - The edge path (using `getBezierPath` or `getSmoothStepPath` from `@xyflow/react`)
   - A floating control panel at the edge midpoint (visible on hover) with two buttons:
     - "← Source" — calls `setCenter(sourceNode.x, sourceNode.y, { zoom, duration: 500 })` and selects the source node
     - "Target →" — calls `setCenter(targetNode.x, targetNode.y, { zoom, duration: 500 })` and selects the target node

2. **Register as custom edge type** — Add `edgeTypes={{ navigableEdge: NavigableEdge }}` to the ReactFlow component, and set `type: "navigableEdge"` on all edges created in [processAST](file:///d:/studio-json-schema/src/utils/processAST.ts#97-185).

3. **Access node positions** — The custom edge receives `sourceX, sourceY, targetX, targetY` as props. For centering, use the `useReactFlow()` hook's `setCenter` and `getNode` to get full node positions.

4. **Styling** — The control panel uses a semi-transparent background with the edge's data color, appearing with a fade-in animation on hover and disappearing on mouse leave. Buttons are compact icons (e.g., `MdNavigateBefore` / `MdNavigateNext` from `react-icons`, already used in the search navigation).

---

## 4. Fix Handle Misalignment on Live Edits

### Current State

In [CustomReactFlowNode.tsx](file:///d:/studio-json-schema/src/components/CustomReactFlowNode.tsx), source handle positions are calculated using `useLayoutEffect` (line 16) which reads `offsetParent` bounding rects and sets `handleOffsets`. The dependency array is `[data.nodeData]`.

**The problem:** When schema edits add/remove keywords, the node's content changes size, but handle offsets may not recalculate if `data.nodeData` reference equality isn't properly invalidated. This causes handles to visually misalign with their corresponding data rows.

### Proposed Approach

1. **Use `ResizeObserver` instead of `useLayoutEffect`** — Replace the current offset calculation with a `ResizeObserver` on the node container element. This ensures handles reposition whenever the node's DOM dimensions change, regardless of what triggered the change.

2. **Add `data.sourceHandles` to dependencies** — If keeping `useLayoutEffect`, also depend on `data.sourceHandles` since the handle list changes on schema edits. Currently only `data.nodeData` is watched.

3. **Force re-measure on React Flow's `onNodesChange`** — After the main nodes state update, trigger a force re-render of affected nodes to recalculate handle positions. This handles edge cases where the node data reference changes but DOM hasn't yet reflowed.

4. **Use React Flow's `updateNodeInternals`** — Call `useUpdateNodeInternals()` hook from `@xyflow/react` after schema edits that add/remove source handles, forcing React Flow to recalculate handle positions and edge paths.

---

## 5. Distinct Visualization for Multi-Type Schemas

### Current State

The `type` keyword handler (line 382 of [processAST.ts](file:///d:/studio-json-schema/src/utils/processAST.ts)) uses [createBasicKeywordHandler("type")](file:///d:/studio-json-schema/src/utils/processAST.ts#250-263), which stores the type value as-is. For `"type": "string"` this works fine, but for `"type": ["string", "null"]` it stores an array. The [inferSchemaType](file:///d:/studio-json-schema/src/utils/inferSchemaType.ts#3-63) function (line 4 of [inferSchemaType.ts](file:///d:/studio-json-schema/src/utils/inferSchemaType.ts)) only checks `typeof nodeData.type?.value === "string"`, so multi-type schemas fall through to keyword-based inference or return `"others"`.

### Proposed Approach

**Goal:** Clear, intuitive rendering for nodes with multiple types.

**Implementation:**

1. **Update [inferSchemaType](file:///d:/studio-json-schema/src/utils/inferSchemaType.ts#3-63)** — Add an `Array.isArray(nodeData.type?.value)` check that returns a new category `"multiType"` with the type array.

2. **Segmented color bar in node header** — When a node has multiple types, render its header as a segmented gradient bar using the neon colors from the existing `neonColors` map. For example, `["string", "null"]` → a header with half magenta (#FF6EFF) and half purple (#A259FF). This gives instant visual recognition.

3. **Type badges in node body** — Display each type as a small colored badge (pill shape) in the node data row:
   ```
   type: [string] [null]
   ```
   Each badge uses the corresponding neon color as its background.

4. **Update [CustomReactFlowNode.tsx](file:///d:/studio-json-schema/src/components/CustomReactFlowNode.tsx)** — Add conditional rendering logic: when the `type` data value is an array, render the segmented header and badges instead of plain text.

5. **Border color** — Use the first type's color as the primary border color, maintaining visual consistency with single-type nodes.

---

## 6. File Upload Support

### Current State

Schemas can only be entered by typing/pasting into the Monaco editor. There is no file upload mechanism. The schema persistence uses `sessionStorage` ([MonacoEditor.tsx](file:///d:/studio-json-schema/src/components/MonacoEditor.tsx) lines 78-90).

### Proposed Approach

**Goal:** Allow users to upload [.json](file:///d:/studio-json-schema/package.json) / `.yaml` JSON Schema files for instant visualization.

**Implementation:**

1. **Add upload button to the navigation bar** — Place a file upload icon button in [NavigationBar.tsx](file:///d:/studio-json-schema/src/components/NavigationBar.tsx) next to the existing controls.

2. **Hidden file input + FileReader** — Use a hidden `<input type="file" accept=".json,.yaml,.yml">` element triggered by the button click. On file selection:
   - Read file contents using `FileReader.readAsText()`
   - Detect format from file extension ([.json](file:///d:/studio-json-schema/package.json) → JSON, `.yaml`/`.yml` → YAML)
   - Update `schemaFormat` via `changeSchemaFormat()` from [AppContext](file:///d:/studio-json-schema/src/contexts/AppContext.tsx#10-25)
   - Set the editor content via `setSchemaText(fileContent)`

3. **Drag-and-drop support** — Add `onDragOver` and `onDrop` handlers to the main editor/visualization container. Show a visual drop zone overlay when a file is being dragged over the app.

4. **Validation on upload** — The existing schema compilation pipeline (lines 217-280 of MonacoEditor.tsx) will automatically validate the uploaded schema since it triggers on `schemaText` changes. No additional validation code needed.

5. **Error handling** — Show a toast notification if the uploaded file isn't valid JSON/YAML (parsing fails), or if it exceeds a reasonable size limit (e.g., 5MB).

---

## 7. Downloadable Visualization Output

### Current State

There is no export functionality. Users cannot save the visualization for documentation or sharing.

### Proposed Approach

**Goal:** Enable exporting the visualization as an image (PNG/SVG) or as the schema file itself.

**Implementation:**

1. **PNG export using `html-to-image`** — Use the `html-to-image` library (or `dom-to-image-more`) to capture the ReactFlow viewport:
   ```ts
   import { toPng } from 'html-to-image';
   
   const exportToPng = async () => {
     const viewport = document.querySelector('.react-flow__viewport');
     const dataUrl = await toPng(viewport, { backgroundColor: '#1a1a2e' });
     // trigger download
   };
   ```
   Use `reactFlowInstance.fitView()` before export to ensure all nodes are captured.

2. **SVG export** — ReactFlow renders edges as SVG paths. Use `toSvg()` from `html-to-image` for a vector-based export that looks crisp at any zoom level.

3. **Schema file download** — Add "Download Schema" option that exports the current editor content as a [.json](file:///d:/studio-json-schema/package.json) or `.yaml` file using `Blob` + `URL.createObjectURL`.

4. **Export controls** — Add a dropdown export button in the navigation bar or bottom bar with options:
   - "Export as PNG" — raster image with background
   - "Export as SVG" — vector image
   - "Download Schema (.json / .yaml)" — raw schema file

5. **Fit-to-content before export** — Before capturing, call `fitView({ padding: 0.1 })` to ensure the full graph is in the viewport. Restore the previous zoom/pan position after export.
