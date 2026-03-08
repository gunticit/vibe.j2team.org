<script setup lang="ts">
import { ref, computed, onUnmounted } from 'vue'
import { RouterLink } from 'vue-router'

// ═══════════════════════════════════════════════════════════════════════════
// TYPES & CONSTANTS
// ═══════════════════════════════════════════════════════════════════════════

type GameState = 'intro' | 'permission' | 'countdown' | 'dance' | 'warning' | 'freeze' | 'busted' | 'result'

interface Round { danceTime: number; freezeTime: number }

const ROUNDS: Round[] = [
  { danceTime: 5000, freezeTime: 3000 },
  { danceTime: 4000, freezeTime: 3500 },
  { danceTime: 3500, freezeTime: 4000 },
  { danceTime: 3000, freezeTime: 4000 },
  { danceTime: 2500, freezeTime: 4500 },
]

const MOTION_THRESHOLD = 30 // pixel diff threshold
const FREEZE_MOTION_LIMIT = 0.025 // max allowed motion % in freeze phase
const SAMPLE_STEP = 4 // skip pixels for performance

interface TierInfo { title: string; icon: string; desc: string; color: string }
const TIERS: Record<string, TierInfo> = {
  statue: { title: 'Tượng Đá', icon: '🗿', desc: 'Bạn đứng yên như tượng! Không ai bắt được bạn. Huyền thoại sân trường!', color: '#4ADE80' },
  flexible: { title: 'Người Linh Hoạt', icon: '💃', desc: 'Nhảy giỏi, đứng cũng ổn. Bạn có tố chất của một nghệ sĩ sân khấu!', color: '#38BDF8' },
  shaky: { title: 'Chân Run', icon: '🦵', desc: 'Cố gắng đứng yên mà tay chân cứ run. Bạn nên đi tập yoga!', color: '#FFB830' },
  pole: { title: 'Cây Cột Điện', icon: '⚡', desc: 'Đứng im cả lúc phải nhảy. Bạn có chắc là đang sống không?', color: '#EF4444' },
}

// ═══════════════════════════════════════════════════════════════════════════
// STATE
// ═══════════════════════════════════════════════════════════════════════════

const gameState = ref<GameState>('intro')
const currentRound = ref(0)
const totalRounds = ROUNDS.length
const motionScore = ref(0) // 0-1 current frame motion
const dancePoints = ref(0)
const lives = ref(3)
const countdownNum = ref(3)
const bustedFlash = ref(false)
const resultTier = ref('statue')
const totalDanceScore = ref(0)
const totalBustedCount = ref(0)
const phaseTimeLeft = ref(0)

const videoRef = ref<HTMLVideoElement | null>(null)
const canvasRef = ref<HTMLCanvasElement | null>(null)
const overlayRef = ref<HTMLCanvasElement | null>(null)

let stream: MediaStream | null = null
let animFrame: number | null = null
let prevFrame: ImageData | null = null
let phaseTimer: ReturnType<typeof setTimeout> | null = null
let countdownTimer: ReturnType<typeof setTimeout> | null = null
let tickInterval: ReturnType<typeof setInterval> | null = null

const canvasW = 320
const canvasH = 240

// ═══════════════════════════════════════════════════════════════════════════
// CAMERA ACCESS (WebRTC)
// ═══════════════════════════════════════════════════════════════════════════

async function requestCamera() {
  gameState.value = 'permission'
  try {
    stream = await navigator.mediaDevices.getUserMedia({
      video: { facingMode: 'user', width: { ideal: 320 }, height: { ideal: 240 } },
      audio: false,
    })
    const video = videoRef.value
    if (video) {
      video.srcObject = stream
      await video.play()
    }
    startCountdown()
  } catch {
    alert('Camera không được cấp quyền! Vui lòng cho phép camera và thử lại.')
    gameState.value = 'intro'
  }
}

// ═══════════════════════════════════════════════════════════════════════════
// FRAME DIFFERENCING ENGINE
// ═══════════════════════════════════════════════════════════════════════════

