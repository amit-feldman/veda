<script setup>
import { ref, reactive, computed, onMounted, onUnmounted } from 'vue'

const GATE_TYPES = {
  INPUT:  { label: 'IN',   inputs: 0, outputs: 1, width: 60, height: 40, color: '#4CAF50' },
  OUTPUT: { label: 'OUT',  inputs: 1, outputs: 0, width: 60, height: 40, color: '#f44336' },
  NOT:    { label: 'NOT',  inputs: 1, outputs: 1, width: 60, height: 40, color: '#9C27B0' },
  AND:    { label: 'AND',  inputs: 2, outputs: 1, width: 70, height: 50, color: '#2196F3' },
  OR:     { label: 'OR',   inputs: 2, outputs: 1, width: 70, height: 50, color: '#FF9800' },
  XOR:    { label: 'XOR',  inputs: 2, outputs: 1, width: 70, height: 50, color: '#00BCD4' },
  NAND:   { label: 'NAND', inputs: 2, outputs: 1, width: 70, height: 50, color: '#3F51B5' },
  NOR:    { label: 'NOR',  inputs: 2, outputs: 1, width: 70, height: 50, color: '#E91E63' },
}

let nextId = 1
const gates = reactive([])
const wires = reactive([])
const selectedTool = ref(null)
const draggingGate = ref(null)
const dragOffset = ref({ x: 0, y: 0 })
const wiringFrom = ref(null)
const mousePos = ref({ x: 0, y: 0 })
const showVerilog = ref(false)
const showProblems = ref(true)
const highlightedGateId = ref(null)
const toasts = reactive([])

function addToast(msg, type = 'error') {
  const t = { id: nextId++, msg, type }
  toasts.push(t)
  setTimeout(() => {
    const idx = toasts.findIndex(e => e.id === t.id)
    if (idx !== -1) toasts.splice(idx, 1)
  }, 3000)
}

function wouldCreateCycle(fromGateId, toGateId) {
  const visited = new Set()
  const stack = [fromGateId]
  while (stack.length) {
    const current = stack.pop()
    if (current === toGateId) return true
    if (visited.has(current)) continue
    visited.add(current)
    for (const w of wires) {
      if (w.toGateId === current) stack.push(w.fromGateId)
    }
  }
  return false
}

function getUnconnectedInputs(gate) {
  const def = GATE_TYPES[gate.type]
  const missing = []
  for (let i = 0; i < def.inputs; i++) {
    if (!wires.some(w => w.toGateId === gate.id && w.toPort === i)) {
      missing.push(i)
    }
  }
  return missing
}

function addGate(type, x, y) {
  const def = GATE_TYPES[type]
  gates.push({
    id: nextId++,
    type,
    x: x - def.width / 2,
    y: y - def.height / 2,
    value: type === 'INPUT' ? 0 : null,
    inputValues: Array(def.inputs).fill(null),
  })
  simulate()
}

function onCanvasClick(e) {
  if (wiringFrom.value) {
    wiringFrom.value = null
    return
  }
  if (!selectedTool.value) return
  const rect = e.currentTarget.getBoundingClientRect()
  addGate(selectedTool.value, e.clientX - rect.left, e.clientY - rect.top)
}

function getInputPortPos(gate, index) {
  const def = GATE_TYPES[gate.type]
  const spacing = def.height / (def.inputs + 1)
  return { x: gate.x, y: gate.y + spacing * (index + 1) }
}

function getOutputPortPos(gate) {
  const def = GATE_TYPES[gate.type]
  return { x: gate.x + def.width, y: gate.y + def.height / 2 }
}

