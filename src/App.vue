<template>
  <div class="h-full w-full flex">
    <!-- Left panel with tabs/config -->
    <aside class="w-96 border-r flex flex-col bg-white/60 backdrop-blur">
      <div class="p-3 font-semibold">Plant L‑system</div>

      <div class="px-3 pb-2">
        <label class="block text-xs text-slate-600 mb-1">Branching preset</label>
        <select v-model="preset" class="w-full border rounded px-2 py-1 text-sm">
          <option value="monopodial">Monopodial (lateral)</option>
          <option value="sympodial">Sympodial (Aono–Kunii inspired)</option>
        </select>
      </div>

      <div class="px-3 py-2 flex items-center gap-3">
        <span class="text-sm">Steps</span>
        <input type="range" min="4" max="24" v-model.number="steps" />
        <span class="w-8 text-right text-sm tabular-nums">{{ steps }}</span>
      </div>
      <div class="px-3 py-2 flex gap-2">
        <button class="btn" @click="seed++">Reseed</button>
        <button class="btn-alt" @click="resetPreset">Reset preset</button>
      </div>
      <div class="px-3 pb-2 text-xs text-slate-600">Schedule: {{ scheduleLabel }}</div>

      <div class="px-3 pt-2 flex gap-2">
        <button :class="tabBtn('controls')" @click="tab='controls'">Controls</button>
        <button :class="tabBtn('dsl')" @click="tab='dsl'">DSL</button>
      </div>

      <div v-if="tab==='controls'" class="p-3 text-xs text-slate-700 leading-relaxed">
        <p>Pick a preset, tweak <b>Steps</b> and <b>Seed</b>. The grammar below updates automatically. Edit the DSL to customize rules; the right side renders the result.</p>
      </div>
      <div v-else class="p-3">
        <textarea v-model="dsl" class="code" spellcheck="false"></textarea>
      </div>
    </aside>

    <!-- Right: viewer fills remaining space -->
    <main class="flex-1 relative">
      <div ref="mount" class="viewer"></div>
    </main>
  </div>
</template>

<script setup>
import { onMounted, onBeforeUnmount, ref, watch, computed } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { mergeGeometries } from 'three/examples/jsm/utils/BufferGeometryUtils.js'

// ---------------------------- Core L-system types ----------------------------
class Sym {
  /** @param {string} type @param {number[]} params */
  constructor(type, params = []) { this.type = type; this.params = params }
}

/** Deterministic RNG (Mulberry32) */
function makeRng (seed = 1) {
  let t = seed >>> 0
  return () => {
    t += 0x6D2B79F5
    let r = Math.imul(t ^ (t >>> 15), 1 | t)
    r ^= r + Math.imul(r ^ (r >>> 7), 61 | r)
    return ((r ^ (r >>> 14)) >>> 0) / 4294967296
  }
}

/** One parallel derivation step (pure) */
function deriveOnce (word, productions, rand) {
  const out = []
  for (let i = 0; i < word.length; i++) {
    const sym = word[i]
    const rules = productions.get(sym.type)
    if (!rules || rules.length === 0) { out.push(sym); continue }
    const left = word[i - 1] || null
    const right = word[i + 1] || null
    const candidates = rules.filter(r => (r.cond ? r.cond(sym.params, left, right) : true))
    if (candidates.length === 0) { out.push(sym); continue }
    let picked = candidates[0]
    if (candidates.some(r => r.prob != null)) {
      let total = 0; for (const c of candidates) total += (c.prob ?? 0)
      const u = rand(); let acc = 0
      for (const c of candidates) { acc += (c.prob ?? 0) / (total || 1); if (u <= acc) { picked = c; break } }
    }
    const succ = picked.succ(sym.params, rand, i, word)
    for (const s of succ) out.push(s)
  }
  return out
}

/** Run with table switching (environmental stages) */
function deriveWithTables (axiom, tables, schedule, steps, rand) {
  let word = axiom.slice()
  const tableByName = new Map(tables.map(t => [t.name, t.productions]))
  let stageIndex = 0; let stageStep = 0; let current = tableByName.get(schedule[0].name)
  for (let n = 0; n < steps; n++) {
    if (!current) throw new Error(`Unknown table ${schedule[stageIndex].name}`)
    word = deriveOnce(word, current, rand)
    stageStep++
    if (stageStep >= (schedule[stageIndex].steps ?? 1) && stageIndex < schedule.length - 1) {
      stageIndex++; stageStep = 0; current = tableByName.get(schedule[stageIndex].name)
    }
  }
  return word
}