function processFrame() {
  const video = videoRef.value
  const canvas = canvasRef.value
  const overlay = overlayRef.value
  if (!video || !canvas || !overlay) { animFrame = requestAnimationFrame(processFrame); return }

  const ctx = canvas.getContext('2d', { willReadFrequently: true })
  const octx = overlay.getContext('2d')
  if (!ctx || !octx) { animFrame = requestAnimationFrame(processFrame); return }

  // Draw video frame (mirrored)
  ctx.save()
  ctx.translate(canvasW, 0)
  ctx.scale(-1, 1)
  ctx.drawImage(video, 0, 0, canvasW, canvasH)
  ctx.restore()

  const currentFrame = ctx.getImageData(0, 0, canvasW, canvasH)

  if (prevFrame) {
    const curr = currentFrame.data
    const prev = prevFrame.data
    const overlayData = octx.createImageData(canvasW, canvasH)
    const od = overlayData.data

    let changedPixels = 0
    let totalPixels = 0

    for (let i = 0; i < curr.length; i += 4 * SAMPLE_STEP) {
      const dr = Math.abs(curr[i]! - prev[i]!)
      const dg = Math.abs(curr[i + 1]! - prev[i + 1]!)
      const db = Math.abs(curr[i + 2]! - prev[i + 2]!)
      const diff = dr + dg + db
      totalPixels++

      if (diff > MOTION_THRESHOLD) {
        changedPixels++
        // Heatmap: red in freeze, green in dance
        const isDance = gameState.value === 'dance'
        const intensity = Math.min(255, diff * 2)
        od[i] = isDance ? 0 : intensity     // R
        od[i + 1] = isDance ? intensity : 0 // G
        od[i + 2] = 0                        // B
        od[i + 3] = Math.min(180, intensity) // A
      }
    }

    motionScore.value = totalPixels > 0 ? changedPixels / totalPixels : 0
    octx.clearRect(0, 0, canvasW, canvasH)
    octx.putImageData(overlayData, 0, 0)
  }

  prevFrame = currentFrame
  animFrame = requestAnimationFrame(processFrame)
}

// ═══════════════════════════════════════════════════════════════════════════
// GAME LOGIC
// ═══════════════════════════════════════════════════════════════════════════

function startCountdown() {
  gameState.value = 'countdown'
  countdownNum.value = 3
  currentRound.value = 0
  dancePoints.value = 0
  lives.value = 3
  totalDanceScore.value = 0
  totalBustedCount.value = 0
  prevFrame = null

  // Start frame processing
  animFrame = requestAnimationFrame(processFrame)

  const tick = () => {
    countdownNum.value--
    if (countdownNum.value <= 0) {
      startDancePhase()
    } else {
      countdownTimer = setTimeout(tick, 1000)
    }
  }
  countdownTimer = setTimeout(tick, 1000)
}

function startDancePhase() {
  const round = ROUNDS[currentRound.value]
  if (!round) { endGame(); return }

  gameState.value = 'dance'
  dancePoints.value = 0
  phaseTimeLeft.value = round.danceTime

  // Accumulate dance points based on motion
  tickInterval = setInterval(() => {
    if (gameState.value === 'dance') {
      dancePoints.value += Math.round(motionScore.value * 100)
      phaseTimeLeft.value = Math.max(0, phaseTimeLeft.value - 100)
    }
  }, 100)

  // Transition to warning first, then freeze
  phaseTimer = setTimeout(() => {
    if (tickInterval) clearInterval(tickInterval)
    totalDanceScore.value += dancePoints.value
    startWarningPhase()
  }, round.danceTime)
}

function startWarningPhase() {
  gameState.value = 'warning'
  phaseTimer = setTimeout(() => {
    startFreezePhase()
  }, 1500)
}

function startFreezePhase() {
  const round = ROUNDS[currentRound.value]
  if (!round) { endGame(); return }

  gameState.value = 'freeze'
  phaseTimeLeft.value = round.freezeTime

  // Monitor motion during freeze
  tickInterval = setInterval(() => {
    if (gameState.value === 'freeze') {
      phaseTimeLeft.value = Math.max(0, phaseTimeLeft.value - 100)

      if (motionScore.value > FREEZE_MOTION_LIMIT) {
        // BUSTED!
        if (tickInterval) clearInterval(tickInterval)
        if (phaseTimer) clearTimeout(phaseTimer)
        handleBusted()
      }
    }
  }, 100)

  phaseTimer = setTimeout(() => {
    // Survived freeze phase
    if (tickInterval) clearInterval(tickInterval)
    nextRound()
  }, round.freezeTime)
}

function handleBusted() {
  lives.value--
  totalBustedCount.value++
  bustedFlash.value = true
  gameState.value = 'busted'

  setTimeout(() => {
    bustedFlash.value = false
    if (lives.value <= 0) {
      endGame()
    } else {
      nextRound()
    }
  }, 1500)
}

