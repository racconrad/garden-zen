const {
  Plugin, ItemView, Keymap, Menu, Notice, debounce, getAllTags, PluginSettingTab, Setting, FuzzySuggestModal, setIcon,
} = require('obsidian');

const VIEW = 'garden-zen';
const SVGNS = 'http://www.w3.org/2000/svg';
const STAGES = ['seed', 'sprout', 'plant', 'shrub', 'tree'];
const STATES = { bloom: 'in bloom', healthy: 'growing', wilted: 'wilted' };
const DAY = 864e5;
// Hues blended into the moss so each zone's island has its own shade.
const ZONE_TINTS = ['#c9d86b', '#7fb8a4', '#d9b86b', '#a3c46b', '#6fa3b8', '#b8a36b', '#8fc27a', '#b0c4a0'];
const DIRS = [[1, 0], [0, 1], [-1, 0], [0, -1], [1, 1], [-1, -1], [1, -1], [-1, 1]];
const TREE_STYLES = ['round', 'pine', 'bonsai'];
const PRESETS = {
  zen: { name: 'Zen', sandColor: '#efe6cf', mossColor: '#6f9a4a', canopyColor: '#5f9445', blossomColor: '#f4b6c6' },
  crimson: { name: 'Crimson', sandColor: '#f1e7da', mossColor: '#7a8f4a', canopyColor: '#b02e3c', blossomColor: '#ffd27a' },
  autumn: { name: 'Autumn', sandColor: '#f1e3c8', mossColor: '#8d9a4c', canopyColor: '#cf6d3a', blossomColor: '#f2c14e' },
  jade: { name: 'Jade', sandColor: '#e7eee6', mossColor: '#5e9c7a', canopyColor: '#3f8a6a', blossomColor: '#f6c1a6' },
};
// Garden level from total growth; thresholds roughly double.
const LEVELS = [[0, 'Wasteland'], [8, 'First sprouts'], [25, 'Orchard'], [60, 'Garden'], [120, 'Lush garden'], [220, 'Forest'], [400, 'Sanctuary']];
const shortDate = t => (window.moment ? window.moment(t).format('D MMM YYYY') : new Date(t).toLocaleDateString(undefined, { day: 'numeric', month: 'short', year: 'numeric' }));
const applyPreset = (settings, id) => {
  const { name, ...colors } = PRESETS[id];
  Object.assign(settings, colors);
};
const GROUPS = { type: 'Type', folder: 'Folder', subfolder: 'Subfolder', tag: 'Tag', none: 'Single island' };
const PALETTE_NAMES = Object.fromEntries(Object.entries(PRESETS).map(([id, p]) => [id, p.name]));
const TREE_NAMES = { mixed: 'Mixed', round: 'Round', pine: 'Pines', bonsai: 'Bonsai' };
const DECOR_NAMES = { none: 'None', some: 'Some', lots: 'Lots' };
const LABEL_NAMES = { mature: 'Mature', always: 'All', hover: 'On hover' };

const DEFAULTS = {
  exclude: '',
  groupBy: 'type',
  growth: 1,
  bloomDays: 7,
  wiltDays: 90,
  grid: 22,
  palette: 'zen',
  treeStyle: 'mixed',
  islandSize: 1,
  rings: 3,
  islandGap: 1.5,
  moonGap: 2.5,
  roughness: 0.5,
  zoneNames: true,
  textColor: '#5a4636',
  decor: 'some',
  // motion
  swayAmp: 2,
  swaySpeed: 5,
  petals: true,
  clouds: true,
  fireflies: true,
  flicker: true,
  bounce: true,
  smoothCamera: true,
  tlDuration: 12,
  tlLoop: false,
  showLog: true,
  sandColor: '#efe6cf',
  mossColor: '#6f9a4a',
  canopyColor: '#5f9445',
  blossomColor: '#f4b6c6',
  // panel state
  panelOpen: false,
  showOrphans: true,
  showWilted: true,
  hiddenZones: [],
  labels: 'hover',
  plantScale: 1,
  sway: true,
};