function onPortClick(gateId, portType, portIndex, e) {
  e.stopPropagation()
  if (!wiringFrom.value) {
    wiringFrom.value = { gateId, portType, portIndex }
    return
  }

  const from = wiringFrom.value
  let source, target

  if (from.portType === 'output' && portType === 'input') {
    source = { gateId: from.gateId }
    target = { gateId, portIndex }
  } else if (from.portType === 'input' && portType === 'output') {
    source = { gateId }
    target = { gateId: from.gateId, portIndex: from.portIndex }
  } else {
    addToast(
      from.portType === portType
        ? `Cannot connect ${portType} to ${portType}`
        : 'Invalid connection'
    )
    wiringFrom.value = null
    return
  }

  // no self-connections
  if (source.gateId === target.gateId) {
    addToast('Cannot connect a gate to itself')
    wiringFrom.value = null
    return
  }

  // duplicate wire check
  const duplicate = wires.find(w => w.fromGateId === source.gateId && w.toGateId === target.gateId && w.toPort === target.portIndex)
  if (duplicate) {
    addToast('This connection already exists')
    wiringFrom.value = null
    return
  }

  // cycle detection
  if (wouldCreateCycle(target.gateId, source.gateId)) {
    addToast('Cannot create a cycle in the circuit')
    wiringFrom.value = null
    return
  }

  // remove existing wire to this input port
  const idx = wires.findIndex(w => w.toGateId === target.gateId && w.toPort === target.portIndex)
  if (idx !== -1) wires.splice(idx, 1)

  wires.push({
    id: nextId++,
    fromGateId: source.gateId,
    toGateId: target.gateId,
    toPort: target.portIndex,
  })
  wiringFrom.value = null
  simulate()
}

function onCanvasMouseMove(e) {
  const rect = e.currentTarget.getBoundingClientRect()
  mousePos.value = { x: e.clientX - rect.left, y: e.clientY - rect.top }

  if (draggingGate.value) {
    const gate = gates.find(g => g.id === draggingGate.value)
    if (gate) {
      gate.x = mousePos.value.x - dragOffset.value.x
      gate.y = mousePos.value.y - dragOffset.value.y
    }
  }
}

function startDrag(gate, e) {
  if (selectedTool.value || wiringFrom.value) return
  e.preventDefault()
  const svg = e.currentTarget.ownerSVGElement
  const rect = svg.getBoundingClientRect()
  draggingGate.value = gate.id
  dragOffset.value = {
    x: (e.clientX - rect.left) - gate.x,
    y: (e.clientY - rect.top) - gate.y,
  }
}

function stopDrag() {
  draggingGate.value = null
}

function toggleInput(gate, e) {
  e.stopPropagation()
  if (gate.type === 'INPUT') {
    gate.value = gate.value ? 0 : 1
    simulate()
  }
}

function deleteGate(gateId) {
  const idx = gates.findIndex(g => g.id === gateId)
  if (idx !== -1) gates.splice(idx, 1)
  for (let i = wires.length - 1; i >= 0; i--) {
    if (wires[i].fromGateId === gateId || wires[i].toGateId === gateId) {
      wires.splice(i, 1)
    }
  }
  simulate()
}

function deleteWire(wireId, e) {
  e.stopPropagation()
  const idx = wires.findIndex(w => w.id === wireId)
  if (idx !== -1) wires.splice(idx, 1)
  simulate()
}

function simulate() {
  for (const gate of gates) {
    if (gate.type !== 'INPUT') {
      gate.value = null
      gate.inputValues = Array(GATE_TYPES[gate.type].inputs).fill(null)
    }
  }

  let changed = true
  let iterations = 0
  while (changed && iterations < 100) {
    changed = false
    iterations++

    for (const wire of wires) {
      const fromGate = gates.find(g => g.id === wire.fromGateId)
      const toGate = gates.find(g => g.id === wire.toGateId)
      if (fromGate && toGate && fromGate.value !== null) {
        if (toGate.inputValues[wire.toPort] !== fromGate.value) {
          toGate.inputValues[wire.toPort] = fromGate.value
          changed = true
        }
      }
    }

    for (const gate of gates) {
      if (gate.type === 'INPUT') continue
      const ins = gate.inputValues
      if (ins.some(v => v === null)) continue

      let val = null
      switch (gate.type) {
        case 'NOT':    val = ins[0] ? 0 : 1; break
        case 'AND':    val = (ins[0] & ins[1]); break
        case 'OR':     val = (ins[0] | ins[1]); break
        case 'XOR':    val = (ins[0] ^ ins[1]); break
        case 'NAND':   val = (ins[0] & ins[1]) ? 0 : 1; break
        case 'NOR':    val = (ins[0] | ins[1]) ? 0 : 1; break
        case 'OUTPUT': val = ins[0]; break
      }
      if (gate.value !== val) {
        gate.value = val
        changed = true
      }
    }
  }
}

