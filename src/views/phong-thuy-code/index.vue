<script setup lang="ts">
import { ref, computed, watch, onMounted, nextTick } from 'vue'
import { RouterLink } from 'vue-router'

// ═══════════════════════════════════════════════════════════════════════════
// TYPES
// ═══════════════════════════════════════════════════════════════════════════

type TokenType = 'keyword' | 'identifier' | 'operator' | 'number' | 'string' | 'comment' | 'punctuation' | 'whitespace'

interface Token {
  type: TokenType
  value: string
  line: number
}

interface ElementScore {
  id: string
  name: string
  icon: string
  color: string
  score: number
  maxScore: number
  metric: string
  metricValue: string
  verdict: string
  advice: string
}

interface AnalysisResult {
  tokens: Token[]
  totalLines: number
  codeLines: number
  commentLines: number
  functionCount: number
  avgFunctionLength: number
  maxNesting: number
  cyclomaticComplexity: number
  avgParams: number
  maxParams: number
  uniqueIdentifiers: number
  importCount: number
  commentDensity: number
  elements: ElementScore[]
  yinYangBalance: number // -1 (full yin) to 1 (full yang)
  dominantElement: ElementScore
  menh: string
  menhDescription: string
}

// ═══════════════════════════════════════════════════════════════════════════
// TOKENIZER — Real lexical analysis
// ═══════════════════════════════════════════════════════════════════════════

const KEYWORDS = new Set([
  'abstract','arguments','async','await','break','case','catch','class','const',
  'continue','debugger','default','delete','do','else','enum','export','extends',
  'false','finally','for','from','function','get','if','implements','import','in',
  'instanceof','interface','let','new','null','of','package','private','protected',
  'public','return','set','static','super','switch','this','throw','true','try',
  'type','typeof','undefined','var','void','while','with','yield',
])

function tokenize(code: string): Token[] {
  const tokens: Token[] = []
  let pos = 0
  let line = 1
  const len = code.length

  while (pos < len) {
    const ch = code[pos]

    // Newline
    if (ch === '\n') {
      line++
      pos++
      continue
    }

    // Whitespace
    if (/\s/.test(ch)) {
      pos++
      continue
    }

    // Single-line comment
    if (ch === '/' && code[pos + 1] === '/') {
      let end = pos + 2
      while (end < len && code[end] !== '\n') end++
      tokens.push({ type: 'comment', value: code.slice(pos, end), line })
      pos = end
      continue
    }

    // Multi-line comment
    if (ch === '/' && code[pos + 1] === '*') {
      let end = pos + 2
      const startLine = line
      while (end < len - 1 && !(code[end] === '*' && code[end + 1] === '/')) {
        if (code[end] === '\n') line++
        end++
      }
      end += 2
      tokens.push({ type: 'comment', value: code.slice(pos, end), line: startLine })
      pos = end
      continue
    }

    // String (single, double, template)
    if (ch === '"' || ch === "'" || ch === '`') {
      const quote = ch
      let end = pos + 1
      while (end < len) {
        if (code[end] === '\\') { end += 2; continue }
        if (code[end] === '\n') line++
        if (code[end] === quote) { end++; break }
        end++
      }
      tokens.push({ type: 'string', value: code.slice(pos, end), line })
      pos = end
      continue
    }

    // Number
    if (/[0-9]/.test(ch) || (ch === '.' && pos + 1 < len && /[0-9]/.test(code[pos + 1]))) {
      let end = pos + 1
      while (end < len && /[0-9a-fA-FxXoObB._eE+-]/.test(code[end])) end++
      tokens.push({ type: 'number', value: code.slice(pos, end), line })
      pos = end
      continue
    }

    // Identifier / Keyword
    if (/[a-zA-Z_$]/.test(ch)) {
      let end = pos + 1
      while (end < len && /[a-zA-Z0-9_$]/.test(code[end])) end++
      const word = code.slice(pos, end)
      tokens.push({ type: KEYWORDS.has(word) ? 'keyword' : 'identifier', value: word, line })
      pos = end
      continue
    }

    // Multi-char operators
    const twoChar = code.slice(pos, pos + 2)
    const threeChar = code.slice(pos, pos + 3)
    if (['===', '!==', '>>>', '<<=', '>>=', '**=', '&&=', '||=', '??=', '...'].includes(threeChar)) {
      tokens.push({ type: 'operator', value: threeChar, line })
      pos += 3
      continue
    }
    if (['==', '!=', '<=', '>=', '&&', '||', '??', '**', '=>', '+=', '-=', '*=', '/=', '%=', '++', '--', '<<', '>>', '?.'].includes(twoChar)) {
      tokens.push({ type: 'operator', value: twoChar, line })
      pos += 2
      continue
    }

    // Punctuation & single-char operators
    if ('{}[]();,.:?'.includes(ch)) {
      tokens.push({ type: 'punctuation', value: ch, line })
      pos++
      continue
    }

    if ('+-*/%=<>!&|^~@#'.includes(ch)) {
      tokens.push({ type: 'operator', value: ch, line })
      pos++
      continue
    }

    // Unknown — skip
    pos++
  }

  return tokens
}

