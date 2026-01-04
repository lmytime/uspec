<template>
  <div class="page">
    <header class="hero">
      <div class="hero-brand">
        <div class="hero-line">
          <div class="app-name">USpec</div>
          <h1 class="hero-title">Interactive explore your spectra</h1>
        </div>
        <div class="designer">
          Designed by <a href="https://lmytime.com" target="_blank" rel="noreferrer">Mingyu Li</a> · <a href="https://lmytime.com" target="_blank" rel="noreferrer">lmytime.com</a>
        </div>
        <div class="hero-note">Test build — may have some bugs while we iterate.</div>
      </div>
    </header>

    <div class="layout">
      <aside
        class="sidebar"
        :style="{ '--sidebar-width': uploadCollapsed ? '320px' : '260px' }"
      >
        <div class="sidebar-card upload-card" :class="{ collapsed: uploadCollapsed }">
          <div class="sidebar-title-row">
            <div class="sidebar-title">Upload CSV spectra</div>
            <button type="button" class="collapse-btn" @click="uploadCollapsed = !uploadCollapsed">
              {{ uploadCollapsed ? "▸" : "▾" }}
            </button>
          </div>
          <div v-if="!uploadCollapsed" class="upload-body">
            <p class="sidebar-copy">Select files or drag them in. Multiple CSV files are supported.</p>
            <div class="controls sidebar-controls">
              <div class="control-group">
                <label for="x-unit-input">Input λ unit</label>
                <select id="x-unit-input" v-model="xUnit">
                  <option value="A">Å</option>
                  <option value="nm">nm</option>
                  <option value="um">µm</option>
                </select>
              </div>
              <div class="control-group">
                <label for="flux-unit-input">Input flux unit</label>
                <select id="flux-unit-input" v-model="fluxUnit">
                  <option value="flambda">erg s⁻¹ cm⁻² Å⁻¹</option>
                  <option value="Jy">Jy</option>
                  <option value="mJy">mJy</option>
                  <option value="uJy">µJy</option>
                  <option value="nJy">nJy</option>
                </select>
              </div>
            </div>
            <label
              class="drop-area"
              :class="{ active: dragActive }"
              @dragover.prevent="dragActive = true"
              @dragleave.prevent="dragActive = false"
              @drop.prevent="handleDrop"
            >
              <input type="file" accept=".csv,text/csv" multiple @change="handleFileInput" />
              <span>Drop CSV files here or click to choose</span>
              <small>Expected columns: wavelength, flux, error</small>
            </label>
          </div>
        </div>

        <div class="sidebar-card">
          <div class="sidebar-title-row">
            <div class="sidebar-title">Loaded files</div>
            <button type="button" class="collapse-btn" @click="exportRedshifts">Export</button>
          </div>
          <ul class="dataset-list" v-if="datasets.length > 0">
            <li v-for="dataset in datasets" :key="dataset.id">
              <button
                type="button"
                class="dataset-item"
                :class="{ active: dataset.id === activeDatasetId }"
                @click="setActiveDataset(dataset.id)"
              >
                <span class="dataset-name">{{ dataset.name }}</span>
                <span class="dataset-meta">{{ datasetMeta(dataset) }}</span>
              </button>
            </li>
          </ul>
          <p v-else class="empty-state">No files yet. Start with the sample data or add your own.</p>
        </div>
      </aside>

      <section class="plot-card">
        <header class="plot-header">
          <h2 class="plot-title">
            {{ activeDataset?.name || "Spectrum" }}
          </h2>
          <div class="plot-subtitle" v-if="activeDataset">
            {{ activeDataset.meta?.xUnit?.toUpperCase?.() || "A" }} ·
            {{ activeDataset.meta?.fluxUnit || "flambda" }}
          </div>
        </header>
        <div class="plot-wrap" ref="plotWrapEl">
          <div ref="chartEl" class="chart-host"></div>
          <div class="cursor-hud">{{ cursorText }}</div>
          <div class="cursor-rest">{{ cursorRestText }}</div>
        </div>

        <div class="z-control">
          <button class="mode-toggle" type="button" @click="toggleMode">
            {{ currentMode === "flam" ? "fλ" : "fν" }}
          </button>
          <button class="mode-toggle" type="button" @click="toggleSteps">
            {{ useSteps ? "step" : "line" }}
          </button>
          <label for="z-input">Redshift z</label>
          <input
            id="z-input"
            type="number"
            step="0.0001"
            v-model="zDisplay"
            @blur="sanitizeZ"
          />
          <button type="button" @click="openLineModal">Add line</button>
        </div>

        <div class="lines-control">
          <div class="line-group" v-for="group in groupedLines" :key="group.name">
            <button type="button" class="line-group-toggle" @click="toggleGroup(group.name)">
              <span>{{ group.label }}</span>
              <span>{{ isGroupOpen(group.name) ? "▾" : "▸" }}</span>
            </button>
            <div class="line-items" v-if="isGroupOpen(group.name)">
              <label v-for="line in group.lines" :key="line.id">
                <input type="checkbox" v-model="line.show" />
                <span>{{ line.label }}</span>
                <span class="line-tooltip">{{ line.rest.toFixed(2) }} Å</span>
              </label>
            </div>
          </div>
        </div>
      </section>
    </div>

    <div
      class="modal-backdrop"
      :class="{ show: showLineModal }"
      @click.self="closeLineModal"
    >
      <div class="modal">
        <h3>Add custom line</h3>
        <form @submit.prevent="submitLine">
          <label>
            Line name
            <input v-model="newLineName" type="text" placeholder="e.g. [OIII]5007" required />
          </label>
          <label>
            Wavelength
            <input
              v-model.number="newLineLambda"
              type="number"
              step="0.0001"
              placeholder="e.g. 5008.24"
              required
            />
          </label>
          <label>
            Unit
            <select v-model="newLineUnit">
              <option value="A">Å</option>
              <option value="um">µm</option>
            </select>
          </label>
          <div class="modal-actions">
            <button type="button" @click="closeLineModal">Cancel</button>
            <button type="submit" id="line-submit">Add</button>
          </div>
        </form>
      </div>
    </div>
  </div>
