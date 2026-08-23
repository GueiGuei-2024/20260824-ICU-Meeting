<template>
  <!-- 滿版全螢幕黑底容器 (覆蓋整張 Slidev 簡報頁面) -->
  <div class="absolute -top-12 -left-16 w-[calc(100%+8rem)] h-[calc(100%+6rem)] bg-black overflow-hidden flex flex-col items-center justify-center select-none z-10">
    <!-- 全螢幕純黑底 Canvas 探照燈質感微光粒子畫布 -->
    <canvas
      ref="canvasRef"
      class="absolute inset-0 w-full h-full cursor-pointer z-0"
      @mousemove="handleMouseMove"
      @mouseleave="handleMouseLeave"
      @touchstart="handleTouchMove"
      @touchmove="handleTouchMove"
    ></canvas>

    <!-- 中央輕度半透明 THANK YOU 結語 (背景通透，沙粒與水波流動可穿透視覺) -->
    <div class="relative z-10 text-center pointer-events-none px-12 py-8 rounded-3xl bg-black/30 backdrop-blur-[3px] border border-white/10 shadow-[0_0_60px_rgba(0,0,0,0.4)]">
      <h1 class="text-6xl font-black tracking-widest text-white drop-shadow-[0_0_25px_rgba(255,255,255,0.85)] mb-3">
        THANK YOU
      </h1>
      <!-- 水波紋流光分隔線 -->
      <div class="h-1 w-28 mx-auto bg-gradient-to-r from-cyan-400 via-teal-300 to-indigo-400 rounded-full mb-4 shadow-[0_0_15px_rgba(34,211,238,0.8)]"></div>
      <p class="text-sm font-bold tracking-[0.25em] text-cyan-300 uppercase font-mono drop-shadow-[0_0_10px_rgba(34,211,238,0.7)]">
        Open for Discussion & Questions
      </p>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

const canvasRef = ref(null)
let ctx = null
let animationFrameId = null
let particles = []
let resizeObserver = null

// 滑鼠近接與大範圍光暈參數 (保持與當前完全一致)
const mouse = {
  x: -1000,
  y: -1000,
  pushRadius: 75,    // 精緻小巧的 75px 滑鼠排斥區
  glowRadius: 450,   // 大範圍光暈半徑 (450px)
  coreRadius: 120    // 核心高亮區半徑
}
let time = 0

// ProximityParticle 保持 100% 原汁原味粒子物理與特性
class ProximityParticle {
  constructor(x, y, baseHue, totalWidth) {
    this.anchorX = x
    this.anchorY = y
    this.originX = x
    this.originY = y
    this.x = x + (Math.random() - 0.5) * 6
    this.y = y + (Math.random() - 0.5) * 6
    this.vx = 0
    this.vy = 0

    // 每個粒子獨立的極慢自主舞動相位與速度
    this.phaseX = Math.random() * Math.PI * 2
    this.phaseY = Math.random() * Math.PI * 2
    this.driftSpeed = Math.random() * 0.008 + 0.006
    this.driftRange = Math.random() * 8 + 4

    // 基礎粒子大小 (1.0px ~ 1.5px)
    this.baseSize = Math.random() * 0.5 + 1.0
    this.currentSize = this.baseSize

    // 超流暢慢速水波阻尼與極致柔軟歸位彈性
    this.friction = 0.95
    this.ease = 0.012

    // 空間漸層色相與透明度
    const spatialOffset = (x / totalWidth) * 40 - 20
    this.hue = (baseHue + spatialOffset + (Math.random() * 14 - 7) + 360) % 360
    
    // 當前透明度與亮度
    this.alpha = 0.04
    this.lightness = 50
  }

  draw() {
    if (this.alpha <= 0.02) return

    ctx.fillStyle = `hsla(${this.hue}, 95%, ${this.lightness}%, ${this.alpha})`
    
    const glowBlur = this.alpha > 0.3 ? (this.alpha * 11) : 0
    if (glowBlur > 0) {
      ctx.shadowBlur = glowBlur
      ctx.shadowColor = `hsla(${this.hue}, 98%, 78%, ${this.alpha})`
    }

    ctx.fillRect(
      this.x - this.currentSize / 2,
      this.y - this.currentSize / 2,
      this.currentSize,
      this.currentSize
    )
    ctx.shadowBlur = 0
  }