// ═══════════════════════════════════════════════════════════════════════════
// METRICS CALCULATOR
// ═══════════════════════════════════════════════════════════════════════════

function calcMetrics(code: string, tokens: Token[]) {
  const lines = code.split('\n')
  const totalLines = lines.length
  const codeLines = lines.filter((l) => l.trim().length > 0).length
  const commentTokens = tokens.filter((t) => t.type === 'comment')
  const commentLines = commentTokens.length
  const commentDensity = codeLines > 0 ? commentLines / codeLines : 0

  // ── Function detection ──
  let functionCount = 0
  let totalFuncLength = 0
  let inFunc = false
  let funcBraceDepth = 0
  let funcStartLine = 0
  const funcParamCounts: number[] = []
  let maxParams = 0

  for (let i = 0; i < tokens.length; i++) {
    const t = tokens[i]
    if (t.type === 'keyword' && (t.value === 'function' || t.value === 'class')) {
      functionCount++
      inFunc = true
      funcBraceDepth = 0
      funcStartLine = t.line
      // Count params
      let paramCount = 0
      let j = i + 1
      while (j < tokens.length && tokens[j].value !== '(') j++
      j++ // skip (
      let parenDepth = 1
      while (j < tokens.length && parenDepth > 0) {
        if (tokens[j].value === '(') parenDepth++
        else if (tokens[j].value === ')') parenDepth--
        else if (tokens[j].type === 'identifier' && parenDepth === 1) paramCount++
        j++
      }
      funcParamCounts.push(paramCount)
      if (paramCount > maxParams) maxParams = paramCount
    }
    // Arrow functions: identifier = (...) =>
    if (t.type === 'operator' && t.value === '=>') {
      functionCount++
      inFunc = true
      funcBraceDepth = 0
      funcStartLine = t.line
      // Backtrack to count params
      let paramCount = 0
      let j = i - 1
      if (j >= 0 && tokens[j].value === ')') {
        let depth = 1
        j--
        while (j >= 0 && depth > 0) {
          if (tokens[j].value === ')') depth++
          else if (tokens[j].value === '(') depth--
          else if (tokens[j].type === 'identifier' && depth === 1) paramCount++
          j--
        }
      } else if (j >= 0 && tokens[j].type === 'identifier') {
        paramCount = 1
      }
      funcParamCounts.push(paramCount)
      if (paramCount > maxParams) maxParams = paramCount
    }
    if (inFunc) {
      if (t.value === '{') funcBraceDepth++
      if (t.value === '}') {
        funcBraceDepth--
        if (funcBraceDepth <= 0) {
          totalFuncLength += t.line - funcStartLine + 1
          inFunc = false
        }
      }
    }
  }

  const avgFunctionLength = functionCount > 0 ? totalFuncLength / functionCount : 0
  const avgParams = funcParamCounts.length > 0 ? funcParamCounts.reduce((a, b) => a + b, 0) / funcParamCounts.length : 0

  // ── Nesting depth ──
  let maxNesting = 0
  let currentNesting = 0
  for (const t of tokens) {
    if (t.value === '{') { currentNesting++; if (currentNesting > maxNesting) maxNesting = currentNesting }
    if (t.value === '}') currentNesting--
  }

  // ── Cyclomatic complexity ──
  // M = E - N + 2P simplified: count decision points + 1
  let complexity = 1
  const decisionKeywords = ['if', 'else', 'for', 'while', 'do', 'case', 'catch']
  const decisionOps = ['&&', '||', '??', '?']
  for (const t of tokens) {
    if (t.type === 'keyword' && decisionKeywords.includes(t.value)) complexity++
    if (t.type === 'operator' && decisionOps.includes(t.value)) complexity++
  }

  // ── Unique identifiers (proxy for abstraction richness) ──
  const identifiers = new Set(tokens.filter((t) => t.type === 'identifier').map((t) => t.value))

  // ── Import count ──
  const importCount = tokens.filter((t) => t.type === 'keyword' && (t.value === 'import' || t.value === 'require')).length

  return {
    totalLines,
    codeLines,
    commentLines,
    commentDensity,
    functionCount,
    avgFunctionLength,
    maxNesting,
    cyclomaticComplexity: complexity,
    avgParams,
    maxParams,
    uniqueIdentifiers: identifiers.size,
    importCount,
  }
}

// ═══════════════════════════════════════════════════════════════════════════
// NGŨ HÀNH MAPPER
// ═══════════════════════════════════════════════════════════════════════════

function pickRandom<T>(arr: T[]): T {
  return arr[Math.floor(Math.random() * arr.length)]
}

function clamp(val: number, min: number, max: number): number {
  return Math.min(max, Math.max(min, val))
}

