<script setup lang="ts">
import { ref, computed, watch, onMounted, onUnmounted, nextTick } from 'vue'
import { RouterLink } from 'vue-router'

// ═══════════════════════════════════════════════════════════════════════════
// TYPES
// ═══════════════════════════════════════════════════════════════════════════

interface Token { type: string; value: string; line: number }

interface DevZodiac {
  id: string; name: string; icon: string; motto: string
  color: string; description: string; strength: string; weakness: string
}

interface StarPoint {
  label: string; value: number; angle: number
  x: number; y: number; baseX: number; baseY: number
  twinklePhase: number; size: number
}

interface Particle { x: number; y: number; vx: number; vy: number; life: number; maxLife: number; size: number; hue: number }

interface CodeMetrics {
  totalLines: number; codeLines: number; commentLines: number; commentDensity: number
  functionCount: number; avgFunctionLength: number; maxNesting: number
  cyclomaticComplexity: number; avgParams: number; maxParams: number
  importCount: number; hasAsync: boolean; hasTryCatch: number
  hasRegex: boolean; hasCSS: boolean; antiPatternCount: number
  uniqueIdentifiers: number; stringCount: number
}

// ═══════════════════════════════════════════════════════════════════════════
// TOKENIZER (proven from Phong Thủy Code)
// ═══════════════════════════════════════════════════════════════════════════

const KEYWORDS = new Set([
  'async','await','break','case','catch','class','const','continue','default',
  'delete','do','else','export','extends','false','finally','for','function',
  'if','import','in','instanceof','let','new','null','of','return','static',
  'super','switch','this','throw','true','try','typeof','var','void','while','yield',
])

function tokenize(code: string): Token[] {
  const tokens: Token[] = []
  let pos = 0; let line = 1; const len = code.length
  while (pos < len) {
    const ch = code[pos]
    if (ch === '\n') { line++; pos++; continue }
    if (/\s/.test(ch)) { pos++; continue }
    if (ch === '/' && code[pos + 1] === '/') {
      let end = pos + 2; while (end < len && code[end] !== '\n') end++
      tokens.push({ type: 'comment', value: code.slice(pos, end), line }); pos = end; continue
    }
    if (ch === '/' && code[pos + 1] === '*') {
      let end = pos + 2; const sl = line
      while (end < len - 1 && !(code[end] === '*' && code[end + 1] === '/')) { if (code[end] === '\n') line++; end++ }
      end += 2; tokens.push({ type: 'comment', value: code.slice(pos, end), line: sl }); pos = end; continue
    }
    if (ch === '/' && pos > 0) {
      // Regex detection
      const prev = tokens[tokens.length - 1]
      if (prev && (prev.type === 'operator' || prev.type === 'punctuation' || prev.type === 'keyword')) {
        let end = pos + 1; let escaped = false
        while (end < len) {
          if (escaped) { escaped = false; end++; continue }
          if (code[end] === '\\') { escaped = true; end++; continue }
          if (code[end] === '/') { end++; while (end < len && /[gimsuy]/.test(code[end])) end++; break }
          if (code[end] === '\n') break
          end++
        }
        if (end > pos + 1 && code[end - 1] !== '\n') {
          tokens.push({ type: 'regex', value: code.slice(pos, end), line }); pos = end; continue
        }
      }
    }
    if (ch === '"' || ch === "'" || ch === '`') {
      const q = ch; let end = pos + 1
      while (end < len) { if (code[end] === '\\') { end += 2; continue }; if (code[end] === '\n') line++; if (code[end] === q) { end++; break }; end++ }
      tokens.push({ type: 'string', value: code.slice(pos, end), line }); pos = end; continue
    }
    if (/[0-9]/.test(ch)) {
      let end = pos + 1; while (end < len && /[0-9a-fA-FxXoObB._eE+-]/.test(code[end])) end++
      tokens.push({ type: 'number', value: code.slice(pos, end), line }); pos = end; continue
    }
    if (/[a-zA-Z_$]/.test(ch)) {
      let end = pos + 1; while (end < len && /[a-zA-Z0-9_$]/.test(code[end])) end++
      const word = code.slice(pos, end)
      tokens.push({ type: KEYWORDS.has(word) ? 'keyword' : 'identifier', value: word, line }); pos = end; continue
    }
    if ('{}[]();,.:?'.includes(ch)) { tokens.push({ type: 'punctuation', value: ch, line }); pos++; continue }
    if ('+-*/%=<>!&|^~@#'.includes(ch)) { tokens.push({ type: 'operator', value: ch, line }); pos++; continue }
    pos++
  }
  return tokens
}

// ═══════════════════════════════════════════════════════════════════════════
// METRICS CALCULATOR
// ═══════════════════════════════════════════════════════════════════════════

