<template>
  <div class="editor-wrapper">
    <!-- ── 左侧划线面板 ── -->
    <Transition name="panel-slide">
      <div v-if="marksPanel.length > 0" class="marks-panel">
        <div class="marks-panel__header">
          <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <path d="M12 20h9"/><path d="M16.5 3.5a2.121 2.121 0 0 1 3 3L7 19l-4 1 1-4L16.5 3.5z"/>
          </svg>
          <span>划线记录</span>
          <span class="marks-panel__count">{{ marksPanel.length }}</span>
        </div>
        <div class="marks-panel__list">
          <div
            v-for="(item, idx) in marksPanel"
            :key="item.id"
            class="mark-item"
            :class="{ 'is-focused': focusedMarkId === item.id }"
            @click="scrollToMark(item)"
          >
            <span class="mark-item__index">{{ idx + 1 }}</span>
            <span class="mark-item__dot" :class="`dot-${item.type}`" />
            <span class="mark-item__text">{{ item.text }}</span>
            <span class="mark-item__badge" :class="`badge-${item.type}`">{{ LINE_STYLE_LABEL[item.type] }}</span>
          </div>
        </div>
      </div>
    </Transition>

    <!-- ── 浮动划线工具栏 ── -->
    <Transition name="toolbar-fade">
      <div
        v-if="toolbarVisible"
        class="float-toolbar"
        :style="toolbarStyle"
        @mousedown.prevent
      >
        <button
          v-for="style in LINE_STYLES"
          :key="style.type"
          class="toolbar-btn"
          :class="{ 'is-active': activeLineType === style.type }"
          :title="style.label"
          @click="handleUnderline(style.type)"
        >
          <span class="btn-preview" :class="`preview-${style.type}`">A</span>
          <span>{{ style.label }}</span>
        </button>

        <div class="toolbar-divider" />

        <button class="toolbar-btn toolbar-btn--cancel" title="取消划线" @click="handleCancelUnderline">
          <svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5">
            <line x1="4" y1="12" x2="20" y2="12"/>
            <line x1="4" y1="6" x2="8" y2="6"/><line x1="16" y1="6" x2="20" y2="6"/>
            <line x1="4" y1="18" x2="8" y2="18"/><line x1="16" y1="18" x2="20" y2="18"/>
          </svg>
          <span>取消</span>
        </button>
      </div>
    </Transition>

    <div class="main-body">
      <!-- 预览区域 -->
      <div
        ref="previewRef"
        class="preview-area"
        @mouseup="onMouseUp"
        @touchend="onMouseUp"
      >
        <div class="preview-content" v-html="renderedHtml" />
      </div>

      <!-- 编辑器区域 -->
      <div class="editor-area">
        <div class="editor-label">编辑区</div>
        <MdEditor
          v-model="mdContent"
          :preview="false"
          theme="light"
          class="md-editor-instance"
          @onSave="syncContent"
        />
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, onBeforeUnmount, nextTick } from 'vue'
import { MdEditor } from 'md-editor-v3'
import 'md-editor-v3/lib/style.css'
import MarkdownIt from 'markdown-it'

// ─── 类型 ────────────────────────────────────────────────────────────────────
type LineType = 'single' | 'double' | 'wavy'

interface URange {
  start: number
  end: number
}

type LineRangeMap = Record<LineType, URange[]>

interface SelectionInfo {
  selStart: number
  selEnd: number
  rect: DOMRect
}

interface MarkItem {
  id: string           // 对应 span 的 data-mark-id
  type: LineType
  text: string         // 划线文字
  paraIndex: number    // 段落在 preview-content 中的顺序（用于排序）
}

// ─── 常量 ────────────────────────────────────────────────────────────────────
const LINE_STYLES: { type: LineType; label: string }[] = [
  { type: 'single', label: '单划线' },
  { type: 'double', label: '双划线' },
  { type: 'wavy',   label: '波浪线' },
]

const LINE_STYLE_LABEL: Record<LineType, string> = {
  single: '单线',
  double: '双线',
  wavy:   '波浪',
}

