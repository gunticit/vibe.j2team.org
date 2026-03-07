<script setup lang="ts">
import { ref, computed, watch } from 'vue'
import { RouterLink } from 'vue-router'

// ─── Types ───────────────────────────────────────────────────────────────────

interface PowerLevel {
  level: number
  name: string
  icon: string
  minEntropy: number
  color: string
  bgGlow: string
}

interface CheckItem {
  label: string
  passed: boolean
}

// ─── Constants ───────────────────────────────────────────────────────────────

const POWER_LEVELS: PowerLevel[] = [
  { level: 0, name: 'Thường dân', icon: '👶', minEntropy: 0, color: 'text-red-500', bgGlow: 'shadow-red-500/20' },
  { level: 1, name: 'Chiến binh tập sự', icon: '🥋', minEntropy: 25, color: 'text-orange-400', bgGlow: 'shadow-orange-400/20' },
  { level: 2, name: 'Hiệp sĩ', icon: '⚔️', minEntropy: 40, color: 'text-accent-amber', bgGlow: 'shadow-amber-400/20' },
  { level: 3, name: 'Dũng sĩ', icon: '🛡️', minEntropy: 55, color: 'text-accent-sky', bgGlow: 'shadow-sky-400/20' },
  { level: 4, name: 'Anh hùng', icon: '🦸', minEntropy: 70, color: 'text-accent-coral', bgGlow: 'shadow-accent-coral/20' },
  { level: 5, name: 'Siêu anh hùng', icon: '🦸‍♂️✨', minEntropy: 85, color: 'text-purple-400', bgGlow: 'shadow-purple-400/30' },
]

const COMMON_PASSWORDS = [
  'password', '123456', '12345678', 'qwerty', 'abc123', 'monkey', 'master',
  'dragon', 'login', 'princess', 'football', 'shadow', 'sunshine', 'trustno1',
  'iloveyou', 'batman', 'access', 'hello', 'charlie', 'donald', '123456789',
  'password1', '1234567', 'letmein', 'welcome', 'admin', 'passw0rd',
  'matkhau', 'yeuem', 'anhyeuem', 'vietnam', '123321', '111111', '000000',
]

const KEYBOARD_PATTERNS = [
  'qwerty', 'qwertyuiop', 'asdfghjkl', 'zxcvbnm', 'qazwsx', '1qaz2wsx',
  'qweasd', 'asdzxc', '!@#$%^', '1234567890',
]

// ─── Roast Messages ─────────────────────────────────────────────────────────

function pickRandom<T>(arr: T[]): T {
  return arr[Math.floor(Math.random() * arr.length)]
}

const roastsByLevel: Record<number, string[]> = {
  0: [
    'Mật khẩu này yếu hơn wifi hàng xóm 📡',
    'Hacker nhìn mật khẩu này xong cười ra nước mắt 😂',
    'Mật khẩu này chỉ cần 1 giây để crack. Một. Giây.',
    'Đặt mật khẩu kiểu này thì khỏi cần đặt luôn cho nhanh 🤷',
    'Mật khẩu "123456" gọi, nó bảo của bạn còn yếu hơn nó.',
  ],
  1: [
    'Có cố gắng hơn rồi, nhưng hacker vẫn uống trà chờ được 🍵',
    'Mật khẩu tầm trung — giống khóa xe đạp giữa bãi xe hơi.',
    'Đủ để chặn em trai tọc mạch, nhưng không đủ chặn hacker.',
    'Level chiến binh tập sự — vẫn đang trong quá trình tu luyện.',
  ],
  2: [
    'Bắt đầu khá rồi! Hacker phải bật máy lên mới crack được 💻',
    'Mật khẩu cấp hiệp sĩ — bảo vệ được lâu đài nhỏ, chưa bảo vệ được vương quốc.',
    'Tạm ổn cho tài khoản ít quan trọng. Tài khoản ngân hàng thì... chưa.',
  ],
  3: [
    'Dũng sĩ xuất hiện! Mật khẩu này đã bắt đầu khiến hacker đau đầu 🤕',
    'Khá mạnh rồi! Hacker cần vài năm để crack. Nhưng vẫn có thể khỏe hơn.',
    'Level dũng sĩ — đủ sức chiến đấu, nhưng boss cuối vẫn là thử thách.',
  ],
  4: [
    'Anh hùng đây rồi! 💪 Mật khẩu này xứng đáng bảo vệ kho báu.',
    'Hacker nhìn mật khẩu này rồi quyết định... đi hack người khác cho lẹ.',
    'Mật khẩu cấp anh hùng — tự tin dùng cho ngân hàng được rồi! 🏦',
  ],
  5: [
    'SIÊU ANH HÙNG! 🦸‍♂️✨ Mật khẩu này bất khả xâm phạm!',
    'Vũ trụ heat death trước khi ai crack được mật khẩu này 🌌',
    'NSA gọi, họ muốn tuyển bạn vào đội bảo mật 🕵️',
    'Mật khẩu mạnh đến mức hacker thấy rồi bỏ nghề chuyển sang bán phở 🍜',
  ],
}