function clearAll() {
  gates.splice(0)
  wires.splice(0)
  wiringFrom.value = null
  selectedTool.value = null
}

function getWirePath(wire) {
  const fromGate = gates.find(g => g.id === wire.fromGateId)
  const toGate = gates.find(g => g.id === wire.toGateId)
  if (!fromGate || !toGate) return ''
  const from = getOutputPortPos(fromGate)
  const to = getInputPortPos(toGate, wire.toPort)
  const dx = Math.abs(to.x - from.x) * 0.5
  return `M ${from.x} ${from.y} C ${from.x + dx} ${from.y}, ${to.x - dx} ${to.y}, ${to.x} ${to.y}`
}

function getWireValue(wire) {
  const fromGate = gates.find(g => g.id === wire.fromGateId)
  return fromGate?.value
}

const getWiringFromPos = computed(() => {
  if (!wiringFrom.value) return { x: 0, y: 0 }
  const gate = gates.find(g => g.id === wiringFrom.value.gateId)
  if (!gate) return { x: 0, y: 0 }
  return wiringFrom.value.portType === 'output'
    ? getOutputPortPos(gate)
    : getInputPortPos(gate, wiringFrom.value.portIndex)
})

const statusText = computed(() => {
  if (wiringFrom.value) return 'Click a port to connect, or canvas to cancel'
  if (selectedTool.value) return `Click canvas to place ${selectedTool.value}`
  if (draggingGate.value) return 'Dragging...'
  return 'Select a gate or drag to move'
})

function exportVerilog() {
  const inputGates = gates.filter(g => g.type === 'INPUT')
  const outputGates = gates.filter(g => g.type === 'OUTPUT')
  const logicGates = gates.filter(g => !['INPUT', 'OUTPUT'].includes(g.type))

  if (!inputGates.length && !outputGates.length) return '// place some gates first\n'

  const vWarnings = []
  if (!inputGates.length) vWarnings.push('// WARNING: no input ports defined')
  if (!outputGates.length) vWarnings.push('// WARNING: no output ports defined')
  const issueCount = lintProblems.value.length
  if (issueCount) vWarnings.push(`// WARNING: ${issueCount} lint issue${issueCount > 1 ? 's' : ''} in circuit`)
  const warningBlock = vWarnings.length ? vWarnings.join('\n') + '\n\n' : ''

  const ins = inputGates.map((_, i) => `in_${i}`)
  const outs = outputGates.map((_, i) => `out_${i}`)
  const ports = [...ins.map(n => `  input  ${n}`), ...outs.map(n => `  output ${n}`)].join(',\n')

  let v = warningBlock + `module circuit (\n${ports}\n);\n\n`

  for (const gate of logicGates) v += `  wire w_${gate.id};\n`
  if (logicGates.length) v += '\n'

  function srcName(gateId) {
    const g = gates.find(g => g.id === gateId)
    if (!g) return "1'b0"
    if (g.type === 'INPUT') return `in_${inputGates.indexOf(g)}`
    return `w_${g.id}`
  }

  function wireInputs(gate) {
    const result = []
    for (let i = 0; i < GATE_TYPES[gate.type].inputs; i++) {
      const w = wires.find(w => w.toGateId === gate.id && w.toPort === i)
      result.push(w ? srcName(w.fromGateId) : "1'b0")
    }
    return result
  }

  for (const gate of logicGates) {
    const [a, b] = wireInputs(gate)
    const ops = { NOT: `~${a}`, AND: `${a} & ${b}`, OR: `${a} | ${b}`, XOR: `${a} ^ ${b}`, NAND: `~(${a} & ${b})`, NOR: `~(${a} | ${b})` }
    v += `  assign w_${gate.id} = ${ops[gate.type]};\n`
  }

  if (logicGates.length) v += '\n'

  for (let i = 0; i < outputGates.length; i++) {
    const w = wires.find(w => w.toGateId === outputGates[i].id && w.toPort === 0)
    v += `  assign out_${i} = ${w ? srcName(w.fromGateId) : "1'b0"};\n`
  }

  v += '\nendmodule\n'
  return v
}