// ---------------------------- Tiny DSL parser for productions ----------------------------
/**
 * Grammar (tiny, whitespace-insensitive):
 *   TABLE <name>:
 *   <Head> [ : <prob> ] -> <RHS>
 *   <Head> := SYM | SYM ( id[, id]* )
 *   <RHS>  := token{space} ...
 *   token  := SYM | SYM(expr[,expr]*) | [ | ] | | | +[(expr)] | -[(expr)] | &[(expr)] | ^[(expr)] | \\[(expr)] | /[(expr)] | ![(expr)]
 * - Functions available in expr: Math.* and rand()
 * - Variables in expr are exactly the Head parameter names.
 */
function parseDSL (text) {
  const lines = text.split(/\n+/)
  /** @type {{name:string, productions: Map<string, any[]>}[]} */
  const tables = []
  let current = null

  function ensureTable (name) {
    current = { name, productions: new Map() }
    tables.push(current)
  }

  for (let raw of lines) {
    let line = raw.trim()
    if (!line || line.startsWith('#') || line.startsWith('//')) continue
    const mTable = line.match(/^TABLE\s+([A-Za-z0-9_\-]+)\s*:\s*$/i)
    if (mTable) { ensureTable(mTable[1]); continue }
    if (!current) throw new Error('No TABLE defined before rules')

    const m = line.match(/^([A-Za-z]+)\s*(?:\(([^)]*)\))?\s*(?::\s*([0-9]*\.?[0-9]+))?\s*->\s*(.+)$/)
    if (!m) throw new Error('Bad rule: ' + line)
    const headSym = m[1]
    const headParams = (m[2] ? m[2].split(',').map(s => s.trim()).filter(Boolean) : [])
    const prob = m[3] != null ? parseFloat(m[3]) : null
    const rhs = m[4]

    const tokens = tokenizeRHS(rhs)
    const builders = tokens.map(tok => compileToken(tok, headParams))
    const rule = {
      prob,
      succ: (params, rand) => {
        const env = Object.fromEntries(headParams.map((k, i) => [k, params[i]]))
        const out = []
        for (const b of builders) {
          const part = b(env, rand)
          if (Array.isArray(part)) out.push(...part)
          else if (part) out.push(part)
        }
        return out
      }
    }
    if (!current.productions.has(headSym)) current.productions.set(headSym, [])
    current.productions.get(headSym).push(rule)
  }
  return tables
}

function tokenizeRHS (rhs) {
  const out = []
  let buf = ''
  let depth = 0
  for (let i = 0; i < rhs.length; i++) {
    const ch = rhs[i]
    if (ch === '[' || ch === ']') {
      if (depth === 0) { if (buf.trim()) { out.push(buf.trim()); buf = '' } out.push(ch); continue }
    }
    if (ch === '(') depth++
    if (ch === ')') depth--
    if (/(\s)/.test(ch) && depth === 0) { if (buf.trim()) { out.push(buf.trim()); buf = '' } }
    else buf += ch
  }
  if (buf.trim()) out.push(buf.trim())
  return out
}

function compileToken (token, headParams) {
  if (token === '[' || token === ']' || token === '|') return () => new Sym(token)
  const m = token.match(/^(.+?)(?:\((.*)\))?$/)
  if (!m) throw new Error('Bad token: ' + token)
  const name = m[1]
  const args = (m[2] ?? '').trim()

  if (!args) {
    return () => new Sym(name)
  }
  // Compile expression list into a function of head params + rand
  const keys = [...headParams, 'rand']
  const body = `return [${args}]`
  const fn = new Function(...keys, body)
  return (env, rand) => {
    const values = headParams.map(k => env[k])
    const arr = fn(...values, rand)
    return new Sym(name, arr)
  }
}

