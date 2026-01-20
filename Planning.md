# Forest Graph Library - Development Plan

## Project Overview

A lightweight Next.js/React graph visualization library focused on rendering tree/forest structures with maximum customization flexibility. Users provide their data structure and custom node components, and the library handles layout, positioning, and basic interactions.

### Core Philosophy
- **User-first customization**: Users define node appearance with their own TSX/HTML/Tailwind components
- **Data-driven**: Simple API to map user data to graph structure
- **Interaction-ready**: Built-in support for click, hover, and basic graph interactions
- **Minimal complexity**: Leverage D3.js for positioning, focus on DX (developer experience)

---

## Technology Stack

### Core Dependencies
- **React 18+** - Component library
- **TypeScript** - Type safety and better DX
- **d3-hierarchy** - Tree layout calculations (lightweight, ~15KB)
- **d3-shape** (optional) - Curved edges if needed

### Build Tools
- **tsup** - Zero-config TypeScript bundler (simpler than Rollup)
- **React** as peer dependency

### Development
- **Storybook** - Component development and documentation
- **Vitest** - Testing framework
- **Example Next.js app** - For testing integration

### Optional Enhancement
- **react-zoom-pan-pinch** - Pan/zoom functionality (can add later)

---

## Project Structure

```
forest-graph-lib/
├── src/
│   ├── components/
│   │   ├── ForestGraph.tsx          # Main graph container
│   │   ├── TreeGraph.tsx            # Single tree renderer
│   │   ├── NodeWrapper.tsx          # Wrapper for user's custom node
│   │   └── Edge.tsx                 # Edge/link component
│   ├── hooks/
│   │   ├── useGraphLayout.ts        # D3 layout calculation
│   │   ├── useGraphInteraction.ts   # Click, hover state management
│   │   └── useGraphZoom.ts          # Pan/zoom controls (future)
│   ├── utils/
│   │   ├── dataTransform.ts         # Convert user data to hierarchy
│   │   └── layoutCalculator.ts      # D3 wrapper functions
│   ├── types/
│   │   └── index.ts                 # TypeScript definitions
│   └── index.ts                     # Public API exports
├── examples/
│   └── nextjs-demo/                 # Example Next.js project
├── stories/                          # Storybook stories
├── tests/
├── dist/                            # Build output (gitignored)
├── package.json
├── tsconfig.json
├── tsup.config.ts
└── README.md
```

---

## API Design

### Simple Usage Example

```tsx
'use client'

import { ForestGraph } from '@yourname/forest-graph';

// User's custom node component
const MyCustomNode = ({ data, isHovered, isSelected }) => (
  <div className={`
    p-4 rounded-lg border-2 
    ${isSelected ? 'border-blue-500 bg-blue-50' : 'border-gray-300'}
    ${isHovered ? 'shadow-lg' : 'shadow'}
  `}>
    <h3 className="font-bold">{data.name}</h3>
    <p className="text-sm text-gray-600">{data.role}</p>
  </div>
);

// User's data
const orgData = [
  { id: '1', name: 'CEO', role: 'Leadership', parentId: null },
  { id: '2', name: 'CTO', role: 'Technology', parentId: '1' },
  { id: '3', name: 'CFO', role: 'Finance', parentId: '1' },
  { id: '4', name: 'Dev Lead', role: 'Engineering', parentId: '2' },
];

// Usage
function MyApp() {
  return (
    <ForestGraph
      data={orgData}
      nodeComponent={MyCustomNode}
      idField="id"
      parentField="parentId"
      onNodeClick={(node) => console.log('Clicked:', node)}
      onNodeHover={(node) => console.log('Hovered:', node)}
      layout={{
        direction: 'vertical', // 'vertical' | 'horizontal' | 'radial'
        nodeSpacing: 100,
        levelSpacing: 150,
      }}
      edgeStyle={{
        stroke: '#94a3b8',
        strokeWidth: 2,
        type: 'straight' // 'straight' | 'curved'
      }}
    />
  );
}
```

---

## Core Components to Build

