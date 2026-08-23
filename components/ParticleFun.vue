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

// 滑鼠近接與大範圍光暈參數
const mouse = {
  x: -1000,
  y: -1000,
  pushRadius: 75,    // 1. 滑鼠中間空白縮小至精緻的 75px
  glowRadius: 450,   // 2. 光圈感應範圍更大 (450px)
  coreRadius: 120    // 核心高亮區範圍
}
let time = 0

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
    this.driftSpeed = Math.random() * 0.008 + 0.006 // 悠揚順暢的自漂速度
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
    
    // 3. 亮度稍微增加 (Glow Blur 與光芒增強)
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

      // 3. 亮度與透明度稍微增加
      this.alpha = 0.04 + proximity * 0.93
      this.currentSize = this.baseSize * (1 + proximity * 1.8)
      this.lightness = 50 + proximity * 38 // 50% -> 88%
    } else {
      this.alpha += (0.04 - this.alpha) * 0.04
      this.currentSize += (this.baseSize - this.currentSize) * 0.04
      this.lightness += (50 - this.lightness) * 0.04
    }

    // 1. 滑鼠中間空白縮小 (pushRadius = 75px)
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
  canvas.width = rect.width
  canvas.height = rect.height
  ctx = canvas.getContext('2d')

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

const handleMouseMove = (e) => {
  const canvas = canvasRef.value
  if (!canvas) return
  const rect = canvas.getBoundingClientRect()
  const scaleX = canvas.width / rect.width
  const scaleY = canvas.height / rect.height
  mouse.x = (e.clientX - rect.left) * scaleX
  mouse.y = (e.clientY - rect.top) * scaleY
}

const handleMouseLeave = () => {
  mouse.x = -1000
  mouse.y = -1000
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
})

onUnmounted(() => {
  if (animationFrameId) cancelAnimationFrame(animationFrameId)
  window.removeEventListener('resize', initCanvas)
})
</script>