function nextRound() {
  currentRound.value++
  if (currentRound.value >= totalRounds) {
    endGame()
  } else {
    startDancePhase()
  }
}

function endGame() {
  if (tickInterval) clearInterval(tickInterval)
  if (phaseTimer) clearTimeout(phaseTimer)
  gameState.value = 'result'

  // Determine tier
  if (totalDanceScore.value < 50) {
    resultTier.value = 'pole' // Didn't dance at all
  } else if (totalBustedCount.value === 0) {
    resultTier.value = 'statue'
  } else if (totalBustedCount.value <= 2) {
    resultTier.value = 'flexible'
  } else {
    resultTier.value = 'shaky'
  }
}

function resetGame() {
  cleanup()
  gameState.value = 'intro'
}

function playAgain() {
  prevFrame = null
  startCountdown()
}

function cleanup() {
  if (animFrame) cancelAnimationFrame(animFrame)
  if (tickInterval) clearInterval(tickInterval)
  if (phaseTimer) clearTimeout(phaseTimer)
  if (countdownTimer) clearTimeout(countdownTimer)
  if (stream) stream.getTracks().forEach((t) => t.stop())
  stream = null; prevFrame = null; animFrame = null
}

onUnmounted(cleanup)

const motionBarHeight = computed(() => `${Math.min(100, motionScore.value * 500)}%`)
const motionBarColor = computed(() => {
  if (gameState.value === 'freeze' || gameState.value === 'warning') {
    return motionScore.value > FREEZE_MOTION_LIMIT ? '#EF4444' : '#4ADE80'
  }
  return '#38BDF8'
})
const livesArray = computed(() => Array.from({ length: 3 }, (_, i) => i < lives.value))
const roundProgress = computed(() => `${currentRound.value + 1} / ${totalRounds}`)
</script>