// ---------------------------- 3D Turtle Interpreter ----------------------------
function buildPlant (word, opts = {}) {
  const angle = opts.angle ?? 25
  const step = opts.step ?? 1
  const radialSegments = opts.radialSegments ?? 8
  const initialWidth = opts.width ?? 0.12 // diameter

  const branchGeoms = []
  const leafTransforms = []

  const H0 = new THREE.Vector3(0, 1, 0)
  const L0 = new THREE.Vector3(-1, 0, 0)
  const U0 = new THREE.Vector3(0, 0, 1)
  const state = { pos: new THREE.Vector3(0, 0, 0), H: H0.clone(), L: L0.clone(), U: U0.clone(), w: initialWidth }
  const stack = []

  const toRad = d => d * Math.PI / 180
  const rotAround = (axis, deg) => {
    const q = new THREE.Quaternion().setFromAxisAngle(axis, toRad(deg))
    state.H.applyQuaternion(q).normalize()
    state.L.applyQuaternion(q).normalize()
    state.U.applyQuaternion(q).normalize()
  }

  const makeCylinder = (p0, p1, diameter) => {
    const dir = new THREE.Vector3().subVectors(p1, p0)
    const len = dir.length(); if (len <= 1e-6) return null
    const mid = new THREE.Vector3().addVectors(p0, p1).multiplyScalar(0.5)
    const geom = new THREE.CylinderGeometry(diameter * 0.5, diameter * 0.5, len, radialSegments)
    const y = new THREE.Vector3(0, 1, 0)
    const q = new THREE.Quaternion().setFromUnitVectors(y, dir.clone().normalize())
    geom.applyQuaternion(q)
    geom.translate(mid.x, mid.y, mid.z)
    return geom
  }

  for (let i = 0; i < word.length; i++) {
    const s = word[i]; const t = s.type; const p = s.params
    if (t === 'F' || t === 'f') {
      const len = p[0] ?? step
      const p0 = state.pos.clone()
      const p1 = state.pos.clone().addScaledVector(state.H, len)
      if (t === 'F') {
        const g = makeCylinder(p0, p1, state.w); if (g) branchGeoms.push(g)
      }
      state.pos.copy(p1)
    } else if (t === '+') rotAround(state.U, +(p[0] ?? angle))
    else if (t === '-') rotAround(state.U, -(p[0] ?? angle))
    else if (t === '&') rotAround(state.L, +(p[0] ?? angle))
    else if (t === '^') rotAround(state.L, -(p[0] ?? angle))
    else if (t === '\\') rotAround(state.H, +(p[0] ?? angle))
    else if (t === '/') rotAround(state.H, -(p[0] ?? angle))
    else if (t === '|') rotAround(state.U, 180)
    else if (t === '[') stack.push({ pos: state.pos.clone(), H: state.H.clone(), L: state.L.clone(), U: state.U.clone(), w: state.w })
    else if (t === ']') { const popped = stack.pop(); if (popped) Object.assign(state, popped) }
    else if (t === '!') { if (p.length) state.w = p[0]; else state.w *= 0.70710678 }
    else if (t === 'L') {
      const size = p[0] ?? (opts.leaf?.size ?? 0.35)
      // Face leaves outward around the stem (normal = turtle's left vector)
      const normal = state.L.clone().normalize()
      const up = state.U.clone().normalize()
      const right = new THREE.Vector3().crossVectors(up, normal).normalize()
      const m = new THREE.Matrix4().makeBasis(right, up, normal)
      m.setPosition(state.pos)
      m.multiply(new THREE.Matrix4().makeScale(size, size, size))
      leafTransforms.push(m)
    }
  }

  const branchGeometry = branchGeoms.length ? mergeGeometries(branchGeoms, false) : new THREE.BufferGeometry()
  const branchMaterial = new THREE.MeshStandardMaterial({ metalness: 0, roughness: 0.9, color: new THREE.Color(0x7a5c3f) })
  const branchMesh = new THREE.Mesh(branchGeometry, branchMaterial)

  const leafGeom = opts.leaf?.geometry ?? new THREE.PlaneGeometry(1, 0.5, 1, 1)
  const leafMat = opts.leaf?.material ?? new THREE.MeshStandardMaterial({ color: new THREE.Color(0x2e8b57), side: THREE.DoubleSide })
  const leafCount = leafTransforms.length
  const leavesMesh = new THREE.InstancedMesh(leafGeom, leafMat, Math.max(leafCount, 1))
  leavesMesh.count = leafCount
  for (let i = 0; i < leafCount; i++) leavesMesh.setMatrixAt(i, leafTransforms[i])
  leavesMesh.instanceMatrix.needsUpdate = true

  const group = new THREE.Group()
  group.add(branchMesh)
  group.add(leavesMesh)
  return group
}