  update(currentTime) {
    this.originX = this.anchorX + Math.sin(currentTime * this.driftSpeed + this.phaseX) * this.driftRange
    this.originY = this.anchorY + Math.cos(currentTime * this.driftSpeed + this.phaseY) * this.driftRange

    const dx = mouse.x - this.x
    const dy = mouse.y - this.y
    const distance = Math.sqrt(dx * dx + dy * dy)

    if (distance < mouse.glowRadius) {
      let proximity = 0
      if (distance < mouse.coreRadius) {
        proximity = 1.0
      } else {
        proximity = 1 - (distance - mouse.coreRadius) / (mouse.glowRadius - mouse.coreRadius)
      }

      this.alpha = 0.04 + proximity * 0.93
      this.currentSize = this.baseSize * (1 + proximity * 1.8)
      this.lightness = 50 + proximity * 38
    } else {
      this.alpha += (0.04 - this.alpha) * 0.04
      this.currentSize += (this.baseSize - this.currentSize) * 0.04
      this.lightness += (50 - this.lightness) * 0.04
    }

    if (distance < mouse.pushRadius) {
      const wave = Math.sin((1 - distance / mouse.pushRadius) * Math.PI)
      const angle = Math.atan2(dy, dx)
      const push = wave * 2.2
      this.vx -= Math.cos(angle) * push
      this.vy -= Math.sin(angle) * push
    }

    this.vx += (this.originX - this.x) * this.ease
    this.vy += (this.originY - this.y) * this.ease

    this.vx *= this.friction
    this.vy *= this.friction

    this.x += this.vx
    this.y += this.vy
  }
}

const initCanvas = () => {
  const canvas = canvasRef.value
  if (!canvas) return
  const rect = canvas.getBoundingClientRect()
  if (rect.width === 0 || rect.height === 0) return

  canvas.width = rect.width
  canvas.height = rect.height
  ctx = canvas.getContext('2d')

  // 特性保持：若滑鼠尚未移動過，剛翻到頁面時預設在畫面中央自動發光亮起一次
  if (mouse.x === -1000 && mouse.y === -1000) {
    mouse.x = canvas.width / 2
    mouse.y = canvas.height / 2
  }

  particles = []
  const gap = 22

  const colorPalettes = [185, 160, 210, 45, 310]
  const baseHue = colorPalettes[Math.floor(Math.random() * colorPalettes.length)]

  for (let x = gap / 2; x < canvas.width; x += gap) {
    for (let y = gap / 2; y < canvas.height; y += gap) {
      particles.push(new ProximityParticle(x, y, baseHue, canvas.width))
    }
  }
}

// 修正 Slidev CSS 縮放導致的滑鼠座標偏移 (Scale Correction)
const handleMouseMove = (e) => {
  const canvas = canvasRef.value
  if (!canvas) return
  const rect = canvas.getBoundingClientRect()
  const scaleX = canvas.width / rect.width
  const scaleY = canvas.height / rect.height
  mouse.x = (e.clientX - rect.left) * scaleX
  mouse.y = (e.clientY - rect.top) * scaleY
}

const handleMouseLeave = (e) => {
  // 離開畫布時維持在中央發光
  if (canvasRef.value) {
    mouse.x = canvasRef.value.width / 2
    mouse.y = canvasRef.value.height / 2
  }
}

const handleTouchMove = (e) => {
  const canvas = canvasRef.value
  if (!canvas || !e.touches.length) return
  const rect = canvas.getBoundingClientRect()
  const scaleX = canvas.width / rect.width
  const scaleY = canvas.height / rect.height
  mouse.x = (e.touches[0].clientX - rect.left) * scaleX
  mouse.y = (e.touches[0].clientY - rect.top) * scaleY
}

const animate = () => {
  const canvas = canvasRef.value
  if (!canvas || !ctx) return

  time += 1
  ctx.fillStyle = '#000000'
  ctx.fillRect(0, 0, canvas.width, canvas.height)

  for (let i = 0; i < particles.length; i++) {
    particles[i].update(time)
    particles[i].draw()
  }

  animationFrameId = requestAnimationFrame(animate)
}

onMounted(() => {
  initCanvas()
  animate()
  window.addEventListener('resize', initCanvas)

  // 利用 ResizeObserver 監聽 Canvas 元素顯示與尺寸變化 (切換至該頁時自動校正 Canvas 尺寸)
  if (window.ResizeObserver && canvasRef.value) {
    resizeObserver = new ResizeObserver(() => {
      initCanvas()
    })
    resizeObserver.observe(canvasRef.value)
  }
})

onUnmounted(() => {
  if (animationFrameId) cancelAnimationFrame(animationFrameId)
  window.removeEventListener('resize', initCanvas)
  if (resizeObserver) resizeObserver.disconnect()
})
</script>
