<script setup lang="ts">
import { ref, onMounted, watch, nextTick, onUnmounted } from 'vue';
import { useFEAStore } from '../store/fea';

const store = useFEAStore();
const canvas = ref<HTMLCanvasElement>();
const container = ref<HTMLDivElement>();

let viewScale = 1;
let viewOffsetX = 0;
let viewOffsetY = 0;
let baseScale = 1;
let isDragging = false;
let lastMouse = { x: 0, y: 0 };
let didPan = false;
const PAN_THRESHOLD = 3;

const scaleDisplay = ref(100);
const showScaleHint = ref(false);
let scaleHintTimer: number | null = null;
let dpr = 1;

const MIN_ZOOM = 0.2;
const MAX_ZOOM = 5;
const SCALE_STEP = 1.1;

function getWorldBounds() {
  const { nodes } = store.model;
  if (nodes.length === 0) {
    return { minX: 0, maxX: 1, minY: 0, maxY: 1 };
  }
  let minX = Infinity, maxX = -Infinity, minY = Infinity, maxY = -Infinity;
  for (const n of nodes) {
    minX = Math.min(minX, n.x);
    maxX = Math.max(maxX, n.x);
    minY = Math.min(minY, n.y);
    maxY = Math.max(maxY, n.y);
  }
  return { minX, maxX, minY, maxY };
}

function worldToScreen(x: number, y: number): [number, number] {
  return [x * viewScale + viewOffsetX, y * viewScale + viewOffsetY];
}

function screenToWorld(sx: number, sy: number): [number, number] {
  return [(sx - viewOffsetX) / viewScale, (sy - viewOffsetY) / viewScale];
}

function fitView() {
  if (!canvas.value) return;
  const W = canvas.value.width / dpr;
  const H = canvas.value.height / dpr;
  const { minX, maxX, minY, maxY } = getWorldBounds();
  const worldW = maxX - minX || 1;
  const worldH = maxY - minY || 1;
  const margin = 80;
  const fitScale = Math.min((W - margin * 2) / worldW, (H - margin * 2) / worldH);
  baseScale = fitScale;
  viewScale = fitScale;
  viewOffsetX = margin - minX * viewScale + (W - margin * 2 - worldW * viewScale) / 2;
  viewOffsetY = margin - minY * viewScale + (H - margin * 2 - worldH * viewScale) / 2;
  updateScaleDisplay();
}

function clampView() {
  if (!canvas.value) return;
  const W = canvas.value.width / dpr;
  const H = canvas.value.height / dpr;
  const { minX, maxX, minY, maxY } = getWorldBounds();

  const [sx1, sy1] = worldToScreen(minX, minY);
  const [sx2, sy2] = worldToScreen(maxX, maxY);
  const modelLeft = Math.min(sx1, sx2);
  const modelRight = Math.max(sx1, sx2);
  const modelTop = Math.min(sy1, sy2);
  const modelBottom = Math.max(sy1, sy2);
  const modelW = modelRight - modelLeft;
  const modelH = modelBottom - modelTop;

  const minVisibleRatio = 0.3;

  if (modelW <= W * minVisibleRatio) {
    const maxOffsetX = W - modelW * minVisibleRatio;
    const minOffsetX = -modelW * (1 - minVisibleRatio);
    viewOffsetX = Math.max(minOffsetX, Math.min(maxOffsetX, viewOffsetX));
  } else {
    const maxOffsetX = W * minVisibleRatio;
    const minOffsetX = -(modelW - W * minVisibleRatio);
    viewOffsetX = Math.max(minOffsetX, Math.min(maxOffsetX, viewOffsetX));
  }

  if (modelH <= H * minVisibleRatio) {
    const maxOffsetY = H - modelH * minVisibleRatio;
    const minOffsetY = -modelH * (1 - minVisibleRatio);
    viewOffsetY = Math.max(minOffsetY, Math.min(maxOffsetY, viewOffsetY));
  } else {
    const maxOffsetY = H * minVisibleRatio;
    const minOffsetY = -(modelH - H * minVisibleRatio);
    viewOffsetY = Math.max(minOffsetY, Math.min(maxOffsetY, viewOffsetY));
  }
}