### 1. `ForestGraph.tsx` (Main Component)
**Purpose**: Top-level container that orchestrates everything

**Responsibilities**:
- Accept user data and configuration
- Transform flat data into hierarchical structure
- Calculate layout using D3
- Manage global state (selected nodes, hovered nodes)
- Render multiple trees if forest structure detected

**Props Interface**:
```typescript
interface ForestGraphProps<T = any> {
  data: T[];
  nodeComponent: React.ComponentType<NodeComponentProps<T>>;
  idField: keyof T;
  parentField: keyof T;
  onNodeClick?: (node: T) => void;
  onNodeHover?: (node: T) => void;
  layout?: LayoutConfig;
  edgeStyle?: EdgeStyle;
  className?: string;
  width?: number;
  height?: number;
}
```

**Implementation Steps**:
1. Validate and transform input data
2. Detect forest structure (multiple root nodes)
3. Calculate layout for each tree using `useGraphLayout`
4. Render SVG container with trees and edges
5. Handle user interactions

---

### 2. `useGraphLayout.ts` (Layout Hook)
**Purpose**: Calculate node positions using D3

**Responsibilities**:
- Use `d3-hierarchy` to create tree structure
- Apply layout algorithm (tree, cluster)
- Return positioned nodes and links

**Implementation**:
```typescript
import { hierarchy, tree } from 'd3-hierarchy';

interface LayoutConfig {
  direction: 'vertical' | 'horizontal' | 'radial';
  nodeSpacing: number;
  levelSpacing: number;
}

function useGraphLayout<T>(
  data: T[],
  idField: keyof T,
  parentField: keyof T,
  config: LayoutConfig
) {
  const [layout, setLayout] = useState(null);

  useEffect(() => {
    // 1. Transform flat data to hierarchical
    const hierarchicalData = stratify()
      .id(d => d[idField])
      .parentId(d => d[parentField])
      (data);

    // 2. Apply D3 tree layout
    const treeLayout = tree()
      .nodeSize([config.nodeSpacing, config.levelSpacing]);
    
    const root = treeLayout(hierarchicalData);

    // 3. Adjust coordinates based on direction
    const nodes = root.descendants().map(node => ({
      ...node.data,
      x: config.direction === 'vertical' ? node.x : node.y,
      y: config.direction === 'vertical' ? node.y : node.x,
    }));

    const links = root.links();

    setLayout({ nodes, links });
  }, [data, config]);

  return layout;
}
```

---

### 3. `NodeWrapper.tsx` (Node Wrapper)
**Purpose**: Wrap user's custom component with interaction logic

**Responsibilities**:
- Position the user's component at calculated coordinates
- Handle hover/click events
- Pass interaction state to user component
- Manage sizing and positioning

**Implementation**:
```typescript
interface NodeWrapperProps<T> {
  node: T & { x: number; y: number };
  UserComponent: React.ComponentType<NodeComponentProps<T>>;
  onHover: (node: T) => void;
  onClick: (node: T) => void;
  isHovered: boolean;
  isSelected: boolean;
}

function NodeWrapper<T>({
  node,
  UserComponent,
  onHover,
  onClick,
  isHovered,
  isSelected
}: NodeWrapperProps<T>) {
  return (
    <foreignObject
      x={node.x - 75} // Center the node (adjust based on size)
      y={node.y - 40}
      width={150}
      height={80}
      onMouseEnter={() => onHover(node)}
      onMouseLeave={() => onHover(null)}
      onClick={() => onClick(node)}
      style={{ overflow: 'visible', cursor: 'pointer' }}
    >
      <UserComponent
        data={node}
        isHovered={isHovered}
        isSelected={isSelected}
      />
    </foreignObject>
  );
}
```

---

### 4. `Edge.tsx` (Edge Component)
**Purpose**: Render connections between nodes

**Responsibilities**:
- Draw lines (straight or curved) between parent and child nodes
- Apply user-defined styling