const lintProblems = computed(() => {
  const problems = []

  // circuit-level checks
  const hasInputs = gates.some(g => g.type === 'INPUT')
  const hasOutputs = gates.some(g => g.type === 'OUTPUT')
  if (gates.length && !hasInputs) problems.push({ severity: 'error', msg: 'Circuit has no INPUT gates', gateId: null, rule: 'no-inputs' })
  if (gates.length && !hasOutputs) problems.push({ severity: 'warning', msg: 'Circuit has no OUTPUT gates', gateId: null, rule: 'no-outputs' })

  for (const gate of gates) {
    const def = GATE_TYPES[gate.type]
    const hasAnyInput = wires.some(w => w.toGateId === gate.id)
    const hasAnyOutput = wires.some(w => w.fromGateId === gate.id)

    // floating gate
    if (def.inputs > 0 && def.outputs > 0 && !hasAnyInput && !hasAnyOutput) {
      problems.push({ severity: 'warning', msg: `${def.label} #${gate.id} is floating (no connections)`, gateId: gate.id, rule: 'floating-gate' })
      continue
    }

    // unconnected inputs
    if (gate.type !== 'INPUT') {
      const missing = getUnconnectedInputs(gate)
      if (missing.length) {
        problems.push({
          severity: 'error',
          msg: `${def.label} #${gate.id}: ${missing.length} unconnected input${missing.length > 1 ? 's' : ''} (port ${missing.join(', ')})`,
          gateId: gate.id,
          rule: 'unconnected-input',
        })
      }
    }

    // dangling output
    if (def.outputs > 0 && !hasAnyOutput) {
      problems.push({ severity: 'warning', msg: `${def.label} #${gate.id}: output not connected`, gateId: gate.id, rule: 'dangling-output' })
    }
  }

  return problems
})

const errorCount = computed(() => lintProblems.value.filter(p => p.severity === 'error').length)
const warningCount = computed(() => lintProblems.value.filter(p => p.severity === 'warning').length)

const gatesWithProblems = computed(() => {
  const map = new Map()
  for (const p of lintProblems.value) {
    if (p.gateId === null) continue
    if (!map.has(p.gateId)) map.set(p.gateId, [])
    map.get(p.gateId).push(p)
  }
  return map
})

const gateSeverity = computed(() => {
  const map = new Map()
  for (const [gateId, problems] of gatesWithProblems.value) {
    map.set(gateId, problems.some(p => p.severity === 'error') ? 'error' : 'warning')
  }
  return map
})

const verilogOutput = computed(() => exportVerilog())

const fanOutPorts = computed(() => {
  const counts = new Map()
  for (const w of wires) {
    counts.set(w.fromGateId, (counts.get(w.fromGateId) || 0) + 1)
  }
  const result = []
  for (const [gateId, count] of counts) {
    if (count > 1) result.push(gateId)
  }
  return result
})

function focusGate(gateId) {
  if (gateId === null) return
  highlightedGateId.value = gateId
  setTimeout(() => {
    if (highlightedGateId.value === gateId) highlightedGateId.value = null
  }, 2000)
}