const patternRoasts: Record<string, string[]> = {
  common: [
    'Mật khẩu này nằm trong top 100 phổ biến nhất thế giới. Seriously.',
    'Đây là mật khẩu mà hacker thử ĐẦU TIÊN. Luôn luôn.',
  ],
  sequential: [
    'Dãy số liên tiếp? Sáng tạo ngang việc đặt tên con là ABC 📝',
    'Mật khẩu kiểu 123... — keyboard đi từ trái sang phải, hacker cũng vậy.',
  ],
  repeated: [
    'Lặp ký tự liên tục? Bàn phím bạn có bị kẹt không? ⌨️',
    'aaaa... hay bbbb... — hacker cần đúng 26 lần thử.',
  ],
  keyboard: [
    'Gõ theo hàng phím — "sáng tạo" kiểu lười biếng nhất vũ trụ ⌨️',
    'Pattern bàn phím — hacker có cả từ điển keyboard patterns đấy.',
  ],
  allNumbers: [
    'Toàn số? PIN ATM 4 số mà ngân hàng còn lo, huống chi...',
    'Mật khẩu toàn số — giới hạn trong 10 ký tự charset, yếu lắm.',
  ],
  allLower: [
    'Toàn chữ thường? 26 ký tự charset thôi — hacker mừng lắm 😏',
    'Không viết hoa, không số, không ký tự đặc biệt — minimalism quá mức.',
  ],
}

// ─── Vietnamese Passphrase Generator ─────────────────────────────────────────

const vnNouns = [
  'Rồng', 'Phượng', 'Trăng', 'Sao', 'Mây', 'Gió', 'Sông', 'Núi',
  'Biển', 'Hổ', 'Đại Bàng', 'Cá Voi', 'Sấm', 'Sen', 'Trúc', 'Mai',
]

const vnAdjectives = [
  'Vàng', 'Bạc', 'Thép', 'Lửa', 'Sắt', 'Ngọc', 'Kim', 'Bão',
]

const vnVerbs = [
  'Bay', 'Lướt', 'Gầm', 'Chiến', 'Phá', 'Xé', 'Đập', 'Rung',
]

const specialChars = '!@#$%^&*'

// ─── Core Logic ──────────────────────────────────────────────────────────────

function calcCharsetSize(password: string): number {
  let size = 0
  if (/[a-z]/.test(password)) size += 26
  if (/[A-Z]/.test(password)) size += 26
  if (/[0-9]/.test(password)) size += 10
  if (/[^a-zA-Z0-9]/.test(password)) size += 33
  return size
}