function calcMetrics(code: string, tokens: Token[]): CodeMetrics {
  const lines = code.split('\n')
  const totalLines = lines.length
  const codeLines = lines.filter((l) => l.trim().length > 0).length
  const commentTokens = tokens.filter((t) => t.type === 'comment')
  const commentDensity = codeLines > 0 ? commentTokens.length / codeLines : 0

  let functionCount = 0; let totalFuncLen = 0; let inFunc = false; let funcBrace = 0; let funcStart = 0
  const paramCounts: number[] = []; let maxParams = 0

  for (let i = 0; i < tokens.length; i++) {
    const t = tokens[i]
    if ((t.type === 'keyword' && t.value === 'function') || (t.type === 'operator' && t.value === '>' && i > 0 && tokens[i - 1]?.value === '=')) {
      functionCount++; inFunc = true; funcBrace = 0; funcStart = t.line
      let pc = 0; let j = i + 1
      while (j < tokens.length && tokens[j]?.value !== '(') j++; j++
      let pd = 1
      while (j < tokens.length && pd > 0) {
        if (tokens[j]?.value === '(') pd++; else if (tokens[j]?.value === ')') pd--
        else if (tokens[j]?.type === 'identifier' && pd === 1) pc++; j++
      }
      paramCounts.push(pc); if (pc > maxParams) maxParams = pc
    }
    if (inFunc) {
      if (t.value === '{') funcBrace++; if (t.value === '}') { funcBrace--; if (funcBrace <= 0) { totalFuncLen += t.line - funcStart + 1; inFunc = false } }
    }
  }

  let maxNesting = 0; let nesting = 0
  for (const t of tokens) { if (t.value === '{') { nesting++; if (nesting > maxNesting) maxNesting = nesting }; if (t.value === '}') nesting-- }

  let complexity = 1
  for (const t of tokens) {
    if (t.type === 'keyword' && ['if', 'else', 'for', 'while', 'do', 'case', 'catch'].includes(t.value)) complexity++
    if (t.type === 'operator' && ['&', '|', '?'].includes(t.value)) complexity++
  }

  const hasAsync = tokens.some((t) => t.type === 'keyword' && (t.value === 'async' || t.value === 'await'))
  const hasTryCatch = tokens.filter((t) => t.type === 'keyword' && t.value === 'try').length
  const hasRegex = tokens.some((t) => t.type === 'regex')
  const hasCSS = /(?:color|background|margin|padding|display|flex|grid|font|border|width|height|position)\s*:/.test(code)
  const identifiers = new Set(tokens.filter((t) => t.type === 'identifier').map((t) => t.value))

  // Anti-patterns
  let antiPatterns = 0
  if (tokens.some((t) => t.type === 'keyword' && t.value === 'var')) antiPatterns++
  if (/[^!=<>]={2}[^=]/.test(code)) antiPatterns++
  if (/:\s*any\b/.test(code)) antiPatterns++
  if (/console\.log/.test(code)) antiPatterns++
  if (/catch\s*\([^)]*\)\s*\{\s*\}/.test(code)) antiPatterns++

  return {
    totalLines, codeLines, commentLines: commentTokens.length, commentDensity,
    functionCount, avgFunctionLength: functionCount > 0 ? totalFuncLen / functionCount : 0,
    maxNesting, cyclomaticComplexity: complexity,
    avgParams: paramCounts.length > 0 ? paramCounts.reduce((a, b) => a + b, 0) / paramCounts.length : 0,
    maxParams, importCount: tokens.filter((t) => t.value === 'import').length,
    hasAsync, hasTryCatch, hasRegex, hasCSS, antiPatternCount: antiPatterns,
    uniqueIdentifiers: identifiers.size, stringCount: tokens.filter((t) => t.type === 'string').length,
  }
}

// ═══════════════════════════════════════════════════════════════════════════
// 12 CUNG DEV
// ═══════════════════════════════════════════════════════════════════════════