**Implementation**:
```typescript
import { linkVertical } from 'd3-shape'; // For curved edges

interface EdgeProps {
  source: { x: number; y: number };
  target: { x: number; y: number };
  style: EdgeStyle;
}

function Edge({ source, target, style }: EdgeProps) {
  if (style.type === 'curved') {
    const link = linkVertical()
      .x(d => d.x)
      .y(d => d.y);
    
    const pathData = link({ source, target });
    
    return (
      <path
        d={pathData}
        fill="none"
        stroke={style.stroke}
        strokeWidth={style.strokeWidth}
      />
    );
  }

  // Straight line
  return (
    <line
      x1={source.x}
      y1={source.y}
      x2={target.x}
      y2={target.y}
      stroke={style.stroke}
      strokeWidth={style.strokeWidth}
    />
  );
}
```

---

### 5. `useGraphInteraction.ts` (Interaction Hook)
**Purpose**: Manage click and hover states

**Implementation**:
```typescript
function useGraphInteraction<T>() {
  const [hoveredNode, setHoveredNode] = useState<T | null>(null);
  const [selectedNode, setSelectedNode] = useState<T | null>(null);

  const handleHover = useCallback((node: T | null) => {
    setHoveredNode(node);
  }, []);

  const handleClick = useCallback((node: T) => {
    setSelectedNode(prev => prev === node ? null : node);
  }, []);

  return {
    hoveredNode,
    selectedNode,
    handleHover,
    handleClick,
  };
}
```

---

## TypeScript Definitions

```typescript
// src/types/index.ts

export interface NodeComponentProps<T = any> {
  data: T;
  isHovered: boolean;
  isSelected: boolean;
}

export interface LayoutConfig {
  direction: 'vertical' | 'horizontal' | 'radial';
  nodeSpacing: number;
  levelSpacing: number;
}

export interface EdgeStyle {
  stroke: string;
  strokeWidth: number;
  type: 'straight' | 'curved';
}

export interface ForestGraphProps<T = any> {
  data: T[];
  nodeComponent: React.ComponentType<NodeComponentProps<T>>;
  idField: keyof T;
  parentField: keyof T;
  onNodeClick?: (node: T) => void;
  onNodeHover?: (node: T) => void;
  layout?: Partial<LayoutConfig>;
  edgeStyle?: Partial<EdgeStyle>;
  className?: string;
  width?: number;
  height?: number;
}
```

---

## Development Steps

### Phase 1: Foundation (Week 1)
1. **Initialize project**
   ```bash
   mkdir forest-graph-lib && cd forest-graph-lib
   npm init -y
   npm install -D typescript react react-dom @types/react @types/react-dom
   npm install -D tsup
   npm install d3-hierarchy d3-shape
   npm install -D @types/d3-hierarchy @types/d3-shape
   ```

2. **Setup tsup configuration**
   ```typescript
   // tsup.config.ts
   import { defineConfig } from 'tsup';

   export default defineConfig({
     entry: ['src/index.ts'],
     format: ['cjs', 'esm'],
     dts: true,
     splitting: false,
     sourcemap: true,
     clean: true,
     external: ['react', 'react-dom'],
   });
   ```

3. **Setup package.json**
   ```json
   {
     "name": "@yourname/forest-graph",
     "version": "0.1.0",
     "main": "./dist/index.js",
     "module": "./dist/index.mjs",
     "types": "./dist/index.d.ts",
     "files": ["dist"],
     "scripts": {
       "build": "tsup",
       "dev": "tsup --watch"
     },
     "peerDependencies": {
       "react": "^18.0.0 || ^19.0.0",
       "react-dom": "^18.0.0 || ^19.0.0"
     }
   }
   ```

4. **Create TypeScript types** (`src/types/index.ts`)

5. **Build basic ForestGraph component** (accept data, render simple SVG)

### Phase 2: Layout & D3 Integration (Week 2)
6. **Implement `useGraphLayout` hook**
   - Transform flat data to hierarchy
   - Apply D3 tree layout
   - Handle multiple roots (forest)

7. **Implement `dataTransform.ts` utility**
   - Data validation
   - Hierarchy creation with `d3-hierarchy.stratify()`