function calcEntropy(password: string): number {
  if (!password) return 0
  const charsetSize = calcCharsetSize(password)
  if (charsetSize === 0) return 0

  // Base entropy from charset × length
  let entropy = Math.log2(charsetSize) * password.length

  // Penalty for detected patterns
  const lower = password.toLowerCase()

  // Common password penalty
  if (COMMON_PASSWORDS.includes(lower)) {
    entropy = Math.min(entropy, 10)
  }

  // Sequential numbers penalty (123, 456, etc.)
  if (/^(\d)\1+$/.test(password) || /012|123|234|345|456|567|678|789|890/.test(password)) {
    entropy *= 0.4
  }

  // Repeated characters penalty
  if (/(.)\1{2,}/.test(password)) {
    entropy *= 0.6
  }

  // Keyboard pattern penalty
  if (KEYBOARD_PATTERNS.some((p) => lower.includes(p))) {
    entropy *= 0.3
  }

  return Math.round(entropy * 10) / 10
}

function detectPatterns(password: string): string[] {
  const patterns: string[] = []
  const lower = password.toLowerCase()

  if (COMMON_PASSWORDS.includes(lower)) {
    patterns.push('common')
  }
  if (/012|123|234|345|456|567|678|789|890|abc|bcd|cde|def|efg/.test(lower)) {
    patterns.push('sequential')
  }
  if (/(.)\1{2,}/.test(password)) {
    patterns.push('repeated')
  }
  if (KEYBOARD_PATTERNS.some((p) => lower.includes(p))) {
    patterns.push('keyboard')
  }
  if (/^\d+$/.test(password)) {
    patterns.push('allNumbers')
  }
  if (/^[a-z]+$/.test(password)) {
    patterns.push('allLower')
  }

  return patterns
}

function estimateCrackTime(password: string): string {
  if (!password) return '—'
  const charsetSize = calcCharsetSize(password)
  if (charsetSize === 0) return '0 giây'

  // Assume 10 billion guesses per second (modern GPU cluster)
  const guessesPerSecond = 10_000_000_000
  const totalCombinations = Math.pow(charsetSize, password.length)
  // On average, found in half the keyspace
  const seconds = totalCombinations / (2 * guessesPerSecond)

  // Common password? Instant.
  if (COMMON_PASSWORDS.includes(password.toLowerCase())) {
    return '< 1 giây 💀'
  }

  if (seconds < 0.001) return '< 0.001 giây 💀'
  if (seconds < 1) return `${seconds.toFixed(3)} giây`
  if (seconds < 60) return `${Math.round(seconds)} giây`
  if (seconds < 3600) return `${Math.round(seconds / 60)} phút`
  if (seconds < 86400) return `${Math.round(seconds / 3600)} giờ`
  if (seconds < 31_536_000) return `${Math.round(seconds / 86400)} ngày`
  if (seconds < 31_536_000 * 1000) return `${Math.round(seconds / 31_536_000)} năm`
  if (seconds < 31_536_000 * 1_000_000) return `${Math.round(seconds / (31_536_000 * 1000))} nghìn năm 🔒`
  if (seconds < 31_536_000 * 1_000_000_000) return `${Math.round(seconds / (31_536_000 * 1_000_000))} triệu năm 🔒`
  return `${(seconds / (31_536_000 * 1_000_000_000)).toExponential(1)} tỷ năm 🌌`
}

function getPowerLevel(entropy: number): PowerLevel {
  let result = POWER_LEVELS[0]
  for (const level of POWER_LEVELS) {
    if (entropy >= level.minEntropy) {
      result = level
    }
  }
  return result
}

