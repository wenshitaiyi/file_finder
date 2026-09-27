<!-- 文件卡片（网格单元）：图片缩略图 / 类型图标 + 元信息 -->
<template>
  <div
    class="card"
    :class="[`m-${mode}`, { selected, favorited: isFavorite, missing: item?.exists_now === false, pressing: isPressing, 'fav-pop': favPopping }]"
    :title="item ? item.rel_path : ''"
    @mousedown="onMouseDown"
    @mousemove="onMouseMove"
    @mouseup="onMouseUp"
    @mouseleave="onMouseLeave"
    @touchstart.passive="onTouchStart"
    @touchmove.passive="onTouchMove"
    @touchend="onTouchEnd"
    @touchcancel="onTouchCancel"
    @click="onClick"
    @dblclick="onDblClick"
    @contextmenu="onContextMenu"
  >
    <div class="thumb">
      <template v-if="item">
        <img
          v-if="showThumb"
          :src="thumbUrl"
          loading="lazy"
          decoding="async"
          draggable="false"
          alt=""
          @error="imgError = true"
        />
        <div v-else-if="imgError" class="missing-thumb">
          <Icon name="warn" :size="mode === 'small' ? 20 : 36" style="color: #d9a53f;" />
          <span v-if="mode !== 'small'" class="missing-text">文件已丢失</span>
        </div>
        <Icon v-else :name="iconName" :size="iconSize" :style="{ color: item.is_dir ? '#e8c468' : fileColor(item) }" />
        <span v-if="isVideo" class="play-badge"><Icon name="play" :size="14" /></span>
        <span class="idx-badge">#{{ (index || 0) + 1 }}</span>
        <span v-if="isFavorite" class="fav-tag">★ 已收藏</span>
        <span v-if="item.is_dir" class="dir-badge"><Icon name="folder" :size="14" /></span>
        <span
          class="fav-badge"
          :class="{ active: isFavorite, 'star-pop': starAnim }"
          :title="isFavorite ? '已收藏（点击取消）' : '点击收藏（亦支持双击/长按/空格）'"
          @click.stop="$emit('fav', item)"
        >
          <Icon name="star" :size="13" />
        </span>
        <span v-if="item.exists_now === false || imgError" class="missing-badge">丢失</span>

        <!-- 长按收藏触发提示气泡 -->
        <transition name="fav-pop">
          <div v-if="favPopping" class="fav-pop-badge">
            <Icon name="star" :size="24" :style="{ color: popFavTarget ? '#f5b942' : '#8b919a' }" />
            <span>{{ popFavTarget ? '已收藏' : '已取消收藏' }}</span>
          </div>
        </transition>
      </template>
      <div v-else class="skeleton"></div>
    </div>
    <div class="meta">
      <div class="name" :title="item?.name">{{ item ? item.name : '' }}</div>
      <div v-if="mode !== 'small' && item" class="sub">{{ formatSize(item.size) }} · {{ formatDate(item.mtime) }}</div>
      <div v-if="mode === 'small' && item" class="sub-sm">{{ item.ext || '文件' }}</div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch, onBeforeUnmount } from 'vue'
import Icon from './Icon.vue'
import { formatSize, formatDate } from '../utils/format'
import { fileIcon, fileColor, classifyExt, previewable } from '../utils/fileTypes'
import { apiFiles } from '../api'

const props = defineProps({
  item: { type: Object, default: null },
  index: { type: Number, default: 0 },
  mode: { type: String, default: 'medium' },
  selected: { type: Boolean, default: false },
  favorite: { type: Boolean, default: false },
})
const emit = defineEmits(['select', 'fav', 'dbl', 'ctx'])

const isFavorite = computed(() => !!(props.favorite || (props.item && props.item.favorite)))
const starAnim = ref(false)
watch(() => isFavorite.value, (val) => {
  if (val) {
    starAnim.value = true
    setTimeout(() => { starAnim.value = false }, 350)
  }
})