const ZODIACS: DevZodiac[] = [
  { id: 'architect', name: 'Kiến Trúc Sư', icon: '🏛️', color: '#38BDF8', motto: '"Thiết kế trước, code sau"', description: 'Bạn là người nhìn xa trông rộng. Code của bạn có cấu trúc rõ ràng, nhiều lớp abstraction, imports gọn gàng. Bạn xây hệ thống, không chỉ viết function.', strength: 'System design, kiến trúc sạch', weakness: 'Đôi khi over-engineer cho task đơn giản' },
  { id: 'ninja', name: 'Ninja', icon: '🥷', color: '#4ADE80', motto: '"Less is more"', description: 'Code bạn ngắn gọn, hiệu quả, mỗi dòng đều có ý nghĩa. Không dư thừa, không rườm rà. Đồng nghiệp đọc code bạn mà tấm tắc khen.', strength: 'Code ngắn nhưng chất, clean', weakness: 'Đôi khi ngắn quá đến khó hiểu' },
  { id: 'warrior', name: 'Chiến Binh', icon: '⚔️', color: '#FF6B4A', motto: '"Không if nào tôi không xử lý"', description: 'Bạn viết logic phức tạp như một chiến binh trên chiến trường. If-else, switch-case, nested loops — bạn xử lý tất cả. Cyclomatic complexity là bạn thân.', strength: 'Xử lý business logic phức tạp', weakness: 'Code đôi khi quá nhiều nhánh, khó maintain' },
  { id: 'wizard', name: 'Pháp Sư', icon: '🧙', color: '#A855F7', motto: '"Regex là phép thuật"', description: 'Regex, string manipulation, data transformation — bạn biến đổi dữ liệu như phép thuật. Đồng nghiệp nhìn regex của bạn mà choáng váng.', strength: 'Data transformation, regex master', weakness: 'Regex phức tạp sáng viết tối quên' },
  { id: 'teacher', name: 'Thầy Giáo', icon: '📚', color: '#FFB830', motto: '"Code không comment = câu đố"', description: 'Bạn comment mọi thứ, viết doc cẩn thận. Code bạn ai đọc cũng hiểu. Bạn là tài sản quý của team vì onboarding dev mới dễ dàng hơn.', strength: 'Documentation, readable code', weakness: 'Comment nhiều quá đôi khi thừa thãi' },
  { id: 'artist', name: 'Mỹ Thuật Gia', icon: '🎨', color: '#F472B6', motto: '"Pixel perfect hoặc không gì cả"', description: 'CSS là ngôn ngữ yêu thích. Bạn quan tâm đến UI/UX hơn bất kỳ ai. Animation, responsive, dark mode — bạn làm tất.', strength: 'UI/UX, frontend excellence', weakness: 'Backend bạn coi là... đất lạ' },
  { id: 'firefighter', name: 'Lính Cứu Hỏa', icon: '🧯', color: '#FB923C', motto: '"try-catch là bạn, error là thù"', description: 'Defensive programming level max. Mọi thứ đều được try-catch, mọi input đều validate. Code bạn không crash — never ever.', strength: 'Error handling, reliability', weakness: 'Code dài hơn vì xử lý edge cases' },
  { id: 'speedster', name: 'Tốc Độ', icon: '⚡', color: '#FBBF24', motto: '"Async/await is life"', description: 'Performance là ưu tiên số 1. Async, Promise, callback — bạn xử lý concurrent operations như uống nước. API calls chạy song song, không bao giờ blocking.', strength: 'Async programming, performance', weakness: 'Race conditions đôi khi rình rập' },
  { id: 'scientist', name: 'Nhà Khoa Học', icon: '🔬', color: '#2DD4BF', motto: '"SOLID không phải buzzword"', description: 'Methodical, structured, clean. Mỗi function một việc, mỗi module một trách nhiệm. SOLID principles bạn thuộc lòng. Code review bạn là nightmare của junior.', strength: 'Clean architecture, SOLID', weakness: 'Perfectionism đôi khi chậm delivery' },
  { id: 'monk', name: 'Đạo Sĩ', icon: '🧘', color: '#6EE7B7', motto: '"Balance in all things"', description: 'Code cân bằng, ít anti-pattern, không extreme. Không quá ngắn, không quá dài. Bạn là Zen master — mọi thứ vừa đủ, hài hòa tự nhiên.', strength: 'Balanced code, no extremes', weakness: 'Không nổi bật ở mặt nào cả' },
  { id: 'madscientist', name: 'Mad Scientist', icon: '🤪', color: '#EF4444', motto: '"It works, don\'t touch it"', description: 'Code bạn... chạy. Nhưng không ai hiểu sao. var, ==, console.log khắp nơi — bạn code bằng bản năng, không theo quy tắc. Legacy code là tác phẩm nghệ thuật.', strength: 'Ship fast, creativity', weakness: 'Đồng nghiệp khóc khi maintain' },
  { id: 'legend', name: 'Huyền Thoại', icon: '🐉', color: '#C084FC', motto: '"Full-stack, full-power"', description: 'Tất cả metrics đều đẹp. Code sạch, logic rõ, comment đủ, error handling đầy đủ, abstraction đúng. Bạn là developer mà mọi team đều mơ ước.', strength: 'Mọi thứ', weakness: 'Bạn không có weakness (hoặc giấu giỏi)' },
]

function determineZodiac(m: CodeMetrics): { primary: DevZodiac; secondary: DevZodiac; scores: Map<string, number> } {
  const scores = new Map<string, number>()

  // Score each zodiac based on metrics
  scores.set('architect', Math.min(100, m.importCount * 15 + m.uniqueIdentifiers * 1.5 + (m.functionCount > 5 ? 30 : 0)))
  scores.set('ninja', Math.min(100, m.codeLines < 30 ? 50 : 0 + (m.avgFunctionLength < 10 ? 40 : 0) + (m.commentDensity < 0.05 ? 10 : 0)))
  scores.set('warrior', Math.min(100, m.cyclomaticComplexity * 6 + m.maxNesting * 8))
  scores.set('wizard', Math.min(100, (m.hasRegex ? 60 : 0) + m.stringCount * 5 + (m.uniqueIdentifiers > 15 ? 20 : 0)))
  scores.set('teacher', Math.min(100, m.commentDensity * 200 + m.commentLines * 8))
  scores.set('artist', Math.min(100, (m.hasCSS ? 70 : 0) + m.stringCount * 3))
  scores.set('firefighter', Math.min(100, m.hasTryCatch * 30 + (m.maxParams > 2 ? 20 : 0) + (m.cyclomaticComplexity > 5 ? 20 : 0)))
  scores.set('speedster', Math.min(100, (m.hasAsync ? 60 : 0) + m.importCount * 8 + (m.functionCount > 3 ? 20 : 0)))
  scores.set('scientist', Math.min(100, (m.functionCount > 4 ? 30 : 0) + (m.avgFunctionLength < 20 ? 30 : 0) + (m.maxNesting < 4 ? 20 : 0) + (m.antiPatternCount === 0 ? 20 : 0)))
  scores.set('monk', Math.min(100, (m.antiPatternCount === 0 ? 40 : 0) + (m.avgFunctionLength > 5 && m.avgFunctionLength < 25 ? 30 : 0) + (m.commentDensity > 0.05 && m.commentDensity < 0.4 ? 30 : 0)))
  scores.set('madscientist', Math.min(100, m.antiPatternCount * 20 + (m.maxNesting > 4 ? 30 : 0) + (m.commentDensity < 0.01 && m.codeLines > 20 ? 20 : 0)))
  scores.set('legend', Math.min(100, (m.functionCount > 3 ? 15 : 0) + (m.avgFunctionLength < 20 ? 15 : 0) + (m.commentDensity > 0.05 ? 15 : 0) + (m.antiPatternCount === 0 ? 20 : 0) + (m.hasTryCatch > 0 ? 15 : 0) + (m.importCount > 0 ? 20 : 0)))

  // Find primary and secondary
  let best = 'ninja'; let bestScore = 0; let second = 'monk'; let secondScore = 0
  for (const [id, score] of scores) {
    if (score > bestScore) { second = best; secondScore = bestScore; best = id; bestScore = score }
    else if (score > secondScore) { second = id; secondScore = score }
  }

  const primary = ZODIACS.find((z) => z.id === best) ?? ZODIACS[1]
  const secondary = ZODIACS.find((z) => z.id === second) ?? ZODIACS[9]
  return { primary, secondary, scores }
}