function generateStrongPassword(): string {
  const lowercase = 'abcdefghijklmnopqrstuvwxyz'
  const uppercase = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ'
  const numbers = '0123456789'
  const symbols = '!@#$%^&*_+-='
  const all = lowercase + uppercase + numbers + symbols

  // Ensure at least one of each type
  let pw = ''
  pw += lowercase[Math.floor(Math.random() * lowercase.length)]
  pw += uppercase[Math.floor(Math.random() * uppercase.length)]
  pw += numbers[Math.floor(Math.random() * numbers.length)]
  pw += symbols[Math.floor(Math.random() * symbols.length)]

  // Fill remaining with random chars
  for (let i = 4; i < 18; i++) {
    pw += all[Math.floor(Math.random() * all.length)]
  }

  // Shuffle
  return pw
    .split('')
    .sort(() => Math.random() - 0.5)
    .join('')
}

function generatePassphrase(): string {
  const noun1 = vnNouns[Math.floor(Math.random() * vnNouns.length)]
  const adj = vnAdjectives[Math.floor(Math.random() * vnAdjectives.length)]
  const verb = vnVerbs[Math.floor(Math.random() * vnVerbs.length)]
  const noun2 = vnNouns[Math.floor(Math.random() * vnNouns.length)]
  const num = Math.floor(Math.random() * 90 + 10)
  const special = specialChars[Math.floor(Math.random() * specialChars.length)]

  return `${noun1}${adj}${verb}${noun2}${num}${special}`
}

// ─── State ───────────────────────────────────────────────────────────────────

const password = ref('')
const showPassword = ref(false)
const generatedPassword = ref('')
const generatedPassphrase = ref('')
const copied = ref(false)
const copiedPhrase = ref(false)
const showLevelUp = ref(false)
const previousLevel = ref(0)

// ─── Computed ────────────────────────────────────────────────────────────────

const entropy = computed(() => calcEntropy(password.value))
const powerLevel = computed(() => getPowerLevel(entropy.value))
const crackTime = computed(() => estimateCrackTime(password.value))
const detectedPatterns = computed(() => detectPatterns(password.value))

const roastMessage = computed(() => {
  if (!password.value) return ''
  // If patterns detected, show pattern-specific roast first
  for (const pattern of detectedPatterns.value) {
    if (patternRoasts[pattern]) {
      return pickRandom(patternRoasts[pattern])
    }
  }
  return pickRandom(roastsByLevel[powerLevel.value.level])
})

const strengthPercent = computed(() => {
  if (!password.value) return 0
  return Math.min(100, Math.round((entropy.value / 100) * 100))
})

const checklist = computed<CheckItem[]>(() => [
  { label: 'Ít nhất 8 ký tự', passed: password.value.length >= 8 },
  { label: 'Ít nhất 12 ký tự (khuyến nghị)', passed: password.value.length >= 12 },
  { label: 'Có chữ hoa (A-Z)', passed: /[A-Z]/.test(password.value) },
  { label: 'Có chữ thường (a-z)', passed: /[a-z]/.test(password.value) },
  { label: 'Có số (0-9)', passed: /[0-9]/.test(password.value) },
  { label: 'Có ký tự đặc biệt (!@#$...)', passed: /[^a-zA-Z0-9]/.test(password.value) },
  { label: 'Không phải mật khẩu phổ biến', passed: password.value.length > 0 && !COMMON_PASSWORDS.includes(password.value.toLowerCase()) },
  { label: 'Không có pattern bàn phím', passed: password.value.length > 0 && !KEYBOARD_PATTERNS.some((p) => password.value.toLowerCase().includes(p)) },
])

const passedChecks = computed(() => checklist.value.filter((c) => c.passed).length)

// ─── Watchers ────────────────────────────────────────────────────────────────

watch(powerLevel, (newLevel, oldLevel) => {
  if (oldLevel && newLevel.level > oldLevel.level) {
    showLevelUp.value = true
    setTimeout(() => {
      showLevelUp.value = false
    }, 1500)
  }
  previousLevel.value = newLevel.level
})

// ─── Actions ─────────────────────────────────────────────────────────────────

function toggleShowPassword() {
  showPassword.value = !showPassword.value
}