function updateScaleDisplay() {
  scaleDisplay.value = Math.round((viewScale / baseScale) * 100);
}

function showScaleIndicator() {
  showScaleHint.value = true;
  if (scaleHintTimer) {
    clearTimeout(scaleHintTimer);
  }
  scaleHintTimer = window.setTimeout(() => {
    showScaleHint.value = false;
  }, 800);
}

function zoomAt(screenX: number, screenY: number, factor: number) {
  const [worldX, worldY] = screenToWorld(screenX, screenY);
  const newScale = viewScale * factor;
  const minScale = baseScale * MIN_ZOOM;
  const maxScale = baseScale * MAX_ZOOM;
  viewScale = Math.max(minScale, Math.min(maxScale, newScale));
  const [newScreenX, newScreenY] = worldToScreen(worldX, worldY);
  viewOffsetX += screenX - newScreenX;
  viewOffsetY += screenY - newScreenY;
  clampView();
  updateScaleDisplay();
  showScaleIndicator();
}

function zoomIn() {
  if (!canvas.value) return;
  const W = canvas.value.width / dpr;
  const H = canvas.value.height / dpr;
  zoomAt(W / 2, H / 2, SCALE_STEP);
  draw();
}

function zoomOut() {
  if (!canvas.value) return;
  const W = canvas.value.width / dpr;
  const H = canvas.value.height / dpr;
  zoomAt(W / 2, H / 2, 1 / SCALE_STEP);
  draw();
}

function resetView() {
  fitView();
  draw();
}