function autoOrganize() {
  if (!gates.length) return

  // topological sort via BFS from inputs
  const adj = new Map()
  const inDeg = new Map()
  for (const g of gates) {
    adj.set(g.id, [])
    inDeg.set(g.id, 0)
  }
  for (const w of wires) {
    adj.get(w.fromGateId)?.push(w.toGateId)
    inDeg.set(w.toGateId, (inDeg.get(w.toGateId) || 0) + 1)
  }

  const layers = []
  let queue = []
  for (const g of gates) {
    if (inDeg.get(g.id) === 0) queue.push(g.id)
  }

  const placed = new Set()
  while (queue.length) {
    layers.push([...queue])
    queue.forEach(id => placed.add(id))
    const next = []
    for (const id of queue) {
      for (const toId of adj.get(id) || []) {
        inDeg.set(toId, inDeg.get(toId) - 1)
        if (inDeg.get(toId) === 0 && !placed.has(toId)) next.push(toId)
      }
    }
    queue = next
  }

  // catch unplaced gates (isolated)
  for (const g of gates) {
    if (!placed.has(g.id)) {
      layers.push([g.id])
      placed.add(g.id)
    }
  }

  const xStart = 80
  const xGap = 140
  const yGap = 30

  for (let col = 0; col < layers.length; col++) {
    const layer = layers[col]
    const totalHeight = layer.reduce((sum, id) => {
      const g = gates.find(g => g.id === id)
      return sum + GATE_TYPES[g.type].height
    }, 0) + (layer.length - 1) * yGap
    let y = Math.max(40, (500 - totalHeight) / 2)

    for (const id of layer) {
      const g = gates.find(g => g.id === id)
      const def = GATE_TYPES[g.type]
      g.x = xStart + col * xGap
      g.y = y
      y += def.height + yGap
    }
  }
}

function onKeyDown(e) {
  if (e.key === 'Escape') {
    selectedTool.value = null
    wiringFrom.value = null
    draggingGate.value = null
  }
}

onMounted(() => window.addEventListener('keydown', onKeyDown))
onUnmounted(() => window.removeEventListener('keydown', onKeyDown))
</script>

