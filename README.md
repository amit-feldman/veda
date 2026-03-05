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
| Export Verilog | Click **Export Verilog** in the sidebar |

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