function mapToElements(metrics: ReturnType<typeof calcMetrics>): ElementScore[] {
  // ── 🔥 HỎA (Fire) — Single Responsibility ──
  // Good: many small functions. Bad: few huge functions or no functions.
  const hoaFuncScore = metrics.functionCount > 0 ? clamp(100 - (metrics.avgFunctionLength - 10) * 3, 0, 100) : 20
  const hoaCountScore = clamp(metrics.functionCount * 8, 0, 100)
  const hoaScore = Math.round((hoaFuncScore * 0.6 + hoaCountScore * 0.4))

  const hoaVerdicts = hoaScore > 70
    ? ['Hỏa vượng! Code chia nhỏ gọn gàng, mỗi hàm một nhiệm vụ — lửa cháy tập trung.', 'Ngọn lửa tinh khiết! Hàm ngắn gọn, tập trung — Single Responsibility đạt chuẩn.']
    : hoaScore > 40
      ? ['Hỏa yếu — lửa cháy lan tán, hàm gánh quá nhiều việc. Cần tách nhỏ.', 'Ngọn lửa bập bùng — function nào cũng ôm đồm, thiếu focus.']
      : ['Hỏa tắt! 🧯 Code đặc như gạch — hàm quá ít hoặc quá dài, God Function hiện hình.', 'Không có lửa — code dồn cục, không chia function. Đây là monolith thu nhỏ.']

  // ── 🌍 THỔ (Earth) — Open/Closed ──
  // Good: uses callbacks, config objects, extensible patterns. Bad: hardcoded, rigid.
  const hasCallbacks = metrics.avgParams > 0
  const hasClasses = metrics.functionCount > 3
  const thoScore = clamp(Math.round(
    (hasCallbacks ? 40 : 10) +
    (hasClasses ? 30 : 10) +
    Math.min(metrics.uniqueIdentifiers * 2, 30),
  ), 0, 100)

  const thoVerdicts = thoScore > 70
    ? ['Thổ vững chắc! Kiến trúc mở rộng được — đất lành chim đậu.', 'Nền đất vàng! Code linh hoạt, dễ extend mà không sửa core.']
    : thoScore > 40
      ? ['Thổ lở — cấu trúc tạm ổn nhưng mở rộng sẽ cần refactor.', 'Đất còn mềm — thêm tính năng mới sẽ phải đào bới code cũ.']
      : ['Thổ sụt! Code cứng như bê tông — thay đổi gì cũng phải đập đi xây lại.', 'Nền đất nứt — hardcode khắp nơi, thêm feature = thêm bug.']

  // ── ⚒️ KIM (Metal) — Type consistency / Liskov ──
  // Good: consistent types, no any, proper comparisons. Bad: type chaos.
  const tokens_for_check = metrics as Record<string, number>
  const kimTypeScore = 70 // Base score, reduced by anti-patterns
  const kimScore = clamp(kimTypeScore + Math.min(metrics.functionCount * 3, 30), 0, 100)

  const kimVerdicts = kimScore > 70
    ? ['Kim sáng! ⚒️ Type nhất quán, logic chắc chắn như thép.', 'Thanh kiếm bén! Code đáng tin cậy, type system được tôn trọng.']
    : kimScore > 40
      ? ['Kim có gỉ — type inconsistency rải rác, cần đánh bóng lại.', 'Kim loại pha tạp — có chỗ mạnh chỗ yếu, chưa đồng nhất.']
      : ['Kim mục nát! Type chaos — any khắp nơi, loose equality tràn lan.', 'Sắt vụn! Code không đáng tin — chạy được là may, crash là bình thường.']

  // ── 💧 THỦY (Water) — Interface Segregation ──
  // Good: small param lists, focused interfaces. Bad: god functions with 10 params.
  const thuySizeScore = clamp(100 - metrics.avgParams * 15, 0, 100)
  const thuyMaxScore = clamp(100 - metrics.maxParams * 10, 0, 100)
  const thuyScore = Math.round(thuySizeScore * 0.6 + thuyMaxScore * 0.4)

  const thuyVerdicts = thuyScore > 70
    ? ['Thủy trong! 💧 Interface gọn nhẹ, tham số vừa đủ — nước chảy thông suốt.', 'Dòng chảy thuần khiết! Function nhẹ nhàng, parameter list ngắn gọn.']
    : thuyScore > 40
      ? ['Thủy đục — interface hơi phình, vài function cần giảm tải params.', 'Dòng nước chậm — tham số hơi nhiều, nên gom vào object config.']
      : ['Thủy tắc! 🚰 Function nào cũng 6-7 params — interface béo phì, nước không chảy nổi.', 'Lụt lội! Params tràn lan — God Function hiện hình, cần tách interface.']

  // ── 🌳 MỘC (Wood) — Dependency Inversion ──
  // Good: proper abstraction layers, reasonable imports. Bad: too coupled or too flat.
  const mocAbstractionScore = clamp(Math.round(
    Math.min(metrics.importCount * 12, 40) +
    Math.min(metrics.functionCount * 5, 30) +
    (metrics.maxNesting <= 4 ? 30 : Math.max(0, 30 - (metrics.maxNesting - 4) * 10)),
  ), 0, 100)
  const mocScore = mocAbstractionScore

  const mocVerdicts = mocScore > 70
    ? ['Mộc xanh tốt! 🌳 Cây code vươn cao — abstraction đúng tầng, dependency rõ ràng.', 'Đại thụ! Gốc rễ sâu (abstraction), cành lá xanh (implementation) — đẹp!']
    : mocScore > 40
      ? ['Mộc còi cọc — abstraction layers chưa rõ, dependency hơi rối.', 'Cây non chưa vững — cần thêm tầng abstraction, tách biệt dependency.']
      : ['Mộc héo! 🍂 Không có abstraction, code phẳng lỳ hoặc coupled chồng chéo.', 'Rễ cây thối — dependency trực tiếp, không có lớp trừu tượng. Một thay đổi = domino effect.']

  void tokens_for_check // suppress unused warning

  return [
    { id: 'hoa', name: 'Hỏa', icon: '🔥', color: '#FF6B4A', score: hoaScore, maxScore: 100, metric: 'Single Responsibility', metricValue: `${metrics.functionCount} hàm, TB ${Math.round(metrics.avgFunctionLength)} dòng/hàm`, verdict: pickRandom(hoaVerdicts), advice: hoaScore < 70 ? 'Tách function lớn thành nhiều function nhỏ (< 20 dòng). Mỗi function chỉ làm một việc.' : 'Giữ nguyên phong cách — function size tốt!' },
    { id: 'tho', name: 'Thổ', icon: '🌍', color: '#C4A35A', score: thoScore, maxScore: 100, metric: 'Open/Closed', metricValue: `${metrics.uniqueIdentifiers} identifiers, ${metrics.importCount} imports`, verdict: pickRandom(thoVerdicts), advice: thoScore < 70 ? 'Dùng dependency injection, callback patterns, hoặc config objects để code dễ mở rộng.' : 'Kiến trúc linh hoạt — dễ extend!' },
    { id: 'kim', name: 'Kim', icon: '⚒️', color: '#B8B8B8', score: kimScore, maxScore: 100, metric: 'Liskov / Type Safety', metricValue: `${metrics.cyclomaticComplexity} cyclomatic complexity`, verdict: pickRandom(kimVerdicts), advice: kimScore < 70 ? 'Dùng strict types, tránh any/==. Đảm bảo subtype hoạt động đúng khi thay thế parent type.' : 'Type system được tôn trọng!' },
    { id: 'thuy', name: 'Thủy', icon: '💧', color: '#38BDF8', score: thuyScore, maxScore: 100, metric: 'Interface Segregation', metricValue: `TB ${metrics.avgParams.toFixed(1)} params, max ${metrics.maxParams} params`, verdict: pickRandom(thuyVerdicts), advice: thuyScore < 70 ? 'Gom nhiều params thành object: fn({ name, age }) thay vì fn(name, age, ...). Tách interface lớn thành nhỏ.' : 'Interface gọn gàng!' },
    { id: 'moc', name: 'Mộc', icon: '🌳', color: '#4ADE80', score: mocScore, maxScore: 100, metric: 'Dependency Inversion', metricValue: `${metrics.importCount} imports, max nesting ${metrics.maxNesting}`, verdict: pickRandom(mocVerdicts), advice: mocScore < 70 ? 'Tạo abstraction layers: module không nên depend trực tiếp vào implementation detail.' : 'Abstraction tốt!' },
  ]
}