// ═══════════════════════════════════════════════════════════════════════════
// STARS & FORTUNE
// ═══════════════════════════════════════════════════════════════════════════

const STAR_NAMES = ['Tử Vi', 'Thiên Cơ', 'Thái Dương', 'Vũ Khúc', 'Thiên Đồng', 'Liêm Trinh', 'Thiên Phủ', 'Thái Âm']

function pickRandom<T>(arr: T[]): T { return arr[Math.floor(Math.random() * arr.length)] }

const fortuneTemplates: Record<string, string[]> = {
  career: [
    'Năm nay sao {star} chiếu mệnh, sự nghiệp code thăng hoa. Đặc biệt thuận lợi khi {action}.',
    'Sao {star} tọa mệnh, công danh rạng rỡ. {action} sẽ mang lại kết quả bất ngờ.',
    '{star} đắc địa, quý nhân phù trợ. Nên {action} để phát huy tối đa.',
  ],
  love: [
    'Đào hoa vượng nhờ sao {star}. Code đẹp tự nhiên có người thương 💕',
    'Sao {star} chiếu Phu Thê, pair programming năm nay rất hợp duyên.',
    'Tình duyên khởi sắc — viết code clean thì crush cũng notice 😏',
  ],
  warning: [
    '⚠️ Cẩn thận sao Kình Dương — tránh merge vào Friday chiều. Hạn kị: deploy sau 10h tối.',
    '⚠️ Sao Đà La quấy phá — backup database trước khi refactor. Taboo: xóa branch mà chưa merge.',
    '⚠️ Hạn Tam Tai — avoid touching legacy code tháng này. Nên: viết test trước khi sửa bug.',
  ],
}

function generateFortune(primary: DevZodiac, m: CodeMetrics): { career: string; love: string; warning: string } {
  const star = pickRandom(STAR_NAMES)
  const actions: Record<string, string[]> = {
    architect: ['thiết kế microservices', 'review kiến trúc team', 'viết RFC cho feature mới'],
    ninja: ['refactor code legacy', 'tối ưu performance', 'viết one-liner functions'],
    warrior: ['giải quyết business logic phức tạp', 'handle edge cases', 'viết integration tests'],
    wizard: ['viết data pipeline', 'build parser/transformer', 'master thêm một ngôn ngữ mới'],
    teacher: ['mentor junior devs', 'viết technical blog', 'update documentation'],
    artist: ['redesign UI', 'implement design system', 'tạo animation library'],
    firefighter: ['improve error handling', 'setup monitoring', 'viết disaster recovery plan'],
    speedster: ['optimize API performance', 'implement caching layer', 'parallelize tasks'],
    scientist: ['apply design patterns mới', 'refactor theo SOLID', 'setup CI/CD pipeline'],
    monk: ['meditate on code quality', 'balance features vs tech debt', 'code review cẩn thận'],
    madscientist: ['learn TypeScript strict mode', 'viết unit tests', 'đọc Clean Code book'],
    legend: ['open source dự án cá nhân', 'speak at tech conference', 'mentor đội dev'],
  }
  const action = pickRandom(actions[primary.id] ?? actions['ninja'])

  return {
    career: pickRandom(fortuneTemplates['career']).replace('{star}', star).replace('{action}', action),
    love: pickRandom(fortuneTemplates['love']).replace('{star}', star),
    warning: pickRandom(fortuneTemplates['warning']),
  }
}

// ═══════════════════════════════════════════════════════════════════════════
// CANVAS CONSTELLATION
// ═══════════════════════════════════════════════════════════════════════════

const canvasRef = ref<HTMLCanvasElement | null>(null)
let animFrameId: number | null = null
let stars: StarPoint[] = []
let particles: Particle[] = []
let canvasW = 0; let canvasH = 0; let time = 0

function initConstellation(m: CodeMetrics) {
  const canvas = canvasRef.value
  if (!canvas) return

  const dpr = window.devicePixelRatio || 1
  const displayW = Math.min(400, window.innerWidth - 32)
  const displayH = displayW
  canvas.width = displayW * dpr; canvas.height = displayH * dpr
  canvas.style.width = `${displayW}px`; canvas.style.height = `${displayH}px`
  canvasW = displayW; canvasH = displayH

  const cx = displayW / 2; const cy = displayH / 2; const maxR = displayW * 0.35
  const metrics = [
    { label: 'Logic', value: Math.min(100, m.cyclomaticComplexity * 8) },
    { label: 'Clean', value: Math.min(100, (m.antiPatternCount === 0 ? 60 : 0) + (m.avgFunctionLength < 20 ? 40 : 0)) },
    { label: 'Docs', value: Math.min(100, m.commentDensity * 250) },
    { label: 'Modular', value: Math.min(100, m.functionCount * 10 + m.importCount * 12) },
    { label: 'Safety', value: Math.min(100, m.hasTryCatch * 25 + (m.antiPatternCount === 0 ? 50 : 0)) },
    { label: 'Depth', value: Math.min(100, m.maxNesting * 15 + m.uniqueIdentifiers * 2) },
    { label: 'Speed', value: Math.min(100, (m.hasAsync ? 50 : 0) + m.functionCount * 8) },
    { label: 'Style', value: Math.min(100, (m.hasCSS ? 40 : 0) + m.stringCount * 4 + (m.hasRegex ? 30 : 0)) },
  ]

  stars = metrics.map((metric, i) => {
    const angle = (i / metrics.length) * Math.PI * 2 - Math.PI / 2
    const r = (metric.value / 100) * maxR
    return {
      label: metric.label, value: metric.value, angle,
      x: cx + r * Math.cos(angle), y: cy + r * Math.sin(angle),
      baseX: cx + maxR * Math.cos(angle), baseY: cy + maxR * Math.sin(angle),
      twinklePhase: Math.random() * Math.PI * 2, size: 2 + (metric.value / 100) * 4,
    }
  })

  particles = []
  for (let i = 0; i < 60; i++) {
    particles.push({
      x: Math.random() * displayW, y: Math.random() * displayH,
      vx: (Math.random() - 0.5) * 0.3, vy: (Math.random() - 0.5) * 0.3,
      life: Math.random() * 200, maxLife: 150 + Math.random() * 150,
      size: 0.5 + Math.random() * 1.5, hue: 20 + Math.random() * 40,
    })
  }

  if (animFrameId) cancelAnimationFrame(animFrameId)
  time = 0
  animate()
}