// ---------------------------- Vue state & Three scene ----------------------------
const mount = ref(null)
const seed = ref(3)
const steps = ref(12)
const schedule = ref([
  { name: 'veg', steps: 8 },
  { name: 'flower', steps: 4 }
])

const scheduleLabel = computed(() => schedule.value.map(s => `${s.name}×${s.steps}`).join(' → '))

// UI tabs + preset handling
const tab = ref('dsl')
const preset = ref('monopodial')
const tabBtn = (n) => `px-2 py-1 rounded text-sm ${tab.value===n? 'bg-slate-900 text-white':'bg-slate-100 text-slate-700'}`

// ---- DSL presets (modular) ----
function dslMonopodial () { return `
TABLE veg:
A(l,w) : 0.38 -> !(w) F(l) [+(25+10*(rand()-0.5)) &(20) !(w*0.7) B(l*0.8,w*0.7)] [-(25+10*(rand()-0.5)) &(20) !(w*0.7) C(l*0.8,w*0.7)] /(137.5) A(l*0.92,w*0.92)
A(l,w) : 0.62 -> !(w) F(l) ! /(137.5) A(l*0.95,w*0.96)
B(l,w) : 0.50 -> !(w) F(l) [+(15+10*(rand()-0.5)) !(w*0.75) B(l*0.85,w*0.85)] /(137.5) B(l*0.9,w*0.92)
B(l,w) : 0.50 -> !(w) F(l) /(137.5) B(l*0.9,w*0.92)
C(l,w) : 0.50 -> !(w) F(l) [-(15+10*(rand()-0.5)) !(w*0.75) C(l*0.85,w*0.85)] /(137.5) C(l*0.9,w*0.92)
C(l,w) : 0.50 -> !(w) F(l) /(137.5) C(l*0.9,w*0.92)

TABLE flower:
A(l,w) -> !(w) F(l*0.7) [&(30) L(0.7)] [^(30) L(0.7)]
B(l,w) -> !(w*0.8) F(l*0.6) [&(20) L(0.55)]
C(l,w) -> !(w*0.8) F(l*0.6) [^(20) L(0.55)]
` }

// Sympodial: main apex only emits trunk segment + two lateral apices; laterals continue similarly
function dslSympodial () { return `
TABLE veg:
# p1: main apex makes trunk then a symmetric pair of lateral apices
A(l,w) -> !(w) F(l) [+(25) &(20) !(w*0.75) B(l*0.85,w*0.75)] [-(25) &(20) !(w*0.75) B(l*0.85,w*0.75)]
# p2: each lateral continues bifurcating (no axial continuation by A)
B(l,w) : 0.55 -> !(w) F(l) [+(22) !(w*0.8) B(l*0.86,w*0.86)] [-(22) !(w*0.8) B(l*0.86,w*0.86)] /(137.5)
B(l,w) : 0.45 -> !(w) F(l*0.9) /(137.5) B(l*0.9,w*0.9)

TABLE flower:
# finish with small leaf clusters at tips (placeholder for inflorescences)
A(l,w) -> !(w) F(l*0.6) [&(25) L(0.6)] [^(25) L(0.6)]
B(l,w) -> !(w) F(l*0.6) [&(20) L(0.5)] [^(20) L(0.5)]
` }

const dsl = ref('')
function resetPreset(){ dsl.value = (preset.value === 'monopodial' ? dslMonopodial() : dslSympodial()) }
watch(preset, resetPreset, { immediate: true }); schedule.value.map(s => `${s.name}×${s.steps}`).join(' → ');

let renderer, scene, camera, controls;
let plantGroup = null