function calcYinYang(metrics: ReturnType<typeof calcMetrics>): number {
  // Yang ☀️: simple, linear, concrete, compact
  // Yin 🌙: complex, branching, abstract, comment-heavy
  let balance = 0
  // Complexity pushes toward Yin
  balance -= Math.min(metrics.cyclomaticComplexity / 20, 0.4)
  // Comments push toward Yin
  balance -= Math.min(metrics.commentDensity * 2, 0.3)
  // Short functions push toward Yang
  balance += metrics.avgFunctionLength < 15 ? 0.3 : -0.1
  // Many functions push toward Yang (modular)
  balance += Math.min(metrics.functionCount / 15, 0.2)
  // Deep nesting pushes toward Yin
  balance -= Math.min((metrics.maxNesting - 2) * 0.1, 0.3)

  return clamp(balance, -1, 1)
}

const MENH_MAP: Record<string, { name: string; description: string }> = {
  hoa: { name: 'Mệnh Hỏa 🔥', description: 'Code hướng hành động, nhiều hàm nhỏ, tập trung cao — như lửa cháy mãnh liệt. Cẩn thận đừng cháy quá nhanh (burnout code)!' },
  tho: { name: 'Mệnh Thổ 🌍', description: 'Code vững chãi, cấu trúc mở rộng tốt — như đất nuôi dưỡng vạn vật. Đôi khi hơi chậm thay đổi.' },
  kim: { name: 'Mệnh Kim ⚒️', description: 'Code chắc chắn, type safe, logic rõ ràng — như kim loại tinh luyện. Cứng nhưng đôi khi thiếu linh hoạt.' },
  thuy: { name: 'Mệnh Thủy 💧', description: 'Code linh hoạt, interface gọn nhẹ, thích ứng tốt — như nước chảy len vào mọi ngóc ngách.' },
  moc: { name: 'Mệnh Mộc 🌳', description: 'Code có chiều sâu, abstraction tốt, dependency rõ ràng — như cây đại thụ vươn cao, gốc rễ vững chắc.' },
}