// ─── 状态 ────────────────────────────────────────────────────────────────────
const mdContent = ref<string>('你真牛我很服\n\n这是第二段内容，可以在这里测试三种划线样式。\n\n第三段内容也可以划线标注。')
const previewRef = ref<HTMLElement | null>(null)
const toolbarVisible = ref(false)
const toolbarLeft = ref(0)
const toolbarTop = ref(0)
const arrowLeft = ref(0)

const currentParagraph = ref<HTMLElement | null>(null)
const lineRangeMap = ref<LineRangeMap>({ single: [], double: [], wavy: [] })
const pendingSelection = ref<SelectionInfo | null>(null)
const activeLineType = ref<LineType | null>(null)

// 左侧面板
const marksPanel = ref<MarkItem[]>([])
const focusedMarkId = ref<string | null>(null)

let markIdCounter = 0
const md = new MarkdownIt()

// ─── 计算属性 ────────────────────────────────────────────────────────────────
const toolbarStyle = computed(() => ({
  left: `${toolbarLeft.value}px`,
  top:  `${toolbarTop.value}px`,
}))

const renderedHtml = computed(() => md.render(mdContent.value))

// ─── 鼠标抬起：检测选区 ──────────────────────────────────────────────────────
function onMouseUp() {
  nextTick(() => {
    const sel = window.getSelection()
    if (!sel || sel.isCollapsed || !sel.rangeCount) {
      toolbarVisible.value = false
      pendingSelection.value = null
      return
    }

    const range = sel.getRangeAt(0)
    const preview = previewRef.value
    if (!preview || !preview.contains(range.commonAncestorContainer)) {
      toolbarVisible.value = false
      return
    }

    const para = findParagraph(range.commonAncestorContainer)
    if (!para) { toolbarVisible.value = false; return }

    currentParagraph.value = para

    const selStart = getTextOffset(para, range.startContainer, range.startOffset)
    const selEnd   = getTextOffset(para, range.endContainer,   range.endOffset)
    if (selStart >= selEnd) { toolbarVisible.value = false; return }

    const rect = range.getBoundingClientRect()
    pendingSelection.value = { selStart, selEnd, rect }
    lineRangeMap.value = getLineRangeMap(para)
    activeLineType.value = detectActiveType(selStart, selEnd, lineRangeMap.value)

    positionToolbar(rect)
    toolbarVisible.value = true
  })
}

function findParagraph(node: Node): HTMLElement | null {
  let n: Node | null = node
  while (n && n !== previewRef.value) {
    if (n.nodeType === Node.ELEMENT_NODE) {
      const el = n as HTMLElement
      if (['P', 'H1', 'H2', 'H3', 'H4', 'H5', 'H6', 'LI'].includes(el.tagName)) return el
    }
    n = n.parentNode
  }
  return null
}

function positionToolbar(rect: DOMRect) {
  // 使用视口坐标（position:fixed），完全不受滚动影响
  const TOOLBAR_W = 280
  const TOOLBAR_H = 40
  const GAP = 10

  const centerX = rect.left + rect.width / 2
  const viewportW = window.innerWidth
  let left = centerX - TOOLBAR_W / 2
  left = Math.max(8, Math.min(left, viewportW - TOOLBAR_W - 8))

  // 上方放不下时显示在选区下方
  let top = rect.top - TOOLBAR_H - GAP
  if (top < 8) top = rect.bottom + GAP

  toolbarLeft.value = left
  toolbarTop.value = top
  arrowLeft.value = Math.min(Math.max(centerX - left - 5, 12), TOOLBAR_W - 24)
}

// ─── DOM 解析 ────────────────────────────────────────────────────────────────
function getLineRangeMap(root: HTMLElement): LineRangeMap {
  const map: LineRangeMap = { single: [], double: [], wavy: [] }
  let pos = 0
  function walk(node: Node) {
    if (node.nodeType === Node.TEXT_NODE) {
      pos += (node as Text).length
    } else if (node.nodeType === Node.ELEMENT_NODE) {
      const el = node as HTMLElement
      const uType = el.dataset.u as LineType | undefined
      if (uType && uType in map) {
        const start = pos
        for (const c of Array.from(el.childNodes)) walk(c)
        map[uType].push({ start, end: pos })
      } else {
        for (const c of Array.from(el.childNodes)) walk(c)
      }
    }
  }
  for (const c of Array.from(root.childNodes)) walk(c)
  return map
}