function animate() {
  const canvas = canvasRef.value
  if (!canvas) return
  const ctx = canvas.getContext('2d')
  if (!ctx) return

  const dpr = window.devicePixelRatio || 1
  ctx.setTransform(dpr, 0, 0, dpr, 0, 0)
  time += 0.016

  // Clear with nebula
  ctx.fillStyle = '#0F1923'
  ctx.fillRect(0, 0, canvasW, canvasH)

  // Nebula glow
  const cx = canvasW / 2; const cy = canvasH / 2
  const nebula = ctx.createRadialGradient(cx, cy, 0, cx, cy, canvasW * 0.45)
  nebula.addColorStop(0, 'rgba(255, 107, 74, 0.08)')
  nebula.addColorStop(0.5, 'rgba(255, 184, 48, 0.04)')
  nebula.addColorStop(1, 'rgba(0, 0, 0, 0)')
  ctx.fillStyle = nebula; ctx.fillRect(0, 0, canvasW, canvasH)

  // Background particles
  for (const p of particles) {
    p.x += p.vx; p.y += p.vy; p.life++
    if (p.life > p.maxLife || p.x < 0 || p.x > canvasW || p.y < 0 || p.y > canvasH) {
      p.x = Math.random() * canvasW; p.y = Math.random() * canvasH
      p.life = 0; p.vx = (Math.random() - 0.5) * 0.3; p.vy = (Math.random() - 0.5) * 0.3
    }
    const alpha = Math.sin((p.life / p.maxLife) * Math.PI) * 0.6
    ctx.beginPath(); ctx.arc(p.x, p.y, p.size, 0, Math.PI * 2)
    ctx.fillStyle = `hsla(${p.hue}, 80%, 75%, ${alpha})`; ctx.fill()
  }

  // Grid rings
  const maxR = canvasW * 0.35
  for (let ring = 1; ring <= 4; ring++) {
    const r = (maxR / 4) * ring
    ctx.beginPath(); ctx.arc(cx, cy, r, 0, Math.PI * 2)
    ctx.strokeStyle = `rgba(37, 53, 73, ${ring === 4 ? 0.5 : 0.2})`; ctx.lineWidth = 0.5; ctx.stroke()
  }

  // Axis lines
  for (const s of stars) {
    ctx.beginPath(); ctx.moveTo(cx, cy); ctx.lineTo(s.baseX, s.baseY)
    ctx.strokeStyle = 'rgba(37, 53, 73, 0.3)'; ctx.lineWidth = 0.5; ctx.stroke()
  }

  // Constellation fill
  ctx.beginPath()
  for (let i = 0; i <= stars.length; i++) {
    const s = stars[i % stars.length]
    if (i === 0) ctx.moveTo(s.x, s.y); else ctx.lineTo(s.x, s.y)
  }
  ctx.fillStyle = 'rgba(255, 107, 74, 0.1)'; ctx.fill()
  ctx.strokeStyle = 'rgba(255, 107, 74, 0.6)'; ctx.lineWidth = 1.5; ctx.stroke()

  // Connection lines (constellation pattern)
  ctx.strokeStyle = 'rgba(255, 184, 48, 0.25)'; ctx.lineWidth = 0.8
  for (let i = 0; i < stars.length; i++) {
    const j = (i + 2) % stars.length
    ctx.beginPath(); ctx.moveTo(stars[i].x, stars[i].y); ctx.lineTo(stars[j].x, stars[j].y); ctx.stroke()
  }

  // Stars with twinkle
  for (const s of stars) {
    const twinkle = 0.6 + 0.4 * Math.sin(time * 3 + s.twinklePhase)
    const glowSize = s.size * 3

    // Glow
    const glow = ctx.createRadialGradient(s.x, s.y, 0, s.x, s.y, glowSize)
    glow.addColorStop(0, `rgba(255, 184, 48, ${0.4 * twinkle})`); glow.addColorStop(1, 'rgba(255, 184, 48, 0)')
    ctx.beginPath(); ctx.arc(s.x, s.y, glowSize, 0, Math.PI * 2); ctx.fillStyle = glow; ctx.fill()

    // Star point
    ctx.beginPath(); ctx.arc(s.x, s.y, s.size * twinkle, 0, Math.PI * 2)
    ctx.fillStyle = `rgba(255, 240, 220, ${twinkle})`; ctx.fill()

    // Label
    const labelR = canvasW * 0.35 + 18
    const lx = cx + labelR * Math.cos(s.angle); const ly = cy + labelR * Math.sin(s.angle)
    ctx.textAlign = 'center'; ctx.textBaseline = 'middle'
    ctx.font = '10px "Be Vietnam Pro", sans-serif'
    ctx.fillStyle = `rgba(139, 157, 181, ${0.7 + 0.3 * twinkle})`
    ctx.fillText(s.label, lx, ly - 6)
    ctx.font = 'bold 10px "Be Vietnam Pro", sans-serif'
    ctx.fillStyle = `rgba(255, 184, 48, ${twinkle})`
    ctx.fillText(String(s.value), lx, ly + 6)
  }

  // Center emblem
  const pulse = 1 + 0.1 * Math.sin(time * 2)
  const cGlow = ctx.createRadialGradient(cx, cy, 0, cx, cy, 20 * pulse)
  cGlow.addColorStop(0, 'rgba(255, 107, 74, 0.3)'); cGlow.addColorStop(1, 'rgba(255, 107, 74, 0)')
  ctx.beginPath(); ctx.arc(cx, cy, 20 * pulse, 0, Math.PI * 2); ctx.fillStyle = cGlow; ctx.fill()
  ctx.beginPath(); ctx.arc(cx, cy, 4, 0, Math.PI * 2); ctx.fillStyle = '#FF6B4A'; ctx.fill()

  animFrameId = requestAnimationFrame(animate)
}