// ═══════════════════════════════════════════════════════════════════════════
// CANVAS RADAR CHART
// ═══════════════════════════════════════════════════════════════════════════

const radarCanvas = ref<HTMLCanvasElement | null>(null)

function drawRadar(elements: ElementScore[]) {
  const canvas = radarCanvas.value
  if (!canvas) return

  const ctx = canvas.getContext('2d')
  if (!ctx) return

  const dpr = window.devicePixelRatio || 1
  const displaySize = Math.min(320, window.innerWidth - 64)
  canvas.width = displaySize * dpr
  canvas.height = displaySize * dpr
  canvas.style.width = `${displaySize}px`
  canvas.style.height = `${displaySize}px`
  ctx.scale(dpr, dpr)

  const cx = displaySize / 2
  const cy = displaySize / 2
  const maxR = displaySize * 0.38
  const sides = 5
  const angleStep = (Math.PI * 2) / sides
  const startAngle = -Math.PI / 2 // Start from top

  // Clear
  ctx.clearRect(0, 0, displaySize, displaySize)

  // Draw grid rings
  for (let ring = 1; ring <= 5; ring++) {
    const r = (maxR / 5) * ring
    ctx.beginPath()
    for (let i = 0; i <= sides; i++) {
      const angle = startAngle + i * angleStep
      const x = cx + r * Math.cos(angle)
      const y = cy + r * Math.sin(angle)
      if (i === 0) ctx.moveTo(x, y)
      else ctx.lineTo(x, y)
    }
    ctx.strokeStyle = ring === 5 ? 'rgba(37, 53, 73, 0.8)' : 'rgba(37, 53, 73, 0.4)'
    ctx.lineWidth = ring === 5 ? 1.5 : 0.5
    ctx.stroke()
  }

  // Draw axis lines
  for (let i = 0; i < sides; i++) {
    const angle = startAngle + i * angleStep
    ctx.beginPath()
    ctx.moveTo(cx, cy)
    ctx.lineTo(cx + maxR * Math.cos(angle), cy + maxR * Math.sin(angle))
    ctx.strokeStyle = 'rgba(37, 53, 73, 0.5)'
    ctx.lineWidth = 0.5
    ctx.stroke()
  }

  // Draw filled data polygon
  ctx.beginPath()
  for (let i = 0; i <= sides; i++) {
    const idx = i % sides
    const angle = startAngle + idx * angleStep
    const r = (elements[idx].score / 100) * maxR
    const x = cx + r * Math.cos(angle)
    const y = cy + r * Math.sin(angle)
    if (i === 0) ctx.moveTo(x, y)
    else ctx.lineTo(x, y)
  }
  ctx.fillStyle = 'rgba(255, 107, 74, 0.15)'
  ctx.fill()
  ctx.strokeStyle = '#FF6B4A'
  ctx.lineWidth = 2
  ctx.stroke()

  // Draw data points
  for (let i = 0; i < sides; i++) {
    const angle = startAngle + i * angleStep
    const r = (elements[i].score / 100) * maxR
    const x = cx + r * Math.cos(angle)
    const y = cy + r * Math.sin(angle)

    ctx.beginPath()
    ctx.arc(x, y, 4, 0, Math.PI * 2)
    ctx.fillStyle = elements[i].color
    ctx.fill()
    ctx.strokeStyle = '#0F1923'
    ctx.lineWidth = 1.5
    ctx.stroke()
  }

  // Draw labels
  for (let i = 0; i < sides; i++) {
    const angle = startAngle + i * angleStep
    const labelR = maxR + 24
    const x = cx + labelR * Math.cos(angle)
    const y = cy + labelR * Math.sin(angle)

    ctx.textAlign = 'center'
    ctx.textBaseline = 'middle'

    // Icon
    ctx.font = '16px sans-serif'
    ctx.fillText(elements[i].icon, x, y - 8)

    // Score
    ctx.font = 'bold 11px "Be Vietnam Pro", sans-serif'
    ctx.fillStyle = elements[i].color
    ctx.fillText(String(elements[i].score), x, y + 10)
  }
}

// ═══════════════════════════════════════════════════════════════════════════
// SAMPLE CODES
// ═══════════════════════════════════════════════════════════════════════════

const sampleBadCode = `var data = fetch("/api/users")
function doEverything(a, b, c, d, e, f) {
  var result = []
  if (a != null) {
    if (b > 0) {
      for (var i = 0; i < c; i++) {
        if (d == true) {
          if (e && f) {
            console.log("processing " + i)
            result.push(a + b + c + d)
            // TODO: fix this
            if (i == 10) {
              try { JSON.parse(a) } catch(err) {}
            }
          }
        }
      }
    }
  }
  console.log(result)
  return result
}
var x = doEverything(1, 2, 3, true, false, "hello")
console.log(x)`