function detectActiveType(selStart: number, selEnd: number, map: LineRangeMap): LineType | null {
  let best: LineType | null = null
  let bestLen = 0
  for (const type of ['single', 'double', 'wavy'] as LineType[]) {
    let covered = 0
    for (const r of map[type]) {
      const lo = Math.max(r.start, selStart)
      const hi = Math.min(r.end, selEnd)
      if (hi > lo) covered += hi - lo
    }
    if (covered > bestLen) { bestLen = covered; best = type }
  }
  return best
}

// ─── 工具函数 ────────────────────────────────────────────────────────────────
function getTextOffset(root: HTMLElement, node: Node, offset: number): number {
  let pos = 0
  const walk = document.createTreeWalker(root, NodeFilter.SHOW_TEXT)
  while (walk.nextNode()) {
    const n = walk.currentNode as Text
    if (n === node) return pos + offset
    pos += n.length
  }
  return pos
}

function mergeRanges(ranges: URange[]): URange[] {
  if (!ranges.length) return []
  const s = [...ranges].sort((a, b) => a.start - b.start)
  const r: URange[] = [{ start: s[0]!.start, end: s[0]!.end }]
  for (let i = 1; i < s.length; i++) {
    const cur = s[i]!
    const last = r[r.length - 1]!
    if (cur.start <= last.end) last.end = Math.max(last.end, cur.end)
    else r.push({ start: cur.start, end: cur.end })
  }
  return r
}

function buildMarked(len: number, ranges: URange[]): boolean[] {
  const m = new Array<boolean>(len).fill(false)
  for (const r of ranges) for (let i = r.start; i < r.end; i++) m[i] = true
  return m
}

// ─── 核心划线逻辑 ────────────────────────────────────────────────────────────
function applyUnderlineLogic(selStart: number, selEnd: number, underlines: URange[], text: string): URange[] {
  const marked = buildMarked(text.length, underlines)
  let hasU = false, hasP = false
  for (let i = selStart; i < selEnd; i++) {
    if (marked[i]) hasU = true; else hasP = true
  }

  if (!hasU) return mergeRanges([...underlines, { start: selStart, end: selEnd }])
  if (!hasP) return underlines.filter(u => !(u.end > selStart && u.start < selEnd))

  const firstMarked = marked[selStart]!
  const lastMarked  = marked[selEnd - 1]!

  if (!firstMarked && lastMarked) {
    const intersecting = underlines.filter(u => u.end > selStart && u.start < selEnd)
    const others = underlines.filter(u => !(u.end > selStart && u.start < selEnd))
    let ns = selStart, ne = selEnd
    for (const u of intersecting) { ns = Math.min(ns, u.start); ne = Math.max(ne, u.end) }
    return mergeRanges([...others, { start: ns, end: ne }])
  }

  if (firstMarked && !lastMarked) {
    const plainSegs: URange[] = []
    let segStart = -1
    for (let i = selStart; i <= selEnd; i++) {
      const isPlain = i < selEnd ? !marked[i] : true
      if (isPlain && segStart === -1) segStart = i
      else if (!isPlain && segStart !== -1) { plainSegs.push({ start: segStart, end: i }); segStart = -1 }
      if (i === selEnd && segStart !== -1) plainSegs.push({ start: segStart, end: i })
    }
    return mergeRanges([...underlines, ...plainSegs])
  }

  const intersecting = underlines.filter(u => u.end > selStart && u.start < selEnd)
  const others = underlines.filter(u => !(u.end > selStart && u.start < selEnd))
  let ns = selStart, ne = selEnd
  for (const u of intersecting) { ns = Math.min(ns, u.start); ne = Math.max(ne, u.end) }
  return mergeRanges([...others, { start: ns, end: ne }])
}