function draw() {
  const ctx = canvas.value?.getContext('2d');
  if (!ctx) return;

  const W = canvas.value!.width / dpr;
  const H = canvas.value!.height / dpr;

  ctx.save();
  ctx.setTransform(dpr, 0, 0, dpr, 0, 0);

  ctx.clearRect(0, 0, W, H);
  ctx.fillStyle = '#0f172a';
  ctx.fillRect(0, 0, W, H);

  drawGrid(ctx, W, H);

  const { nodes, elements, loads } = store.model;
  if (nodes.length === 0) {
    ctx.fillStyle = '#94a3b8';
    ctx.font = '14px sans-serif';
    ctx.textAlign = 'center';
    ctx.fillText('选择一个预设模型开始分析', W / 2, H / 2);
    return;
  }

  function toScreen(x: number, y: number): [number, number] {
    return worldToScreen(x, y);
  }

  for (const el of elements) {
    const n1 = nodes.find((n) => n.id === el.nodeIds[0]);
    const n2 = nodes.find((n) => n.id === el.nodeIds[1]);
    if (!n1 || !n2) continue;

    const [x1, y1] = toScreen(n1.x, n1.y);
    const [x2, y2] = toScreen(n2.x, n2.y);
    const color = store.elementColors.get(el.id) || '#6b7280';
    const isSelected = store.selectedElement === el.id;

    ctx.beginPath();
    ctx.moveTo(x1, y1);
    ctx.lineTo(x2, y2);
    ctx.strokeStyle = color;
    ctx.lineWidth = isSelected ? 4 : 2.5;
    ctx.stroke();

    if (isSelected) {
      ctx.strokeStyle = '#ffffff';
      ctx.lineWidth = 1;
      ctx.setLineDash([4, 3]);
      ctx.beginPath();
      ctx.moveTo(x1, y1);
      ctx.lineTo(x2, y2);
      ctx.stroke();
      ctx.setLineDash([]);
    }
  }

  if (store.showDeformed && store.result) {
    ctx.setLineDash([5, 3]);
    for (const el of elements) {
      const n1 = nodes.find((n) => n.id === el.nodeIds[0]);
      const n2 = nodes.find((n) => n.id === el.nodeIds[1]);
      if (!n1 || !n2) continue;

      const s = store.deformationScale;
      const [x1, y1] = toScreen(n1.x + n1.displacementX * s, n1.y + n1.displacementY * s);
      const [x2, y2] = toScreen(n2.x + n2.displacementX * s, n2.y + n2.displacementY * s);

      ctx.beginPath();
      ctx.moveTo(x1, y1);
      ctx.lineTo(x2, y2);
      ctx.strokeStyle = 'rgba(251,191,36,0.5)';
      ctx.lineWidth = 1.5;
      ctx.stroke();
    }
    ctx.setLineDash([]);
  }

  for (const node of nodes) {
    const [x, y] = toScreen(node.x, node.y);

    if (node.fixed) {
      ctx.beginPath();
      ctx.moveTo(x, y - 8);
      ctx.lineTo(x - 6, y + 4);
      ctx.lineTo(x + 6, y + 4);
      ctx.closePath();
      ctx.fillStyle = '#f97316';
      ctx.fill();
      ctx.strokeStyle = '#ea580c';
      ctx.lineWidth = 1.5;
      ctx.stroke();

      ctx.strokeStyle = '#f97316';
      ctx.lineWidth = 1;
      for (let i = -8; i <= 8; i += 4) {
        ctx.beginPath();
        ctx.moveTo(x + i, y + 5);
        ctx.lineTo(x + i - 3, y + 10);
        ctx.stroke();
      }
    } else {
      ctx.beginPath();
      ctx.arc(x, y, 4, 0, Math.PI * 2);
      ctx.fillStyle = '#e2e8f0';
      ctx.fill();
      ctx.strokeStyle = '#64748b';
      ctx.lineWidth = 1;
      ctx.stroke();
    }
  }

  for (const load of loads) {
    const node = nodes.find((n) => n.id === load.nodeId);
    if (!node) continue;
    const [x, y] = toScreen(node.x, node.y);

    const mag = Math.sqrt(load.fx ** 2 + load.fy ** 2);
    if (mag === 0) continue;

    const arrowLen = 30;
    const dx = (load.fx / mag) * arrowLen;
    const dy = (load.fy / mag) * arrowLen;

    ctx.beginPath();
    ctx.moveTo(x - dx, y - dy);
    ctx.lineTo(x, y);
    ctx.strokeStyle = '#ef4444';
    ctx.lineWidth = 2.5;
    ctx.stroke();

    const headLen = 8;
    const angle = Math.atan2(dy, dx);
    ctx.beginPath();
    ctx.moveTo(x, y);
    ctx.lineTo(x - headLen * Math.cos(angle - 0.4), y - headLen * Math.sin(angle - 0.4));
    ctx.moveTo(x, y);
    ctx.lineTo(x - headLen * Math.cos(angle + 0.4), y - headLen * Math.sin(angle + 0.4));
    ctx.strokeStyle = '#ef4444';
    ctx.lineWidth = 2;
    ctx.stroke();

    ctx.fillStyle = '#fca5a5';
    ctx.font = '10px sans-serif';
    ctx.textAlign = 'center';
    ctx.fillText(`${(mag / 1000).toFixed(1)}kN`, x - dx / 2, y - dy / 2 - 6);
  }

  const legendX = W - 40;
  const legendY = 30;
  const legendH = H - 60;
  const legendW = 15;

  const gradient = ctx.createLinearGradient(0, legendY, 0, legendY + legendH);
  gradient.addColorStop(0, 'rgb(255,0,0)');
  gradient.addColorStop(0.25, 'rgb(255,255,0)');
  gradient.addColorStop(0.5, 'rgb(0,255,0)');
  gradient.addColorStop(0.75, 'rgb(0,255,255)');
  gradient.addColorStop(1, 'rgb(0,0,128)');

  ctx.fillStyle = gradient;
  ctx.fillRect(legendX, legendY, legendW, legendH);
  ctx.strokeStyle = '#475569';
  ctx.lineWidth = 1;
  ctx.strokeRect(legendX, legendY, legendW, legendH);

  ctx.fillStyle = '#94a3b8';
  ctx.font = '10px sans-serif';
  ctx.textAlign = 'left';

  let maxVal = 0;
  if (store.result) {
    switch (store.heatmapMode) {
      case 'stress':
        maxVal = Math.max(...store.result.stresses.map(Math.abs));
        break;
      case 'strain':
        maxVal = Math.max(...store.result.strains.map(Math.abs));
        break;
      case 'force':
        maxVal = Math.max(...elements.map((e) => Math.abs(e.force)));
        break;
    }
  }

  const unit = store.heatmapMode === 'stress' ? 'MPa' :
    store.heatmapMode === 'strain' ? '%' : 'kN';

  ctx.textAlign = 'right';
  ctx.fillText(`${maxVal.toExponential(1)} ${unit}`, legendX - 4, legendY + 8);
  ctx.fillText('0', legendX - 4, legendY + legendH);

  ctx.save();
  ctx.translate(legendX + legendW + 10, legendY + legendH / 2);
  ctx.rotate(-Math.PI / 2);
  ctx.textAlign = 'center';
  ctx.fillStyle = '#64748b';
  ctx.font = '11px sans-serif';
  ctx.fillText(store.heatmapMode.toUpperCase(), 0, 0);
  ctx.restore();

  ctx.restore();
}