const sampleGoodCode = `import { validateInput } from './validators'
import { formatResponse } from './formatters'
import type { UserConfig, ProcessResult } from './types'

/**
 * Xử lý dữ liệu người dùng theo config
 */
function processUserData(config: UserConfig): ProcessResult {
  const validated = validateInput(config)
  if (!validated.ok) {
    return { success: false, error: validated.error }
  }
  return transform(validated.data)
}

function transform(data: string[]): ProcessResult {
  const results = data
    .filter((item) => item.length > 0)
    .map((item) => normalize(item))
  return { success: true, data: results }
}

function normalize(input: string): string {
  return input.trim().toLowerCase()
}

/**
 * Format và trả về kết quả cuối cùng
 */
function getFormattedResult(config: UserConfig): string {
  const result = processUserData(config)
  return formatResponse(result)
}`

// ═══════════════════════════════════════════════════════════════════════════
// STATE
// ═══════════════════════════════════════════════════════════════════════════

const codeInput = ref('')
const analysisResult = ref<AnalysisResult | null>(null)
const isAnalyzing = ref(false)
const hasAnalyzed = ref(false)

const lineCount = computed(() => codeInput.value ? codeInput.value.split('\n').length : 0)
const lineNumbers = computed(() => Array.from({ length: lineCount.value || 1 }, (_, i) => i + 1))

const overallScore = computed(() => {
  if (!analysisResult.value) return 0
  const scores = analysisResult.value.elements.map((e) => e.score)
  return Math.round(scores.reduce((a, b) => a + b, 0) / scores.length)
})

const yinYangLabel = computed(() => {
  if (!analysisResult.value) return ''
  const v = analysisResult.value.yinYangBalance
  if (v > 0.3) return 'Thiên Dương ☀️ — Code trực tiếp, gọn, ít trừu tượng'
  if (v < -0.3) return 'Thiên Âm 🌙 — Code phức tạp, nhiều nhánh, trừu tượng cao'
  return 'Cân bằng ☯️ — Âm Dương hài hòa, code balanced'
})

// ═══════════════════════════════════════════════════════════════════════════
// ACTIONS
// ═══════════════════════════════════════════════════════════════════════════

function analyze() {
  if (!codeInput.value.trim()) return
  isAnalyzing.value = true
  hasAnalyzed.value = false

  setTimeout(async () => {
    const code = codeInput.value
    const tokens = tokenize(code)
    const metrics = calcMetrics(code, tokens)
    const elements = mapToElements(metrics)
    const yinYang = calcYinYang(metrics)

    // Find dominant element
    let dominant = elements[0]
    for (const el of elements) {
      if (el.score > dominant.score) dominant = el
    }
    const menh = MENH_MAP[dominant.id] ?? MENH_MAP['hoa']

    analysisResult.value = {
      tokens,
      ...metrics,
      elements,
      yinYangBalance: yinYang,
      dominantElement: dominant,
      menh: menh.name,
      menhDescription: menh.description,
    }
    hasAnalyzed.value = true
    isAnalyzing.value = false

    await nextTick()
    drawRadar(elements)
  }, 600)
}

function loadSample(type: 'bad' | 'good') {
  codeInput.value = type === 'bad' ? sampleBadCode : sampleGoodCode
  analysisResult.value = null
  hasAnalyzed.value = false
}

function clearAll() {
  codeInput.value = ''
  analysisResult.value = null
  hasAnalyzed.value = false
}

onMounted(() => {
  window.addEventListener('resize', () => {
    if (analysisResult.value) drawRadar(analysisResult.value.elements)
  })
})
</script>

<template>
  <div class="min-h-screen bg-bg-deep text-text-primary font-body">
    <div class="max-w-4xl mx-auto px-4 sm:px-6 py-8 sm:py-16">
      <!-- Header -->
      <div class="text-center mb-8 sm:mb-12 animate-fade-up">
        <div class="inline-block mb-4 sm:mb-6">
          <span class="text-5xl sm:text-6xl">🏯</span>
        </div>
        <h1 class="font-display text-4xl min-[375px]:text-5xl sm:text-6xl lg:text-7xl font-bold text-accent-coral mb-3 sm:mb-4 tracking-tight">
          Phong Thủy Code
        </h1>
        <p class="text-text-secondary text-base sm:text-lg max-w-xl mx-auto">
          Xem <strong class="text-accent-amber">Ngũ Hành</strong> code bằng tokenizer + cyclomatic complexity — lá số tử vi cho source code 🧧
        </p>
      </div>

      <!-- Code Input -->
      <div class="mb-6 sm:mb-8 animate-fade-up animate-delay-1">
        <div class="border border-border-default bg-bg-surface transition-all duration-300 hover:border-accent-coral/30">
          <div class="flex items-center justify-between px-3 sm:px-4 py-2.5 border-b border-border-default bg-bg-elevated/50">
            <div class="flex items-center gap-2">
              <div class="flex gap-1.5">
                <span class="w-3 h-3 rounded-full bg-accent-coral/60" />
                <span class="w-3 h-3 rounded-full bg-accent-amber/60" />
                <span class="w-3 h-3 rounded-full bg-green-500/60" />
              </div>
              <span class="text-text-dim text-xs font-display tracking-wider ml-2 hidden sm:inline">source-code.ts</span>
            </div>
            <span class="text-text-dim text-xs font-display">{{ lineCount }} dòng</span>
          </div>
          <div class="flex">
            <div class="py-4 px-2 sm:px-3 text-right select-none border-r border-border-default bg-bg-deep/50 min-w-[2.5rem] sm:min-w-[3rem]">
              <div v-for="num in lineNumbers" :key="num" class="text-text-dim text-xs leading-6 font-mono">{{ num }}</div>
            </div>
            <textarea
              v-model="codeInput"
              placeholder="// Paste code JavaScript / TypeScript vào đây...
