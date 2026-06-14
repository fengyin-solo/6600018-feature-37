<template>
  <div
    ref="viewportRef"
    class="relative w-full h-full overflow-hidden select-none"
    @mousedown="onMouseDown"
    @mousemove="onMouseMove"
    @mouseup="onMouseUp"
    @mouseleave="onMouseUp"
    @wheel.prevent="onWheel"
    @contextmenu.prevent
  >
    <div
      class="absolute top-0 left-0 origin-top-left"
      :style="{
        transform: `translate(${offsetX}px, ${offsetY}px) scale(${scale})`,
        width: `${contentWidth}px`,
        height: `${contentHeight}px`,
      }"
    >
      <img
        v-if="store.currentDoc?.imageUrl"
        :src="store.currentDoc.imageUrl"
        class="absolute top-0 left-0 block"
        :style="{ width: `${contentWidth}px`, height: `${contentHeight}px` }"
        ref="imgRef"
        @load="onImageLoad"
        draggable="false"
      />
      <canvas
        v-else
        ref="mockCanvas"
        class="absolute top-0 left-0 block"
        :width="contentWidth"
        :height="contentHeight"
        :style="{ width: `${contentWidth}px`, height: `${contentHeight}px` }"
      />

      <svg
        class="absolute top-0 left-0 pointer-events-none"
        :width="contentWidth"
        :height="contentHeight"
      >
        <g v-for="r in store.currentDoc?.results || []" :key="r.id">
          <rect
            :x="r.bbox[0]"
            :y="r.bbox[1]"
            :width="r.bbox[2]"
            :height="r.bbox[3]"
            fill="none"
            stroke="rgba(251,191,36,0.6)"
            stroke-width="2"
          />
          <text :x="r.bbox[0]" :y="r.bbox[1] - 5" fill="#fbbf24" font-size="12">
            {{ r.text }}
          </text>
        </g>
        <g v-for="a in store.currentDoc?.annotations || []" :key="a.id">
          <rect
            :x="a.bbox[0]"
            :y="a.bbox[1]"
            :width="a.bbox[2]"
            :height="a.bbox[3]"
            fill="rgba(59,130,246,0.15)"
            stroke="#3b82f6"
            stroke-width="2"
            stroke-dasharray="5,5"
          />
          <text :x="a.bbox[0]" :y="a.bbox[1] - 5" fill="#3b82f6" font-size="11">
            {{ a.label }}
          </text>
        </g>
        <rect
          v-if="isSelecting"
          :x="selectionImageRect.x"
          :y="selectionImageRect.y"
          :width="selectionImageRect.w"
          :height="selectionImageRect.h"
          fill="rgba(16,185,129,0.1)"
          stroke="#10b981"
          stroke-width="2"
          stroke-dasharray="4,4"
        />
      </svg>
    </div>

    <div class="absolute bottom-4 right-4 flex items-center gap-2 bg-gray-800/90 backdrop-blur rounded-lg px-3 py-2 shadow-lg">
      <button
        @click="zoomOut"
        class="w-7 h-7 flex items-center justify-center bg-gray-700 rounded hover:bg-gray-600 text-white text-lg font-bold transition-colors"
        title="缩小"
      >
        −
      </button>
      <span class="w-14 text-center text-white text-xs font-medium tabular-nums">
        {{ Math.round(scale * 100) }}%
      </span>
      <button
        @click="zoomIn"
        class="w-7 h-7 flex items-center justify-center bg-gray-700 rounded hover:bg-gray-600 text-white text-lg font-bold transition-colors"
        title="放大"
      >
        +
      </button>
      <div class="w-px h-5 bg-gray-600 mx-1"></div>
      <button
        @click="resetView"
        class="px-2 h-7 flex items-center justify-center bg-gray-700 rounded hover:bg-gray-600 text-white text-xs transition-colors"
        title="重置视图"
      >
        适应
      </button>
    </div>

    <div class="absolute top-4 left-4 bg-gray-800/80 backdrop-blur rounded px-2 py-1 text-gray-400 text-xs">
      滚轮缩放 · 中键拖拽平移 · 左键框选标注
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, reactive, onMounted, nextTick, watch } from 'vue'
import { useOcrStore } from '../store/ocr'