function drawGrid(ctx: CanvasRenderingContext2D, W: number, H: number) {
  ctx.strokeStyle = 'rgba(71, 85, 105, 0.3)';
  ctx.lineWidth = 0.5;

  const gridSize = 50;
  const startX = viewOffsetX % gridSize;
  const startY = viewOffsetY % gridSize;

  for (let x = startX; x < W; x += gridSize) {
    ctx.beginPath();
    ctx.moveTo(x, 0);
    ctx.lineTo(x, H);
    ctx.stroke();
  }

  for (let y = startY; y < H; y += gridSize) {
    ctx.beginPath();
    ctx.moveTo(0, y);
    ctx.lineTo(W, y);
    ctx.stroke();
  }
}

function getCanvasMouse(e: MouseEvent): { x: number; y: number } {
  const rect = canvas.value!.getBoundingClientRect();
  return {
    x: e.clientX - rect.left,
    y: e.clientY - rect.top,
  };
}

function handleMouseDown(e: MouseEvent) {
  if (e.button !== 0) return;
  isDragging = true;
  didPan = false;
  const m = getCanvasMouse(e);
  lastMouse = { x: m.x, y: m.y };
  if (canvas.value) {
    canvas.value.style.cursor = 'grabbing';
  }
}

function handleMouseMove(e: MouseEvent) {
  if (!isDragging) return;
  const m = getCanvasMouse(e);
  const dx = m.x - lastMouse.x;
  const dy = m.y - lastMouse.y;
  if (Math.abs(dx) > PAN_THRESHOLD || Math.abs(dy) > PAN_THRESHOLD) {
    didPan = true;
  }
  viewOffsetX += dx;
  viewOffsetY += dy;
  clampView();
  lastMouse = { x: m.x, y: m.y };
  draw();
}

function handleMouseUp() {
  isDragging = false;
  if (canvas.value) {
    canvas.value.style.cursor = 'grab';
  }
}

function handleWheel(e: WheelEvent) {
  e.preventDefault();
  const m = getCanvasMouse(e);
  const factor = e.deltaY > 0 ? 1 / SCALE_STEP : SCALE_STEP;
  zoomAt(m.x, m.y, factor);
  draw();
}