// ─── 重建段落 DOM + 注入 mark-id ─────────────────────────────────────────────
function applyAllRangesToParagraph(para: HTMLElement, map: LineRangeMap) {
  const text = para.textContent ?? ''
  const len = text.length
  const charType = new Array<LineType | null>(len).fill(null)

  for (const type of ['single', 'double', 'wavy'] as LineType[]) {
    for (const r of mergeRanges(map[type])) {
      for (let i = r.start; i < r.end; i++) charType[i] = type
    }
  }

  para.innerHTML = ''
  let i = 0
  while (i < len) {
    const ct = charType[i]
    let j = i + 1
    while (j < len && charType[j] === ct) j++
    const chunk = text.slice(i, j)
    if (ct === null) {
      para.appendChild(document.createTextNode(chunk))
    } else {
      const sp = document.createElement('span')
      sp.dataset.u = ct
      sp.dataset.markId = `mark-${++markIdCounter}`
      sp.textContent = chunk
      para.appendChild(sp)
    }
    i = j
  }
}

// ─── 同步面板：扫描 preview 所有 span[data-u] ────────────────────────────────
function syncMarksPanel() {
  const preview = previewRef.value
  if (!preview) return

  const content = preview.querySelector('.preview-content') as HTMLElement | null
  if (!content) return

  const paras = Array.from(content.children) as HTMLElement[]
  const items: MarkItem[] = []

  paras.forEach((para, paraIndex) => {
    const spans = Array.from(para.querySelectorAll('span[data-u]')) as HTMLElement[]
    spans.forEach(span => {
      const type = span.dataset.u as LineType
      const id   = span.dataset.markId ?? ''
      const text = span.textContent ?? ''
      if (text.trim()) {
        items.push({ id, type, text, paraIndex })
      }
    })
  })

  marksPanel.value = items
}

// ─── 点击面板项 → 滚动定位 + 高亮闪烁 ──────────────────────────────────────
function scrollToMark(item: MarkItem) {
  const preview = previewRef.value
  if (!preview) return

  const span = preview.querySelector(`span[data-mark-id="${item.id}"]`) as HTMLElement | null
  if (!span) return

  focusedMarkId.value = item.id

  // 滚动到目标位置（居中）
  const spanRect    = span.getBoundingClientRect()
  const previewRect = preview.getBoundingClientRect()
  const scrollTarget = preview.scrollTop + (spanRect.top - previewRect.top) - preview.clientHeight / 2 + spanRect.height / 2
  preview.scrollTo({ top: scrollTarget, behavior: 'smooth' })

  // 高亮闪烁
  span.classList.add('mark-flash')
  setTimeout(() => {
    span.classList.remove('mark-flash')
    focusedMarkId.value = null
  }, 1800)
}

// ─── 按钮处理 ────────────────────────────────────────────────────────────────
function handleUnderline(lineType: LineType) {
  if (!pendingSelection.value || !currentParagraph.value) return
  const { selStart, selEnd } = pendingSelection.value
  const para = currentParagraph.value
  const text = para.textContent ?? ''
  const map  = lineRangeMap.value

  const newRanges = applyUnderlineLogic(selStart, selEnd, map[lineType], text)
  const newMap: LineRangeMap = { ...map, [lineType]: newRanges }

  applyAllRangesToParagraph(para, newMap)
  lineRangeMap.value = newMap
  toolbarVisible.value = false
  window.getSelection()?.removeAllRanges()
  nextTick(syncMarksPanel)
}

function handleCancelUnderline() {
  if (!pendingSelection.value || !currentParagraph.value) return
  const { selStart, selEnd } = pendingSelection.value
  const para = currentParagraph.value
  const map  = lineRangeMap.value

  const newMap: LineRangeMap = { single: [], double: [], wavy: [] }
  for (const type of ['single', 'double', 'wavy'] as LineType[]) {
    newMap[type] = mergeRanges(
      map[type].flatMap(u => {
        const segs: URange[] = []
        if (u.start < selStart) segs.push({ start: u.start, end: Math.min(u.end, selStart) })
        if (u.end > selEnd)     segs.push({ start: Math.max(u.start, selEnd), end: u.end })
        if (u.end <= selStart || u.start >= selEnd) segs.push({ start: u.start, end: u.end })
        return segs
      })
    )
  }

  applyAllRangesToParagraph(para, newMap)
  lineRangeMap.value = newMap
  toolbarVisible.value = false
  window.getSelection()?.removeAllRanges()
  nextTick(syncMarksPanel)
}

function syncContent() {
  toolbarVisible.value = false
}