const store = useOcrStore()
const viewportRef = ref<HTMLDivElement | null>(null)
const mockCanvas = ref<HTMLCanvasElement | null>(null)
const imgRef = ref<HTMLImageElement | null>(null)

const scale = ref(1)
const offsetX = ref(0)
const offsetY = ref(0)

const contentWidth = ref(1000)
const contentHeight = ref(700)

const isPanning = ref(false)
const isSelecting = ref(false)
const panStart = reactive({ x: 0, y: 0 })
const panOffsetStart = reactive({ x: 0, y: 0 })

const selectionStart = reactive({ x: 0, y: 0 })
const selectionImageRect = reactive({ x: 0, y: 0, w: 0, h: 0 })

const MIN_SCALE = 0.1
const MAX_SCALE = 5
const ZOOM_STEP = 0.1

onMounted(() => {
  if (mockCanvas.value) {
    drawMockCanvas()
  }
  nextTick(() => {
    fitToContainer()
  })
})

watch(
  () => store.currentDoc?.id,
  () => {
    nextTick(() => {
      if (store.currentDoc?.imageUrl && imgRef.value?.complete) {
        onImageLoad()
      } else if (!store.currentDoc?.imageUrl && mockCanvas.value) {
        drawMockCanvas()
        fitToContainer()
      }
    })
  }
)

function drawMockCanvas() {
  if (!mockCanvas.value) return
  const ctx = mockCanvas.value.getContext('2d')!
  const w = contentWidth.value
  const h = contentHeight.value

  ctx.fillStyle = '#f5e6c8'
  ctx.fillRect(0, 0, w, h)

  for (let i = 0; i < 5000; i++) {
    ctx.fillStyle = `rgba(139,90,43,${Math.random() * 0.1})`
    ctx.fillRect(Math.random() * w, Math.random() * h, 2, 2)
  }

  ctx.fillStyle = '#2d1810'
  ctx.font = 'bold 48px serif'
  const columns = [
    ['子', '曰', '學', '而', '時', '習', '之', '不', '亦', '説', '乎'],
    ['有', '朋', '自', '遠', '方', '來', '不', '亦', '樂', '乎'],
    ['人', '不', '知', '而', '不', '慍', '不', '亦', '君', '子', '乎'],
  ]
  columns.forEach((col, ci) => {
    const x = w - 150 - ci * 180
    col.forEach((ch, ri) => {
      ctx.fillText(ch, x, 80 + ri * 65)
    })
  })
}

function onImageLoad() {
  if (!imgRef.value) return
  contentWidth.value = imgRef.value.naturalWidth
  contentHeight.value = imgRef.value.naturalHeight
  fitToContainer()
}

function fitToContainer() {
  if (!viewportRef.value) return
  const vw = viewportRef.value.clientWidth
  const vh = viewportRef.value.clientHeight
  if (vw === 0 || vh === 0) return

  const scaleX = vw / contentWidth.value
  const scaleY = vh / contentHeight.value
  const newScale = Math.min(scaleX, scaleY, 1)

  const scaledW = contentWidth.value * newScale
  const scaledH = contentHeight.value * newScale

  scale.value = newScale
  offsetX.value = (vw - scaledW) / 2
  offsetY.value = (vh - scaledH) / 2
}

function screenToImage(sx: number, sy: number): [number, number] {
  return [(sx - offsetX.value) / scale.value, (sy - offsetY.value) / scale.value]
}