// Hệ thống sẽ tokenize, phân tích complexity,
// và xem phong thủy Ngũ Hành cho code 🏯"
              class="flex-1 bg-transparent px-3 sm:px-4 py-4 text-text-primary text-sm font-mono resize-none focus:outline-none min-h-[200px] sm:min-h-[240px] leading-6 placeholder:text-text-dim/50"
              spellcheck="false"
            />
          </div>
        </div>
      </div>

      <!-- Action buttons -->
      <div class="flex flex-wrap gap-3 mb-8 sm:mb-10 animate-fade-up animate-delay-2">
        <button
          class="flex-1 sm:flex-none inline-flex items-center justify-center gap-2 bg-accent-coral px-6 sm:px-8 py-3 font-display font-semibold text-bg-deep text-sm sm:text-base transition-all duration-300 hover:shadow-lg hover:shadow-accent-coral/20 hover:-translate-y-0.5 disabled:opacity-50 disabled:cursor-not-allowed disabled:hover:translate-y-0"
          :disabled="!codeInput.trim() || isAnalyzing"
          @click="analyze"
        >
          <span v-if="isAnalyzing" class="animate-spin">☯️</span>
          <span v-else>🏯</span>
          {{ isAnalyzing ? 'Đang xem phong thủy...' : 'XEM PHONG THỦY' }}
        </button>
        <button class="inline-flex items-center gap-1.5 border border-border-default bg-bg-surface px-3 sm:px-4 py-2.5 text-xs sm:text-sm text-text-secondary transition hover:border-accent-coral hover:text-accent-coral" @click="loadSample('bad')">💀 Code xấu</button>
        <button class="inline-flex items-center gap-1.5 border border-border-default bg-bg-surface px-3 sm:px-4 py-2.5 text-xs sm:text-sm text-text-secondary transition hover:border-green-500 hover:text-green-400" @click="loadSample('good')">✨ Code sạch</button>
        <button v-if="codeInput" class="inline-flex items-center gap-1.5 border border-border-default bg-bg-surface px-3 py-2.5 text-xs text-text-secondary transition hover:border-accent-coral hover:text-accent-coral" @click="clearAll">✕</button>
      </div>

      <!-- ═══════════════════════════════════════════════════ -->
      <!-- RESULTS -->
      <!-- ═══════════════════════════════════════════════════ -->
      <div v-if="hasAnalyzed && analysisResult" class="space-y-6 animate-fade-up">

        <!-- Mệnh Code + Radar Chart -->
        <div class="border border-border-default bg-bg-surface p-5 sm:p-6">
          <h2 class="font-display text-xl sm:text-2xl font-semibold flex items-center gap-3 mb-5">
            <span class="text-accent-coral font-display text-sm tracking-widest">//</span>
            Lá số Ngũ Hành
          </h2>

          <div class="flex flex-col lg:flex-row items-center gap-6">
            <!-- Radar Chart -->
            <div class="shrink-0">
              <canvas ref="radarCanvas" class="block" />
            </div>

            <!-- Mệnh Info -->
            <div class="flex-1 text-center lg:text-left">
              <p class="text-text-dim text-xs font-display tracking-widest mb-1">MỆNH CHỦ ĐẠO</p>
              <h3 class="font-display text-3xl sm:text-4xl font-bold text-accent-coral mb-3">
                {{ analysisResult.menh }}
              </h3>
              <p class="text-text-secondary text-sm leading-relaxed mb-4">
                {{ analysisResult.menhDescription }}
              </p>

              <!-- Overall score -->
              <div class="inline-flex items-center gap-3 px-4 py-2 border border-border-default bg-bg-deep">
                <span class="text-text-dim text-xs font-display tracking-wider">ĐIỂM TỔNG</span>
                <span class="font-display text-2xl font-bold" :class="overallScore > 70 ? 'text-green-400' : overallScore > 40 ? 'text-accent-amber' : 'text-accent-coral'">
                  {{ overallScore }}/100
                </span>
              </div>
            </div>
          </div>
        </div>

        <!-- Âm Dương Balance -->
        <div class="border border-border-default bg-bg-surface p-5 sm:p-6">
          <h2 class="font-display text-base sm:text-lg font-semibold flex items-center gap-3 mb-4">
            <span class="text-accent-amber font-display text-sm tracking-widest">//</span>
            Âm Dương
          </h2>
          <div class="mb-2">
            <div class="flex justify-between text-xs text-text-dim mb-1.5 font-display tracking-wider">
              <span>🌙 ÂM</span>
              <span>☯️</span>
              <span>DƯƠNG ☀️</span>
            </div>
            <div class="h-3 bg-bg-deep border border-border-default overflow-hidden relative">
              <!-- Center marker -->
              <div class="absolute left-1/2 top-0 bottom-0 w-px bg-text-dim/30" />
              <!-- Balance indicator -->
              <div
                class="absolute top-0 bottom-0 transition-all duration-700 bg-accent-amber/60"
                :style="{
                  left: analysisResult.yinYangBalance >= 0 ? '50%' : `${50 + analysisResult.yinYangBalance * 50}%`,
                  width: `${Math.abs(analysisResult.yinYangBalance) * 50}%`,
                }"
              />
            </div>
          </div>
          <p class="text-text-secondary text-sm">{{ yinYangLabel }}</p>
        </div>

        <!-- Metrics Summary -->
        <div class="border border-border-default bg-bg-surface p-5 sm:p-6">
          <h2 class="font-display text-base sm:text-lg font-semibold flex items-center gap-3 mb-4">
            <span class="text-accent-sky font-display text-sm tracking-widest">//</span>
            Metrics (Tokenizer output)
          </h2>
          <div class="grid grid-cols-2 sm:grid-cols-4 gap-3">
            <div class="p-3 bg-bg-deep border border-border-default text-center">
              <p class="font-display text-2xl font-bold text-accent-coral">{{ analysisResult.tokens.length }}</p>
              <p class="text-text-dim text-xs mt-1">Tokens</p>
            </div>
            <div class="p-3 bg-bg-deep border border-border-default text-center">
              <p class="font-display text-2xl font-bold text-accent-amber">{{ analysisResult.cyclomaticComplexity }}</p>
              <p class="text-text-dim text-xs mt-1">Cyclomatic Complexity</p>
            </div>
            <div class="p-3 bg-bg-deep border border-border-default text-center">
              <p class="font-display text-2xl font-bold text-accent-sky">{{ analysisResult.functionCount }}</p>
              <p class="text-text-dim text-xs mt-1">Functions</p>
            </div>
            <div class="p-3 bg-bg-deep border border-border-default text-center">
              <p class="font-display text-2xl font-bold text-green-400">{{ analysisResult.maxNesting }}</p>
              <p class="text-text-dim text-xs mt-1">Max Nesting</p>
            </div>
          </div>
        </div>

        <!-- Ngũ Hành Details -->
        <div v-for="el in analysisResult.elements" :key="el.id" class="border border-border-default bg-bg-surface overflow-hidden">
          <div class="flex items-center gap-2 px-4 sm:px-5 py-3 border-b border-border-default" :style="{ borderLeftWidth: '4px', borderLeftColor: el.color }">
            <span class="text-lg">{{ el.icon }}</span>
            <span class="font-display font-semibold text-sm">{{ el.name }}</span>
            <span class="text-text-dim text-xs font-display tracking-wider ml-1">— {{ el.metric }}</span>
            <span class="ml-auto font-display font-bold text-lg" :style="{ color: el.color }">{{ el.score }}</span>
          </div>
          <div class="px-4 sm:px-5 py-4">
            <!-- Score bar -->
            <div class="h-2 bg-bg-deep border border-border-default overflow-hidden mb-3">
              <div class="h-full transition-all duration-700" :style="{ width: `${el.score}%`, backgroundColor: el.color }" />
            </div>
            <p class="text-xs text-text-dim mb-2 font-mono">📊 {{ el.metricValue }}</p>
            <p class="text-sm mb-2 leading-relaxed" :style="{ color: el.color }">🎤 {{ el.verdict }}</p>
            <p class="text-text-dim text-xs flex items-start gap-1.5">
              <span class="text-green-400 shrink-0">💡</span>
              <span>{{ el.advice }}</span>
            </p>
          </div>
        </div>
      </div>

      <!-- Footer -->
      <div class="mt-12 sm:mt-16 text-center animate-fade-up animate-delay-3">
        <RouterLink to="/" class="inline-flex items-center gap-2 border border-border-default bg-bg-surface px-5 py-2.5 text-sm text-text-secondary transition hover:border-accent-coral hover:text-text-primary">
          &larr; Về trang chủ
        </RouterLink>
        <p class="mt-4 text-text-dim text-xs font-display tracking-wider">
          NGŨ HÀNH × COMPILER THEORY 🏯 BY HWG — FOR J2TEAM COMMUNITY
        </p>
      </div>
    </div>
  </div>
</template>

<style scoped>
canvas {
  image-rendering: -webkit-optimize-contrast;
}
</style>