const imgError = ref(false)
// 关键修复：当条目发生变更（如删除补位或虚拟滚动复用卡片）时，必须重置 imgError 状态，
// 否则后续补位上来的正常完好图片会被锁死在 imgError=true 状态导致缩略图空白！
watch(() => props.item?.id, () => {
  imgError.value = false
})

const isVideo = computed(() => props.item && classifyExt(props.item.ext) === 'video')
const iconName = computed(() => props.item ? fileIcon(props.item) : 'file')
const iconSize = computed(() => (props.mode === 'small' ? 30 : props.mode === 'large' ? 56 : 42))
const showThumb = computed(() => {
  if (!props.item || !props.item.id || imgError.value) return false
  const k = classifyExt(props.item.ext)
  return (k === 'image') && props.mode !== 'small'
})
const thumbUrl = computed(() => {
  if (!props.item || !showThumb.value) return ''
  const cfg = { small: 96, medium: 160, large: 240 }
  return apiFiles.thumbUrl(props.item.id, cfg[props.mode] || 160)
})

/* ---------- 长按收藏 / 取消收藏（320ms 阈值，带容差与即时视觉反馈） ---------- */
const LONG_PRESS_DURATION = 320
let pressTimer = null
let startX = 0
let startY = 0
let didLongPress = false
const isPressing = ref(false)
const favPopping = ref(false)
const popFavTarget = ref(false)

function triggerLongPress() {
  if (!props.item) return
  didLongPress = true
  isPressing.value = false
  pressTimer = null
  popFavTarget.value = !isFavorite.value
  if (typeof navigator !== 'undefined' && navigator.vibrate) {
    try { navigator.vibrate(40) } catch {}
  }
  emit('fav', props.item)
  favPopping.value = true
  setTimeout(() => { favPopping.value = false }, 700)
}

function cancelPress() {
  if (pressTimer) {
    clearTimeout(pressTimer)
    pressTimer = null
  }
  isPressing.value = false
}

function onMouseDown(e) {
  if (e.button !== 0 || !props.item) return
  startX = e.clientX
  startY = e.clientY
  didLongPress = false
  isPressing.value = true
  pressTimer = setTimeout(triggerLongPress, LONG_PRESS_DURATION)
}

function onMouseMove(e) {
  if (!pressTimer) return
  const dx = Math.abs(e.clientX - startX)
  const dy = Math.abs(e.clientY - startY)
  // 放宽到 20px 容差，防止鼠标按下时的指尖微移误杀长按
  if (dx > 20 || dy > 20) cancelPress()
}

function onMouseUp() {
  cancelPress()
}

function onMouseLeave() {
  cancelPress()
}

function onTouchStart(e) {
  if (!props.item || !e.touches || e.touches.length !== 1) return
  const t = e.touches[0]
  startX = t.clientX
  startY = t.clientY
  didLongPress = false
  isPressing.value = true
  pressTimer = setTimeout(triggerLongPress, LONG_PRESS_DURATION)
}

function onTouchMove(e) {
  if (!pressTimer || !e.touches || !e.touches.length) return
  const t = e.touches[0]
  const dx = Math.abs(t.clientX - startX)
  const dy = Math.abs(t.clientY - startY)
  if (dx > 20 || dy > 20) cancelPress()
}

function onTouchEnd() {
  cancelPress()
}

function onTouchCancel() {
  cancelPress()
}

function onContextMenu(e) {
  cancelPress()
  e.preventDefault()
  emit('ctx', e)
}

/* ---------- 点击与双击（模拟双击检测 + 原生双击兜底 + 防重） ---------- */
let lastClickTime = 0
let lastClickX = 0
let lastClickY = 0
let lastDblTriggerTime = 0

function handleDblTrigger(e) {
  cancelPress()
  if (!props.item || e?.altKey || e?.ctrlKey || e?.metaKey) return
  const now = Date.now()
  if (now - lastDblTriggerTime < 300) return
  lastDblTriggerTime = now
  emit('dbl', props.item)
}