function onMouseDown(e: MouseEvent) {
  if (!viewportRef.value) return
  const rect = viewportRef.value.getBoundingClientRect()
  const sx = e.clientX - rect.left
  const sy = e.clientY - rect.top

  if (e.button === 1) {
    isPanning.value = true
    isSelecting.value = false
    panStart.x = sx
    panStart.y = sy
    panOffsetStart.x = offsetX.value
    panOffsetStart.y = offsetY.value
    viewportRef.value.style.cursor = 'grabbing'
    e.preventDefault()
  } else if (e.button === 0) {
    isSelecting.value = true
    isPanning.value = false
    selectionStart.x = sx
    selectionStart.y = sy
    const [ix, iy] = screenToImage(sx, sy)
    selectionImageRect.x = ix
    selectionImageRect.y = iy
    selectionImageRect.w = 0
    selectionImageRect.h = 0
  }
}

function onMouseMove(e: MouseEvent) {
  if (!viewportRef.value) return
  const rect = viewportRef.value.getBoundingClientRect()
  const sx = e.clientX - rect.left
  const sy = e.clientY - rect.top

  if (isPanning.value) {
    offsetX.value = panOffsetStart.x + (sx - panStart.x)
    offsetY.value = panOffsetStart.y + (sy - panStart.y)
  } else if (isSelecting.value) {
    const [startIx, startIy] = screenToImage(selectionStart.x, selectionStart.y)
    const [curIx, curIy] = screenToImage(sx, sy)
    selectionImageRect.x = Math.min(startIx, curIx)
    selectionImageRect.y = Math.min(startIy, curIy)
    selectionImageRect.w = Math.abs(curIx - startIx)
    selectionImageRect.h = Math.abs(curIy - startIy)
  }
}

function onMouseUp(e: MouseEvent) {
  if (isPanning.value) {
    isPanning.value = false
    if (viewportRef.value) {
      viewportRef.value.style.cursor = ''
    }
  }

  if (isSelecting.value) {
    isSelecting.value = false
    if (selectionImageRect.w > 10 && selectionImageRect.h > 10) {
      const label = prompt('标注标签（如：章节/段落/异体字）') || 'region'
      const content = prompt('标注内容') || ''
      const bbox: [number, number, number, number] = [
        selectionImageRect.x,
        selectionImageRect.y,
        selectionImageRect.w,
        selectionImageRect.h,
      ]
      store.addAnnotation('region', bbox, label, content)
    }
  }
}

function onWheel(e: WheelEvent) {
  if (!viewportRef.value) return
  const rect = viewportRef.value.getBoundingClientRect()
  const sx = e.clientX - rect.left
  const sy = e.clientY - rect.top

  const [ix, iy] = screenToImage(sx, sy)

  const delta = e.deltaY > 0 ? -ZOOM_STEP : ZOOM_STEP
  const newScale = Math.max(MIN_SCALE, Math.min(MAX_SCALE, scale.value + delta * scale.value))

  offsetX.value = sx - ix * newScale
  offsetY.value = sy - iy * newScale
  scale.value = newScale
}

function zoomIn() {
  if (!viewportRef.value) return
  const cx = viewportRef.value.clientWidth / 2
  const cy = viewportRef.value.clientHeight / 2
  const [ix, iy] = screenToImage(cx, cy)
  const newScale = Math.min(MAX_SCALE, scale.value * 1.2)
  offsetX.value = cx - ix * newScale
  offsetY.value = cy - iy * newScale
  scale.value = newScale
}

function zoomOut() {
  if (!viewportRef.value) return
  const cx = viewportRef.value.clientWidth / 2
  const cy = viewportRef.value.clientHeight / 2
  const [ix, iy] = screenToImage(cx, cy)
  const newScale = Math.max(MIN_SCALE, scale.value / 1.2)
  offsetX.value = cx - ix * newScale
  offsetY.value = cy - iy * newScale
  scale.value = newScale
}

function resetView() {
  fitToContainer()
}
</script>