function handleClick(e: MouseEvent) {
  if (didPan) {
    return;
  }

  const m = getCanvasMouse(e);
  const mx = m.x;
  const my = m.y;

  const { nodes, elements } = store.model;
  let bestDist = 15;
  let bestId: number | null = null;

  for (const el of elements) {
    const n1 = nodes.find((n) => n.id === el.nodeIds[0]);
    const n2 = nodes.find((n) => n.id === el.nodeIds[1]);
    if (!n1 || !n2) continue;

    const [x1, y1] = worldToScreen(n1.x, n1.y);
    const [x2, y2] = worldToScreen(n2.x, n2.y);

    const dx = x2 - x1;
    const dy = y2 - y1;
    const len2 = dx * dx + dy * dy;
    if (len2 === 0) continue;
    let t = ((mx - x1) * dx + (my - y1) * dy) / len2;
    t = Math.max(0, Math.min(1, t));
    const px = x1 + t * dx;
    const py = y1 + t * dy;
    const dist = Math.sqrt((mx - px) ** 2 + (my - py) ** 2);

    if (dist < bestDist) {
      bestDist = dist;
      bestId = el.id;
    }
  }

  store.selectElement(bestId);
  draw();
}

function handleResize() {
  if (!canvas.value || !container.value) return;
  const rect = container.value.getBoundingClientRect();
  dpr = window.devicePixelRatio || 1;
  canvas.value.width = rect.width * dpr;
  canvas.value.height = rect.height * dpr;
  canvas.value.style.width = `${rect.width}px`;
  canvas.value.style.height = `${rect.height}px`;
  fitView();
  draw();
}

onMounted(() => {
  nextTick(() => {
    handleResize();
    window.addEventListener('resize', handleResize);
    if (canvas.value) {
      canvas.value.style.cursor = 'grab';
    }
  });
});

onUnmounted(() => {
  window.removeEventListener('resize', handleResize);
  if (scaleHintTimer) {
    clearTimeout(scaleHintTimer);
  }
});

watch(
  () => store.model,
  () => {
    nextTick(() => {
      fitView();
      draw();
    });
  },
  { deep: true }
);

watch(
  () => [
    store.result,
    store.showDeformed,
    store.deformationScale,
    store.selectedElement,
    store.heatmapMode,
    store.elementColors,
  ],
  () => {
    nextTick(draw);
  },
  { deep: true }
);

defineExpose({
  zoomIn,
  zoomOut,
  resetView,
});
</script>

<template>
  <div ref="container" class="relative w-full h-full">
    <canvas
      ref="canvas"
      class="rounded-lg border border-slate-700 select-none"
      @mousedown="handleMouseDown"
      @mousemove="handleMouseMove"
      @mouseup="handleMouseUp"
      @mouseleave="handleMouseUp"
      @wheel="handleWheel"
      @click="handleClick"
    />

    <div
      v-if="showScaleHint"
      class="absolute top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 pointer-events-none"
    >
      <div class="bg-slate-800/90 text-slate-200 px-4 py-2 rounded-lg text-sm font-mono backdrop-blur-sm border border-slate-600 transition-opacity duration-200">
        {{ scaleDisplay }}%
      </div>
    </div>

    <div class="absolute bottom-3 right-3 flex flex-col gap-1">
      <button
        @click="zoomIn"
        class="w-8 h-8 bg-slate-800/90 hover:bg-slate-700 text-slate-300 rounded border border-slate-600 text-lg font-bold flex items-center justify-center backdrop-blur-sm transition-colors"
        title="放大"
      >
        +
      </button>
      <button
        @click="zoomOut"
        class="w-8 h-8 bg-slate-800/90 hover:bg-slate-700 text-slate-300 rounded border border-slate-600 text-lg font-bold flex items-center justify-center backdrop-blur-sm transition-colors"
        title="缩小"
      >
        −
      </button>
      <button
        @click="resetView"
        class="w-8 h-8 bg-slate-800/90 hover:bg-slate-700 text-slate-300 rounded border border-slate-600 text-xs flex items-center justify-center backdrop-blur-sm transition-colors"
        title="重置视图"
      >
        ⟲
      </button>
    </div>

    <div class="absolute bottom-3 left-3 bg-slate-800/80 text-slate-400 px-2 py-1 rounded text-xs backdrop-blur-sm border border-slate-700">
      缩放: {{ scaleDisplay }}%
    </div>
  </div>
</template>