const plain = s => String(s).normalize('NFD').replace(/[̀-ͯ]/g, '').toLowerCase().trim();
const esc = s => String(s).replace(/[&<>"]/g, c => ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;' })[c]);

// Growth = substance (words) + how rooted it is (backlinks weigh double).
function stage(words, inLinks, outLinks, growth = 1) {
  const score = (Math.min(words, 2000) / 200 + inLinks * 2 + outLinks) * growth;
  return score < 1 ? 0 : score < 4 ? 1 : score < 9 ? 2 : score < 16 ? 3 : 4;
}

// Frontmatter `stage: tree` (or 0-4) overrides the computed stage.
function parseStage(value) {
  if (value == null) return null;
  const n = Number(value);
  if (Number.isInteger(n) && n >= 0 && n < STAGES.length) return n;
  const i = STAGES.map(plain).indexOf(plain(value));
  return i < 0 ? null : i;
}

function health(mtime, now, bloomDays = 7, wiltDays = 90) {
  const days = (now - mtime) / DAY;
  return days <= bloomDays ? 'bloom' : days > wiltDays ? 'wilted' : 'healthy';
}

function hash(s) {
  let h = 2166136261;
  for (const c of s) h = Math.imul(h ^ c.charCodeAt(0), 16777619);
  return h >>> 0;
}

const fmt = a => a.map(([x, y]) => `${x.toFixed(1)},${y.toFixed(1)}`).join(' ');

// Low-poly blob: base tone, a shaded lower-right facet and a lit top facet.
function facet(cx, cy, rx, ry, seed, cls) {
  const pts = [];
  for (let i = 0; i < 8; i++) {
    const a = ((i - 0.5) / 8) * 2 * Math.PI, j = 0.86 + ((seed >>> (i * 3)) & 7) / 30;
    pts.push([cx + Math.cos(a) * rx * j, cy + Math.sin(a) * ry * j]);
  }
  const c = [cx, cy - ry * 0.1];
  return `<polygon class="${cls}" points="${fmt(pts)}"/>`
    + `<polygon class="${cls} shade" points="${fmt([c, pts[0], pts[1], pts[2], pts[3]])}"/>`
    + `<polygon class="${cls} lit" points="${fmt([c, pts[5], pts[6], pts[7]])}"/>`;
}

function rock(size, seed, mossy = false) {
  const j = n => 1 + (((seed >>> n) & 7) - 3.5) / 18;
  const p = [[-size, 0], [-size * 0.8 * j(0), -size * 0.9 * j(3)], [-size * 0.15, -size * 1.5 * j(6)], [size * 0.65 * j(9), -size * 1.1 * j(12)], [size, 0]];
  const mid = [0, -size * 0.4];
  return `<polygon class="rock" points="${fmt(p)}"/>`
    + `<polygon class="rock lit" points="${fmt([p[1], p[2], mid])}"/>`
    + `<polygon class="rock shade" points="${fmt([p[2], p[3], p[4], [size * 0.1, 0], mid])}"/>`
    + (mossy ? `<polygon class="rock-moss" points="${fmt([p[1], p[2], [p[2][0] + size * 0.3, p[2][1] + size * 0.35], [p[1][0] + size * 0.25, p[1][1] + size * 0.3]])}"/>` : '');
}

// Stone lantern (tōrō) with a warm light.
function lantern() {
  return '<ellipse class="ground-shadow" cx="1.5" cy="0.5" rx="5" ry="2.5"/>'
    + '<polygon class="stone" points="-3.5,0 3.5,0 3,-2 -3,-2"/><polygon class="stone shade" points="0,0 3.5,0 3,-2 0,-2"/>'
    + '<rect class="stone" x="-1.2" y="-8" width="2.4" height="6"/><rect class="stone shade" x="0" y="-8" width="1.2" height="6"/>'
    + '<polygon class="stone" points="-3,-8 3,-8 2.6,-9.5 -2.6,-9.5"/>'
    + '<rect class="lantern-light" x="-2" y="-13" width="4" height="3.5"/>'
    + '<polygon class="stone" points="-4.2,-13 4.2,-13 0,-16.5"/><polygon class="stone shade" points="0,-13 4.2,-13 0,-16.5"/>'
    + '<circle class="stone" cx="0" cy="-17.2" r="0.9"/>';
}

// Lone sand rocks and stone lanterns beside the islands; deterministic so the garden stays put.
function decorations(cells, beds, amount) {
  if (amount === 'none' || !cells.length) return [];
  const taken = new Set(cells.map(c => c.join()));
  const near = (c, r, d) => cells.some(([x, y]) => Math.max(Math.abs(x - c), Math.abs(y - r)) <= d);
  const cs = cells.map(c => c[0]), rs = cells.map(c => c[1]);
  const spots = [];
  for (let c = Math.min(...cs) - 3; c <= Math.max(...cs) + 3; c++) {
    for (let r = Math.min(...rs) - 3; r <= Math.max(...rs) + 3; r++) {
      if (!near(c, r, 2)) spots.push({ c, r, h: hash(`${c},${r}`) });
    }
  }
  // A few rocks, kept apart from each other, more for bigger gardens.
  const want = Math.max(2, Math.ceil(cells.length / (amount === 'lots' ? 2 : 6))), out = [];
  for (const spot of spots.sort((a, b) => a.h - b.h)) {
    if (out.length >= want) break;
    if (out.some(o => Math.max(Math.abs(o.c - spot.c), Math.abs(o.r - spot.r)) < 3)) continue;
    out.push({ type: 'rock', c: spot.c, r: spot.r, seed: hash(`roca${spot.c},${spot.r}`) /* seed salt kept so rock shapes don't change */ });
  }
  const zones = new Map();
  cells.forEach((cell, i) => {
    if (!zones.has(beds[i])) zones.set(beds[i], []);
    zones.get(beds[i]).push(cell);
  });
  for (const [bed, list] of zones) {
    const h = hash(bed);
    for (let k = 0; k < (amount === 'lots' ? 2 : 1) && k < list.length; k++) {
      const [c, r] = list[(h >>> (k * 5)) % list.length];
      const spot = [...DIRS.keys()].map(d => DIRS[(d + h + k * 3) % 8]).map(([dc, dr]) => [c + dc, r + dr]).find(cell => !taken.has(cell.join()));
      if (!spot) continue;
      taken.add(spot.join());
      out.push({ type: 'lantern', c: spot[0], r: spot[1], seed: h });
    }
  }
  return out;
}

// Canopy lobes, back to front; one more lobe per pair of relations.
const LOBES = [[-0.3, -0.55], [0.7, 0.3], [-0.7, 0.35], [0, 0]];

// One note = one element of the zen garden, standing on (0,0).
// Stage picks the element, relations add canopy lobes, recent edits blossom pink, forgotten notes turn to stone.
function gardenLevel(snaps, relations) {
  const score = snaps.reduce((t, v) => t + (v.level + 1) * (v.state === 'wilted' ? 0.4 : 1), 0) + relations * 0.5;
  let n = 0;
  while (n + 1 < LEVELS.length && score >= LEVELS[n + 1][0]) n++;
  const [from, name] = LEVELS[n], to = LEVELS[n + 1]?.[0];
  return { n: n + 1, name, score, progress: to ? (score - from) / (to - from) : 1 };
}

function pine(level, lobes, canopy) {
  const h = [0, 0, 12, 16, 26][level], w = [0, 0, 5.5, 7, 9][level], tiers = Math.min(4, 1 + lobes);
  const lift = level >= 3 ? 4 : 1;
  let out = level >= 3 ? `<rect class="trunk" x="-1" y="${-lift - 2}" width="2" height="${lift + 2}"/>` : '';
  for (let t = 0; t < tiers; t++) {
    const k = 1 - t * 0.2, y0 = -lift - (t * h) / (tiers + 1), y1 = y0 - h * 0.55 * k, hw = w * k;
    // jagged lower edge reads as needles
    out += `<polygon class="${canopy} lit" points="${fmt([[-hw, y0], [-hw * 0.5, y0 - 1], [0, y0 + 0.8], [0, y1]])}"/>`
      + `<polygon class="${canopy} shade" points="${fmt([[0, y0 + 0.8], [hw * 0.5, y0 - 1], [hw, y0], [0, y1]])}"/>`;
  }
  return out;
}

const PADS = [[-5, -17.5, 5.5], [4.5, -12.5, 4.5], [-0.5, -22, 5], [6, -19, 4]];
function bonsai(level, lobes, seed, canopy) {
  const scale = [0, 0, 0.55, 0.75, 1][level];
  const trunk = '<path class="trunk" d="M-1.5 0C-1 -4 3 -6 1 -10C-1 -13 -4 -14 -5 -16.5L-3.6 -17.2C-2 -14.6 1 -13 2.6 -10C4.6 -6 1 -4 1.6 0Z"/>';
  const pads = PADS.slice(0, Math.max(1, lobes)).map(([x, y, r], n) => facet(x, y, r, r * 0.42, seed >>> n, canopy)).join('');
  return `<g transform="scale(${scale})">${trunk}${pads}</g>`;
}

const DOTS = [[-0.45, -0.25], [0.4, -0.45], [0.1, 0.2], [-0.15, -0.8]];

function element(level, state, leaves, seed, style = 'round') {
  const shade = [3, 4, 6, 8, 11][level];
  let body = `<ellipse class="ground-shadow" cx="1.5" cy="0.5" rx="${shade}" ry="${shade / 2}"/>`;
  if (state === 'wilted') return body + rock(2.5 + level * 1.4, seed, leaves > 0);
  const bloom = state === 'bloom';
  if (level === 0) {
    body += '<path class="grass" d="M-3.4 0L-2.8 -4.6L-2 0ZM-1.2 0L0 -6.8L0.9 0ZM1.6 0L3.2 -4.2L3.5 0Z"/>'
      + '<path class="grass lit" d="M-2.2 0L-1.4 -5.6L-0.6 0ZM0.4 0L1.5 -5L2 0Z"/>';
  } else if (level === 1) {
    const stalks = [[-1.6, 13], [1.4, 10], [3.6, 7], [-3.8, 8]].slice(0, 2 + Math.min(2, leaves));
    body += stalks.map(([x, h]) => {
      const side = x < 0 ? -1 : 1;
      return `<rect class="bamboo" x="${x - 0.7}" y="${-h}" width="1.4" height="${h}"/>`
        + [0.35, 0.7].map(f => `<rect class="bamboo-node" x="${x - 0.8}" y="${(-h * f).toFixed(1)}" width="1.6" height="0.5"/>`).join('')
        + `<path class="bamboo-leaf" d="M${x} ${-h + 1}l${side * 4.5} -1.6l${-side * 4.5} 0.4Z"/>`
        + `<path class="bamboo-leaf lit" d="M${x} ${(-h * 0.6).toFixed(1)}l${-side * 3.8} -1.2l${side * 3.8} 0.5Z"/>`;
    }).join('');
  } else {
    const [rx, ry, lift] = [null, null, [6, 4.5, 4.5], [7.5, 5.5, 6], [8.5, 6, 20]][level];
    const lobes = Math.min(level === 2 ? 2 : 4, 1 + Math.floor(leaves / 2));
    const canopy = bloom ? 'canopy sakura' : 'canopy';
    if (style === 'pine') return body + pine(level, lobes, canopy);
    if (style === 'bonsai') return body + bonsai(level, lobes, seed, canopy);
    if (level === 4) body += '<path class="trunk" d="M-1.6 0C-1 -6 1.2 -10 -0.6 -16L1 -16.4C3 -10 1.2 -6 1.6 0ZM0.2 -12L5 -17L5.6 -16.4L1 -11ZM-0.4 -13.5L-4.6 -17.5L-5 -16.8L-0.8 -12.6Z"/>';
    LOBES.slice(LOBES.length - lobes).forEach(([dx, dy], n, all) => {
      const main = n === all.length - 1;
      body += facet(dx * rx, -lift + dy * ry * 1.4, rx * (main ? 1 : 0.75), ry * (main ? 1 : 0.75), seed >>> n, canopy);
    });
    // Blossoms on recent notes, small fruit on well-connected ones.
    const dots = bloom ? 4 : Math.min(3, Math.floor(leaves / 3));
    body += DOTS.slice(0, dots).map(([dx, dy]) => `<circle class="${bloom ? 'petal' : 'fruit'}" cx="${(dx * rx).toFixed(1)}" cy="${(-lift + dy * ry).toFixed(1)}" r="${bloom ? 1.1 : 0.8}"/>`).join('');
  }
  if (bloom && level < 2) body += '<circle class="petal" cx="-2.6" cy="-6" r="1.1"/><circle class="petal" cx="2.4" cy="-4.8" r="1"/>';
  return body;
}

// Isometric projection of the garden plane; the ground group uses the same matrix(1 .5 -1 .5 0 0).
const iso = (u, v) => [u - v, (u + v) / 2];

// Each bed is laid out on its own (linked notes pull together, then snap to free grid cells),
// then the islands are packed around the center, biggest first, with open sand between them.
// shortcut: O(n²) collision pass per bed, fine for a few hundred notes; use a spatial grid if the vault grows past that.
function layout(plants, edges, gap = 1, shape = {}) {
  const cells = [];
  const groups = new Map();
  plants.forEach((p, i) => { if (!groups.has(p.bed)) groups.set(p.bed, []); groups.get(p.bed).push(i); });
  const islands = [...groups].map(([bed, ids]) => {
    const local = new Map(ids.map((g, l) => [g, l]));
    const own = edges.filter(([a, b]) => local.has(a) && local.has(b)).map(([a, b]) => [local.get(a), local.get(b)]);
    const c = cluster(ids.map(i => plants[i]), own, shape);
    const r = Math.max(0, ...c.map(([q, v]) => Math.hypot(q, v)));
    return { bed, ids, c, r };
  }).sort((x, y) => y.ids.length - x.ids.length || x.bed.localeCompare(y.bed));

  // Spacing 0 lets the shores nearly touch; each step widens the sand channel.
  const margin = 2 + 4 * gap;
  const placed = [];
  for (const isl of islands) {
    let at = [0, 0];
    if (placed.length) {
      // Golden-angle spiral: the first free spot is the closest one, so islands fill the map evenly.
      for (let k = 1; ; k++) {
        const d = Math.sqrt(k) * 0.8, a = k * 2.39996;
        const x = Math.cos(a) * d, y = Math.sin(a) * d;
        if (placed.every(o => Math.hypot(x - o.at[0], y - o.at[1]) >= isl.r + o.r + margin)) { at = [x, y]; break; }
      }
    }
    isl.at = [Math.round(at[0]), Math.round(at[1])];
    placed.push(isl);
    isl.ids.forEach((g, l) => { cells[g] = [isl.c[l][0] + isl.at[0], isl.c[l][1] + isl.at[1]]; });
  }
  return cells;
}

// One island as a planet: the parent note sits in the middle, its children fill rings around it.
// Parent = note named like its folder (Opsys/Opsys.md), else the most linked; linked children sit closest.
function cluster(plants, edges, { moonGap = 2.5, roughness = 0.5 } = {}) {
  const n = plants.length;
  const degree = Array(n).fill(0), nb = plants.map(() => new Set());
  edges.forEach(([a, b]) => { degree[a]++; degree[b]++; nb[a].add(b); nb[b].add(a); });
  const named = plants.findIndex(p => !p.moon && p.file.basename === p.bed);
  const hub = named >= 0 ? named : degree.indexOf(Math.max(0, ...degree));
  const order = [...Array(n).keys()].filter(i => i !== hub && !plants[i].moon).sort((a, b) =>
    nb[hub].has(b) - nb[hub].has(a) || degree[b] - degree[a] || plants[a].file.path.localeCompare(plants[b].file.path));
  if (hub >= 0 && !plants[hub].moon) order.unshift(hub);
  const moons = [...Array(n).keys()].filter(i => plants[i].moon).sort((a, b) => plants[b].mtime - plants[a].mtime);
  // Each planet gets its own lumpy outline, and grown planets leave clearings so they spread wider.
  const seed = hash(plants[0]?.bed ?? ''), rnd = k => ((seed >>> k) & 255) / 255;
  // Roughness scales the lobes (capped so the outline never folds) and adds ragged per-cell noise.
  const amp = Math.min(roughness * 2, 1.6);
  const lump = [(0.08 + rnd(0) * 0.2) * amp, rnd(8) * 6.3, (0.05 + rnd(16) * 0.15) * amp, rnd(24) * 6.3];
  const warp = ([q, r]) => Math.hypot(q, r) / (1 + lump[0] * Math.sin(2 * Math.atan2(r, q) + lump[1]) + lump[2] * Math.sin(3 * Math.atan2(r, q) + lump[3]))
    + roughness * 1.5 * ((hash(`${seed}~${q},${r}`) & 255) / 255 - 0.5);
  const core = order.map(i => plants[i]);
  const maturity = core.length ? core.reduce((t, p) => t + (p.forced ?? stage(p.words, p.ins?.size ?? 0, p.outs?.size ?? 0)), 0) / core.length / (STAGES.length - 1) : 0;
  const clearing = 0.05 + 0.35 * maturity;
  const R = Math.ceil(Math.sqrt(n / Math.PI) * 2 + moonGap) + 5, spots = [];
  for (let q = -R; q <= R; q++) for (let r = -R; r <= R; r++) spots.push([q, r]);
  spots.sort((x, y) => warp(x) - warp(y) || Math.atan2(x[1], x[0]) - Math.atan2(y[1], y[0]));
  const skip = (c, p) => (hash(`${seed}:${c}`) & 1023) / 1023 < p;
  const free = spots.filter((c, k) => k === 0 || !skip(c, clearing));
  const cells = [];
  order.forEach((i, k) => { cells[i] = free[k]; });
  // Moons scatter over an uneven belt, at least moonGap cells from any planet note.
  const edge = order.length ? Math.max(...order.map(i => warp(cells[i]))) : -2;
  const near = c => order.some(i => Math.hypot(c[0] - cells[i][0], c[1] - cells[i][1]) < moonGap);
  const belt = spots.filter(c => warp(c) >= edge + moonGap - 1 && !skip(`m${c}`, 0.45) && !near(c));
  moons.forEach((i, k) => { cells[i] = belt[k]; });
  return cells;
}

const describe = (p, v) => `${p.file.basename} · ${STAGES[v.level]}${v.state === 'healthy' ? '' : `, ${STATES[v.state]}`} · ${v.leaves} links`;

class NoteModal extends FuzzySuggestModal {
  constructor(app, items, onPick) {
    super(app);
    this.items = items;
    this.onPick = onPick;
    this.setPlaceholder('Go to a plant in the garden…');
  }
  getItems() { return this.items; }
  getItemText(item) { return item.p.file.path.replace(/\.md$/, ''); }
  onChooseItem(item) { this.onPick(item.i); }
}

class GardenView extends ItemView {
  constructor(leaf, plugin) {
    super(leaf);
    this.plugin = plugin;
    this.run = 0;
    this.data = null;
    this.vb = null; // kept across refreshes so editing a note doesn't reset the camera
    this.search = '';
    this.progress = null; // null = the garden today; 0..1 = growth animation position
    this.frame = null;
    this.speed = 1;
    this.current = null;
    this.plantEls = new Map();
    this.labelEls = new Map();
    this.vineEls = [];
  }
  getViewType() { return VIEW; }
  getDisplayText() { return 'Garden Zen'; }
  getIcon() { return 'sprout'; }
  get settings() { return this.plugin.settings; }

  async onOpen() {
    const el = this.contentEl;
    el.empty();
    el.addClass('mind-garden');
    el.tabIndex = -1; // keyboard shortcuts need focus inside the view
    this.box = el.createDiv('mg-map');
    this.svg = document.createElementNS(SVGNS, 'svg');
    this.svg.setAttribute('class', 'mg-svg mg-canvas');
    this.box.appendChild(this.svg);
    this.info = this.box.createDiv('mg-info');
    this.petals = this.box.createDiv('mg-petals');
    this.clouds = this.box.createDiv('mg-clouds');
    for (let n = 0; n < 3; n++) this.clouds.createSpan('mg-cloud');
    this.card = this.box.createDiv('mg-card');
    this.buildControls();
    const gear = this.box.createDiv({ cls: 'mg-panel-button clickable-icon', attr: { 'aria-label': 'Filters and options' } });
    setIcon(gear, 'sliders-horizontal');
    this.panel = this.box.createDiv('mg-panel');
    gear.addEventListener('click', () => {
      this.settings.panelOpen = !this.settings.panelOpen;
      this.panel.toggleClass('is-hidden', !this.settings.panelOpen);
      this.plugin.saveSettings(false);
    });
    this.buildPanel();
    this.wire();
    this.caption();
    this.registerDomEvent(el, 'keydown', e => this.onKey(e));
    await this.load();
  }

  async onClose() { this.stop(); }

  async load() {
    const run = ++this.run;
    try {
      const data = await this.plugin.collect();
      if (run !== this.run) return; // a newer load started meanwhile
      this.data = data;
      this.dirty = true;
      this.beds = [...new Set(data.plants.map(p => p.bed))].sort();
      this.buildZones();
      if (!this.frame) this.progress = null; // a paused growth never outlives a refresh
      this.draw();
    } catch (error) {
      console.error('Garden Zen:', error);
      this.info.setText(`Could not draw the garden: ${error?.message ?? error}`);
    }
  }

  tint(bed) {
    return ZONE_TINTS[this.beds.indexOf(bed) % ZONE_TINTS.length];
  }

  buildPanel() {
    const s = this.settings, panel = this.panel;
    const save = () => this.plugin.saveSettings(false);
    const redraw = () => { save(); this.draw(); };
    const relayout = () => { save(); this.dirty = true; this.draw(); };
    const row = name => new Setting(panel).setName(name);
    const section = name => panel.createDiv({ cls: 'mg-section', text: name });
    panel.empty();
    panel.toggleClass('is-hidden', !s.panelOpen);

    new Setting(panel).addSearch(c => {
      this.searchInput = c.inputEl;
      c.setPlaceholder('Search').setValue(this.search).onChange(v => { this.search = v; this.draw(); });
    });
    row('Unlinked notes').addToggle(t => t.setValue(s.showOrphans).onChange(v => { s.showOrphans = v; redraw(); }));
    row('Stones').addToggle(t => t.setValue(s.showWilted).onChange(v => { s.showWilted = v; redraw(); }));
    this.zonesEl = panel.createDiv('mg-zones');
    if (this.beds) this.buildZones();

    section('Islands');
    row('Group by').addDropdown(d => d.addOptions(GROUPS).setValue(s.groupBy)
      .onChange(v => { s.groupBy = v; save(); this.vb = null; this.load(); }));
    row('Size').addSlider(c => c.setLimits(0.7, 1.4, 0.05).setValue(s.islandSize).setDynamicTooltip().onChange(v => { s.islandSize = v; redraw(); }));
    row('Spacing').addSlider(c => c.setLimits(0, 3, 0.1).setValue(s.islandGap).setDynamicTooltip().onChange(v => { s.islandGap = v; relayout(); }));
    row('Moon orbit').addSlider(c => c.setLimits(1, 6, 0.5).setValue(s.moonGap).setDynamicTooltip().onChange(v => { s.moonGap = v; relayout(); }));
    row('Roughness').addSlider(c => c.setLimits(0, 1, 0.05).setValue(s.roughness).setDynamicTooltip().onChange(v => { s.roughness = v; relayout(); }));
    row('Furrows').addSlider(c => c.setLimits(0, 6, 1).setValue(s.rings).setDynamicTooltip().onChange(v => { s.rings = v; redraw(); }));
    row('Names').addToggle(t => t.setValue(s.zoneNames).onChange(v => { s.zoneNames = v; redraw(); }));

    section('Appearance');
    row('Palette').addDropdown(d => d.addOptions(PALETTE_NAMES).setValue(s.palette)
      .onChange(v => { s.palette = v; applyPreset(s, v); redraw(); }));
    row('Trees').addDropdown(d => d.addOptions(TREE_NAMES).setValue(s.treeStyle).onChange(v => { s.treeStyle = v; redraw(); }));
    row('Decoration').addDropdown(d => d.addOptions(DECOR_NAMES).setValue(s.decor).onChange(v => { s.decor = v; relayout(); }));
    row('Labels').addDropdown(d => d.addOptions(LABEL_NAMES).setValue(s.labels).onChange(v => { s.labels = v; redraw(); }));
    row('Plant size').addSlider(c => c.setLimits(0.6, 1.6, 0.1).setValue(s.plantScale).setDynamicTooltip().onChange(v => { s.plantScale = v; redraw(); }));

    section('Motion');
    row('Sway').addToggle(t => t.setValue(s.sway).onChange(v => { s.sway = v; redraw(); }));
    row('Strength').addSlider(c => c.setLimits(0.5, 6, 0.5).setValue(s.swayAmp).setDynamicTooltip().onChange(v => { s.swayAmp = v; redraw(); }));
    row('Pace').addSlider(c => c.setLimits(2, 10, 0.5).setValue(s.swaySpeed).setDynamicTooltip().onChange(v => { s.swaySpeed = v; redraw(); }));
    row('Drifting petals').addToggle(t => t.setValue(s.petals).onChange(v => { s.petals = v; redraw(); }));
    row('Clouds').addToggle(t => t.setValue(s.clouds).onChange(v => { s.clouds = v; redraw(); }));
    row('Fireflies').addToggle(t => t.setValue(s.fireflies).onChange(v => { s.fireflies = v; redraw(); }));
    row('Flickering lanterns').addToggle(t => t.setValue(s.flicker).onChange(v => { s.flicker = v; redraw(); }));
    row('Hop on hover').addToggle(t => t.setValue(s.bounce).onChange(v => { s.bounce = v; redraw(); }));
    row('Smooth camera').addToggle(t => t.setValue(s.smoothCamera).onChange(v => { s.smoothCamera = v; save(); }));

    section('Growth');
    row('Duration (s)').addSlider(c => c.setLimits(5, 60, 1).setValue(s.tlDuration).setDynamicTooltip().onChange(v => { s.tlDuration = v; save(); }));
    row('Loop').addToggle(t => t.setValue(s.tlLoop).onChange(v => { s.tlLoop = v; save(); this.caption(); }));
    row('Change log').addToggle(t => t.setValue(s.showLog).onChange(v => { s.showLog = v; save(); this.caption(); this.renderLog(true); }));

    new Setting(panel)
      .addButton(b => b.setButtonText('Go to note').onClick(() => this.pick()))
      .addExtraButton(b => b.setIcon('maximize').setTooltip('Fit view (F)').onClick(() => this.fitView()))
      .addExtraButton(b => b.setIcon('settings').setTooltip('Settings').onClick(() => {
        this.app.setting?.open();
        this.app.setting?.openTabById(this.plugin.manifest.id);
      }));
  }

  buildZones() {
    const s = this.settings;
    this.zonesEl.empty();
    for (const bed of this.beds) {
      const chip = this.zonesEl.createDiv({ cls: 'mg-zone', attr: { 'aria-label': 'Show or hide zone' } });
      chip.toggleClass('off', s.hiddenZones.includes(bed));
      chip.createSpan('mg-swatch').style.background = `color-mix(in srgb, var(--mg-moss) 75%, ${this.tint(bed)})`;
      chip.appendText(bed);
      chip.addEventListener('click', () => {
        s.hiddenZones = s.hiddenZones.includes(bed) ? s.hiddenZones.filter(z => z !== bed) : [...s.hiddenZones, bed];
        chip.toggleClass('off', s.hiddenZones.includes(bed));
        this.plugin.saveSettings(false);
        this.draw();
      });
    }
  }

  // Counts and growth for note i. g is how far it has grown (1 = fully grown, as it is today).
  snapshot(i, exists, g, now) {
    const s = this.settings, p = this.data.plants[i];
    const count = set => { let n = 0; for (const k of set) if (exists(k)) n++; return n; };
    const inc = count(p.ins), out = count(p.outs), leaves = count(p.links);
    const full = p.forced ?? stage(p.words, inc, out, s.growth);
    // While growing, the note climbs through every stage and eases its size inside each one.
    const level = g >= 1 ? full : Math.min(full, Math.floor(g * (full + 1)));
    const t = Math.min(1, g * (full + 1) - level); // progress inside the current stage
    // Sprouts pop out of the ground with a small overshoot; later stages ease up from a slight dip.
    const scale = g >= 1 ? 1 : level === 0 ? 1 + 2.70158 * (t - 1) ** 3 + 1.70158 * (t - 1) ** 2 : 0.85 + 0.15 * (1 - (1 - t) ** 3);
    const state = g < 1 ? 'healthy' : health(p.mtime, now, s.bloomDays, s.wiltDays);
    return { level, state, leaves, words: p.words, inc, out, scale };
  }

  draw() {
    if (!this.data) return;
    const s = this.settings, { plants, edges } = this.data, now = Date.now();
    const q = this.search.trim().toLowerCase();
    const base = [];
    plants.forEach((p, i) => {
      if (s.hiddenZones.includes(p.bed)) return;
      if (q && !p.file.path.toLowerCase().includes(q)) return;
      if (!s.showOrphans && !p.links.size) return;
      if (!s.showWilted && health(p.mtime, now, s.bloomDays, s.wiltDays) === 'wilted') return;
      base.push(i);
    });

    // Layout covers the filtered set; the growth animation reuses it so plants appear in place.
    const key = `${base.join()}|${s.decor}`;
    if (this.dirty || key !== this.layoutKey) {
      if (key !== this.layoutKey) this.vb = null;
      this.layoutKey = key;
      this.dirty = false;
      this.sig = null;
      const local = new Map(base.map((g, l) => [g, l]));
      const localEdges = edges.filter(([a, b]) => local.has(a) && local.has(b)).map(([a, b]) => [local.get(a), local.get(b)]);
      const cells = layout(base.map(i => plants[i]), localEdges, s.islandGap, { moonGap: s.moonGap, roughness: s.roughness });
      this.cells = new Map(base.map((g, l) => [g, cells[l]]));
      this.decor = decorations(cells, base.map(i => plants[i].bed), s.decor);
      // Older notes sprout first; each note takes a quarter of the animation to grow.
      const ranked = [...base].sort((a, b) => plants[a].ctime - plants[b].ctime || plants[a].file.path.localeCompare(plants[b].file.path));
      this.appear = new Map(ranked.map((i, r) => [i, (r / Math.max(ranked.length, 1)) * 0.75]));
      this.changes = this.buildChanges(base, now);
    }

    const S = s.grid, prog = this.progress, prevSnap = this.snap;
    const exists = prog == null ? () => true : i => prog >= this.appear.get(i);
    const growth = i => (prog == null ? 1 : Math.min(1, (prog - this.appear.get(i)) / 0.25));
    const shown = base.filter(exists);
    const shownSet = new Set(shown);
    this.snap = new Map(shown.map(i => [i, this.snapshot(i, exists, growth(i), now)]));

    // Mid-animation, if no note appeared or changed stage, only resize the growing ones.
    const sig = shown.map(i => { const v = this.snap.get(i); return `${i}:${v.level}:${v.state}:${v.leaves}`; }).join();
    if (prog != null && sig === this.sig) {
      this.snap.forEach((v, i) => { const a = this.animEls.get(i); if (a) a.style.transform = `scale(${v.scale})`; });
      return;
    }
    this.sig = sig;

    const uv = ([c, r]) => [c * S, r * S];
    const px = cell => iso(...uv(cell));
    const depth = ([c, r]) => c + r;
    const zones = new Map();
    for (const i of shown) {
      if (!zones.has(plants[i].bed)) zones.set(plants[i].bed, []);
      zones.get(plants[i].bed).push(i);
    }
    const R0 = S * 0.92 * s.islandSize, GAP = S * 0.2, RR = S * 0.3;

    let scene = '', names = '';
    const allCells = [...base.map(i => this.cells.get(i)), ...this.decor.map(d => [d.c, d.r])];
    if (base.length) {
      const pad = S * 1.2 + R0 * 0.6 + s.rings * GAP, d = S * 1.2;
      const cs = allCells.map(c => c[0]), rs = allCells.map(c => c[1]);
      const u0 = Math.min(...cs) * S - pad, u1 = Math.max(...cs) * S + pad, v0 = Math.min(...rs) * S - pad, v1 = Math.max(...rs) * S + pad;
      const [T, R, B, L] = [iso(u0, v0), iso(u1, v0), iso(u1, v1), iso(u0, v1)];
      const down = ([x, y], h = d) => [x, y + h];
      const sides = `<polygon class="side-left" points="${fmt([L, B, down(B), down(L)])}"/>`
        + `<polygon class="side-right" points="${fmt([B, R, down(R), down(B)])}"/>`
        + `<polygon class="lip" points="${fmt([L, B, R, down(R, 2.5), down(B, 2.5), down(L, 2.5)])}"/>`;

      // Raked sand: each ring level paints a furrow-colored union, then a slightly smaller sand union,
      // leaving only the union's outline, so the rings flow around islands and rocks like in a karesansui.
      const pts = shown.filter(i => !plants[i].moon).map(i => uv(this.cells.get(i)));
      const rocks = this.decor.filter(dd => dd.type === 'rock').map(dd => uv([dd.c, dd.r]));
      const circles = (list, r, cls, off = 0) => list.map(([u, v]) => `<circle class="${cls}" cx="${(u + off).toFixed(1)}" cy="${(v + off).toFixed(1)}" r="${r.toFixed(1)}"/>`).join('');
      let ground = `<rect class="raked-sand" x="${u0}" y="${v0}" width="${u1 - u0}" height="${v1 - v0}"/>`;
      for (let l = s.rings; l >= 1; l--) {
        ground += circles(pts, R0 + l * GAP, 'furrow') + circles(rocks, RR + l * GAP, 'furrow')
          + circles(pts, R0 + l * GAP - 0.9, 'sand') + circles(rocks, RR + l * GAP - 0.9, 'sand');
      }
      ground += circles(pts, R0, 'island-shadow', S * 0.14);
      for (const [bed, list] of zones) {
        const specks = list.filter(i => !plants[i].moon).map(i => {
          const h = hash(plants[i].file.path), [u, v] = uv(this.cells.get(i));
          return [0, 8].map(b => `<circle class="speck" cx="${(u + (((h >>> b) & 15) / 15 - 0.5) * R0).toFixed(1)}" cy="${(v + (((h >>> (b + 4)) & 15) / 15 - 0.5) * R0).toFixed(1)}" r="0.7"/>`).join('');
        }).join('');
        const zp = list.filter(i => !plants[i].moon).map(i => uv(this.cells.get(i)));
        ground += `<g style="--zone:${this.tint(bed)}">${circles(zp, R0, 'moss-edge')}${circles(zp, R0 - 1.2, 'moss')}${specks}</g>`;
        if (s.fireflies) {
          for (const [n, i] of list.slice(0, 2).entries()) {
            const h = hash(`${plants[i].file.path}${n}`), [x, y] = px(this.cells.get(i));
            names += `<circle class="firefly" cx="${(x + ((h & 31) - 16) * S * 0.05).toFixed(1)}" cy="${(y - S * 0.6 - ((h >>> 5) & 15) * S * 0.04).toFixed(1)}" r="1.3" `
              + `style="animation-delay:-${(h % 60) / 10}s,-${(h % 23) / 10}s"/>`;
          }
        }
        if (s.zoneNames && zones.size > 1) {
          const sp = list.map(i => px(this.cells.get(i))), x = sp.reduce((a, q2) => a + q2[0], 0) / sp.length;
          const y = Math.max(...sp.map(q2 => q2[1])) + R0 * 0.55 + s.rings * GAP * 0.5 + S * 0.45;
          names += `<text class="mg-island-name${this.coveredZones?.has(bed) ? ' covered' : ''}" data-zone="${esc(bed)}" data-n="${list.length}" x="${x.toFixed(1)}" y="${y.toFixed(1)}">${esc(bed)}</text>`;
        }
      }
      // Relations stay hidden until hover: stepping stones from the linking note to the linked one.
      const stones = edges.filter(([a, b]) => shownSet.has(a) && shownSet.has(b)).map(([a, b, mutual]) => {
        const [x1, y1] = uv(this.cells.get(a)), [x2, y2] = uv(this.cells.get(b));
        return `<line class="mg-link${mutual ? ' mutual' : ''}" data-a="${a}" data-b="${b}" x1="${x1}" y1="${y1}" x2="${x2}" y2="${y2}"/>`;
      }).join('');
      scene = '<defs><pattern id="mg-rake" width="5" height="5" patternUnits="userSpaceOnUse">'
        + '<rect class="sand" width="5" height="5"/><line class="furrow-fine" x1="0" y1="2.5" x2="5" y2="2.5"/></pattern></defs>'
        + `${sides}<g transform="matrix(1 0.5 -1 0.5 0 0)">${ground}${stones}</g>`;
      // Camera frames the notes themselves (the slab may run off the edges) so names stay readable.
      const np = base.map(i => px(this.cells.get(i))), xs = np.map(q2 => q2[0]), ys = np.map(q2 => q2[1]);
      this.fitBox = { x: Math.min(...xs) - S * 2.2, y: Math.min(...ys) - S * 2.6, w: Math.max(...xs) - Math.min(...xs) + S * 4.4, h: Math.max(...ys) - Math.min(...ys) + S * 4.4 };
    } else {
      this.fitBox = { x: -100, y: -100, w: 200, h: 200 };
    }

    const k = this.k = s.plantScale * (S / 22) * 1.9;
    let labels = '';
    const items = shown.map(i => {
      const p = plants[i], v = this.snap.get(i), seed = hash(p.file.path), cell = this.cells.get(i), [x, y] = px(cell);
      const style = p.style ?? (s.treeStyle === 'mixed' ? TREE_STYLES[seed % 3] : s.treeStyle);
      const sway = s.sway && v.state !== 'wilted' ? 'mg-sway' : '';
      const changed = prog != null && prevSnap?.has(i) && prevSnap.get(i).level !== v.level;
      const name = p.file.basename.length > 24 ? `${p.file.basename.slice(0, 22)}…` : p.file.basename;
      if (p.moon) {
        return [depth(cell), `<g class="mg-p mg-moon" data-i="${i}" tabindex="0" role="button" aria-label="${esc(describe(p, v))}" `
          + `transform="translate(${x.toFixed(1)} ${y.toFixed(1)}) scale(${(k * 0.55).toFixed(3)})">`
          + `<title>${esc(p.file.basename)}</title><rect class="mg-hit" x="-8" y="-8" width="16" height="12"/><ellipse class="ground-shadow" cx="1.5" cy="0.5" rx="5" ry="2.5"/>${rock(3, seed, false)}</g>`];
      }
      labels += `<text class="mg-label${this.shownLabels?.has(i) ? ' shown' : ''}" data-i="${i}" x="${x.toFixed(1)}" y="${(y + 9 * k).toFixed(1)}">${esc(name)}</text>`;
      return [depth(cell), `<g class="mg-p" data-i="${i}" tabindex="0" role="button" aria-label="${esc(describe(p, v))}" `
        + `transform="translate(${x.toFixed(1)} ${y.toFixed(1)}) scale(${k.toFixed(3)})">`
        + '<rect class="mg-hit" x="-12" y="-34" width="24" height="38"/>'
        + `<g class="mg-anim" data-i="${i}" style="transform:scale(${v.scale})"><g class="mg-pop${changed ? ' changed' : ''}">`
        + `<g class="${sway}" style="animation-delay:-${seed % 5000}ms">${element(v.level, v.state, v.leaves, seed, style)}</g></g></g></g>`];
    });
    for (const dd of this.decor) {
      const [x, y] = px([dd.c, dd.r]);
      items.push([depth([dd.c, dd.r]), `<g class="mg-deco" transform="translate(${x.toFixed(1)} ${y.toFixed(1)}) scale(${k.toFixed(3)})">`
        + `${dd.type === 'lantern' ? `<g transform="scale(0.8)">${lantern()}</g>` : `<ellipse class="ground-shadow" cx="1.5" cy="0.5" rx="5" ry="2.5"/>${rock(3 + (dd.seed % 3), dd.seed, dd.seed % 3 === 0)}`}</g>`]);
    }
    const flora = items.sort((a, b) => a[0] - b[0]).map(it => it[1]).join('');
    this.vb ??= this.fitted();

    for (const key of ['sand', 'moss', 'canopy', 'blossom', 'text']) this.contentEl.style.setProperty(`--mg-${key}`, s[`${key}Color`]);
    this.contentEl.style.setProperty('--mg-sway-amp', `${s.swayAmp}deg`);
    this.contentEl.style.setProperty('--mg-sway-dur', `${s.swaySpeed}s`);
    this.contentEl.toggleClass('no-flicker', !s.flicker);
    this.contentEl.toggleClass('no-bounce', !s.bounce);
    this.clouds.toggleClass('is-hidden', !s.clouds);
    // Parsed as XML instead of innerHTML; every note-derived string in the markup goes through esc().
    const doc = new DOMParser().parseFromString(`<svg xmlns="${SVGNS}">${scene}<g>${flora}</g><g>${names}${labels}</g></svg>`, 'image/svg+xml');
    this.svg.replaceChildren(...[...doc.documentElement.childNodes].map(n => document.importNode(n, true)));
    this.plantEls = new Map();
    this.svg.querySelectorAll('.mg-p').forEach(g => this.plantEls.set(+g.dataset.i, g));
    this.animEls = new Map();
    this.svg.querySelectorAll('.mg-anim').forEach(g => this.animEls.set(+g.dataset.i, g));
    this.vineEls = [...this.svg.querySelectorAll('.mg-link')];
    this.labelEls = new Map();
    this.svg.querySelectorAll('.mg-label').forEach(t => this.labelEls.set(+t.dataset.i, t));
    this.current = null;
    this.applyVb();
    this.info.setText(shown.length ? '' : 'Nothing to show with these filters');
    this.drawPetals();

    // Garden level card and growth caption.
    const snaps = [...this.snap.values()], wilted = snaps.filter(v => v.state === 'wilted').length;
    const visibleEdges = edges.filter(([a, b]) => shownSet.has(a) && shownSet.has(b)).length;
    const lvl = gardenLevel(snaps, visibleEdges);
    this.card.empty();
    this.card.createDiv({ cls: 'mg-level', text: `Level ${lvl.n} · ${lvl.name}` });
    this.card.createDiv('mg-bar').createDiv('mg-bar-fill').style.width = `${Math.round(lvl.progress * 100)}%`;
    this.card.createDiv({ cls: 'mg-vitality', text: `${shown.length} notes · ${shown.length ? Math.round((1 - wilted / shown.length) * 100) : 0}% alive` });
    this.growCount = `${shown.length} of ${base.length}`;
    this.caption();
  }

  // Floating playback bar: just "Watch it grow" at rest, full controls while the garden grows.
  buildControls() {
    const bar = this.ctrl = this.box.createDiv('mg-controls');
    const btn = (icon, label, fn, cls = '') => {
      const b = bar.createDiv({ cls: `mg-ctrl clickable-icon ${cls}`, attr: { 'aria-label': label } });
      setIcon(b, icon);
      b.addEventListener('click', fn);
      return b;
    };
    this.ctrlPlay = btn('play', 'Play or pause (T)', () => this.togglePlay(), 'mg-play');
    this.playBtn = { setIcon: icon => setIcon(this.ctrlPlay, icon) };
    bar.createSpan({ cls: 'mg-hint', text: 'Watch it grow' }).addEventListener('click', () => this.togglePlay());
    btn('rotate-ccw', 'From the start (R)', () => this.restart());
    this.ctrlLoop = btn('repeat', 'Loop', () => {
      this.settings.tlLoop = !this.settings.tlLoop;
      this.plugin.saveSettings(false);
      this.caption();
    });
    this.ctrlSpeed = bar.createDiv({ cls: 'mg-ctrl mg-speed', attr: { 'aria-label': 'Speed' } });
    this.ctrlSpeed.addEventListener('click', () => { this.speed = { 1: 2, 2: 0.5, 0.5: 1 }[this.speed]; this.caption(); });
    const range = bar.createEl('input', { cls: 'mg-progress', attr: { type: 'range', min: '0', max: '1000', value: '1000', 'aria-label': 'Position' } });
    range.addEventListener('input', () => { this.stop(); this.setProgress(+range.value / 1000); });
    this.timeSlider = { inputEl: range };
    this.ctrlText = bar.createDiv('mg-ctrl-text');
    this.ctrlLog = btn('scroll-text', 'Change log (L)', () => this.toggleLog());
    btn('x', 'Back to today (Esc)', () => { this.stop(); this.setProgress(null); });
    this.log = this.box.createDiv('mg-log');
  }

  caption() {
    if (!this.ctrl) return;
    const active = this.progress != null;
    this.ctrl.toggleClass('active', active);
    this.ctrlLoop.toggleClass('on', this.settings.tlLoop);
    this.ctrlLog.toggleClass('on', this.settings.showLog);
    this.ctrlSpeed.setText(`${this.speed}×`);
    this.ctrlText.setText(active ? this.growCount ?? '' : '');
    this.log.toggleClass('is-hidden', !active || !this.settings.showLog);
  }

  toggleLog() {
    this.settings.showLog = !this.settings.showLog;
    this.plugin.saveSettings(false);
    this.caption();
    this.renderLog(true);
  }

  // Every sprout (with its real creation date) and stage change of the growth animation.
  // Stage changes carry no date: Obsidian keeps no edit history, so any date there would be invented.
  buildChanges(base, now) {
    const out = [];
    for (const i of base) {
      const p = this.data.plants[i], full = this.snapshot(i, () => true, 1, now).level, at = this.appear.get(i);
      out.push({ at, date: p.ctime, text: `Sprouts ${p.file.basename}`, i });
      for (let lv = 1; lv <= full; lv++) {
        const f = lv / (full + 1);
        out.push({ at: at + 0.25 * f, text: `↳ ${p.file.basename} → ${STAGES[lv]}`, i });
      }
    }
    return out.sort((a, b) => a.at - b.at);
  }

  renderLog(force = false) {
    if (!this.changes || this.progress == null || !this.settings.showLog) { this.logCount = -1; return; }
    const done = this.changes.filter(c => c.at <= this.progress);
    if (!force && done.length === this.logCount) return;
    this.logCount = done.length;
    this.log.empty();
    this.log.createDiv({ cls: 'mg-log-title', text: 'Changes' });
    for (const c of done.slice(-40).reverse()) {
      const row = this.log.createDiv('mg-log-row');
      row.createSpan({ cls: 'mg-log-date', text: c.date ? shortDate(c.date) : '' });
      row.toggleClass('stage', !c.date);
      row.createSpan({ text: c.text });
      row.addEventListener('click', () => this.centerOn(c.i));
    }
  }

  drawPetals() {
    this.petals.toggleClass('is-hidden', !this.settings.petals);
    if (!this.settings.petals || this.petals.childElementCount) return;
    for (let n = 0; n < 12; n++) {
      const petal = this.petals.createSpan('mg-petal');
      petal.style.left = `${(n * 37) % 100}%`;
      petal.style.animationDelay = `-${(n * 1.7) % 11}s`;
      petal.style.animationDuration = `${9 + (n % 5) * 2}s`;
    }
  }

  applyVb() {
    if (!this.vb) return;
    this.svg.setAttribute('viewBox', `${this.vb.x} ${this.vb.y} ${this.vb.w} ${this.vb.h}`);
    // Keep names ~11px on screen at any zoom (they live inside the scaled plant group).
    const ctm = this.svg.getScreenCTM()?.a || 1;
    this.svg.style.setProperty('--mg-txt', `${(12.5 / ctm).toFixed(2)}px`);
    this.svg.style.setProperty('--mg-txt-island', `${(13 / ctm).toFixed(2)}px`);
    this.placeLabels();
  }

  // Greedy label placement: most important names first, skip any that would overlap one already shown.
  placeLabels() {
    if (this.labelFrame) return;
    this.labelFrame = requestAnimationFrame(() => {
      this.labelFrame = null;
      const s = this.settings, f = this.current, near = f == null ? null : this.data.plants[f].links;
      const want = [];
      this.labelEls.forEach((label, i) => {
        const v = this.snap.get(i);

        const prio = f != null ? (i === f ? 1e6 : near.has(i) ? 1e3 + v.level * 10 + v.leaves : -1)
          : s.labels === 'always' || (s.labels === 'mature' && v.level >= 3) ? v.level * 100 + v.leaves : -1;
        // Names already on screen keep their spot, so growing plants don't make labels swap and blink.
        if (prio >= 0) want.push([prio + (f == null && this.shownLabels?.has(i) ? 60 : 0), label, i]);
      });
      want.sort((a, b) => b[0] - a[0]);
      const placed = [];
      const free = r => !placed.some(o => r.left < o.right + 6 && r.right > o.left - 6 && r.top < o.bottom + 3 && r.bottom > o.top - 3);
      // Island names go first (biggest islands win), then note names.
      const islands = [...this.svg.querySelectorAll('.mg-island-name')].sort((x, y) => y.dataset.n - x.dataset.n);
      // While a note is focused island names are dimmed, so they don't reserve space.
      if (f == null) {
        this.coveredZones = new Set();
        for (const name of islands) {
          const r = name.getBoundingClientRect(), ok = free(r);
          name.classList.toggle('covered', !ok);
          if (ok) placed.push(r); else this.coveredZones.add(name.dataset.zone);
        }
      }
      const shown = new Set();
      for (const [, label, i] of want) {
        const r = label.getBoundingClientRect();
        if (free(r)) { placed.push(r); shown.add(i); }
      }
      this.labelEls.forEach((label, i) => label.classList.toggle('shown', shown.has(i)));
      if (f == null) this.shownLabels = shown;
    });
  }

  focus(i) {
    if (i === this.current) return;
    this.current = i;
    const p = i == null ? null : this.data.plants[i];
    this.svg.classList.toggle('focus', i != null);
    this.plantEls.forEach((g, k) => g.classList.toggle('active', i != null && (k === i || p.links.has(k))));
    this.vineEls.forEach(v => v.classList.toggle('active', i != null && (+v.dataset.a === i || +v.dataset.b === i)));
    this.info.setText(i == null ? '' : describe(p, this.snap.get(i)));
    this.placeLabels();
  }

  wire() {
    const svg = this.svg;
    const indexOf = target => {
      const g = target.closest?.('.mg-p');
      return g ? +g.dataset.i : null;
    };
    const open = (i, e) => this.app.workspace.getLeaf(Keymap.isModEvent(e)).openFile(this.data.plants[i].file);

    let drag = null;
    svg.addEventListener('pointerdown', e => {
      this.contentEl.focus({ preventScroll: true });
      if (!this.vb || e.button !== 0) return;
      const zone = e.target.closest?.('.mg-island-name')?.dataset.zone;
      drag = { x: e.clientX, y: e.clientY, vb: { ...this.vb }, scale: 1 / svg.getScreenCTM().a, plant: indexOf(e.target), zone, moved: false };
      svg.setPointerCapture(e.pointerId);
    });
    svg.addEventListener('pointermove', e => {
      if (!drag) {
        const i = indexOf(e.target);
        if (i != null && i !== this.current) {
          this.app.workspace.trigger('hover-link', {
            event: e, source: VIEW, hoverParent: this, targetEl: e.target.closest('.mg-p'), linktext: this.data.plants[i].file.path, sourcePath: '',
          });
        }
        return this.focus(i);
      }
      const dx = e.clientX - drag.x, dy = e.clientY - drag.y;
      if (Math.hypot(dx, dy) > 4) drag.moved = true;
      if (!drag.moved) return;
      this.vb.x = drag.vb.x - dx * drag.scale;
      this.vb.y = drag.vb.y - dy * drag.scale;
      this.applyVb();
    });
    svg.addEventListener('pointerup', e => {
      if (drag && !drag.moved && drag.plant != null) open(drag.plant, e);
      if (drag && !drag.moved && drag.zone != null) this.zoomToZone(drag.zone);
      drag = null;
    });
    svg.addEventListener('contextmenu', e => {
      const i = indexOf(e.target);
      if (i == null) return;
      e.preventDefault();
      this.menu(i, e);
    });
    svg.addEventListener('pointerleave', () => { if (!drag) this.focus(null); });
    svg.addEventListener('wheel', e => {
      e.preventDefault();
      const pt = new DOMPoint(e.clientX, e.clientY).matrixTransform(svg.getScreenCTM().inverse());
      this.zoom(Math.exp(e.deltaY * 0.0015), pt.x, pt.y);
    }, { passive: false });
    svg.addEventListener('dblclick', () => this.fitView());
    svg.addEventListener('focusin', e => this.focus(indexOf(e.target)));
    svg.addEventListener('focusout', () => this.focus(null));
    svg.addEventListener('keydown', e => {
      const i = indexOf(e.target);
      if (e.key === 'Enter' && i != null) open(i, e);
    });
  }

  zoom(factor, cx, cy) {
    if (!this.vb) return;
    cx ??= this.vb.x + this.vb.w / 2;
    cy ??= this.vb.y + this.vb.h / 2;
    const w = Math.min(Math.max(this.vb.w * factor, 60), 20000), ratio = w / this.vb.w;
    this.vb = { x: cx - (cx - this.vb.x) * ratio, y: cy - (cy - this.vb.y) * ratio, w, h: this.vb.h * ratio };
    this.applyVb();
  }

  // Fit the garden, but never zoom in past MAX_ZOOM so a small vault still reads as miniatures.
  fitted() {
    const MAX_ZOOM = 4, f = this.fitBox;
    const w = Math.max(f.w, (this.svg.clientWidth || 800) / MAX_ZOOM), h = Math.max(f.h, (this.svg.clientHeight || 500) / MAX_ZOOM);
    return { x: f.x + f.w / 2 - w / 2, y: f.y + f.h / 2 - h / 2, w, h };
  }

  fitView() {
    if (this.fitBox) this.animateTo(this.fitted());
  }

  // Ease the camera to a new view box (or jump, if smooth camera is off).
  animateTo(target) {
    cancelAnimationFrame(this.camFrame);
    if (!this.settings.smoothCamera || !this.vb) {
      this.vb = target;
      return this.applyVb();
    }
    const from = { ...this.vb }, start = performance.now();
    const step = now => {
      const t = Math.min(1, (now - start) / 350), e = 1 - (1 - t) ** 3;
      this.vb = Object.fromEntries(Object.keys(from).map(key => [key, from[key] + (target[key] - from[key]) * e]));
      this.applyVb();
      if (t < 1) this.camFrame = requestAnimationFrame(step);
    };
    this.camFrame = requestAnimationFrame(step);
  }

  menu(i, e) {
    const file = this.data.plants[i].file, ws = this.app.workspace, menu = new Menu();
    menu.addItem(it => it.setTitle('Open').setIcon('file').onClick(() => ws.getLeaf(false).openFile(file)));
    menu.addItem(it => it.setTitle('Open in new tab').setIcon('file-plus').onClick(() => ws.getLeaf('tab').openFile(file)));
    menu.addItem(it => it.setTitle('Open to the right').setIcon('separator-vertical').onClick(() => ws.getLeaf('split').openFile(file)));
    menu.addSeparator();
    menu.addItem(it => it.setTitle('Center here').setIcon('locate').onClick(() => this.centerOn(i)));
    menu.addItem(it => it.setTitle('Zoom to its island').setIcon('filter').onClick(() => this.zoomToZone(this.data.plants[i].bed)));
    // Watering touches the note (frontmatter `watered: <today>`), so it counts as recently edited and blooms.
    menu.addItem(it => it.setTitle('Water (mark as reviewed today)').setIcon('droplet').onClick(() => {
      this.app.fileManager.processFrontMatter(file, fm => { fm.watered = new Date().toISOString().slice(0, 10); });
    }));
    menu.addItem(it => it.setTitle('Copy link').setIcon('link').onClick(() => {
      navigator.clipboard.writeText(this.app.fileManager.generateMarkdownLink(file, ''));
      new Notice('Link copied');
    }));
    menu.showAtMouseEvent(e);
  }

  zoomToZone(bed) {
    const pts = [...this.plantEls.keys()].filter(i => this.data.plants[i].bed === bed)
      .map(i => iso(...this.cells.get(i).map(c => c * this.settings.grid)));
    if (!pts.length) return;
    const S = this.settings.grid, xs = pts.map(p => p[0]), ys = pts.map(p => p[1]);
    const box = { x: Math.min(...xs) - S * 2.5, y: Math.min(...ys) - S * 3, w: Math.max(...xs) - Math.min(...xs) + S * 5, h: Math.max(...ys) - Math.min(...ys) + S * 5 };
    const aspect = (this.svg.clientWidth || 800) / (this.svg.clientHeight || 500);
    const w = Math.max(box.w, box.h * aspect), h = w / aspect;
    this.animateTo({ x: box.x + box.w / 2 - w / 2, y: box.y + box.h / 2 - h / 2, w, h });
  }

  centerOn(i) {
    if (!this.plantEls.has(i)) return new Notice('That note is hidden by the filters or the timelapse.');
    const [c, r] = this.cells.get(i), [x, y] = iso(c * this.settings.grid, r * this.settings.grid);
    const w = Math.min(this.vb.w, this.settings.grid * 30), h = (this.vb.h * w) / this.vb.w;
    this.animateTo({ x: x - w / 2, y: y - h / 2, w, h });
    this.current = null;
    this.focus(i);
  }

  reveal(file) {
    const i = this.data?.plants.findIndex(p => p.file === file) ?? -1;
    if (i < 0) return new Notice('That note is not in the garden.');
    this.centerOn(i);
  }

  pick() {
    if (!this.data) return;
    const items = [...this.plantEls.keys()].map(i => ({ i, p: this.data.plants[i] }));
    new NoteModal(this.app, items, i => this.centerOn(i)).open();
  }

  setProgress(p) {
    this.progress = p == null || p >= 1 ? null : Math.max(0, p);
    this.draw();
    this.renderLog();
    if (this.timeSlider) this.timeSlider.inputEl.value = String(Math.round((this.progress ?? 1) * 1000));
  }

  // Growth animation: every note sprouts in creation order and climbs through its stages.
  play(seconds = this.settings.tlDuration) {
    this.stop();
    if (!this.data?.plants.length) return;
    this.playBtn?.setIcon('pause');
    let p = this.progress ?? 0, last = performance.now();
    const tick = now => {
      p += ((now - last) / 1000 / seconds) * this.speed;
      last = now;
      if (p >= 1 && this.settings.tlLoop) p = 0;
      this.setProgress(p);
      if (this.progress == null) return this.stop();
      this.frame = requestAnimationFrame(tick);
    };
    this.frame = requestAnimationFrame(tick);
  }

  restart() {
    this.stop();
    this.progress = 0;
    this.play();
  }

  togglePlay() {
    if (this.frame) return this.stop();
    if (this.progress == null) this.progress = 0;
    this.play();
  }

  stop() {
    cancelAnimationFrame(this.frame);
    this.frame = null;
    this.playBtn?.setIcon('play');
    this.caption();
  }

  onKey(e) {
    const t = e.target;
    if (t instanceof HTMLInputElement || t instanceof HTMLTextAreaElement || t instanceof HTMLSelectElement) return;
    if (e.ctrlKey || e.metaKey || e.altKey) return;
    const actions = {
      '/': () => { this.settings.panelOpen = true; this.panel.removeClass('is-hidden'); this.searchInput?.focus(); },
      f: () => this.fitView(),
      g: () => this.pick(),
      t: () => this.togglePlay(),
      r: () => this.restart(),
      l: () => this.toggleLog(),
      ArrowLeft: () => { this.stop(); this.setProgress((this.progress ?? 1) - 0.05); },
      ArrowRight: () => { this.stop(); if (this.progress != null) this.setProgress(this.progress + 0.05); },
      '+': () => this.zoom(0.8),
      '=': () => this.zoom(0.8),
      '-': () => this.zoom(1.25),
      Escape: () => {
        if (this.progress == null) return this.focus(null);
        this.stop();
        this.setProgress(null);
      },
    };
    const action = actions[e.key.length === 1 ? e.key.toLowerCase() : e.key];
    if (!action) return;
    e.preventDefault();
    action();
  }
}

class GardenSettings extends PluginSettingTab {
  constructor(app, plugin) {
    super(app, plugin);
    this.plugin = plugin;
  }

  display() {
    const { containerEl } = this, s = this.plugin.settings;
    const save = debounce(() => this.plugin.saveSettings(), 400, true);
    const row = (name, desc) => new Setting(containerEl).setName(name).setDesc(desc ?? '');
    const slider = (name, key, min, max, step, desc) => row(name, desc)
      .addSlider(c => c.setLimits(min, max, step).setValue(s[key]).setDynamicTooltip().onChange(v => { s[key] = v; save(); }));
    containerEl.empty();

    row('Excluded folders', 'One per line; a folder name matches at any depth, e.g. memories/.')
      .addTextArea(c => c.setValue(s.exclude).onChange(v => { s.exclude = v; save(); }));

    new Setting(containerEl).setName('Islands').setHeading();
    row('Group by', 'Each island is a topic. `island: Name` in a note sends it to that island.')
      .addDropdown(c => c.addOptions(GROUPS).setValue(s.groupBy).onChange(v => { s.groupBy = v; save(); }));
    slider('Size', 'islandSize', 0.7, 1.4, 0.05);
    slider('Spacing', 'islandGap', 0, 3, 0.1);
    slider('Moon orbit', 'moonGap', 1, 6, 0.5);
    slider('Roughness', 'roughness', 0, 1, 0.05);
    slider('Sand furrows', 'rings', 0, 6, 1);
    row('Island names').addToggle(c => c.setValue(s.zoneNames).onChange(v => { s.zoneNames = v; save(); }));
    row('Decoration', 'Rocks in the sand and lanterns beside the islands.')
      .addDropdown(c => c.addOptions(DECOR_NAMES).setValue(s.decor).onChange(v => { s.decor = v; save(); }));

    new Setting(containerEl).setName('Plants').setHeading();
    row('Trees', '`plant: pine`, `round` or `bonsai` in a note picks its own.')
      .addDropdown(c => c.addOptions(TREE_NAMES).setValue(s.treeStyle).onChange(v => { s.treeStyle = v; save(); }));
    slider('Growth rate', 'growth', 0.5, 2, 0.1, 'Higher means they grow sooner.');
    slider('Days in bloom', 'bloomDays', 1, 30, 1);
    slider('Days until turning to stone', 'wiltDays', 14, 365, 1);
    slider('Plant spacing', 'grid', 14, 36, 1);

    new Setting(containerEl).setName('Motion').setHeading();
    row('Sway').addToggle(c => c.setValue(s.sway).onChange(v => { s.sway = v; save(); }));
    slider('Sway strength', 'swayAmp', 0.5, 6, 0.5);
    slider('Sway pace (s)', 'swaySpeed', 2, 10, 0.5);
    row('Drifting petals').addToggle(c => c.setValue(s.petals).onChange(v => { s.petals = v; save(); }));
    for (const [key, name] of Object.entries({ clouds: 'Clouds', fireflies: 'Fireflies', flicker: 'Flickering lanterns', bounce: 'Hop on hover', smoothCamera: 'Smooth camera' })) {
      row(name).addToggle(c => c.setValue(s[key]).onChange(v => { s[key] = v; save(); }));
    }
    slider('Growth duration (s)', 'tlDuration', 5, 60, 1);
    row('Loop growth').addToggle(c => c.setValue(s.tlLoop).onChange(v => { s.tlLoop = v; save(); }));
    row('Change log', 'Dated list of what sprouts and grows during the animation.')
      .addToggle(c => c.setValue(s.showLog).onChange(v => { s.showLog = v; save(); }));

    new Setting(containerEl).setName('Colors').setHeading();
    row('Palette').addDropdown(c => c.addOptions(PALETTE_NAMES).setValue(s.palette)
      .onChange(v => { s.palette = v; applyPreset(s, v); save(); this.display(); }));
    const colors = { sand: 'Sand', moss: 'Moss', canopy: 'Canopies', blossom: 'Blossoms', text: 'Island names' };
    for (const [key, name] of Object.entries(colors)) {
      row(name).addColorPicker(c => c.setValue(s[`${key}Color`]).onChange(v => { s[`${key}Color`] = v; save(); }));
    }

    row('Reset').addButton(c => c.setButtonText('Restore defaults').onClick(async () => {
      this.plugin.settings = { ...DEFAULTS, hiddenZones: [] };
      await this.plugin.saveSettings();
      this.display();
    }));
  }
}

module.exports = class MindGarden extends Plugin {
  async onload() {
    this.settings = { ...DEFAULTS, ...(await this.loadData()) };
    // shortcut: settings saved while the plugin was in Spanish get their old keys/values renamed; drop once nobody has old data.
    const LEGACY = {
      keys: { colorArena: 'sandColor', colorMusgo: 'mossColor', colorCopa: 'canopyColor', colorFlor: 'blossomColor', colorTexto: 'textColor' },
      values: {
        groupBy: { tipo: 'type', carpeta: 'folder', subcarpeta: 'subfolder', etiqueta: 'tag', ninguna: 'none' },
        treeStyle: { mixto: 'mixed', redondo: 'round', pino: 'pine' },
        decor: { ninguna: 'none', poca: 'some', mucha: 'lots' },
        labels: { maduras: 'mature', siempre: 'always' },
        palette: { carmesi: 'crimson', otono: 'autumn' },
      },
    };
    for (const [from, to] of Object.entries(LEGACY.keys)) if (from in this.settings) { this.settings[to] = this.settings[from]; delete this.settings[from]; }
    for (const [key, map] of Object.entries(LEGACY.values)) this.settings[key] = map[this.settings[key]] ?? this.settings[key];
    // shortcut: values saved under the previous look are swapped for the new defaults; drop once nobody has old data.
    const OLD = { grid: [9, 10, 16, 18], blossomColor: ['#f0a35e', '#f2a65a'] };
    for (const [key, values] of Object.entries(OLD)) if (values.includes(this.settings[key])) this.settings[key] = DEFAULTS[key];
    delete this.settings.growIn; // growth now only starts by hand
    if (!PRESETS[this.settings.palette]) {
      this.settings.palette = 'zen';
      applyPreset(this.settings, 'zen');
    }
    this.registerView(VIEW, leaf => new GardenView(leaf, this));
    this.registerHoverLinkSource(VIEW, { display: 'Garden Zen', defaultMod: true });
    this.addSettingTab(new GardenSettings(this.app, this));
    this.addRibbonIcon('sprout', 'Open Garden Zen', () => this.activate());
    this.addCommand({ id: 'open', name: 'Open garden', callback: () => this.activate() });
    this.addCommand({ id: 'go-to-note', name: 'Go to a note in the garden', callback: async () => (await this.activate())?.pick() });
    this.addCommand({
      id: 'reveal-active', name: 'Show the active note in the garden',
      callback: async () => {
        const file = this.app.workspace.getActiveFile();
        const view = await this.activate();
        if (file) view?.reveal(file);
      },
    });
    this.addCommand({ id: 'timelapse', name: 'Watch the garden grow', callback: async () => (await this.activate())?.togglePlay() });

    const refresh = debounce(() => this.views().forEach(v => v.load()), 1500, true);
    this.registerEvent(this.app.metadataCache.on('resolved', refresh));
    this.registerEvent(this.app.vault.on('rename', refresh));
    this.registerEvent(this.app.vault.on('delete', refresh));
  }

  views() {
    return this.app.workspace.getLeavesOfType(VIEW).map(l => l.view).filter(v => v instanceof GardenView);
  }

  async saveSettings(reload = true) {
    await this.saveData(this.settings);
    if (!reload) return;
    for (const view of this.views()) {
      view.buildPanel();
      view.vb = null;
      view.load();
    }
  }

  async activate() {
    let leaf = this.app.workspace.getLeavesOfType(VIEW)[0];
    if (!leaf) {
      leaf = this.app.workspace.getLeaf('tab');
      await leaf.setViewState({ type: VIEW, active: true });
    }
    await this.app.workspace.revealLeaf(leaf);
    await leaf.loadIfDeferred?.();
    return leaf.view instanceof GardenView ? leaf.view : null;
  }

  async collect() {
    const { vault, metadataCache } = this.app, s = this.settings;
    const exclude = s.exclude.split('\n').map(x => x.trim()).filter(Boolean);
    const links = metadataCache.resolvedLinks;
    const files = vault.getMarkdownFiles().filter(f => !exclude.some(p => ('/' + f.path).includes('/' + p)));
    const all = (await Promise.all(files.map(async file => {
      const cache = metadataCache.getFileCache(file);
      const fm = cache?.frontmatter ?? {};
      if (fm.garden === false) return null;
      const text = await vault.cachedRead(file);
      const body = cache?.frontmatterPosition ? text.slice(cache.frontmatterPosition.end.offset) : text;
      const folder = file.parent?.isRoot() ? 'Root' : file.path.split('/')[0];
      // Memory notes orbit their project as moons; Areas notes are planets of their own.
      const moon = file.parent?.name === 'memories';
      const dir = moon ? file.parent.parent : file.parent;
      // shortcut: islands use the folder name only, so two folders with the same name share an island.
      const sub = !dir || dir.isRoot() ? 'Root' : dir.name === 'Areas' ? file.basename : dir.name;
      const zone = fm.island ? String(fm.island) : {
        type: fm.type ? String(fm.type) : folder,
        folder: folder,
        subfolder: sub,
        tag: (cache ? getAllTags(cache) ?? [] : [])[0]?.replace(/^#/, '') ?? 'Untagged',
        none: 'Garden',
      }[s.groupBy] ?? folder;
      const style = TREE_STYLES.find(st => st === plain(fm.plant ?? ''));
      return {
        file,
        words: (body.match(/\S+/g) || []).length,
        ctime: file.stat.ctime,
        mtime: file.stat.mtime,
        forced: parseStage(fm.stage),
        bed: moon && s.groupBy !== 'subfolder' && s.groupBy !== 'folder' ? sub : zone,
        moon,
        style,
        outs: new Set(), ins: new Set(), links: new Set(),
      };
    }))).filter(Boolean);

    // Repeated memories ("Session summary - 748", "- 750"…) collapse into their newest copy.
    const newest = new Map();
    for (const p of all) {
      if (!p.moon) continue;
      const key = `${p.bed}|${p.file.basename.replace(/ - \d+$/, '')}`;
      if (!newest.has(key) || newest.get(key).mtime < p.mtime) newest.set(key, p);
    }
    const plants = all.filter(p => !p.moon || newest.get(`${p.bed}|${p.file.basename.replace(/ - \d+$/, '')}`) === p);

    // One edge per linked pair; `mutual` when both notes link to each other.
    const index = new Map(plants.map((p, i) => [p.file.path, i]));
    const pairs = new Map();
    plants.forEach((p, a) => {
      for (const dst in links[p.file.path] || {}) {
        const b = index.get(dst);
        if (b == null || b === a) continue;
        p.outs.add(b); plants[b].ins.add(a);
        p.links.add(b); plants[b].links.add(a);
        const key = a < b ? `${a},${b}` : `${b},${a}`;
        if (pairs.has(key)) pairs.get(key)[2] = true;
        else pairs.set(key, [a, b, false]);
      }
    });
    return { plants, edges: [...pairs.values()] };
  }
};

Object.assign(module.exports, { stage, parseStage, health, layout, element, iso, gardenLevel, decorations, lantern });