// ═══════════════════════════════════════════════════════════════════════════
// SAMPLE CODES
// ═══════════════════════════════════════════════════════════════════════════

const sampleArchitect = `import { UserService } from './services/user'
import { OrderService } from './services/order'
import { NotificationService } from './services/notification'
import type { AppConfig, ProcessResult } from './types'

/**
 * Application controller - orchestrates all services
 */
class AppController {
  constructor(
    private userService: UserService,
    private orderService: OrderService,
    private notifier: NotificationService,
  ) {}

  async processOrder(userId: string, items: string[]): Promise<ProcessResult> {
    const user = await this.userService.getById(userId)
    if (!user) {
      return { success: false, error: 'User not found' }
    }

    try {
      const order = await this.orderService.create(user, items)
      await this.notifier.send(user.email, 'Order confirmed')
      return { success: true, data: order }
    } catch (error) {
      await this.notifier.sendAlert('Order failed', error)
      throw error
    }
  }
}`

const sampleChaos = `var data = fetch("/api")
var x = 10
var temp = ""
function doStuff(a, b, c, d, e) {
  if (a != null) {
    if (b == true) {
      for (var i = 0; i < c; i++) {
        if (d) {
          if (e && x > 5) {
            console.log("yo " + a + " " + b)
            temp = temp + "data" + i
            // TODO: fix later
            try { JSON.parse(temp) } catch(err) {}
          }
        }
      }
    }
  }
  console.log(temp)
  return temp
}`

// ═══════════════════════════════════════════════════════════════════════════
// STATE
// ═══════════════════════════════════════════════════════════════════════════

const codeInput = ref('')
const hasResult = ref(false)
const isAnalyzing = ref(false)
const metrics = ref<CodeMetrics | null>(null)
const primaryZodiac = ref<DevZodiac | null>(null)
const secondaryZodiac = ref<DevZodiac | null>(null)
const fortune = ref<{ career: string; love: string; warning: string } | null>(null)
const zodiacScores = ref<Map<string, number>>(new Map())

const lineCount = computed(() => codeInput.value ? codeInput.value.split('\n').length : 0)
const lineNumbers = computed(() => Array.from({ length: lineCount.value || 1 }, (_, i) => i + 1))

const topZodiacs = computed(() => {
  const entries = Array.from(zodiacScores.value.entries()).sort((a, b) => b[1] - a[1]).slice(0, 5)
  return entries.map(([id, score]) => ({ zodiac: ZODIACS.find((z) => z.id === id) ?? ZODIACS[0], score }))
})

// ═══════════════════════════════════════════════════════════════════════════
// ACTIONS
// ═══════════════════════════════════════════════════════════════════════════

function analyze() {
  if (!codeInput.value.trim()) return
  isAnalyzing.value = true; hasResult.value = false

  setTimeout(async () => {
    const tokens = tokenize(codeInput.value)
    const m = calcMetrics(codeInput.value, tokens)
    metrics.value = m

    const result = determineZodiac(m)
    primaryZodiac.value = result.primary
    secondaryZodiac.value = result.secondary
    zodiacScores.value = result.scores

    fortune.value = generateFortune(result.primary, m)

    isAnalyzing.value = false; hasResult.value = true

    await nextTick()
    initConstellation(m)
  }, 800)
}

function loadSample(type: 'architect' | 'chaos') {
  codeInput.value = type === 'architect' ? sampleArchitect : sampleChaos
  hasResult.value = false
}

function clearAll() { codeInput.value = ''; hasResult.value = false; metrics.value = null }

onMounted(() => { window.addEventListener('resize', () => { if (metrics.value) initConstellation(metrics.value) }) })
onUnmounted(() => { if (animFrameId) cancelAnimationFrame(animFrameId) })
</script>