function onClick(e) {
  if (didLongPress) {
    e.preventDefault()
    e.stopPropagation()
    setTimeout(() => { didLongPress = false }, 100)
    return
  }
  if (!props.item) return

  const now = Date.now()
  const dx = Math.abs(e.clientX - lastClickX)
  const dy = Math.abs(e.clientY - lastClickY)

  // 连续两次点击（放宽到间隔 < 420ms 且位移 < 24px）：稳健判定为双击收藏！
  if (now - lastClickTime < 420 && dx < 24 && dy < 24) {
    lastClickTime = 0
    handleDblTrigger(e)
    return
  }

  lastClickTime = now
  lastClickX = e.clientX
  lastClickY = e.clientY
  emit('select', props.item, e)
}

function onDblClick(e) {
  handleDblTrigger(e)
}

onBeforeUnmount(() => {
  cancelPress()
})
</script>

<style scoped>
.card {
  box-sizing: border-box;
  height: 100%;
  padding: 6px;
  display: flex;
  flex-direction: column;
  border-radius: 8px;
  cursor: default;
  border: 1px solid transparent;
  overflow: hidden;
  user-select: none;
  transition: transform 0.2s cubic-bezier(0.2, 0.8, 0.2, 1), box-shadow 0.2s ease, border-color 0.2s ease, background 0.15s ease;
}
.card:hover { background: rgba(255, 255, 255, 0.05); }
.card.selected { background: rgba(76, 139, 245, 0.18); border-color: rgba(76, 139, 245, 0.55); }
.card.favorited {
  border-color: rgba(245, 185, 66, 0.85) !important;
  background: rgba(245, 185, 66, 0.12) !important;
  box-shadow: 0 0 10px rgba(245, 185, 66, 0.25) !important;
}
.card.favorited.selected {
  border-color: #f5b942 !important;
  box-shadow: 0 0 0 2px rgba(76, 139, 245, 0.8), 0 0 14px rgba(245, 185, 66, 0.4) !important;
}
.card.missing { opacity: 0.55; }

/* 长按按压中的动效反馈（不收缩几何尺寸，避免鼠标滑出卡片触发 mouseleave 误取消） */
.card.pressing {
  border-color: rgba(245, 185, 66, 0.75) !important;
  box-shadow: 0 0 0 2px rgba(245, 185, 66, 0.4), 0 0 12px rgba(245, 185, 66, 0.25) !important;
  background: rgba(245, 185, 66, 0.1) !important;
}

/* 收藏爆开弹动 */
.card.fav-pop {
  animation: favCardPop 0.45s cubic-bezier(0.175, 0.885, 0.32, 1.275);
}
@keyframes favCardPop {
  0% { transform: scale(0.96); }
  50% { transform: scale(1.03); }
  100% { transform: scale(1); }
}