function onDocClick(e: MouseEvent) {
  const toolbar = document.querySelector('.float-toolbar')
  if (toolbar && toolbar.contains(e.target as Node)) return
  toolbarVisible.value = false
}

onMounted(() => document.addEventListener('mousedown', onDocClick))
onBeforeUnmount(() => document.removeEventListener('mousedown', onDocClick))
</script>

<style scoped>
.editor-wrapper {
  display: flex;
  flex-direction: row;
  font-family: 'Noto Serif SC', 'Source Han Serif CN', Georgia, serif;
  height: 100vh;
  overflow: hidden;
  position: relative;
}

/* ── 左侧划线面板 ── */
.marks-panel {
  width: 200px;
  flex-shrink: 0;
  display: flex;
  flex-direction: column;
  background: #f5f3ef;
  border-right: 1px solid #e8e4de;
  overflow: hidden;
  z-index: 10;
}

.marks-panel__header {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 10px 14px;
  font-size: 12px;
  font-weight: 600;
  letter-spacing: 0.05em;
  color: #5a5650;
  border-bottom: 1px solid #e8e4de;
  background: #eeeae4;
  flex-shrink: 0;
}

.marks-panel__count {
  margin-left: auto;
  background: #c0864a;
  color: #fff;
  font-size: 11px;
  font-weight: 700;
  border-radius: 10px;
  padding: 1px 7px;
  line-height: 1.6;
}

.marks-panel__list {
  flex: 1;
  overflow-y: auto;
  padding: 8px 0;
}

.mark-item {
  display: flex;
  align-items: center;
  gap: 7px;
  padding: 7px 12px;
  cursor: pointer;
  transition: background 0.12s;
  border-radius: 0;
  position: relative;
}
.mark-item:hover { background: rgba(0,0,0,0.04); }
.mark-item.is-focused { background: rgba(192, 134, 74, 0.10); }

.mark-item__index {
  font-size: 10px;
  color: #bbb;
  width: 14px;
  text-align: right;
  flex-shrink: 0;
}