<template>
  <div class="app">
    <div class="sidebar">
      <div class="brand">
        <h1>VEDA</h1>
        <span class="subtitle">visual eda</span>
      </div>

      <div class="section">
        <div class="section-title">Gates</div>
        <div class="gate-grid">
          <button
            v-for="(def, type) in GATE_TYPES"
            :key="type"
            class="gate-btn"
            :class="{ active: selectedTool === type }"
            :style="{ '--c': def.color }"
            @click="selectedTool = selectedTool === type ? null : type"
          >{{ def.label }}</button>
        </div>
      </div>

      <div class="section">
        <div class="section-title">Actions</div>
        <button class="action-btn" @click="selectedTool = null; wiringFrom = null">Pointer</button>
        <button class="action-btn" @click="autoOrganize">Auto Organize</button>
        <button class="action-btn" @click="clearAll">Clear All</button>
        <button class="action-btn" @click="showVerilog = !showVerilog">
          {{ showVerilog ? 'Hide' : 'Export' }} Verilog
        </button>
      </div>

      <div v-if="showVerilog" class="verilog-box">
        <pre>{{ verilogOutput }}</pre>
      </div>

      <div class="help">
        <p>Click gate + canvas to place</p>
        <p>Click output port then input port to wire</p>
        <p>Double-click IN to toggle</p>
        <p>Right-click to delete gate/wire</p>
        <p>Drag to move (pointer mode)</p>
        <p>Esc to cancel</p>
      </div>

      <div class="status">{{ statusText }}</div>
    </div>

    <div class="main-area">
    <svg
      class="canvas"
      @click="onCanvasClick"
      @mousemove="onCanvasMouseMove"
      @mouseup="stopDrag"
    >
      <defs>
        <pattern id="grid" width="20" height="20" patternUnits="userSpaceOnUse">
          <path d="M 20 0 L 0 0 0 20" fill="none" stroke="#1a1a2e" stroke-width="0.5" />
        </pattern>
      </defs>
      <rect width="100%" height="100%" fill="url(#grid)" />

      <!-- wires -->
      <path
        v-for="wire in wires"
        :key="'w-' + wire.id"
        :d="getWirePath(wire)"
        fill="none"
        :stroke="getWireValue(wire) === 1 ? '#4CAF50' : getWireValue(wire) === 0 ? '#555' : '#333'"
        :stroke-width="getWireValue(wire) === 1 ? 3 : 2"
        class="wire"
        @contextmenu.prevent="deleteWire(wire.id, $event)"
      />

      <!-- wiring preview -->
      <line
        v-if="wiringFrom"
        :x1="getWiringFromPos.x" :y1="getWiringFromPos.y"
        :x2="mousePos.x" :y2="mousePos.y"
        stroke="#888" stroke-width="1.5" stroke-dasharray="6,4"
      />

      <!-- gates -->
      <g
        v-for="gate in gates"
        :key="'g-' + gate.id"
        :transform="`translate(${gate.x}, ${gate.y})`"
        class="gate-node"
        @click.stop
        @mousedown="startDrag(gate, $event)"
        @dblclick="toggleInput(gate, $event)"
        @contextmenu.prevent="deleteGate(gate.id)"
      >
        <!-- body -->
        <rect
          :width="GATE_TYPES[gate.type].width"
          :height="GATE_TYPES[gate.type].height"
          :fill="GATE_TYPES[gate.type].color + '20'"
          :stroke="highlightedGateId === gate.id ? '#fff' : GATE_TYPES[gate.type].color"
          :stroke-width="highlightedGateId === gate.id ? 3 : 2"
          rx="6"
          :class="{ 'gate-highlighted': highlightedGateId === gate.id }"
        />
        <!-- lint severity underline -->
        <line
          v-if="gateSeverity.has(gate.id)"
          :x1="4" :y1="GATE_TYPES[gate.type].height + 4"
          :x2="GATE_TYPES[gate.type].width - 4" :y2="GATE_TYPES[gate.type].height + 4"
          :stroke="gateSeverity.get(gate.id) === 'error' ? '#f44336' : '#FF9800'"
          stroke-width="2"
          stroke-dasharray="3,2"
          style="pointer-events: none"
        />
        <!-- label -->
        <text
          :x="GATE_TYPES[gate.type].width / 2"
          :y="GATE_TYPES[gate.type].height / 2"
          text-anchor="middle"
          dominant-baseline="central"
          fill="#ddd"
          font-size="13"
          font-weight="bold"
          style="pointer-events: none; user-select: none"
        >{{ GATE_TYPES[gate.type].label }}</text>

        <!-- lint badge -->
        <g v-if="gateSeverity.has(gate.id)">
          <title>{{ gatesWithProblems.get(gate.id).map(p => p.msg).join('\n') }}</title>
          <circle
            :cx="GATE_TYPES[gate.type].width + 2"
            :cy="-2"
            r="8"
            :fill="gateSeverity.get(gate.id) === 'error' ? '#f44336' : '#FF9800'"
            opacity="0.9"
          />
          <text
            :x="GATE_TYPES[gate.type].width + 2"
            :y="-2"
            text-anchor="middle"
            dominant-baseline="central"
            fill="#fff"
            font-size="11"
            font-weight="bold"
            style="pointer-events: none"
          >{{ gateSeverity.get(gate.id) === 'error' ? 'E' : 'W' }}</text>
        </g>

        <!-- fan-out junction dot -->
        <circle
          v-if="fanOutPorts.includes(gate.id)"
          :cx="GATE_TYPES[gate.type].width"
          :cy="GATE_TYPES[gate.type].height / 2"
          r="3"
          fill="#4CAF50"
          style="pointer-events: none"
        />

        <!-- INPUT value text -->
        <text
          v-if="gate.type === 'INPUT'"
          :x="GATE_TYPES[gate.type].width / 2"
          :y="-8"
          text-anchor="middle"
          :fill="gate.value ? '#4CAF50' : '#777'"
          font-size="14"
          font-weight="bold"
          style="pointer-events: none"
        >{{ gate.value }}</text>

        <!-- OUTPUT indicator -->
        <circle
          v-if="gate.type === 'OUTPUT'"
          :cx="GATE_TYPES[gate.type].width / 2"
          :cy="-10"
          r="6"
          :fill="gate.value === 1 ? '#4CAF50' : gate.value === 0 ? '#f44336' : '#333'"
          stroke="#222"
          stroke-width="1"
        />

        <!-- input ports -->
        <circle
          v-for="(n, i) in GATE_TYPES[gate.type].inputs"
          :key="'ip-' + i"
          :cx="0"
          :cy="GATE_TYPES[gate.type].height / (GATE_TYPES[gate.type].inputs + 1) * (i + 1)"
          r="5"
          fill="#0d0d1a"
          :stroke="gate.inputValues[i] === 1 ? '#4CAF50' : '#555'"
          stroke-width="2"
          class="port"
          @mousedown.stop.prevent
          @click.stop="onPortClick(gate.id, 'input', i, $event)"
        />

        <!-- output port -->
        <circle
          v-if="GATE_TYPES[gate.type].outputs"
          :cx="GATE_TYPES[gate.type].width"
          :cy="GATE_TYPES[gate.type].height / 2"
          r="5"
          fill="#0d0d1a"
          :stroke="gate.value === 1 ? '#4CAF50' : '#555'"
          stroke-width="2"
          class="port"
          @mousedown.stop.prevent
          @click.stop="onPortClick(gate.id, 'output', 0, $event)"
        />
      </g>
    </svg>

    <div class="toast-container">
      <div v-for="t in toasts" :key="t.id" :class="['toast', 'toast-' + t.type]">
        {{ t.msg }}
      </div>
    </div>

    <!-- problems panel  -->
    <div class="problems-panel" :class="{ collapsed: !showProblems }">
      <div class="problems-header" @click="showProblems = !showProblems">
        <span class="problems-title">PROBLEMS</span>
        <span class="problems-counts">
          <span v-if="errorCount" class="count-error">{{ errorCount }}</span>
          <span v-if="warningCount" class="count-warning">{{ warningCount }}</span>
          <span v-if="!errorCount && !warningCount" class="count-ok">0</span>
        </span>
        <span class="problems-toggle">{{ showProblems ? '\u25BC' : '\u25B2' }}</span>
      </div>
      <div v-if="showProblems" class="problems-list">
        <div v-if="!lintProblems.length" class="problems-empty">No problems detected</div>
        <div
          v-for="(problem, i) in lintProblems"
          :key="i"
          :class="['problem-item', 'problem-' + problem.severity]"
          @click="focusGate(problem.gateId)"
        >
          <span class="problem-icon">{{ problem.severity === 'error' ? '\u2716' : '\u26A0' }}</span>
          <span class="problem-msg">{{ problem.msg }}</span>
          <span class="problem-rule">{{ problem.rule }}</span>
        </div>
      </div>
    </div>
    </div>
  </div>