8. **Test with sample data** in example Next.js app

### Phase 3: Custom Nodes (Week 3)
9. **Build `NodeWrapper` component**
   - foreignObject SVG positioning
   - Pass user component
   - Handle sizing

10. **Implement interaction state management**
    - Create `useGraphInteraction` hook
    - Wire up hover/click handlers

11. **Test with custom Tailwind components**

### Phase 4: Edges & Polish (Week 4)
12. **Build `Edge` component**
    - Straight lines
    - Curved paths with d3-shape

13. **Add layout direction support**
    - Vertical (top-down)
    - Horizontal (left-right)
    - Coordinate transformation

14. **Configuration defaults**
    - Sensible spacing defaults
    - Edge styling defaults

### Phase 5: Testing & Documentation
15. **Write README with examples**
16. **Create Storybook stories**
17. **Write basic tests**
18. **Publish v0.1.0 to npm**

---

## Build & Publish Commands

```bash
# Development
npm run dev          # Watch mode

# Build
npm run build        # Create dist/

# Test locally
npm link             # In library directory
npm link @yourname/forest-graph  # In example app

# Publish
npm run build
npm publish --access public
```

---

## Example Next.js Integration

```tsx
// app/page.tsx
'use client'

import { ForestGraph } from '@yourname/forest-graph';

const CustomNode = ({ data, isHovered, isSelected }) => (
  <div className={`
    px-6 py-4 rounded-xl border-2 transition-all
    ${isSelected ? 'border-blue-600 bg-blue-100 scale-110' : 'border-gray-300 bg-white'}
    ${isHovered ? 'shadow-2xl' : 'shadow-md'}
  `}>
    <div className="font-bold text-lg">{data.title}</div>
    <div className="text-sm text-gray-600">{data.subtitle}</div>
  </div>
);

export default function Home() {
  const data = [
    { id: 1, title: 'Root', subtitle: 'Top level', parentId: null },
    { id: 2, title: 'Child 1', subtitle: 'Branch A', parentId: 1 },
    { id: 3, title: 'Child 2', subtitle: 'Branch B', parentId: 1 },
  ];

  return (
    <div className="p-8">
      <h1 className="text-3xl font-bold mb-8">My Organization</h1>
      <ForestGraph
        data={data}
        nodeComponent={CustomNode}
        idField="id"
        parentField="parentId"
        onNodeClick={(node) => alert(`Clicked: ${node.title}`)}
      />
    </div>
  );
}
```

---

## Future Enhancements (Post v0.1.0)

- **Pan/Zoom**: Integrate react-zoom-pan-pinch
- **Animations**: Smooth node transitions
- **Auto-sizing**: Dynamic node dimensions based on content
- **Export**: SVG/PNG download
- **Minimap**: Overview for large graphs
- **Search/Filter**: Highlight specific nodes
- **More layouts**: Radial, force-directed

---

## Success Criteria

✅ Users can pass any TSX component as node renderer  
✅ Graph automatically positions nodes in tree layout  
✅ Hover and click interactions work out of the box  
✅ Works seamlessly in Next.js (SSR compatible)  
✅ TypeScript support with full type inference  
✅ Bundle size < 50KB (excluding React)  
✅ Clear documentation with examples  

---

## Timeline Estimate

- **Week 1-2**: Core foundation + D3 layout
- **Week 3**: Custom node rendering + interactions  
- **Week 4**: Edge rendering + polish
- **Week 5**: Documentation + publish

**Total: ~5 weeks for v0.1.0**

---

## Key Design Decisions

1. **Use `foreignObject` for custom nodes**: Allows full HTML/CSS/Tailwind inside SVG
2. **D3 only for math**: Keep rendering in React for better integration
3. **Peer dependencies for React**: Users bring their own React version
4. **TypeScript-first**: Better DX and fewer bugs
5. **Simple API**: One component, clear props, easy to understand

---

This plan gives you a clear roadmap to build a practical, user-friendly graph library without over-engineering. Start with Phase 1 and iterate based on feedback!