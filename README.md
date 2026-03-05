# VEDA

**Visual Electronic Design Automation**

A lightweight, browser-based chip design tool. Drop logic gates, wire them up, and simulate — no HDL syntax required.

Built as a graphical alternative to writing raw Verilog for simple combinational logic.

---

## Preview

```
  ┌──────┐        ┌──────┐
  │  IN  │───┐    │      │
  └──────┘   ├───▶│ AND  │───┐    ┌──────┐
  ┌──────┐   │    │      │   ├───▶│ OUT  │  ●
  │  IN  │───┘    └──────┘   │    └──────┘
  └──────┘                   │
                             │
  ┌──────┐    ┌──────┐       │
  │  IN  │───▶│ NOT  │───────┘
  └──────┘    └──────┘
```

## Features

- **8 logic gates** — AND, OR, NOT, XOR, NAND, NOR + input/output nodes
- **Visual wiring** — click output port, then input port. Bezier curves connect them
- **Live simulation** — toggle inputs and watch values propagate instantly. Wires glow green on logic 1
- **Drag & rearrange** — reposition gates freely on the canvas
- **Circuit linter** — real-time validation panel with error/warning diagnostics (see below)
- **Auto organize** — topological sort layout to clean up your schematic in one click
- **Verilog export** — generate synthesizable `assign`-based combinational Verilog from your schematic
- **Zero dependencies** — pure Vue 3, no external UI libraries

## Getting Started

```bash
git clone https://github.com/amit-feldman/veda.git
cd veda
npm install
npm run dev
```

Open [http://localhost:5173](http://localhost:5173)

## How to Use

| Action | How |
|---|---|
| Place a gate | Select from sidebar, click on canvas |
| Wire two gates | Click an output port (right side), then an input port (left side) |
| Toggle input value | Double-click an **IN** gate |
| Move a gate | Switch to **Pointer** mode, drag the gate |
| Delete a gate | Right-click the gate |
| Delete a wire | Right-click the wire |
| Cancel operation | Press `Esc` |
| Auto organize | Click **Auto Organize** in the sidebar |
| Export Verilog | Click **Export Verilog** in the sidebar |

## Circuit Linter

A VS Code-style problems panel at the bottom validates your circuit in real-time:

| Rule | Severity | Description |
|---|---|---|
| `no-inputs` | Error | Circuit has no INPUT gates |
| `no-outputs` | Warning | Circuit has no OUTPUT gates |
| `unconnected-input` | Error | Gate has input ports with no wire connected |
| `dangling-output` | Warning | Gate output isn't driving anything |
| `floating-gate` | Warning | Gate has zero connections at all |

Connection-time validation prevents:
- **Cycles** — combinational loops are blocked with feedback
- **Duplicate wires** — same source-to-destination connection
- **Self-connections** — a gate wired to itself
- **Same-type port connections** — output-to-output or input-to-input

Click any diagnostic in the problems panel to highlight the offending gate on the canvas.

## Verilog Export

Design a circuit visually, then export it:

```verilog
module circuit (
  input  in_0,
  input  in_1,
  output out_0
);

  wire w_3;

  assign w_3 = in_0 & in_1;

  assign out_0 = w_3;

endmodule
```

## Tech

- [Vue 3](https://vuejs.org/) — composition API, single file component
- [Vite](https://vitejs.dev/) — dev server & build
- SVG — all rendering is native SVG, no canvas or WebGL

## License

MIT