</template>

<style>
* { margin: 0; padding: 0; box-sizing: border-box; }

body {
  background: #0d0d1a;
  color: #ccc;
  font-family: 'Segoe UI', system-ui, -apple-system, sans-serif;
  overflow: hidden;
}

.app {
  display: flex;
  height: 100vh;
}

.sidebar {
  width: 200px;
  background: #111122;
  border-right: 1px solid #222;
  padding: 16px;
  display: flex;
  flex-direction: column;
  gap: 16px;
  overflow-y: auto;
}

.brand h1 {
  font-size: 18px;
  color: #fff;
  letter-spacing: 1px;
}

.brand .subtitle {
  font-size: 11px;
  color: #666;
  text-transform: uppercase;
  letter-spacing: 2px;
}

.section-title {
  font-size: 11px;
  color: #666;
  text-transform: uppercase;
  letter-spacing: 1px;
  margin-bottom: 8px;
}

.gate-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 6px;
}

.gate-btn {
  background: transparent;
  border: 1px solid var(--c);
  color: var(--c);
  padding: 6px 0;
  border-radius: 4px;
  cursor: pointer;
  font-size: 12px;
  font-weight: 600;
  transition: background 0.15s;
}

.gate-btn:hover {
  background: color-mix(in srgb, var(--c) 15%, transparent);
}

.gate-btn.active {
  background: color-mix(in srgb, var(--c) 30%, transparent);
  box-shadow: 0 0 8px color-mix(in srgb, var(--c) 30%, transparent);
}

.action-btn {
  display: block;
  width: 100%;
  background: #1a1a2e;
  border: 1px solid #333;
  color: #aaa;
  padding: 6px 10px;
  border-radius: 4px;
  cursor: pointer;
  font-size: 12px;
  margin-bottom: 4px;
  text-align: left;
}

.action-btn:hover {
  background: #222244;
  color: #ddd;
}