function onGeneratePassword() {
  generatedPassword.value = generateStrongPassword()
  copied.value = false
}

function onGeneratePassphrase() {
  generatedPassphrase.value = generatePassphrase()
  copiedPhrase.value = false
}

function copyToClipboard(text: string, type: 'password' | 'phrase') {
  navigator.clipboard.writeText(text)
  if (type === 'password') {
    copied.value = true
    setTimeout(() => { copied.value = false }, 2000)
  } else {
    copiedPhrase.value = true
    setTimeout(() => { copiedPhrase.value = false }, 2000)
  }
}

function useGenerated(text: string) {
  password.value = text
  showPassword.value = true
}
</script>

<template>
  <div class="min-h-screen bg-bg-deep text-text-primary font-body relative overflow-hidden">
    <!-- Level up flash -->
    <Transition name="flash">
      <div
        v-if="showLevelUp"
        class="fixed inset-0 pointer-events-none z-50 bg-accent-coral/10 animate-pulse"
      />
    </Transition>

    <!-- Power particles when strong -->
    <div
      v-if="powerLevel.level >= 4 && password"
      class="fixed inset-0 pointer-events-none z-10"
    >
      <div
        v-for="n in 15"
        :key="n"
        class="power-particle"
        :style="{
          left: `${Math.random() * 100}%`,
          animationDelay: `${Math.random() * 4}s`,
          animationDuration: `${3 + Math.random() * 3}s`,
          background: powerLevel.level === 5 ? '#a855f7' : '#FF6B4A',
        }"
      />
    </div>

    <div class="relative z-20 max-w-3xl mx-auto px-4 sm:px-6 py-8 sm:py-16">
      <!-- Header -->
      <div class="text-center mb-8 sm:mb-12 animate-fade-up">
        <div class="inline-block mb-4 sm:mb-6">
          <span class="text-5xl sm:text-6xl">🦸</span>
        </div>
        <h1 class="font-display text-4xl min-[375px]:text-5xl sm:text-6xl lg:text-7xl font-bold text-accent-coral mb-3 sm:mb-4 tracking-tight">
          Mật Khẩu Siêu Nhân
        </h1>
        <p class="text-text-secondary text-base sm:text-lg max-w-lg mx-auto">
          Mật khẩu của bạn mạnh cỡ nào? Nhập vào để biết power level 🦸‍♂️
        </p>
      </div>

      <!-- Password Input -->
      <div class="mb-6 sm:mb-8 animate-fade-up animate-delay-1">
        <div
          class="border bg-bg-surface transition-all duration-500"
          :class="password ? 'border-accent-coral/50 shadow-lg ' + powerLevel.bgGlow : 'border-border-default'"
        >
          <div class="flex items-center px-3 sm:px-4 py-2 border-b border-border-default bg-bg-elevated/50">
            <span class="text-text-dim text-xs font-display tracking-wider">MẬT KHẨU CỦA BẠN</span>
          </div>
          <div class="flex items-center p-3 sm:p-4">
            <input
              v-model="password"
              :type="showPassword ? 'text' : 'password'"
              placeholder="Nhập mật khẩu để kiểm tra..."
              class="flex-1 bg-transparent text-lg sm:text-xl text-text-primary font-mono focus:outline-none placeholder:text-text-dim/40"
              autocomplete="off"
              spellcheck="false"
            >
            <button
              class="ml-2 px-3 py-1.5 text-text-dim hover:text-text-primary transition text-sm border border-border-default hover:border-accent-coral/50"
              @click="toggleShowPassword"
            >
              {{ showPassword ? '🙈' : '👁️' }}
            </button>
          </div>
        </div>
      </div>

      <!-- Results Panel -->
      <div
        v-if="password"
        class="space-y-5 animate-fade-up"
      >
        <!-- Power Level Display -->
        <div class="border border-border-default bg-bg-surface p-5 sm:p-6">
          <div class="flex items-center justify-between mb-4">
            <h2 class="font-display text-xl sm:text-2xl font-semibold flex items-center gap-3">
              <span class="text-accent-coral font-display text-sm tracking-widest">//</span>
              Power Level
            </h2>
            <div class="text-right">
              <span class="text-3xl sm:text-4xl block leading-none">{{ powerLevel.icon }}</span>
            </div>
          </div>

          <!-- Power level name + level up animation -->
          <div class="relative mb-4">
            <p
              class="font-display text-2xl sm:text-3xl font-bold transition-all duration-500"
              :class="powerLevel.color"
            >
              {{ powerLevel.name }}
            </p>
            <Transition name="level-up">
              <span
                v-if="showLevelUp"
                class="absolute -top-6 left-0 font-display text-sm font-bold text-accent-amber animate-bounce"
              >
                ⬆️ LEVEL UP!
              </span>
            </Transition>
          </div>

          <!-- Power bar -->
          <div class="mb-3">
            <div class="flex justify-between text-xs text-text-dim mb-1.5 font-display tracking-wider">
              <span>👶 YẾU</span>
              <span>{{ Math.round(entropy) }} bits entropy</span>
              <span>MẠNH 🦸</span>
            </div>
            <div class="h-4 bg-bg-deep border border-border-default overflow-hidden relative">
              <div
                class="h-full transition-all duration-700 ease-out power-bar"
                :style="{ width: `${strengthPercent}%` }"
                :class="{
                  'bg-red-500': powerLevel.level === 0,
                  'bg-orange-400': powerLevel.level === 1,
                  'bg-accent-amber': powerLevel.level === 2,
                  'bg-accent-sky': powerLevel.level === 3,
                  'bg-accent-coral': powerLevel.level === 4,
                  'bg-purple-500 animate-pulse': powerLevel.level === 5,
                }"
              />
              <!-- Level markers -->
              <div class="absolute inset-0 flex">
                <div
                  v-for="i in 4"
                  :key="i"
                  class="border-r border-text-dim/20"
                  :style="{ width: '20%' }"
                />
              </div>
            </div>
          </div>

          <!-- Crack time -->
          <div class="flex items-center gap-2 p-3 bg-bg-deep border border-border-default mb-4">
            <span class="text-lg">⏱️</span>
            <div>
              <p class="text-text-dim text-xs font-display tracking-wider">THỜI GIAN BỊ CRACK (10 tỷ lần thử/giây)</p>
              <p
                class="font-display font-bold text-base sm:text-lg"
                :class="powerLevel.color"
              >
                {{ crackTime }}
              </p>
            </div>
          </div>

          <!-- Roast message -->
          <div class="p-3 sm:p-4 bg-bg-deep border border-border-default">
            <p class="text-accent-coral text-sm sm:text-base leading-relaxed">
              🎤 {{ roastMessage }}
            </p>
          </div>
        </div>

        <!-- Pattern warnings -->
        <div
          v-if="detectedPatterns.length > 0"
          class="border border-accent-coral/30 bg-accent-coral/5 p-4 sm:p-5"
        >
          <h3 class="font-display font-semibold text-accent-coral text-sm mb-3 flex items-center gap-2">
            ⚠️ Pattern nguy hiểm phát hiện
          </h3>
          <ul class="space-y-2">
            <li
              v-for="pattern in detectedPatterns"
              :key="pattern"
              class="flex items-start gap-2 text-sm text-text-secondary"
            >
              <span class="text-accent-coral shrink-0">•</span>
              <span>
                {{ pattern === 'common' ? 'Mật khẩu quá phổ biến — nằm trong danh sách bị crack đầu tiên' :
                   pattern === 'sequential' ? 'Dãy ký tự liên tiếp (123, abc...) — dễ đoán' :
                   pattern === 'repeated' ? 'Ký tự lặp lại — giảm đáng kể độ phức tạp' :
                   pattern === 'keyboard' ? 'Pattern bàn phím (qwerty, asdf...) — hacker có từ điển sẵn' :
                   pattern === 'allNumbers' ? 'Chỉ dùng số — charset quá nhỏ (10 ký tự)' :
                   pattern === 'allLower' ? 'Chỉ chữ thường — charset nhỏ (26 ký tự)' : pattern }}
              </span>
            </li>
          </ul>
        </div>

        <!-- Security Checklist -->
        <div class="border border-border-default bg-bg-surface p-5 sm:p-6">
          <h2 class="font-display text-lg sm:text-xl font-semibold flex items-center gap-3 mb-4">
            <span class="text-accent-amber font-display text-sm tracking-widest">//</span>
            Checklist bảo mật
            <span class="ml-auto text-sm text-text-dim font-display">{{ passedChecks }}/{{ checklist.length }}</span>
          </h2>
          <div class="grid gap-2">
            <div
              v-for="(item, idx) in checklist"
              :key="idx"
              class="flex items-center gap-2.5 p-2 transition-colors duration-300"
              :class="item.passed ? 'text-text-primary' : 'text-text-dim'"
            >
              <span
                class="w-5 h-5 flex items-center justify-center border text-xs transition-all duration-300 shrink-0"
                :class="item.passed ? 'border-green-500 bg-green-500/10 text-green-400' : 'border-border-default bg-bg-deep text-text-dim'"
              >
                {{ item.passed ? '✓' : '' }}
              </span>
              <span class="text-sm">{{ item.label }}</span>
            </div>
          </div>
        </div>
      </div>

      <!-- Password Generator -->
      <div class="mt-6 sm:mt-8 animate-fade-up animate-delay-2">
        <div class="border border-border-default bg-bg-surface p-5 sm:p-6">
          <h2 class="font-display text-lg sm:text-xl font-semibold flex items-center gap-3 mb-5">
            <span class="text-accent-sky font-display text-sm tracking-widest">//</span>
            Tạo mật khẩu mạnh
          </h2>

          <div class="space-y-4">
            <!-- Random password -->
            <div>
              <div class="flex items-center gap-2 mb-2">
                <button
                  class="inline-flex items-center gap-1.5 bg-accent-coral/10 border border-accent-coral/30 px-4 py-2 text-sm text-accent-coral font-display transition hover:bg-accent-coral/20"
                  @click="onGeneratePassword"
                >
                  🎲 Random Password
                </button>
                <span class="text-text-dim text-xs">(18 ký tự, đầy đủ charset)</span>
              </div>
              <div
                v-if="generatedPassword"
                class="flex items-center gap-2 p-3 bg-bg-deep border border-border-default"
              >
                <code class="flex-1 text-accent-amber font-mono text-sm break-all">{{ generatedPassword }}</code>
                <button
                  class="shrink-0 px-3 py-1.5 border border-border-default text-xs text-text-dim hover:text-text-primary hover:border-accent-coral transition"
                  @click="copyToClipboard(generatedPassword, 'password')"
                >
                  {{ copied ? '✅ Copied!' : '📋 Copy' }}
                </button>
                <button
                  class="shrink-0 px-3 py-1.5 border border-border-default text-xs text-text-dim hover:text-accent-sky hover:border-accent-sky transition"
                  @click="useGenerated(generatedPassword)"
                >
                  🧪 Test
                </button>
              </div>
            </div>

            <!-- Vietnamese passphrase -->
            <div>
              <div class="flex items-center gap-2 mb-2">
                <button
                  class="inline-flex items-center gap-1.5 bg-accent-sky/10 border border-accent-sky/30 px-4 py-2 text-sm text-accent-sky font-display transition hover:bg-accent-sky/20"
                  @click="onGeneratePassphrase"
                >
                  🇻🇳 Passphrase tiếng Việt
                </button>
                <span class="text-text-dim text-xs">(dễ nhớ, vẫn mạnh)</span>
              </div>
              <div
                v-if="generatedPassphrase"
                class="flex items-center gap-2 p-3 bg-bg-deep border border-border-default"
              >
                <code class="flex-1 text-accent-sky font-mono text-sm break-all">{{ generatedPassphrase }}</code>
                <button
                  class="shrink-0 px-3 py-1.5 border border-border-default text-xs text-text-dim hover:text-text-primary hover:border-accent-coral transition"
                  @click="copyToClipboard(generatedPassphrase, 'phrase')"
                >
                  {{ copiedPhrase ? '✅ Copied!' : '📋 Copy' }}
                </button>
                <button
                  class="shrink-0 px-3 py-1.5 border border-border-default text-xs text-text-dim hover:text-accent-sky hover:border-accent-sky transition"
                  @click="useGenerated(generatedPassphrase)"
                >
                  🧪 Test
                </button>
              </div>
            </div>
          </div>

          <!-- Tips -->
          <div class="mt-5 p-3 bg-bg-deep border border-border-default">
            <p class="text-text-dim text-xs font-display tracking-wider mb-2">💡 MẸO BẢO MẬT</p>
            <ul class="space-y-1 text-text-secondary text-xs">
              <li>• Dùng mật khẩu khác nhau cho mỗi tài khoản</li>
              <li>• Sử dụng Password Manager (Bitwarden, 1Password...)</li>
              <li>• Bật xác thực 2 lớp (2FA) khi có thể</li>
              <li>• Mật khẩu dài + đa dạng ký tự > mật khẩu ngắn + phức tạp</li>
            </ul>
          </div>
        </div>
      </div>

      <!-- Footer -->
      <div class="mt-12 sm:mt-16 text-center animate-fade-up animate-delay-3">
        <RouterLink
          to="/"
          class="inline-flex items-center gap-2 border border-border-default bg-bg-surface px-5 py-2.5 text-sm text-text-secondary transition hover:border-accent-coral hover:text-text-primary"
        >
          &larr; Về trang chủ
        </RouterLink>
        <p class="mt-4 text-text-dim text-xs font-display tracking-wider">
          POWER UP YOUR PASSWORD 🦸 BY HWG — FOR J2TEAM COMMUNITY
        </p>
      </div>
    </div>
  </div>