<template>
  <div class="min-h-screen bg-bg-deep text-text-primary font-body relative overflow-hidden" :class="{ 'busted-shake': bustedFlash }">

    <!-- ═══════ INTRO ═══════ -->
    <div v-if="gameState === 'intro'" class="min-h-screen flex flex-col items-center justify-center px-4 py-10">
      <div class="text-center animate-fade-up">
        <span class="text-6xl sm:text-7xl block mb-4">🗿</span>
        <h1 class="font-display text-4xl min-[375px]:text-5xl sm:text-6xl lg:text-7xl font-bold text-accent-coral mb-3 tracking-tight">
          1-2-3<br>Đứng Hình!
        </h1>
        <p class="text-text-secondary text-base sm:text-lg max-w-md mx-auto mb-2">
          Camera phát hiện chuyển động thật — nhảy khi hô, đứng yên khi bắt!
        </p>
        <p class="text-text-dim text-xs font-display tracking-wider mb-8">WEBRTC × CANVAS FRAME DIFFERENCING</p>

        <!-- How to play -->
        <div class="max-w-sm mx-auto border border-border-default bg-bg-surface p-5 mb-8 text-left animate-fade-up animate-delay-1">
          <h2 class="font-display text-sm font-semibold text-accent-amber mb-3 tracking-wider">// CÁCH CHƠI</h2>
          <div class="space-y-2 text-sm text-text-secondary">
            <p>🎵 <strong class="text-green-400">"NHẢY ĐI!"</strong> → Cử động thật nhiều trước camera = ghi điểm</p>
            <p>🤫 <strong class="text-accent-coral">"ĐỨNG HÌNH!"</strong> → Đứng yên! Cử động = mất mạng</p>
            <p>❤️ Bạn có <strong class="text-accent-coral">3 mạng</strong>. 5 rounds, khó dần!</p>
          </div>
        </div>

        <button class="inline-flex items-center gap-2 bg-accent-coral px-8 py-3.5 font-display font-bold text-bg-deep text-lg transition-all hover:shadow-lg hover:shadow-accent-coral/20 hover:-translate-y-1 animate-fade-up animate-delay-2" @click="requestCamera">
          📸 BẬT CAMERA & CHƠI
        </button>
      </div>
    </div>

    <!-- ═══════ PERMISSION ═══════ -->
    <div v-if="gameState === 'permission'" class="min-h-screen flex items-center justify-center px-4">
      <div class="text-center border border-border-default bg-bg-surface p-8 max-w-sm">
        <span class="text-5xl block mb-4">📸</span>
        <h2 class="font-display text-xl font-semibold mb-2">Cho phép Camera</h2>
        <p class="text-text-secondary text-sm">Ứng dụng cần camera để phát hiện cử động của bạn.</p>
        <div class="mt-4 flex justify-center gap-2">
          <span class="w-2 h-2 rounded-full bg-accent-coral animate-bounce" style="animation-delay: 0s" />
          <span class="w-2 h-2 rounded-full bg-accent-amber animate-bounce" style="animation-delay: 0.2s" />
          <span class="w-2 h-2 rounded-full bg-accent-sky animate-bounce" style="animation-delay: 0.4s" />
        </div>
      </div>
    </div>

    <!-- ═══════ GAME UI (countdown / dance / warning / freeze / busted) ═══════ -->
    <div v-if="['countdown','dance','warning','freeze','busted'].includes(gameState)" class="min-h-screen flex flex-col items-center justify-center px-4 py-6 relative">

      <!-- Status Bar -->
      <div class="absolute top-4 left-4 right-4 z-30 flex items-center justify-between">
        <div class="flex items-center gap-3">
          <!-- Lives -->
          <div class="flex gap-1">
            <span v-for="(alive, idx) in livesArray" :key="idx" class="text-lg transition-all duration-300" :class="alive ? 'opacity-100 scale-100' : 'opacity-20 scale-75'">❤️</span>
          </div>
          <!-- Round -->
          <span class="text-text-dim text-xs font-display tracking-wider">ROUND {{ roundProgress }}</span>
        </div>
        <!-- Dance Score -->
        <div class="text-right">
          <p class="text-text-dim text-xs font-display tracking-wider">ĐIỂM</p>
          <p class="font-display font-bold text-accent-amber text-lg tabular-nums">{{ totalDanceScore + dancePoints }}</p>
        </div>
      </div>

      <!-- Camera Feed + Overlay -->
      <div class="relative border-2 transition-colors duration-300"
        :class="{
          'border-green-500/60': gameState === 'dance',
          'border-amber-500/60': gameState === 'warning',
          'border-red-500/60 animate-pulse': gameState === 'freeze',
          'border-red-600': gameState === 'busted',
          'border-border-default': gameState === 'countdown',
        }"
      >
        <video ref="videoRef" :width="canvasW" :height="canvasH" autoplay playsinline muted class="block" style="transform: scaleX(-1); object-fit: cover;" />
        <canvas ref="canvasRef" :width="canvasW" :height="canvasH" class="absolute inset-0 opacity-0 pointer-events-none" />
        <canvas ref="overlayRef" :width="canvasW" :height="canvasH" class="absolute inset-0 pointer-events-none mix-blend-screen" style="transform: scaleX(-1);" />

        <!-- Phase Overlay -->
        <div v-if="gameState === 'countdown'" class="absolute inset-0 bg-bg-deep/70 flex items-center justify-center z-10">
          <span class="font-display text-7xl sm:text-8xl font-bold text-accent-coral animate-bounce">{{ countdownNum }}</span>
        </div>

        <div v-if="gameState === 'dance'" class="absolute top-3 left-1/2 -translate-x-1/2 z-10">
          <div class="bg-green-500/90 px-4 py-1.5 font-display font-bold text-bg-deep text-sm tracking-wider animate-pulse">
            🎵 NHẢY ĐI! — {{ Math.ceil(phaseTimeLeft / 1000) }}s
          </div>
        </div>

        <div v-if="gameState === 'warning'" class="absolute inset-0 bg-amber-500/20 flex items-center justify-center z-10">
          <div class="text-center">
            <p class="font-display text-3xl sm:text-4xl font-bold text-amber-400 animate-pulse">1... 2... 3...</p>
          </div>
        </div>

        <div v-if="gameState === 'freeze'" class="absolute top-3 left-1/2 -translate-x-1/2 z-10">
          <div class="bg-red-600/90 px-4 py-1.5 font-display font-bold text-white text-sm tracking-wider">
            🤫 ĐỨNG HÌNH! — {{ Math.ceil(phaseTimeLeft / 1000) }}s
          </div>
        </div>

        <div v-if="gameState === 'busted'" class="absolute inset-0 bg-red-600/40 flex items-center justify-center z-10">
          <div class="text-center">
            <p class="font-display text-4xl sm:text-5xl font-bold text-red-400">BUSTED!</p>
            <p class="text-white text-sm mt-2">Bạn cử động rồi! -1 ❤️</p>
          </div>
        </div>

        <!-- Scan line effect during freeze -->
        <div v-if="gameState === 'freeze'" class="absolute inset-0 pointer-events-none z-5 overflow-hidden">
          <div class="scan-line" />
        </div>
      </div>

      <!-- Motion Meter -->
      <div class="absolute right-4 top-1/2 -translate-y-1/2 z-20 flex flex-col items-center gap-1 hidden sm:flex">
        <p class="text-text-dim text-xs font-display tracking-wider rotate-180" style="writing-mode: vertical-rl;">MOTION</p>
        <div class="w-3 h-32 bg-bg-deep border border-border-default overflow-hidden relative">
          <div class="absolute bottom-0 left-0 right-0 transition-all duration-150" :style="{ height: motionBarHeight, backgroundColor: motionBarColor }" />
        </div>
        <p class="font-display text-xs font-bold tabular-nums" :style="{ color: motionBarColor }">{{ Math.round(motionScore * 100) }}</p>
      </div>

      <!-- Current dance points -->
      <div v-if="gameState === 'dance'" class="mt-3 text-center">
        <p class="font-display text-sm text-green-400">Điểm nhảy: <span class="font-bold text-lg">+{{ dancePoints }}</span></p>
      </div>
    </div>

    <!-- ═══════ RESULT ═══════ -->
    <div v-if="gameState === 'result'" class="min-h-screen flex flex-col items-center justify-center px-4 py-10">
      <div class="text-center max-w-md w-full animate-fade-up">
        <!-- Tier Badge -->
        <div class="mb-6">
          <span class="text-6xl block mb-3">{{ TIERS[resultTier]?.icon }}</span>
          <h2 class="font-display text-3xl sm:text-4xl font-bold mb-2" :style="{ color: TIERS[resultTier]?.color }">
            {{ TIERS[resultTier]?.title }}
          </h2>
          <p class="text-text-secondary text-sm">{{ TIERS[resultTier]?.desc }}</p>
        </div>

        <!-- Stats -->
        <div class="border border-border-default bg-bg-surface p-5 mb-6">
          <div class="grid grid-cols-3 gap-3">
            <div class="text-center">
              <p class="font-display text-2xl font-bold text-accent-amber">{{ totalDanceScore }}</p>
              <p class="text-text-dim text-xs">Điểm nhảy</p>
            </div>
            <div class="text-center">
              <p class="font-display text-2xl font-bold text-accent-coral">{{ totalBustedCount }}</p>
              <p class="text-text-dim text-xs">Lần bị bắt</p>
            </div>
            <div class="text-center">
              <p class="font-display text-2xl font-bold text-green-400">{{ currentRound }}</p>
              <p class="text-text-dim text-xs">Rounds qua</p>
            </div>
          </div>
        </div>

        <!-- Actions -->
        <div class="flex flex-wrap gap-3 justify-center">
          <button class="inline-flex items-center gap-2 bg-accent-coral px-6 py-3 font-display font-bold text-bg-deep text-sm transition-all hover:shadow-lg hover:-translate-y-0.5" @click="playAgain">
            🔄 CHƠI LẠI
          </button>
          <button class="inline-flex items-center gap-2 border border-border-default bg-bg-surface px-4 py-2.5 text-sm text-text-secondary transition hover:border-accent-coral hover:text-accent-coral" @click="resetGame">
            🏠 Về intro
          </button>
        </div>
      </div>

      <!-- Footer -->
      <div class="mt-10 text-center">
        <RouterLink to="/" class="inline-flex items-center gap-2 border border-border-default bg-bg-surface px-5 py-2.5 text-sm text-text-secondary transition hover:border-accent-coral hover:text-text-primary">
          &larr; Về trang chủ
        </RouterLink>
        <p class="mt-4 text-text-dim text-xs font-display tracking-wider">1-2-3 ĐỨNG HÌNH 🗿 WEBRTC × CANVAS — BY HWG</p>
      </div>
    </div>
  </div>
</template>

<style scoped>
@keyframes scan {
  0% { top: -2px; }
  100% { top: 100%; }
}
.scan-line {
  position: absolute;
  left: 0; right: 0; height: 2px;
  background: linear-gradient(90deg, transparent 0%, rgba(239, 68, 68, 0.8) 50%, transparent 100%);
  box-shadow: 0 0 8px rgba(239, 68, 68, 0.5);
  animation: scan 2s linear infinite;
}

@keyframes busted-shake {
  0%, 100% { transform: translate(0); }
  10%, 30%, 50%, 70%, 90% { transform: translate(-4px, 2px); }
  20%, 40%, 60%, 80% { transform: translate(4px, -2px); }
}
.busted-shake {
  animation: busted-shake 0.4s ease-in-out;
}
</style>