.verilog-box {
  background: #0a0a14;
  border: 1px solid #333;
  border-radius: 4px;
  padding: 8px;
  max-height: 200px;
  overflow: auto;
}

.verilog-box pre {
  font-size: 10px;
  color: #8f8;
  white-space: pre-wrap;
  word-break: break-all;
  font-family: 'SF Mono', 'Fira Code', monospace;
}

.help {
  margin-top: auto;
}

.help p {
  font-size: 10px;
  color: #555;
  line-height: 1.6;
}

.status {
  font-size: 11px;
  color: #888;
  padding-top: 8px;
  border-top: 1px solid #222;
}

.main-area {
  flex: 1;
  display: flex;
  flex-direction: column;
  min-width: 0;
}

.canvas {
  flex: 1;
  background: #0d0d1a;
  cursor: crosshair;
  width: 100%;
}

.gate-node {
  cursor: grab;
}

.gate-node:active {
  cursor: grabbing;
}

.port {
  cursor: pointer;
  transition: r 0.1s;
}

.port:hover {
  r: 7;
}

.wire {
  cursor: pointer;
  transition: stroke 0.2s;
}

.wire:hover {
  stroke: #fff !important;
  filter: drop-shadow(0 0 4px rgba(255,255,255,0.3));
}

.gate-highlighted {
  animation: highlight-pulse 0.6s ease-in-out 3;
}

@keyframes highlight-pulse {
  0%, 100% { filter: none; }
  50% { filter: drop-shadow(0 0 8px #fff); }
}

.toast-container {
  position: fixed;
  top: 16px;
  right: 16px;
  display: flex;
  flex-direction: column;
  gap: 8px;
  z-index: 100;
  pointer-events: none;
}

.toast {
  padding: 8px 16px;
  border-radius: 6px;
  font-size: 12px;
  font-weight: 500;
  box-shadow: 0 4px 12px rgba(0,0,0,0.4);
  animation: toast-in 0.2s ease-out;
}

.toast-error { background: #b71c1c; color: #fff; }
.toast-warning { background: #e65100; color: #fff; }

@keyframes toast-in {
  from { opacity: 0; transform: translateX(20px); }
  to { opacity: 1; transform: translateX(0); }
}

.problems-panel {
  background: #111122;
  border-top: 1px solid #222;
  font-size: 12px;
  max-height: 200px;
  display: flex;
  flex-direction: column;
}

.problems-panel.collapsed {
  max-height: none;
}

.problems-header {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 6px 12px;
  cursor: pointer;
  user-select: none;
  border-bottom: 1px solid #222;
  flex-shrink: 0;
}

.problems-header:hover {
  background: #1a1a2e;
}

.problems-title {
  font-size: 11px;
  font-weight: 600;
  color: #888;
  letter-spacing: 1px;
}

.problems-counts {
  display: flex;
  gap: 8px;
}

.count-error {
  color: #f44336;
  font-weight: 600;
}

.count-error::before { content: '\2716 '; font-size: 10px; }

.count-warning {
  color: #FF9800;
  font-weight: 600;
}

.count-warning::before { content: '\26A0 '; font-size: 10px; }

.count-ok {
  color: #4CAF50;
  font-weight: 600;
}

.count-ok::before { content: '\2714 '; font-size: 10px; }

.problems-toggle {
  margin-left: auto;
  color: #555;
  font-size: 10px;
}

.problems-list {
  overflow-y: auto;
  flex: 1;
}

.problems-empty {
  padding: 12px;
  color: #4CAF50;
  text-align: center;
  font-style: italic;
}

.problem-item {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 4px 12px;
  cursor: pointer;
  border-bottom: 1px solid #1a1a2e;
}

.problem-item:hover {
  background: #1a1a2e;
}

.problem-error .problem-icon { color: #f44336; }
.problem-warning .problem-icon { color: #FF9800; }

.problem-msg {
  flex: 1;
  color: #ccc;
}

.problem-rule {
  color: #555;
  font-family: 'SF Mono', 'Fira Code', monospace;
  font-size: 10px;
}
</style>