.thumb {
  flex: 1;
  min-height: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  background: rgba(255, 255, 255, 0.04);
  border-radius: 6px;
  overflow: hidden;
  position: relative;
}
.thumb img {
  max-width: 100%;
  max-height: 100%;
  object-fit: contain;
  border-radius: 4px;
}
.missing-thumb {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 6px;
  width: 100%;
  height: 100%;
  background: rgba(224, 92, 92, 0.08);
  border: 1px dashed rgba(217, 165, 63, 0.4);
  border-radius: 6px;
  user-select: none;
}
.missing-text {
  font-size: 11px;
  color: #d9a53f;
  font-weight: 500;
}
.skeleton {
  width: 70%;
  height: 60%;
  border-radius: 6px;
  background: linear-gradient(90deg, rgba(255,255,255,.05), rgba(255,255,255,.1), rgba(255,255,255,.05));
  background-size: 200% 100%;
  animation: sh 1.2s infinite;
}
@keyframes sh { to { background-position: -200% 0; } }
.meta { padding: 4px 2px 0; min-width: 0; }
.name {
  font-size: 12px;
  color: #e3e6eb;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  line-height: 1.3;
}
.sub { font-size: 10.5px; color: #8b919a; margin-top: 2px; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.sub-sm { font-size: 10px; color: #6c727c; margin-top: 1px; }
.play-badge {
  position: absolute; bottom: 6px; right: 6px;
  width: 24px; height: 24px; border-radius: 50%;
  background: rgba(0,0,0,.65); color: #fff;
  display: flex; align-items: center; justify-content: center;
}
.idx-badge {
  position: absolute;
  top: 5px;
  left: 5px;
  font-size: 10px;
  font-family: Consolas, Monaco, monospace;
  padding: 1px 5px;
  border-radius: 4px;
  background: rgba(0, 0, 0, 0.55);
  color: #adb5c2;
  pointer-events: none;
  z-index: 2;
  transition: background 0.15s, color 0.15s;
}
.card:hover .idx-badge {
  background: rgba(0, 0, 0, 0.85);
  color: #ffffff;
}
.dir-badge {
  position: absolute; top: 5px; left: 42px;
  width: 22px; height: 22px; border-radius: 6px;
  background: rgba(232,196,104,.22); color: #e8c468;
  display: flex; align-items: center; justify-content: center;
}
.fav-tag {
  position: absolute;
  top: 5px;
  left: 38px;
  font-size: 10px;
  font-weight: 600;
  padding: 1px 6px;
  border-radius: 4px;
  background: rgba(245, 185, 66, 0.95);
  color: #1a1c20;
  box-shadow: 0 2px 6px rgba(0,0,0,0.35);
  pointer-events: none;
  z-index: 3;
}
.fav-badge {
  position: absolute;
  top: 5px;
  right: 5px;
  width: 24px;
  height: 24px;
  border-radius: 50%;
  background: rgba(0, 0, 0, 0.45);
  color: rgba(255, 255, 255, 0.35);
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: all 0.18s ease;
  z-index: 5;
}
.card:hover .fav-badge {
  color: rgba(255, 255, 255, 0.85);
  background: rgba(0, 0, 0, 0.65);
}
.fav-badge.active {
  background: rgba(232, 170, 40, 0.95) !important;
  color: #ffffff !important;
  box-shadow: 0 0 10px rgba(245, 185, 66, 0.6) !important;
}
.fav-badge:hover {
  transform: scale(1.22);
}
.fav-badge.star-pop {
  animation: starBounce 0.35s cubic-bezier(0.175, 0.885, 0.32, 1.275);
}
@keyframes starBounce {
  0% { transform: scale(0.4) rotate(-30deg); opacity: 0.3; }
  60% { transform: scale(1.35) rotate(15deg); opacity: 1; }
  100% { transform: scale(1) rotate(0deg); opacity: 1; }
}
.missing-badge {
  position: absolute; top: 6px; left: 6px;
  font-size: 10px; padding: 1px 6px; border-radius: 8px;
  background: rgba(224,92,92,.9); color: #fff;
}

/* 浮动提示 */
.fav-pop-badge {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 4px;
  padding: 8px 12px;
  border-radius: 8px;
  background: rgba(18, 22, 28, 0.92);
  border: 1px solid rgba(245, 185, 66, 0.35);
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.45);
  color: #e3e6eb;
  font-size: 11px;
  font-weight: 500;
  pointer-events: none;
  z-index: 10;
}
.fav-pop-enter-active,
.fav-pop-leave-active {
  transition: opacity 0.25s ease, transform 0.25s cubic-bezier(0.175, 0.885, 0.32, 1.275);
}
.fav-pop-enter-from {
  opacity: 0;
  transform: translate(-50%, -40%) scale(0.7);
}
.fav-pop-leave-to {
  opacity: 0;
  transform: translate(-50%, -60%) scale(0.9);
}
</style>