</template>

<script setup>
import { computed, onBeforeUnmount, onMounted, reactive, ref, watch } from "vue";
import uPlot from "uplot";

const C_CM_PER_S = 2.99792458e10;
const CM_PER_ANG = 1e-8;

const yIncrements = [
  1e-30, 2e-30, 5e-30,
  1e-29, 2e-29, 5e-29,
  1e-28, 2e-28, 5e-28,
  1e-27, 2e-27, 5e-27,
  1e-26, 2e-26, 5e-26,
  1e-25, 2e-25, 5e-25,
  1e-24, 2e-24, 5e-24,
  1e-23, 2e-23, 5e-23,
  1e-22, 2e-22, 5e-22,
  1e-21, 2e-21, 5e-21,
  1e-20, 2e-20, 5e-20,
  1e-19, 2e-19, 5e-19,
  1e-18, 2e-18, 5e-18,
  1e-17, 2e-17, 5e-17,
  1e-16, 2e-16, 5e-16,
  1e-15, 2e-15, 5e-15,
  1e-14, 2e-14, 5e-14,
  1e-13, 2e-13, 5e-13,
  1e-12, 2e-12, 5e-12,
  1e-11, 2e-11, 5e-11,
  1e-10, 2e-10, 5e-10,
  1e-9, 2e-9, 5e-9,
  1e-8, 2e-8, 5e-8,
  1e-7, 2e-7, 5e-7,
  1e-6, 2e-6, 5e-6,
  1e-5, 2e-5, 5e-5,
  1e-4, 2e-4, 5e-4,
  1e-3, 2e-3, 5e-3,
  1e-2, 2e-2, 5e-2,
  1e-1, 2e-1, 5e-1,
  1, 2, 5,
  1e1, 2e1, 5e1,
  1e2, 2e2, 5e2,
  1e3, 2e3, 5e3,
  1e4, 2e4, 5e4,
  1e5, 2e5, 5e5,
  1e6, 2e6, 5e6,
  1e7, 2e7, 5e7,
  1e8, 2e8, 5e8,
  1e9, 2e9, 5e9,
  1e10, 2e10, 5e10,
  1e11, 2e11, 5e11,
  1e12, 2e12, 5e12,
  1e13, 2e13, 5e13,
  1e14, 2e14, 5e14,
  1e15, 2e15, 5e15,
  1e16, 2e16, 5e16,
  1e17, 2e17, 5e17,
  1e18, 2e18, 5e18,
  1e19, 2e19, 5e19,
  1e20, 2e20, 5e20,
];