</template>

<style scoped>
/* Power bar glow */
.power-bar {
  box-shadow: 0 0 12px currentColor, 0 0 4px currentColor inset;
}

/* Power particles for strong passwords */
.power-particle {
  position: fixed;
  bottom: -10px;
  width: 6px;
  height: 6px;
  border-radius: 50%;
  animation: float-up linear infinite;
  opacity: 0;
  box-shadow: 0 0 8px currentColor;
}

@keyframes float-up {
  0% {
    transform: translateY(0) scale(1);
    opacity: 0;
  }
  15% {
    opacity: 0.6;
  }
  100% {
    transform: translateY(-100vh) scale(0);
    opacity: 0;
  }
}

/* Level up flash */
.flash-enter-active {
  animation: flash 0.5s ease-out;
}
.flash-leave-active {
  animation: flash 0.5s ease-out reverse;
}

@keyframes flash {
  0% { opacity: 0; }
  50% { opacity: 1; }
  100% { opacity: 0; }
}

/* Level up text */
.level-up-enter-active {
  animation: pop-up 0.5s ease-out;
}
.level-up-leave-active {
  animation: pop-up 0.5s ease-out reverse;
}

@keyframes pop-up {
  0% { transform: translateY(10px) scale(0.5); opacity: 0; }
  50% { transform: translateY(-5px) scale(1.1); opacity: 1; }
  100% { transform: translateY(0) scale(1); opacity: 1; }
}

/* Input focus glow */
input:focus {
  caret-color: #FF6B4A;
}
</style>