<template>
  <div class="min-h-screen bg-bg-deep text-text-primary font-body">
    <div class="max-w-4xl mx-auto px-4 sm:px-6 py-8 sm:py-16">
      <!-- Header -->
      <div class="text-center mb-8 sm:mb-12 animate-fade-up">
        <div class="inline-block mb-4"><span class="text-5xl sm:text-6xl">🌌</span></div>
        <h1 class="font-display text-4xl min-[375px]:text-5xl sm:text-6xl lg:text-7xl font-bold text-accent-coral mb-3 tracking-tight">Tử Vi Dev</h1>
        <p class="text-text-secondary text-base sm:text-lg max-w-xl mx-auto">Paste code → xem <strong class="text-accent-amber">lá số tử vi lập trình viên</strong> với chòm sao animated 🌠</p>
      </div>

      <!-- Code Input -->
      <div class="mb-6 sm:mb-8 animate-fade-up animate-delay-1">
        <div class="border border-border-default bg-bg-surface transition-all duration-300 hover:border-accent-coral/30">
          <div class="flex items-center justify-between px-3 sm:px-4 py-2.5 border-b border-border-default bg-bg-elevated/50">
            <div class="flex items-center gap-2">
              <div class="flex gap-1.5"><span class="w-3 h-3 rounded-full bg-accent-coral/60" /><span class="w-3 h-3 rounded-full bg-accent-amber/60" /><span class="w-3 h-3 rounded-full bg-green-500/60" /></div>
              <span class="text-text-dim text-xs font-display tracking-wider ml-2 hidden sm:inline">your-code.ts</span>
            </div>
            <span class="text-text-dim text-xs font-display">{{ lineCount }} dòng</span>
          </div>
          <div class="flex">
            <div class="py-4 px-2 sm:px-3 text-right select-none border-r border-border-default bg-bg-deep/50 min-w-[2.5rem] sm:min-w-[3rem]">
              <div v-for="num in lineNumbers" :key="num" class="text-text-dim text-xs leading-6 font-mono">{{ num }}</div>
            </div>
            <textarea v-model="codeInput" placeholder="// Paste code JS/TS vào đây...&#10;// Hệ thống sẽ tokenize, phân tích,&#10;// và xem lá số tử vi dev 🌌" class="flex-1 bg-transparent px-3 sm:px-4 py-4 text-text-primary text-sm font-mono resize-none focus:outline-none min-h-[180px] sm:min-h-[220px] leading-6 placeholder:text-text-dim/50" spellcheck="false" />
          </div>
        </div>
      </div>

      <!-- Actions -->
      <div class="flex flex-wrap gap-3 mb-8 animate-fade-up animate-delay-2">
        <button class="flex-1 sm:flex-none inline-flex items-center justify-center gap-2 bg-accent-coral px-6 sm:px-8 py-3 font-display font-semibold text-bg-deep text-sm sm:text-base transition-all duration-300 hover:shadow-lg hover:shadow-accent-coral/20 hover:-translate-y-0.5 disabled:opacity-50 disabled:cursor-not-allowed disabled:hover:translate-y-0" :disabled="!codeInput.trim() || isAnalyzing" @click="analyze">
          <span v-if="isAnalyzing" class="animate-spin">🌀</span><span v-else>🌌</span>
          {{ isAnalyzing ? 'Đang xem tử vi...' : 'XEM TỬ VI DEV' }}
        </button>
        <button class="inline-flex items-center gap-1.5 border border-border-default bg-bg-surface px-3 sm:px-4 py-2.5 text-xs sm:text-sm text-text-secondary transition hover:border-accent-sky hover:text-accent-sky" @click="loadSample('architect')">🏛️ Code kiến trúc</button>
        <button class="inline-flex items-center gap-1.5 border border-border-default bg-bg-surface px-3 sm:px-4 py-2.5 text-xs sm:text-sm text-text-secondary transition hover:border-accent-coral hover:text-accent-coral" @click="loadSample('chaos')">🤪 Code hỗn loạn</button>
        <button v-if="codeInput" class="inline-flex items-center gap-1.5 border border-border-default bg-bg-surface px-3 py-2.5 text-xs text-text-secondary transition hover:border-accent-coral hover:text-accent-coral" @click="clearAll">✕</button>
      </div>

      <!-- ═══════ RESULTS ═══════ -->
      <div v-if="hasResult && primaryZodiac && secondaryZodiac && fortune && metrics" class="space-y-6 animate-fade-up">

        <!-- Zodiac Card + Constellation -->
        <div class="border border-border-default bg-bg-surface p-5 sm:p-6">
          <h2 class="font-display text-xl sm:text-2xl font-semibold flex items-center gap-3 mb-5">
            <span class="text-accent-coral font-display text-sm tracking-widest">//</span> Lá số Tử Vi Dev
          </h2>
          <div class="flex flex-col lg:flex-row items-center gap-6">
            <!-- Canvas -->
            <div class="shrink-0"><canvas ref="canvasRef" class="block" /></div>
            <!-- Zodiac Info -->
            <div class="flex-1 text-center lg:text-left">
              <p class="text-text-dim text-xs font-display tracking-widest mb-1">CUNG CHỦ ĐẠO</p>
              <div class="flex items-center gap-3 justify-center lg:justify-start mb-2">
                <span class="text-4xl sm:text-5xl">{{ primaryZodiac.icon }}</span>
                <h3 class="font-display text-3xl sm:text-4xl font-bold" :style="{ color: primaryZodiac.color }">{{ primaryZodiac.name }}</h3>
              </div>
              <p class="text-accent-amber text-sm font-display italic mb-3">{{ primaryZodiac.motto }}</p>
              <p class="text-text-secondary text-sm leading-relaxed mb-4">{{ primaryZodiac.description }}</p>
              <div class="flex flex-wrap gap-2">
                <span class="inline-flex items-center gap-1 px-3 py-1 border border-green-500/30 bg-green-500/5 text-green-400 text-xs">💪 {{ primaryZodiac.strength }}</span>
                <span class="inline-flex items-center gap-1 px-3 py-1 border border-accent-coral/30 bg-accent-coral/5 text-accent-coral text-xs">😅 {{ primaryZodiac.weakness }}</span>
              </div>
            </div>
          </div>
        </div>

        <!-- Secondary Zodiac -->
        <div class="border border-border-default bg-bg-surface p-4 sm:p-5 flex items-center gap-4">
          <span class="text-3xl">{{ secondaryZodiac.icon }}</span>
          <div class="flex-1">
            <p class="text-text-dim text-xs font-display tracking-widest">CUNG PHỤ</p>
            <p class="font-display font-semibold" :style="{ color: secondaryZodiac.color }">{{ secondaryZodiac.name }}</p>
            <p class="text-text-dim text-xs italic">{{ secondaryZodiac.motto }}</p>
          </div>
          <div class="text-right">
            <p class="text-text-dim text-xs font-display">SAO CHIẾU MỆNH</p>
            <p class="font-display font-bold text-accent-amber text-lg">{{ STAR_NAMES[Math.floor(Math.random() * STAR_NAMES.length)] }}</p>
          </div>
        </div>

        <!-- Fortune -->
        <div class="border border-border-default bg-bg-surface p-5 sm:p-6">
          <h2 class="font-display text-lg font-semibold flex items-center gap-3 mb-4">
            <span class="text-accent-amber font-display text-sm tracking-widest">//</span> Lời phán năm {{ new Date().getFullYear() }}
          </h2>
          <div class="space-y-3">
            <div class="p-3 bg-bg-deep border border-border-default"><p class="text-text-dim text-xs font-display tracking-wider mb-1">🏆 SỰ NGHIỆP</p><p class="text-text-secondary text-sm">{{ fortune.career }}</p></div>
            <div class="p-3 bg-bg-deep border border-border-default"><p class="text-text-dim text-xs font-display tracking-wider mb-1">💕 ĐÀO HOA</p><p class="text-text-secondary text-sm">{{ fortune.love }}</p></div>
            <div class="p-3 bg-bg-deep border border-accent-coral/20"><p class="text-accent-coral text-sm">{{ fortune.warning }}</p></div>
          </div>
        </div>

        <!-- Top 5 Zodiac Ranking -->
        <div class="border border-border-default bg-bg-surface p-5 sm:p-6">
          <h2 class="font-display text-lg font-semibold flex items-center gap-3 mb-4">
            <span class="text-accent-sky font-display text-sm tracking-widest">//</span> Bảng xếp hạng cung
          </h2>
          <div class="space-y-2">
            <div v-for="(item, idx) in topZodiacs" :key="item.zodiac.id" class="flex items-center gap-3 p-2">
              <span class="font-display font-bold text-text-dim text-sm w-6">{{ idx + 1 }}</span>
              <span class="text-xl">{{ item.zodiac.icon }}</span>
              <span class="font-display font-semibold text-sm flex-1" :style="{ color: item.zodiac.color }">{{ item.zodiac.name }}</span>
              <div class="w-24 sm:w-32 h-2 bg-bg-deep border border-border-default overflow-hidden">
                <div class="h-full transition-all duration-700" :style="{ width: `${item.score}%`, backgroundColor: item.zodiac.color }" />
              </div>
              <span class="font-display text-xs font-bold w-8 text-right" :style="{ color: item.zodiac.color }">{{ item.score }}</span>
            </div>
          </div>
        </div>

        <!-- Metrics -->
        <div class="border border-border-default bg-bg-surface p-5 sm:p-6">
          <h2 class="font-display text-base font-semibold flex items-center gap-3 mb-3">
            <span class="text-accent-coral font-display text-sm tracking-widest">//</span> Tokenizer Metrics
          </h2>
          <div class="grid grid-cols-2 sm:grid-cols-4 gap-2">
            <div class="p-2 bg-bg-deep border border-border-default text-center"><p class="font-display text-xl font-bold text-accent-coral">{{ metrics.codeLines }}</p><p class="text-text-dim text-xs">Code lines</p></div>
            <div class="p-2 bg-bg-deep border border-border-default text-center"><p class="font-display text-xl font-bold text-accent-amber">{{ metrics.cyclomaticComplexity }}</p><p class="text-text-dim text-xs">Complexity</p></div>
            <div class="p-2 bg-bg-deep border border-border-default text-center"><p class="font-display text-xl font-bold text-accent-sky">{{ metrics.functionCount }}</p><p class="text-text-dim text-xs">Functions</p></div>
            <div class="p-2 bg-bg-deep border border-border-default text-center"><p class="font-display text-xl font-bold text-green-400">{{ metrics.antiPatternCount }}</p><p class="text-text-dim text-xs">Anti-patterns</p></div>
          </div>
        </div>
      </div>

      <!-- Footer -->
      <div class="mt-12 sm:mt-16 text-center animate-fade-up animate-delay-3">
        <RouterLink to="/" class="inline-flex items-center gap-2 border border-border-default bg-bg-surface px-5 py-2.5 text-sm text-text-secondary transition hover:border-accent-coral hover:text-text-primary">&larr; Về trang chủ</RouterLink>
        <p class="mt-4 text-text-dim text-xs font-display tracking-wider">TỬ VI DEV 🌌 TOKENIZER × CANVAS × VĂN HÓA — BY HWG</p>
      </div>
    </div>
  </div>
</template>

<style scoped>
canvas { image-rendering: -webkit-optimize-contrast; }
textarea::-webkit-scrollbar { width: 6px; }
textarea::-webkit-scrollbar-track { background: transparent; }
textarea::-webkit-scrollbar-thumb { background: #253549; border-radius: 3px; }
</style>