.mark-item__dot {
  width: 7px;
  height: 7px;
  border-radius: 50%;
  flex-shrink: 0;
}
.dot-single { background: #c0864a; }
.dot-double { background: #5b8dd9; }
.dot-wavy   { background: #4caa72; }

.mark-item__text {
  font-size: 13px;
  color: #3a3630;
  flex: 1;
  overflow: hidden;
  white-space: nowrap;
  text-overflow: ellipsis;
  line-height: 1.4;
}

.mark-item__badge {
  font-size: 10px;
  padding: 1px 5px;
  border-radius: 4px;
  flex-shrink: 0;
  font-weight: 500;
}
.badge-single { background: rgba(192,134,74,0.15); color: #a06a30; }
.badge-double { background: rgba(91,141,217,0.15); color: #3a6db5; }
.badge-wavy   { background: rgba(76,170,114,0.15); color: #2e8a56; }

/* 面板进出动画 */
.panel-slide-enter-active,
.panel-slide-leave-active { transition: width 0.22s ease, opacity 0.22s ease; overflow: hidden; }
.panel-slide-enter-from,
.panel-slide-leave-to { width: 0; opacity: 0; }

/* ── 主体区域 ── */
.main-body {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  position: relative;
}

/* ── 浮动工具栏 ── */
.float-toolbar {
  position: fixed;
  z-index: 1000;
  display: flex;
  align-items: center;
  gap: 2px;
  padding: 5px 8px;
  background: #1a1a2e;
  border-radius: 8px;
  box-shadow: 0 4px 24px rgba(0, 0, 0, 0.4);
  white-space: nowrap;
  pointer-events: all;
}

.float-toolbar::after {
  content: '';
  position: absolute;
  bottom: -5px;
  left: v-bind('arrowLeft + "px"');
  width: 10px;
  height: 10px;
  background: #1a1a2e;
  transform: rotate(45deg);
  border-radius: 2px;
  z-index: -1;
}

.toolbar-btn {
  display: flex;
  align-items: center;
  gap: 5px;
  padding: 4px 9px;
  border: none;
  border-radius: 5px;
  background: transparent;
  color: #d8d8ec;
  font-size: 12px;
  font-family: inherit;
  cursor: pointer;
  transition: background 0.15s, color 0.15s;
  letter-spacing: 0.02em;
}
.toolbar-btn:hover,
.toolbar-btn.is-active { background: rgba(255,255,255,0.14); color: #fff; }

.toolbar-btn--cancel { color: #f0a0a0; }
.toolbar-btn--cancel:hover { background: rgba(240,100,100,0.15); color: #ff8080; }

.btn-preview {
  font-size: 13px; font-weight: 700; font-style: italic; line-height: 1;
}
.preview-single { text-decoration: underline; text-underline-offset: 3px; text-decoration-color: #c0864a; text-decoration-thickness: 1.5px; }
.preview-double { text-decoration: underline; text-underline-offset: 3px; text-decoration-style: double; text-decoration-color: #5b8dd9; }
.preview-wavy   { text-decoration: underline wavy; text-underline-offset: 3px; text-decoration-color: #4caa72; text-decoration-thickness: 1.5px; }

.toolbar-divider { width: 1px; height: 16px; background: rgba(255,255,255,0.15); margin: 0 3px; }

.toolbar-fade-enter-active,
.toolbar-fade-leave-active { transition: opacity 0.15s ease, transform 0.15s ease; }
.toolbar-fade-enter-from,
.toolbar-fade-leave-to { opacity: 0; transform: translateY(4px); }

/* ── 预览区 ── */
.preview-area {
  flex: 1;
  overflow-y: auto;
  position: relative;
  background: #fafaf8;
  border-bottom: 1px solid #e8e4de;
}

.preview-content {
  max-width: 720px;
  margin: 0 auto;
  padding: 2rem 2.5rem;
  font-size: 17px;
  line-height: 1.9;
  color: #2c2a26;
}
.preview-content :deep(p) { margin: 0 0 1.2em; }
.preview-content :deep(h1),
.preview-content :deep(h2),
.preview-content :deep(h3) { font-weight: 600; line-height: 1.4; margin: 1.5em 0 0.6em; color: #1a1816; }

/* 三种划线 */
.preview-content :deep(span[data-u="single"]) {
  text-decoration: underline;
  text-underline-offset: 5px;
  text-decoration-color: #c0864a;
  text-decoration-thickness: 2px;
  background: linear-gradient(to bottom, transparent 65%, rgba(192,134,74,0.10) 65%);
  transition: background 0.2s;
}
.preview-content :deep(span[data-u="double"]) {
  text-decoration: underline;
  text-underline-offset: 6px;
  text-decoration-style: double;
  text-decoration-color: #5b8dd9;
  text-decoration-thickness: 1px;
  background: linear-gradient(to bottom, transparent 65%, rgba(91,141,217,0.10) 65%);
  transition: background 0.2s;
}
.preview-content :deep(span[data-u="wavy"]) {
  text-decoration: underline wavy;
  text-underline-offset: 5px;
  text-decoration-color: #4caa72;
  text-decoration-thickness: 1.5px;
  background: linear-gradient(to bottom, transparent 65%, rgba(76,170,114,0.10) 65%);
  transition: background 0.2s;
}

/* 点击定位后的高亮闪烁 */
.preview-content :deep(span.mark-flash) {
  animation: flash-highlight 1.8s ease forwards;
}
@keyframes flash-highlight {
  0%   { background: rgba(255, 210, 100, 0.55); }
  60%  { background: rgba(255, 210, 100, 0.30); }
  100% { background: transparent; }
}

/* ── 编辑器区 ── */
.editor-area {
  flex: 0 0 40%;
  display: flex;
  flex-direction: column;
  background: #ffffff;
  border-top: 2px solid #e8e4de;
  overflow: hidden;
}
.editor-label {
  padding: 6px 16px;
  font-size: 11px;
  letter-spacing: 0.08em;
  color: #999;
  background: #f5f3ef;
  border-bottom: 1px solid #e8e4de;
  text-transform: uppercase;
}
.md-editor-instance { flex: 1; height: 100% !important; border: none !important; box-shadow: none !important; }
:deep(.md-editor) { border: none !important; height: 100% !important; }
:deep(.md-editor-toolbar) { border-bottom: 1px solid #e8e4de !important; background: #f9f7f4 !important; }
</style>