function rebuildScene (word) {
  if (!scene) return
  // Remove old plant
  if (plantGroup) {
    scene.remove(plantGroup)
    plantGroup.traverse(obj => {
      if (obj.isMesh) {
        obj.geometry?.dispose?.()
        if (Array.isArray(obj.material)) obj.material.forEach(m => m.dispose())
        else obj.material?.dispose?.()
      }
    })
    plantGroup = null
  }
  // Build & add new plant
  plantGroup = buildPlant(word, { angle: 25, step: 0.6, width: 0.12, radialSegments: 8, leaf: { size: 0.35 } })
  scene.add(plantGroup)
  // Fit camera to bbox
  const bbox = new THREE.Box3().setFromObject(plantGroup)
  const size = bbox.getSize(new THREE.Vector3())
  if (isFinite(size.y) && size.y > 0) {
    camera.position.set(size.y * 0.6, size.y * 0.8, size.y * 0.9)
    camera.lookAt(0, size.y * 0.4, 0)
    controls?.update?.()
  }
}

function deriveWord () {
  const rand = makeRng(seed.value)
  let tables
  try {
    tables = parseDSL(dsl.value)
  } catch (e) {
    console.error('DSL parse error:', e)
    // Fallback to a minimal table if parse fails
    tables = [{ name: 'veg', productions: new Map([['A', [{ succ: ([l,w]) => [new Sym('!', [w]), new Sym('F', [l]), new Sym('A', [l*0.9, w*0.95])]}]]]) }]
  }
  const axiom = [ new Sym('[', []), new Sym('A', [1.2, 0.13]), new Sym(']', []) ]
  const w = deriveWithTables(axiom, tables, schedule.value, steps.value, rand)
  return w
}

onMounted(() => {
  const el = mount.value
  // Ensure visible height
  if (!el.style.height) el.style.height = 'calc(100vh - 56px)'

  renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true })
  renderer.setPixelRatio(window.devicePixelRatio || 1)
  renderer.setSize(el.clientWidth, el.clientHeight)
  el.appendChild(renderer.domElement)

  scene = new THREE.Scene()
  scene.background = new THREE.Color(0xf5f7fa)

  camera = new THREE.PerspectiveCamera(45, el.clientWidth / el.clientHeight, 0.1, 1000)
  camera.position.set(6, 6, 10)
  camera.lookAt(0, 3, 0)

  controls = new OrbitControls(camera, renderer.domElement)

  const hemi = new THREE.HemisphereLight(0xffffff, 0x444444, 1.0)
  hemi.position.set(0, 1, 0); scene.add(hemi)
  const dir = new THREE.DirectionalLight(0xffffff, 0.9)
  dir.position.set(5, 10, 4); scene.add(dir)

  const ground = new THREE.Mesh(
    new THREE.CircleGeometry(6, 48),
    new THREE.MeshStandardMaterial({ color: 0xe8ecef, roughness: 1.0, metalness: 0.0 })
  )
  ground.rotateX(-Math.PI / 2); ground.position.y = -0.01; scene.add(ground)

  rebuildScene(deriveWord())

  let rafId
  const render = () => { rafId = requestAnimationFrame(render); controls.update(); renderer.render(scene, camera) }
  render()

  const handleResize = () => {
    renderer.setSize(el.clientWidth, el.clientHeight)
    camera.aspect = el.clientWidth / el.clientHeight
    camera.updateProjectionMatrix()
  }
  window.addEventListener('resize', handleResize)

  const stopW = watch([seed, steps, schedule, dsl], () => rebuildScene(deriveWord()))

  onBeforeUnmount(() => {
    stopW()
    cancelAnimationFrame(rafId)
    window.removeEventListener('resize', handleResize)
    el.removeChild(renderer.domElement)
    renderer.dispose()
  })
})
</script>

<style>
html, body, #app { width: 100%; height: 100%; margin: 0; }
.viewer { flex: 1 1 auto; min-height: 420px; height: calc(100vh - 0px); }
.btn { background:#0f172a; color:white; padding:0.25rem 0.6rem; border-radius:0.6rem; font-size:0.875rem }
.btn-alt { background:#e2e8f0; color:#0f172a; padding:0.25rem 0.6rem; border-radius:0.6rem; font-size:0.875rem }
.code { width:100%; height: 280px; font: 12px/1.4 ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono", "Courier New", monospace; border:1px solid #e2e8f0; border-radius:12px; padding:10px; background:white; }
</style>