function deriveGroup(label = "") {
  const clean = label.trim();
  if (/^PAH/i.test(clean)) return "PAH";
  if (/^H2/i.test(clean)) return "H2";
  if (/^(HI|Ly|Hα|Hβ|Hγ|Hδ|Hε|Hζ|Hη|H∞|Pa|Br|Pf)/i.test(clean)) return "H";
  const match = clean.match(/^\[?([A-Za-z]+)/);
  return match ? match[1].toUpperCase() : "Other";
}

const DEFAULT_SAMPLE_Z = 5.1455;
const DEFAULT_UPLOAD_Z = 2.3456;

const datasets = ref([]);
const activeDatasetId = ref(null);
const cursorText = ref("—");
const cursorRestText = ref("—");
const currentMode = ref("flam");
const xUnit = ref("A"); // applies to next upload
const fluxUnit = ref("flambda"); // applies to next upload
const useSteps = ref(true);
const zValue = ref(DEFAULT_SAMPLE_Z);
const zDisplay = computed({
  get: () => (Number.isFinite(zValue.value) ? zValue.value.toFixed(4) : "0.0000"),
  set: (val) => {
    const num = Number.parseFloat(val);
    if (Number.isFinite(num)) {
      zValue.value = Number(num.toFixed(4));
    }
  },
});
const dragActive = ref(false);
const uploadCollapsed = ref(false);

const showLineModal = ref(false);
const newLineName = ref("");
const newLineLambda = ref("");
const newLineUnit = ref("A");

const linesAll = reactive([
  { id: "oiii", label: "[OIII]", rest: 5007, show: true, group: deriveGroup("[OIII]") },
  { id: "hb", label: "Hβ", rest: 4861, show: true, group: deriveGroup("Hβ") },
  { id: "ha", label: "Hα", rest: 6563, show: true, group: deriveGroup("Hα") },
]);

const chartEl = ref(null);
const plotWrapEl = ref(null);
const plotRef = ref(null);
let resizeObserver = null;
let zDrag = null;
let hasInteracted = false;
let hideLines = false;

const activeDataset = computed(() => {
  return datasets.value.find((d) => d.id === activeDatasetId.value) ?? datasets.value[0] ?? null;
});

const activeUnits = computed(() => {
  const meta = activeDataset.value?.meta ?? {};
  return {
    xUnit: meta.xUnit ?? xUnit.value,
    fluxUnit: meta.fluxUnit ?? fluxUnit.value,
  };
});

const activeRawData = computed(() => activeDataset.value?.raw ?? null);

const baseData = computed(() => convertToBase(activeRawData.value, activeUnits.value.xUnit, activeUnits.value.fluxUnit));
const displayData = computed(() => computeDisplay(baseData.value, currentMode.value));
const groupStates = reactive({});
const groupLabels = {
  H: "Hydrogen (H)",
  HE: "Helium (He)",
  C: "Carbon (C)",
  N: "Nitrogen (N)",
  O: "Oxygen (O)",
  NE: "Neon (Ne)",
  NA: "Sodium (Na)",
  MG: "Magnesium (Mg)",
  AL: "Aluminum (Al)",
  SI: "Silicon (Si)",
  P: "Phosphorus (P)",
  S: "Sulfur (S)",
  CL: "Chlorine (Cl)",
  AR: "Argon (Ar)",
  K: "Potassium (K)",
  CA: "Calcium (Ca)",
  SC: "Scandium (Sc)",
  TI: "Titanium (Ti)",
  V: "Vanadium (V)",
  CR: "Chromium (Cr)",
  MN: "Manganese (Mn)",
  FE: "Iron (Fe)",
  CO: "Cobalt (Co)",
  NI: "Nickel (Ni)",
  CU: "Copper (Cu)",
  ZN: "Zinc (Zn)",
  PAH: "PAH (molecular)",
  H2: "Molecular Hydrogen (H2)",
  OTHER: "Other",
};
const groupedLines = computed(() => {
  const groups = new Map();
  linesAll.forEach((line) => {
    const group = line.group || deriveGroup(line.label);
    if (!groups.has(group)) groups.set(group, []);
    groups.get(group).push(line);
    if (!(group in groupStates)) groupStates[group] = true;
  });
  return Array.from(groups.entries())
    .map(([name, lines]) => ({
      name,
      label: groupLabels[name] || name,
      lines: lines.slice().sort((a, b) => a.rest - b.rest),
    }))
    .sort((a, b) => {
      const order = [
        "H",
        "HE",
        "C",
        "N",
        "O",
        "NE",
        "NA",
        "MG",
        "AL",
        "SI",
        "P",
        "S",
        "CL",
        "AR",
        "K",
        "CA",
        "SC",
        "TI",
        "V",
        "CR",
        "MN",
        "FE",
        "CO",
        "NI",
        "CU",
        "ZN",
        "OTHER",
        "PAH",
        "H2",
      ];
      const norm = (name) => (name || "").toUpperCase();
      const ai = order.indexOf(norm(a.name));
      const bi = order.indexOf(norm(b.name));
      const aIdx = ai === -1 ? order.indexOf("OTHER") : ai;
      const bIdx = bi === -1 ? order.indexOf("OTHER") : bi;
      if (aIdx !== bIdx) return aIdx - bIdx;
      return norm(a.name).localeCompare(norm(b.name));
    });
});

const stepPath = uPlot.paths.stepped({ align: 0, ascDesc: false });

watch([displayData, useSteps], () => {
  updatePlot();
});

watch(zValue, () => {
  if (activeDataset.value) {
    activeDataset.value.meta = {
      ...(activeDataset.value.meta ?? {}),
      z: Number.isFinite(zValue.value) ? Number(zValue.value.toFixed(4)) : 0,
    };
  }
  if (plotRef.value) plotRef.value.redraw();
});

watch(
  linesAll,
  () => {
    if (plotRef.value) plotRef.value.redraw();
  },
  { deep: true },
);

watch(
  activeDataset,
  (dataset) => {
    if (dataset?.meta && Number.isFinite(dataset.meta.z)) {
      zValue.value = Number(dataset.meta.z.toFixed(4));
    } else {
      zValue.value = 0;
    }
    updatePlot();
  },
  { immediate: true },
);

onMounted(async () => {
  await Promise.all([loadLines(), loadSampleSpectrum()]);
  setupResizeObserver();
  window.addEventListener("keydown", handleKeyDown);
  window.addEventListener("keyup", handleKeyUp);
  window.addEventListener("beforeunload", handleBeforeUnload);
});

onBeforeUnmount(() => {
  if (resizeObserver && plotWrapEl.value) {
    resizeObserver.disconnect();
  }
  if (plotRef.value) {
    plotRef.value.destroy();
    plotRef.value = null;
  }
  window.removeEventListener("keydown", handleKeyDown);
  window.removeEventListener("keyup", handleKeyUp);
  window.removeEventListener("beforeunload", handleBeforeUnload);
});

function setupResizeObserver() {
  if (!plotWrapEl.value) return;
  resizeObserver = new ResizeObserver(() => {
    if (plotRef.value && plotWrapEl.value) {
      const rect = plotWrapEl.value.getBoundingClientRect();
      plotRef.value.setSize({ width: rect.width, height: rect.height });
    }
  });
  resizeObserver.observe(plotWrapEl.value);
}

function formatWave(v) {
  return Number.isFinite(v) ? `${v.toFixed(2)} Å` : "—";
}

function formatFlux(v) {
  if (!Number.isFinite(v)) return "—";
  if (currentMode.value === "flam") return `${v.toExponential(3)} erg s⁻¹ cm⁻² Å⁻¹`;
  return `${v.toExponential(3)} Jy`;
}

function formatAxisFlux(v) {
  if (!Number.isFinite(v)) return "";
  const abs = Math.abs(v);
  if (abs === 0) return "0";
  if (currentMode.value === "flam") return v.toExponential(2);
  if (abs >= 1e4 || abs < 1e-3) return v.toExponential(2);
  return Number(v.toPrecision(3)).toString();
}

function fnuJyFromFlam(flamAA, lamA) {
  const lamCm = lamA * CM_PER_ANG;
  return (flamAA * 1e8 * lamCm * lamCm) / C_CM_PER_S / 1e-23;
}

function flamFromFnuJy(fnuJy, lamA) {
  const lamCm = lamA * CM_PER_ANG;
  const fnuCgs = fnuJy * 1e-23;
  return (fnuCgs * C_CM_PER_S) / (lamCm * lamCm) * 1e-8;
}

function makeStepSeries(xArr, ...yArrays) {
  if (!xArr || xArr.length === 0) return { x: [], ys: yArrays.map(() => []) };
  const n = xArr.length;
  const xs = [];
  const ys = yArrays.map(() => []);

  for (let i = 0; i < n; i += 1) {
    const x = xArr[i];
    const prev = i === 0 ? xArr[0] - (xArr[1] - xArr[0]) / 2 : (xArr[i - 1] + x) / 2;
    const next =
      i === n - 1
        ? xArr[n - 1] + (xArr[n - 1] - xArr[n - 2]) / 2
        : (x + xArr[i + 1]) / 2;

    xs.push(prev, next);
    yArrays.forEach((arr, idx) => {
      ys[idx].push(arr[i], arr[i]);
    });
  }

  return { x: xs, ys };
}

function makeTicks(min, max, incrs) {
  if (!Number.isFinite(min) || !Number.isFinite(max)) return [];
  if (min === max) {
    const pad = Math.abs(min) || 1;
    min -= pad * 0.1;
    max += pad * 0.1;
  }
  const span = max - min || 1;
  const targetCount = 6;
  const step = incrs.find((inc) => span / inc <= targetCount) ?? incrs[incrs.length - 1];
  const first = Math.ceil(min / step) * step;
  const ticks = [];
  for (let v = first; v <= max + step * 0.5 && ticks.length < 200; v += step) {
    ticks.push(v);
  }
  if (ticks.length < 2) {
    const mid = (min + max) / 2;
    return [min, mid, max];
  }
  return ticks;
}

function bandPlugin({ upperSeries, lowerSeries, fill }) {
  return {
    hooks: {
      draw: (u) => {
        const xData = u.data[0];
        const upper = u.data[upperSeries];
        const lower = u.data[lowerSeries];

        if (!xData || !upper || !lower || xData.length === 0) return;

        const { left, top, width, height } = u.bbox;
        const ctx = u.ctx;

        ctx.save();
        ctx.beginPath();
        ctx.rect(left, top, width, height);
        ctx.clip();
        ctx.fillStyle = fill;

        ctx.beginPath();
        ctx.moveTo(
          u.valToPos(xData[0], "x", true),
          u.valToPos(upper[0], "y", true),
        );

        for (let i = 1; i < xData.length; i += 1) {
          const xPos = u.valToPos(xData[i], "x", true);
          const yPos = u.valToPos(upper[i], "y", true);
          ctx.lineTo(xPos, yPos);
        }

        for (let i = xData.length - 1; i >= 0; i -= 1) {
          const xPos = u.valToPos(xData[i], "x", true);
          const yPos = u.valToPos(lower[i], "y", true);
          ctx.lineTo(xPos, yPos);
        }

        ctx.closePath();
        ctx.fill();
        ctx.restore();
      },
    },
  };
}

function cursorReadoutPlugin(update) {
  return {
    hooks: {
      init: (u) => {
        u.over.addEventListener("mouseleave", () => update(null, null));
      },
      setCursor: (u) => {
        const { left, top } = u.cursor;
        if (left == null || top == null) {
          update(null, null);
          return;
        }
        const xVal = u.posToVal(left, "x");
        const yVal = u.posToVal(top, "y");
        update(xVal, yVal);
      },
    },
  };
}

function createZLinesPlugin() {
  return {
    hooks: {
      draw: (u) => {
        if (hideLines) return;
        const { top, height } = u.bbox;
        const xMin = u.scales.x.min;
        const xMax = u.scales.x.max;
        const ctx = u.ctx;
        ctx.save();
        ctx.strokeStyle = "rgba(71, 82, 177, 0.7)";
        ctx.lineWidth = 3;
        ctx.fillStyle = "rgba(47, 55, 134, 0.95)";
        ctx.font = "24px Space Grotesk";
        ctx.textBaseline = "top";
        ctx.textAlign = "left";

        const drawLine = (obs, label, color, textY) => {
          if (!Number.isFinite(obs) || obs < xMin || obs > xMax) return;
          const xPos = u.valToPos(obs, "x", true);
          ctx.strokeStyle = color;
          ctx.beginPath();
          ctx.moveTo(xPos, top);
          ctx.lineTo(xPos, top + height);
          ctx.stroke();
          ctx.fillText(label, xPos + 4, textY);
        };

        const visible = linesAll.filter((l) => l.show).sort((a, b) => a.rest - b.rest);
        const laneHeight = 24;
        const laneCount = 3;

        visible.forEach((line, idx) => {
          const obs = line.rest * (1 + zValue.value);
          const lane = idx % laneCount;
          const textY = top + 4 + lane * laneHeight;
          drawLine(obs, line.label, "rgba(71, 82, 177, 0.7)", textY);
        });

        ctx.restore();
      },
    },
  };
}

function hintPlugin(text) {
  return {
    hooks: {
      draw: (u) => {
        if (hasInteracted) return;
        const { left, top, width, height } = u.bbox;
        const ctx = u.ctx;
        ctx.save();
        ctx.fillStyle = "rgba(0, 0, 0, 0.28)";
        ctx.font = "bold 64px Space Grotesk";
        ctx.textAlign = "center";
        ctx.textBaseline = "middle";
        ctx.fillText(text, left + width / 2, top + height / 2);
        ctx.restore();
      },
      ready: (u) => {
        u.over.addEventListener("dblclick", () => {
          hasInteracted = true;
          u.redraw();
        });
      },
    },
  };
}

function handleKeyDown(ev) {
  if (ev.altKey) {
    hideLines = true;
    if (plotRef.value) plotRef.value.redraw();
  }
}

function handleKeyUp(ev) {
  if (!ev.altKey) {
    hideLines = false;
    if (plotRef.value) plotRef.value.redraw();
  }
}

function handleBeforeUnload(ev) {
  ev.preventDefault();
  ev.returnValue = "";
}

function parseCsv(text) {
  const lines = text.trim().split(/\r?\n/);
  const wavelength = [];
  const flux = [];
  const err = [];

  for (let i = 1; i < lines.length; i += 1) {
    const [wave, f, e] = lines[i].split(",").map(Number);
    if (!Number.isFinite(wave)) continue;
    wavelength.push(wave);
    flux.push(Number.isFinite(f) ? f : 0);
    err.push(Number.isFinite(e) ? e : 0);
  }

  return { wavelength, flux, err };
}

async function loadSampleSpectrum() {
  try {
    const response = await fetch("./data.csv");
    if (!response.ok) return;
    const text = await response.text();
    const raw = parseCsv(text);
    addDataset(
      {
        id: "sample-spectrum",
        name: "Sample spectrum",
        raw,
        meta: {
          source: "sample",
          points: raw.wavelength.length,
          xUnit: xUnit.value,
          fluxUnit: fluxUnit.value,
          z: DEFAULT_SAMPLE_Z,
        },
      },
      true,
    );
  } catch (err) {
    console.warn("Could not load sample spectrum", err);
  }
}

async function loadLines() {
  try {
    const response = await fetch("./astro_lines.json");
    if (!response.ok) return;
    const lines = await response.json();
    linesAll.splice(
      0,
      linesAll.length,
      ...lines.map((l, idx) => ({
        id: `line-${idx}`,
        label: l.name ?? l.label,
        rest: l.lambda ?? l.rest,
        show: l.checked ?? true,
        group: l.group ?? deriveGroup(l.name ?? l.label),
      })),
    );
  } catch (err) {
    console.warn("Could not load astro_lines.json, using defaults", err);
  }
}

function convertToBase(raw, waveUnit, fluxUnit) {
  if (!raw) return null;
  const waveUnitLower = (waveUnit || "A").toLowerCase();
  const fluxUnitLower = (fluxUnit || "flambda").toLowerCase();

  const waveA = raw.wavelength.map((w) => {
    if (waveUnitLower === "nm") return w * 10;
    if (waveUnitLower === "um") return w * 1e4;
    return w;
  });

  const fJy = [];
  const fJyUp = [];
  const fJyLo = [];

  for (let i = 0; i < raw.wavelength.length; i += 1) {
    const lamA = waveA[i];
    let jy;
    let jyErr;

    if (fluxUnitLower === "flambda") {
      const fLam = raw.flux[i];
      const eLam = raw.err[i];
      jy = fnuJyFromFlam(fLam, lamA);
      jyErr = fnuJyFromFlam(eLam, lamA);
    } else {
      const factorMap = { jy: 1, mjy: 1e-3, ujy: 1e-6, njy: 1e-9 };
      const factor = factorMap[fluxUnitLower] ?? 1;
      jy = raw.flux[i] * factor;
      jyErr = raw.err[i] * factor;
    }

    fJy.push(jy);
    fJyUp.push(jy + jyErr);
    fJyLo.push(jy - jyErr);
  }

  return { waveA, fJy, fJyUp, fJyLo };
}

function computeDisplay(base) {
  if (!base) return null;
  if (currentMode.value === "flam") {
    const flam = [];
    const flamUp = [];
    const flamLo = [];
    for (let i = 0; i < base.waveA.length; i += 1) {
      flam.push(flamFromFnuJy(base.fJy[i], base.waveA[i]));
      flamUp.push(flamFromFnuJy(base.fJyUp[i], base.waveA[i]));
      flamLo.push(flamFromFnuJy(base.fJyLo[i], base.waveA[i]));
    }
    return {
      x: base.waveA,
      y: flam,
      yUp: flamUp,
      yLo: flamLo,
      yLabel: "Flux (erg s⁻¹ cm⁻² Å⁻¹)",
    };
  }

  return {
    x: base.waveA,
    y: base.fJy,
    yUp: base.fJyUp,
    yLo: base.fJyLo,
    yLabel: "Flux (Jy)",
  };
}

function toggleMode() {
  currentMode.value = currentMode.value === "fnu" ? "flam" : "fnu";
  updatePlot();
}

function toggleSteps() {
  useSteps.value = !useSteps.value;
  updatePlot();
}

function sanitizeZ() {
  if (!Number.isFinite(zValue.value)) {
    zValue.value = 0;
  } else {
    zValue.value = Number(zValue.value.toFixed(4));
  }
}

function openLineModal() {
  newLineName.value = "";
  newLineLambda.value = "";
  newLineUnit.value = "A";
  showLineModal.value = true;
}

function closeLineModal() {
  showLineModal.value = false;
}

function submitLine() {
  const name = newLineName.value.trim();
  const lam = Number.parseFloat(newLineLambda.value);
  const unit = (newLineUnit.value || "A").toLowerCase();
  if (!name || !Number.isFinite(lam) || lam <= 0) return;
  const lamA = unit === "um" ? lam * 1e4 : lam;
  linesAll.push({
    id: `custom-${Date.now()}`,
    label: name,
    rest: lamA,
    show: true,
    group: deriveGroup(name) || "Custom",
  });
  showLineModal.value = false;
  if (plotRef.value) plotRef.value.redraw();
}

function datasetMeta(dataset) {
  const pts = dataset?.raw?.wavelength?.length ?? 0;
  const zVal = Number.isFinite(dataset?.meta?.z) ? dataset.meta.z.toFixed(4) : null;
  const zText = zVal ? ` · z=${zVal}` : "";
  return pts ? `${pts} pts${zText}` : `Ready${zText}`;
}

function setActiveDataset(id) {
  activeDatasetId.value = id;
}

function addDataset(entry, makeActive = false) {
  const existing = datasets.value.find((d) => d.id === entry.id);
  if (existing) {
    Object.assign(existing, entry);
    existing.meta = { ...(existing.meta ?? {}), ...(entry.meta ?? {}) };
  } else {
    datasets.value.push({
      ...entry,
      meta: {
        ...(entry.meta ?? {}),
      },
    });
  }
  if (makeActive || !activeDatasetId.value) {
    activeDatasetId.value = entry.id;
  }

  if (entry.id !== "sample-spectrum") {
    const sampleIdx = datasets.value.findIndex((d) => d.id === "sample-spectrum");
    if (sampleIdx !== -1) {
      const removingActive = activeDatasetId.value === "sample-spectrum";
      datasets.value.splice(sampleIdx, 1);
      if (removingActive) {
        activeDatasetId.value = entry.id;
      }
    }
  }

  updatePlot();
}

function exportRedshifts() {
  if (!datasets.value.length) return;
  const header = "filename,z";
  const rows = datasets.value.map((ds) => {
    const name = (ds.name ?? "unnamed").replace(/"/g, '""');
    const z = Number.isFinite(ds.meta?.z) ? ds.meta.z.toFixed(4) : "0";
    return `"${name}",${z}`;
  });
  const csv = [header, ...rows].join("\n");
  const blob = new Blob([csv], { type: "text/csv;charset=utf-8;" });
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url;
  a.download = "redshifts.csv";
  a.click();
  URL.revokeObjectURL(url);
}

function isGroupOpen(name) {
  if (!(name in groupStates)) groupStates[name] = true;
  return groupStates[name];
}

function toggleGroup(name) {
  groupStates[name] = !isGroupOpen(name);
}

async function ingestFiles(fileList) {
  const files = Array.from(fileList || []).filter((f) => f && f.name.toLowerCase().endsWith(".csv"));
  dragActive.value = false;
  if (files.length === 0) return;
  let first = true;
  for (const file of files) {
    try {
      const text = await file.text();
      const raw = parseCsv(text);
      if (!raw.wavelength.length) continue;
      const id = `${file.name}-${Date.now()}-${Math.random().toString(16).slice(2)}`;
      addDataset(
        {
          id,
          name: file.name,
          raw,
          meta: {
            source: "upload",
            size: file.size,
            points: raw.wavelength.length,
            xUnit: xUnit.value,
            fluxUnit: fluxUnit.value,
            z: DEFAULT_UPLOAD_Z,
          },
        },
        first,
      );
      first = false;
    } catch (err) {
      console.warn("Failed to load file", file?.name, err);
    }
  }
}

function handleFileInput(ev) {
  const files = ev.target.files;
  ingestFiles(files);
  ev.target.value = "";
}

function handleDrop(ev) {
  ingestFiles(ev.dataTransfer?.files);
}

function getXValFromEvent(u, ev) {
  if (!u) return null;
  const rect = u.over.getBoundingClientRect();
  const cssX = ev.clientX - rect.left;
  return u.posToVal(cssX, "x");
}

function attachZDrag(u) {
  if (!u) return;
  u.over.addEventListener(
    "pointerdown",
    (ev) => {
      if (!ev.shiftKey || ev.button !== 0) return;
      const startVal = getXValFromEvent(u, ev);
      if (!Number.isFinite(startVal)) return;
      zDrag = { startVal, startZ: zValue.value };
      ev.preventDefault();
    },
    { passive: false },
  );

  u.over.addEventListener("pointermove", (ev) => {
    if (!zDrag || !ev.shiftKey) return;
    const currentVal = getXValFromEvent(u, ev);
    if (!Number.isFinite(currentVal) || !Number.isFinite(zDrag.startVal) || zDrag.startVal === 0) return;
    const rest = zDrag.startVal / (1 + zDrag.startZ);
    if (!Number.isFinite(rest) || rest === 0) return;
    zValue.value = currentVal / rest - 1;
  });

  u.over.addEventListener("pointerup", () => {
    zDrag = null;
  });

  u.over.addEventListener("pointerleave", () => {
    zDrag = null;
  });
}

function buildPlot(display, data) {
  const rect = plotWrapEl.value?.getBoundingClientRect() ?? { width: 900, height: 420 };

  const opts = {
    width: rect.width,
    height: rect.height,
    scales: {
      x: { time: false},
      y: { auto: true },
    },
    cursor: {
      drag: { x: true, y: true, uni: 50 },
    },
    axes: [
      {
        label: "Wavelength (Å)",
        font: "12px Space Grotesk",
        labelFont: "12px Space Grotesk",
        size: 30,
        labelGap: 6,
      },
      {
        label: display.yLabel,
        font: "12px Space Grotesk",
        labelFont: "12px Space Grotesk",
        incrs: yIncrements,
        size: 70,
        labelGap: 6,
        space: (u, axisIdx, min, max, fullDim) => Math.max(42, fullDim * 0.1),
        splits: (u, axisIdx, min, max) => makeTicks(min, max, yIncrements),
        values: (u, vals) => vals.map((v) => formatAxisFlux(v)),
      },
    ],
    series: [
      {},
      {
        label: "Flux",
        stroke: "#264653",
        width: 2,
        points: { show: false },
        paths: (u, sidx, i0, i1) =>
          useSteps.value ? stepPath(u, sidx, i0, i1) : uPlot.paths.linear()(u, sidx, i0, i1),
      },
      { label: "Upper", stroke: "rgba(42, 157, 143, 0.4)", width: 0, show: false },
      { label: "Lower", stroke: "rgba(42, 157, 143, 0.4)", width: 0, show: false },
    ],
    plugins: [
      bandPlugin({ upperSeries: 2, lowerSeries: 3, fill: "rgba(42, 157, 143, 0.18)" }),
      cursorReadoutPlugin((x, y) => {
        cursorText.value = Number.isFinite(x) && Number.isFinite(y)
          ? `${formatWave(x)}  •  ${formatFlux(y)}`
          : "";
        const rest = Number.isFinite(x) && Number.isFinite(zValue.value) ? x / (1 + zValue.value) : null;
        cursorRestText.value = Number.isFinite(rest) ? `Rest λ: ${rest.toFixed(2)} Å` : "";
      }),
      createZLinesPlugin(),
      hintPlugin("Double-click to start."),
    ],
    hooks: {
      ready: [
        (u) => {
          u.setScale("x", { min: null, max: null });
          u.setScale("y", { min: null, max: null });
        },
      ],
    },
  };

  return new uPlot(opts, [data.x, ...data.ys], chartEl.value);
}

function updatePlot() {
  const display = displayData.value;
  if (!display || !chartEl.value) return;

  const data = useSteps.value
    ? makeStepSeries(display.x, display.y, display.yUp, display.yLo)
    : { x: display.x, ys: [display.y, display.yUp, display.yLo] };

  if (!plotRef.value) {
    plotRef.value = buildPlot(display, data);
    attachZDrag(plotRef.value);
    plotRef.value.setScale("x", { min: null, max: null });
    plotRef.value.setScale("y", { min: null, max: null });
    if (resizeObserver && plotWrapEl.value) {
      resizeObserver.disconnect();
      setupResizeObserver();
    }
    return;
  }

  plotRef.value.series[1].paths = (u, sidx, i0, i1) =>
    useSteps.value ? stepPath(u, sidx, i0, i1) : uPlot.paths.linear()(u, sidx, i0, i1);
  plotRef.value.axes[1].label = display.yLabel;
  plotRef.value.setData([data.x, ...data.ys]);
  plotRef.value.setScale("x", { min: null, max: null });
  plotRef.value.setScale("y", { min: null, max: null });
  plotRef.value.redraw();
}
</script>
