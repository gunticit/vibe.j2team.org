<script setup lang="ts">
import { ref, computed } from 'vue'
import { RouterLink } from 'vue-router'

// ─── Types ───────────────────────────────────────────────────────────────────

interface RoastRule {
  id: string
  name: string
  test: (code: string) => RoastMatch[]
  severity: 1 | 2 | 3
}

interface RoastMatch {
  line: number
  snippet: string
  roast: string
  fix: string
}

interface RoastResult {
  ruleId: string
  ruleName: string
  severity: 1 | 2 | 3
  matches: RoastMatch[]
}

// ─── Roast Templates ────────────────────────────────────────────────────────

function pickRandom<T>(arr: T[]): T {
  return arr[Math.floor(Math.random() * arr.length)]
}

const singleCharVarRoasts = [
  'Biến `{var}` — ngắn gọn như tình yêu thời đại, ai cũng thấy nhưng chẳng ai hiểu.',
  'Biến `{var}`? Bạn đang viết code hay đánh mật mã Morse?',
  'Đặt tên biến `{var}` xong 3 ngày sau chính bạn cũng không nhớ nó là gì.',
  'Senior dev nhìn biến `{var}` rồi âm thầm cập nhật CV.',
  'Biến `{var}` — tiết kiệm keystroke nhưng tốn 3 tiếng debug.',
]

const consoleLogRoasts = [
  'console.log ở dòng {line}? Debug bằng print statement à? Năm 2024 rồi bạn ơi.',
  'console.log — vũ khí tối thượng của dev khi debugger quá phức tạp.',
  'Thấy console.log rải khắp nơi như confetti. Đẹp thì đẹp nhưng production sẽ khóc.',
  'console.log — "temporary" mà ở đây 6 tháng rồi, như cái bàn tạm ở phòng khách.',
]

const magicNumberRoasts = [
  'Số {num} ở dòng {line} — magic number! Bạn là phù thủy hay lập trình viên?',
  'Số {num} xuất hiện không giải thích. Bí ẩn hơn tam giác quỷ Bermuda.',
  'Thấy số {num} — chắc có ý nghĩa sâu xa lắm nhưng không ai biết, kể cả bạn.',
  'Magic number {num} — "trust me, it works" nhưng không ai dám sửa.',
]

const deepNestRoasts = [
  'Nested {level} cấp ở dòng {line}! Code hay mê cung Minotaur vậy?',
  'Indent sâu quá, scroll phải cũng không thấy code. Ultra-wide monitor cũng bó tay.',
  '{level} cấp nested — thang máy cũng không xuống sâu vậy.',
  'Code nested kiểu này đọc xong cần GPS để tìm đường ra.',
]

const longFuncRoasts = [
  'Function {lines} dòng? Đây là function hay tiểu thuyết?',
  'Function dài {lines} dòng — người đọc sẽ cần nghỉ giải lao giữa chừng.',
  'Một function {lines} dòng = một function làm tất cả mọi thứ. Single Responsibility nghe quen không?',
]

const varKeywordRoasts = [
  '`var` ở dòng {line}? Welcome back to 2015! `let` và `const` gửi lời hỏi thăm.',
  '`var` — hoisting party! Biến nào cũng bay lên đầu function, vui như Tết.',
  'Dùng `var` năm 2024 giống đi xe đạp trên đường cao tốc — technically works nhưng...',
]

const looseEqualityRoasts = [
  '`==` ở dòng {line}? Bạn thích sống mạo hiểm nhỉ! `0 == ""` là true đấy.',
  '`==` — "gần đúng cũng được" — triết lý sống hay triết lý code?',
  'Loose equality `==` — type coercion sẽ biến code thành show ảo thuật.',
]

const anyTypeRoasts = [
  '`any` ở dòng {line}? Viết TypeScript để rồi dùng any thì viết JavaScript cho nhẹ bạn ơi.',
  'Loại `any` — TypeScript nhìn thấy mà khóc thầm. Sao phải khổ như vậy?',
  '`any` — "tôi không biết type gì nhưng trust me bro it works".',
]

const todoRoasts = [
  'TODO ở dòng {line} — "I\'ll do it later" said every developer ever. Spoiler: they didn\'t.',
  'TODO comment — di sản văn hóa để lại cho dev sau. Archaeological artifact.',
  'Thấy TODO ở dòng {line}. Nó ở đây bao lâu rồi? Cần carbon dating không?',
]

const importantCssRoasts = [
  '`!important` — khi CSS specificity quá phức tạp, ta dùng búa tạ.',
  '`!important` — "tôi không hiểu CSS cascade nhưng tôi biết cách dùng sức mạnh".',
  'Dùng `!important` giống hét to hơn trong cuộc cãi nhau — thắng tạm thời nhưng vấn đề vẫn còn.',
]

const emptyCatchRoasts = [
  'Catch trống ở dòng {line}? Error xảy ra rồi bạn... nuốt luôn? Yum yum.',
  'Empty catch block — "error gì cũng được, tôi không quan tâm" — famous last words.',
  'Catch trống — silent failure đẹp nhất vũ trụ. Bug sẽ ẩn nấp như ninja.',
]

const stringConcatRoasts = [
  'String concatenation bằng `+` ở dòng {line}? Template literals đã ra đời từ ES6 rồi bạn.',
  'Nối string bằng `+` — nostalgic nhưng hơi cổ. Thử dùng backtick `` ` `` đi!',
]

const tooManyParamsRoasts = [
  'Function có {count} parameters ở dòng {line}? Đây là function hay form đăng ký?',
  '{count} params — chắc cần truyền thêm vài cái nữa là đủ bộ sưu tập.',
  'Function {count} tham số — nên gom vào object config cho gọn bạn ơi.',
]

const noCommentRoasts = [
  'Code {lines} dòng không một dòng comment. Tự tin hoặc... bất cần đời.',
  '{lines} dòng code, 0 dòng comment — "code tự giải thích" họ nói. Narrator: it didn\'t.',
]

// ─── Rule Engine ─────────────────────────────────────────────────────────────

function getLines(code: string): string[] {
  return code.split('\n')
}

const rules: RoastRule[] = [
  {
    id: 'single-char-var',
    name: 'Biến một chữ cái',
    severity: 2,
    test(code: string): RoastMatch[] {
      const matches: RoastMatch[] = []
      const lines = getLines(code)
      // Match variable declarations with single-char names, excluding loop vars i,j,k,_
      const regex = /\b(?:let|const|var)\s+([a-hlo-zA-HLO-Z])\b/g
      for (let i = 0; i < lines.length; i++) {
        let match: RegExpExecArray | null = null
        regex.lastIndex = 0
        while ((match = regex.exec(lines[i])) !== null) {
          const varName = match[1]
          matches.push({
            line: i + 1,
            snippet: lines[i].trim(),
            roast: pickRandom(singleCharVarRoasts).replace('{var}', varName),
            fix: `Đặt tên biến có ý nghĩa, ví dụ: \`${varName}\` → \`${varName === 'x' ? 'positionX' : varName === 'n' ? 'count' : 'meaningfulName'}\``,
          })
        }
      }
      return matches
    },
  },
  {
    id: 'console-log',
    name: 'Console.log debugging',
    severity: 1,
    test(code: string): RoastMatch[] {
      const matches: RoastMatch[] = []
      const lines = getLines(code)
      for (let i = 0; i < lines.length; i++) {
        if (/console\.(log|warn|error|info|debug)\s*\(/.test(lines[i]) && !/\/\//.test(lines[i].split('console')[0])) {
          matches.push({
            line: i + 1,
            snippet: lines[i].trim(),
            roast: pickRandom(consoleLogRoasts).replace('{line}', String(i + 1)),
            fix: 'Dùng debugger hoặc logging library thay vì console.log. Nếu cần log, nhớ xóa trước khi commit.',
          })
        }
      }
      return matches
    },
  },
  {
    id: 'magic-number',
    name: 'Magic Numbers',
    severity: 2,
    test(code: string): RoastMatch[] {
      const matches: RoastMatch[] = []
      const lines = getLines(code)
      const ignore = new Set([0, 1, 2, -1, 100, 1000])
      for (let i = 0; i < lines.length; i++) {
        const line = lines[i]
        // Skip comments, imports, array indices
        if (/^\s*(\/\/|\/\*|\*|import|export)/.test(line)) continue
        // Skip lines that are just variable declarations with obvious numbers
        const numMatches = line.matchAll(/(?<![.\w])(\d{2,}(?:\.\d+)?)(?![.\w])/g)
        for (const m of numMatches) {
          const num = parseFloat(m[1])
          if (!ignore.has(num) && num > 1 && !/(?:length|size|index|port|width|height|timeout|delay|duration|margin|padding|fontSize|opacity)\s*[:=]/.test(line)) {
            matches.push({
              line: i + 1,
              snippet: line.trim(),
              roast: pickRandom(magicNumberRoasts).replace('{num}', m[1]).replace('{line}', String(i + 1)),
              fix: `Tạo constant có tên rõ ràng: \`const MEANINGFUL_NAME = ${m[1]}\``,
            })
            break // One per line is enough
          }
        }
      }
      return matches
    },
  },
  {
    id: 'deep-nesting',
    name: 'Nested quá sâu',
    severity: 3,
    test(code: string): RoastMatch[] {
      const matches: RoastMatch[] = []
      const lines = getLines(code)
      let maxLevel = 0
      let maxLine = 0
      let braceCount = 0
      for (let i = 0; i < lines.length; i++) {
        for (const ch of lines[i]) {
          if (ch === '{') braceCount++
          if (ch === '}') braceCount--
        }
        if (braceCount > maxLevel) {
          maxLevel = braceCount
          maxLine = i + 1
        }
      }
      if (maxLevel > 3) {
        matches.push({
          line: maxLine,
          snippet: lines[maxLine - 1]?.trim() ?? '',
          roast: pickRandom(deepNestRoasts).replace('{level}', String(maxLevel)).replace('{line}', String(maxLine)),
          fix: 'Tách logic ra thành function riêng, sử dụng early return, hoặc dùng guard clauses.',
        })
      }
      return matches
    },
  },
  {
    id: 'long-function',
    name: 'Function quá dài',
    severity: 2,
    test(code: string): RoastMatch[] {
      const matches: RoastMatch[] = []
      const lines = getLines(code)
      const funcRegex = /(?:function\s+\w+|(?:const|let|var)\s+\w+\s*=\s*(?:async\s*)?\(?|(?:async\s+)?(?:\w+\s*)?\()/
      let funcStart = -1
      let braceDepth = 0
      let inFunc = false

      for (let i = 0; i < lines.length; i++) {
        if (!inFunc && funcRegex.test(lines[i]) && lines[i].includes('{')) {
          funcStart = i
          inFunc = true
          braceDepth = 0
        }
        if (inFunc) {
          for (const ch of lines[i]) {
            if (ch === '{') braceDepth++
            if (ch === '}') braceDepth--
          }
          if (braceDepth <= 0 && funcStart >= 0) {
            const funcLen = i - funcStart + 1
            if (funcLen > 30) {
              matches.push({
                line: funcStart + 1,
                snippet: lines[funcStart].trim(),
                roast: pickRandom(longFuncRoasts).replace('{lines}', String(funcLen)),
                fix: 'Tách function lớn thành các function nhỏ hơn, mỗi function chỉ làm một việc (Single Responsibility Principle).',
              })
            }
            inFunc = false
            funcStart = -1
          }
        }
      }
      return matches
    },
  },
  {
    id: 'var-keyword',
    name: 'Dùng var',
    severity: 2,
    test(code: string): RoastMatch[] {
      const matches: RoastMatch[] = []
      const lines = getLines(code)
      for (let i = 0; i < lines.length; i++) {
        if (/\bvar\s+\w+/.test(lines[i]) && !/\/\//.test(lines[i].split('var')[0])) {
          matches.push({
            line: i + 1,
            snippet: lines[i].trim(),
            roast: pickRandom(varKeywordRoasts).replace('{line}', String(i + 1)),
            fix: 'Thay `var` bằng `const` (nếu không reassign) hoặc `let` (nếu cần reassign).',
          })
        }
      }
      return matches
    },
  },
  {
    id: 'loose-equality',
    name: 'Loose equality (==)',
    severity: 2,
    test(code: string): RoastMatch[] {
      const matches: RoastMatch[] = []
      const lines = getLines(code)
      for (let i = 0; i < lines.length; i++) {
        // Match == but not === and not !== and not inside strings
        if (/[^!=<>]={2}[^=]/.test(lines[i]) || /^==[^=]/.test(lines[i])) {
          matches.push({
            line: i + 1,
            snippet: lines[i].trim(),
            roast: pickRandom(looseEqualityRoasts).replace('{line}', String(i + 1)),
            fix: 'Dùng `===` (strict equality) thay vì `==` để tránh type coercion bất ngờ.',
          })
        }
      }
      return matches
    },
  },
  {
    id: 'any-type',
    name: 'TypeScript any',
    severity: 3,
    test(code: string): RoastMatch[] {
      const matches: RoastMatch[] = []
      const lines = getLines(code)
      for (let i = 0; i < lines.length; i++) {
        if (/:\s*any\b/.test(lines[i]) || /as\s+any\b/.test(lines[i]) || /<any>/.test(lines[i])) {
          matches.push({
            line: i + 1,
            snippet: lines[i].trim(),
            roast: pickRandom(anyTypeRoasts).replace('{line}', String(i + 1)),
            fix: 'Định nghĩa interface/type cụ thể thay vì dùng `any`. TypeScript chỉ hữu ích khi bạn dùng đúng type.',
          })
        }
      }
      return matches
    },
  },
  {
    id: 'todo-comment',
    name: 'TODO bỏ quên',
    severity: 1,
    test(code: string): RoastMatch[] {
      const matches: RoastMatch[] = []
      const lines = getLines(code)
      for (let i = 0; i < lines.length; i++) {
        if (/\/\/\s*TODO/i.test(lines[i]) || /\/\/\s*FIXME/i.test(lines[i]) || /\/\/\s*HACK/i.test(lines[i])) {
          matches.push({
            line: i + 1,
            snippet: lines[i].trim(),
            roast: pickRandom(todoRoasts).replace('{line}', String(i + 1)),
            fix: 'Hoặc làm ngay TODO đó, hoặc tạo issue/ticket để track. Đừng để TODO thành di sản.',
          })
        }
      }
      return matches
    },
  },
  {
    id: 'important-css',
    name: 'CSS !important',
    severity: 2,
    test(code: string): RoastMatch[] {
      const matches: RoastMatch[] = []
      const lines = getLines(code)
      for (let i = 0; i < lines.length; i++) {
        if (/!important/.test(lines[i])) {
          matches.push({
            line: i + 1,
            snippet: lines[i].trim(),
            roast: pickRandom(importantCssRoasts),
            fix: 'Tìm hiểu CSS specificity và sửa selector thay vì dùng `!important`. Override bằng selector cụ thể hơn.',
          })
        }
      }
      return matches
    },
  },
  {
    id: 'empty-catch',
    name: 'Catch trống',
    severity: 3,
    test(code: string): RoastMatch[] {
      const matches: RoastMatch[] = []
      const lines = getLines(code)
      for (let i = 0; i < lines.length; i++) {
        // Look for catch followed by empty block
        if (/catch\s*\([^)]*\)\s*\{\s*\}/.test(lines[i])) {
          matches.push({
            line: i + 1,
            snippet: lines[i].trim(),
            roast: pickRandom(emptyCatchRoasts).replace('{line}', String(i + 1)),
            fix: 'Luôn xử lý error trong catch: log, throw lại, hoặc hiển thị cho user. Đừng nuốt error.',
          })
        }
        // Multi-line empty catch
        if (/catch\s*\([^)]*\)\s*\{\s*$/.test(lines[i]) && i + 1 < lines.length && /^\s*\}/.test(lines[i + 1])) {
          matches.push({
            line: i + 1,
            snippet: lines[i].trim(),
            roast: pickRandom(emptyCatchRoasts).replace('{line}', String(i + 1)),
            fix: 'Luôn xử lý error trong catch: log, throw lại, hoặc hiển thị cho user. Đừng nuốt error.',
          })
        }
      }
      return matches
    },
  },
  {
    id: 'string-concat',
    name: 'String concatenation',
    severity: 1,
    test(code: string): RoastMatch[] {
      const matches: RoastMatch[] = []
      const lines = getLines(code)
      for (let i = 0; i < lines.length; i++) {
        const line = lines[i]
        // Detect patterns like "string" + variable or variable + "string"
        if (/["'][^"']*["']\s*\+\s*\w/.test(line) || /\w\s*\+\s*["'][^"']*["']/.test(line)) {
          // Skip import/require lines
          if (/import|require/.test(line)) continue
          matches.push({
            line: i + 1,
            snippet: line.trim(),
            roast: pickRandom(stringConcatRoasts).replace('{line}', String(i + 1)),
            fix: 'Dùng template literals: `Hello ${name}` thay vì "Hello " + name.',
          })
        }
      }
      return matches
    },
  },
  {
    id: 'too-many-params',
    name: 'Quá nhiều tham số',
    severity: 2,
    test(code: string): RoastMatch[] {
      const matches: RoastMatch[] = []
      const lines = getLines(code)
      const funcDeclRegex = /(?:function\s+\w+|(?:const|let|var)\s+\w+\s*=\s*(?:async\s*)?)\s*\(([^)]{20,})\)/
      for (let i = 0; i < lines.length; i++) {
        const match = funcDeclRegex.exec(lines[i])
        if (match) {
          const params = match[1].split(',').filter((p) => p.trim().length > 0)
          if (params.length >= 4) {
            matches.push({
              line: i + 1,
              snippet: lines[i].trim(),
              roast: pickRandom(tooManyParamsRoasts).replace('{count}', String(params.length)).replace('{line}', String(i + 1)),
              fix: 'Gom các tham số liên quan vào một object config: `function doThing({ name, age, email }: UserConfig)`.',
            })
          }
        }
      }
      return matches
    },
  },
  {
    id: 'no-comments',
    name: 'Không có comment',
    severity: 1,
    test(code: string): RoastMatch[] {
      const lines = getLines(code)
      if (lines.length < 20) return [] // Too short to complain about
      const commentLines = lines.filter((l) => /^\s*(\/\/|\/\*|\*)/.test(l)).length
      if (commentLines === 0) {
        return [
          {
            line: 1,
            snippet: lines[0].trim(),
            roast: pickRandom(noCommentRoasts).replace('{lines}', String(lines.length)),
            fix: 'Thêm comment giải thích WHY (tại sao) chứ không phải WHAT (cái gì). Comment ở đầu function/class là minimum.',
          },
        ]
      }
      return []
    },
  },
]

// ─── Sample Codes ────────────────────────────────────────────────────────────

const sampleBadCode = `var x = 10;
var y = 20;
var a = "hello";

function processData(name, age, email, phone, address, city) {
  var result = "";
  if (name != null) {
    if (age > 0) {
      if (email != "") {
        if (phone != null) {
          console.log("processing...");
          result = "Name: " + name + ", Age: " + age;
          // TODO: fix this later
          var total = 86400;
          if (total == "86400") {
            try {
              JSON.parse(name);
            } catch (e) {}
            console.log("done: " + result);
          }
        }
      }
    }
  }
  return result;
}`

const sampleGoodCode = `const MAX_SESSION_SECONDS = 86_400

interface UserProfile {
  name: string
  age: number
  email: string
}

function formatUserSummary(user: UserProfile): string {
  const { name, age, email } = user
  return \`\${name} (\${age}) — \${email}\`
}

function isValidSession(startTime: number): boolean {
  const elapsed = Date.now() - startTime
  return elapsed < MAX_SESSION_SECONDS * 1000
}`

// ─── State ───────────────────────────────────────────────────────────────────

const codeInput = ref('')
const results = ref<RoastResult[]>([])
const hasAnalyzed = ref(false)
const isAnalyzing = ref(false)
const showFireworks = ref(false)

// ─── Computed ────────────────────────────────────────────────────────────────

const totalIssues = computed(() => results.value.reduce((sum, r) => sum + r.matches.length, 0))

const burnLevel = computed(() => {
  const issues = totalIssues.value
  if (issues === 0) return 0
  if (issues <= 2) return 1
  if (issues <= 5) return 2
  if (issues <= 8) return 3
  if (issues <= 12) return 4
  return 5
})

const burnEmoji = computed(() => '🔥'.repeat(burnLevel.value) || '✨')

const verdict = computed(() => {
  const level = burnLevel.value
  if (level === 0) return { text: 'Code sạch đẹp! Không có gì để roast 🎉', color: 'text-green-400' }
  if (level === 1) return { text: 'Hơi ấm thôi — vài chỗ cần sửa nhẹ', color: 'text-accent-amber' }
  if (level === 2) return { text: 'Nóng rồi đấy — nên refactor sớm', color: 'text-orange-400' }
  if (level === 3) return { text: 'Cháy to! Code này cần được cứu hộ 🚒', color: 'text-accent-coral' }
  if (level === 4) return { text: 'CHÁY LỚN! Gọi 114 ngay! 🧯', color: 'text-red-500' }
  return { text: '☠️ THẢM HOẠ! Code này nên được xóa và viết lại từ đầu', color: 'text-red-600' }
})

const lineCount = computed(() => {
  if (!codeInput.value) return 0
  return codeInput.value.split('\n').length
})

const lineNumbers = computed(() => {
  return Array.from({ length: lineCount.value || 1 }, (_, i) => i + 1)
})

// ─── Actions ─────────────────────────────────────────────────────────────────

function analyzeCode() {
  if (!codeInput.value.trim()) return
  isAnalyzing.value = true
  results.value = []
  hasAnalyzed.value = false

  // Simulate processing time for dramatic effect
  setTimeout(() => {
    const allResults: RoastResult[] = []

    for (const rule of rules) {
      const matches = rule.test(codeInput.value)
      if (matches.length > 0) {
        allResults.push({
          ruleId: rule.id,
          ruleName: rule.name,
          severity: rule.severity,
          matches,
        })
      }
    }

    // Sort by severity (highest first)
    allResults.sort((a, b) => b.severity - a.severity)

    results.value = allResults
    hasAnalyzed.value = true
    isAnalyzing.value = false

    if (allResults.length === 0) {
      showFireworks.value = true
      setTimeout(() => {
        showFireworks.value = false
      }, 3000)
    }
  }, 800)
}

function loadSample(type: 'bad' | 'good') {
  codeInput.value = type === 'bad' ? sampleBadCode : sampleGoodCode
  results.value = []
  hasAnalyzed.value = false
}

function clearCode() {
  codeInput.value = ''
  results.value = []
  hasAnalyzed.value = false
}

function severityIcon(severity: number): string {
  if (severity === 3) return '🔥🔥🔥'
  if (severity === 2) return '🔥🔥'
  return '🔥'
}

function severityLabel(severity: number): string {
  if (severity === 3) return 'Nghiêm trọng'
  if (severity === 2) return 'Đáng chú ý'
  return 'Nhẹ'
}
</script>

<template>
  <div class="min-h-screen bg-bg-deep text-text-primary font-body relative overflow-hidden">
    <!-- Fire particles background when burning -->
    <div
      v-if="hasAnalyzed && burnLevel >= 3"
      class="fixed inset-0 pointer-events-none z-10"
    >
      <div
        v-for="n in 20"
        :key="n"
        class="fire-particle"
        :style="{
          left: `${Math.random() * 100}%`,
          animationDelay: `${Math.random() * 3}s`,
          animationDuration: `${2 + Math.random() * 3}s`,
        }"
      />
    </div>

    <!-- Sparkles when code is clean -->
    <div
      v-if="showFireworks"
      class="fixed inset-0 pointer-events-none z-10"
    >
      <div
        v-for="n in 30"
        :key="n"
        class="sparkle-particle"
        :style="{
          left: `${Math.random() * 100}%`,
          top: `${Math.random() * 100}%`,
          animationDelay: `${Math.random() * 2}s`,
        }"
      />
    </div>

    <div class="relative z-20 max-w-4xl mx-auto px-4 sm:px-6 py-8 sm:py-16">
      <!-- Header -->
      <div class="text-center mb-8 sm:mb-12 animate-fade-up">
        <div class="inline-block mb-4 sm:mb-6">
          <span class="text-5xl sm:text-6xl">🔥</span>
        </div>
        <h1 class="font-display text-4xl min-[375px]:text-5xl sm:text-6xl lg:text-7xl font-bold text-accent-coral mb-3 sm:mb-4 tracking-tight">
          Code Roast
        </h1>
        <p class="text-text-secondary text-base sm:text-lg max-w-lg mx-auto">
          Paste code của bạn vào — để senior dev 10 năm kinh nghiệm "review" theo phong cách roast comedy 🎤
        </p>
      </div>

      <!-- Code Input Area -->
      <div class="mb-6 sm:mb-8 animate-fade-up animate-delay-1">
        <div class="border border-border-default bg-bg-surface transition-all duration-300 hover:border-accent-coral/30">
          <!-- Editor toolbar -->
          <div class="flex items-center justify-between px-3 sm:px-4 py-2.5 border-b border-border-default bg-bg-elevated/50">
            <div class="flex items-center gap-2">
              <div class="flex gap-1.5">
                <span class="w-3 h-3 rounded-full bg-accent-coral/60" />
                <span class="w-3 h-3 rounded-full bg-accent-amber/60" />
                <span class="w-3 h-3 rounded-full bg-green-500/60" />
              </div>
              <span class="text-text-dim text-xs font-display tracking-wider ml-2 hidden sm:inline">
                your-code.js
              </span>
            </div>
            <div class="flex items-center gap-2 text-xs">
              <span class="text-text-dim font-display">
                {{ lineCount }} dòng
              </span>
            </div>
          </div>

          <!-- Code editor with line numbers -->
          <div class="flex">
            <!-- Line numbers -->
            <div class="py-4 px-2 sm:px-3 text-right select-none border-r border-border-default bg-bg-deep/50 min-w-[2.5rem] sm:min-w-[3rem]">
              <div
                v-for="num in lineNumbers"
                :key="num"
                class="text-text-dim text-xs leading-6 font-mono"
              >
                {{ num }}
              </div>
            </div>

            <!-- Textarea -->
            <textarea
              v-model="codeInput"
              placeholder="// Paste code JavaScript/TypeScript/CSS của bạn vào đây...
// Ví dụ:
var x = 10;
console.log(x);"
              class="flex-1 bg-transparent px-3 sm:px-4 py-4 text-text-primary text-sm font-mono resize-none focus:outline-none min-h-[200px] sm:min-h-[280px] leading-6 placeholder:text-text-dim/50"
              spellcheck="false"
            />
          </div>
        </div>
      </div>

      <!-- Action buttons -->
      <div class="flex flex-wrap gap-3 mb-8 sm:mb-10 animate-fade-up animate-delay-2">
        <button
          class="roast-button group relative flex-1 sm:flex-none inline-flex items-center justify-center gap-2 bg-accent-coral px-6 sm:px-8 py-3 font-display font-semibold text-bg-deep text-sm sm:text-base transition-all duration-300 hover:shadow-lg hover:shadow-accent-coral/20 hover:-translate-y-0.5 disabled:opacity-50 disabled:cursor-not-allowed disabled:hover:translate-y-0 disabled:hover:shadow-none"
          :disabled="!codeInput.trim() || isAnalyzing"
          @click="analyzeCode"
        >
          <span
            v-if="isAnalyzing"
            class="animate-spin"
          >⚙️</span>
          <span v-else>🔥</span>
          {{ isAnalyzing ? 'Đang phân tích...' : 'ROAST MY CODE' }}
        </button>

        <div class="flex gap-2">
          <button
            class="inline-flex items-center gap-1.5 border border-border-default bg-bg-surface px-3 sm:px-4 py-2.5 text-xs sm:text-sm text-text-secondary transition hover:border-accent-amber hover:text-accent-amber"
            @click="loadSample('bad')"
          >
            💀 Code xấu mẫu
          </button>
          <button
            class="inline-flex items-center gap-1.5 border border-border-default bg-bg-surface px-3 sm:px-4 py-2.5 text-xs sm:text-sm text-text-secondary transition hover:border-green-500 hover:text-green-400"
            @click="loadSample('good')"
          >
            ✨ Code sạch mẫu
          </button>
          <button
            v-if="codeInput"
            class="inline-flex items-center gap-1.5 border border-border-default bg-bg-surface px-3 sm:px-4 py-2.5 text-xs sm:text-sm text-text-secondary transition hover:border-accent-coral hover:text-accent-coral"
            @click="clearCode"
          >
            ✕
          </button>
        </div>
      </div>

      <!-- Results -->
      <div
        v-if="hasAnalyzed"
        class="space-y-6 animate-fade-up"
      >
        <!-- Burn Level Meter -->
        <div class="border border-border-default bg-bg-surface p-5 sm:p-6">
          <div class="flex items-center justify-between mb-4">
            <h2 class="font-display text-xl sm:text-2xl font-semibold flex items-center gap-3">
              <span class="text-accent-coral font-display text-sm tracking-widest">//</span>
              Kết quả Roast
            </h2>
            <span class="text-2xl sm:text-3xl">{{ burnEmoji }}</span>
          </div>

          <!-- Burn level bar -->
          <div class="mb-4">
            <div class="flex justify-between text-xs text-text-dim mb-2 font-display tracking-wider">
              <span>MÁT MẺ</span>
              <span>CHÁY LỚN</span>
            </div>
            <div class="h-3 bg-bg-deep border border-border-default overflow-hidden">
              <div
                class="h-full transition-all duration-1000 ease-out burn-bar"
                :style="{ width: `${burnLevel * 20}%` }"
                :class="{
                  'bg-green-500': burnLevel === 0,
                  'bg-accent-amber': burnLevel === 1,
                  'bg-orange-400': burnLevel === 2,
                  'bg-accent-coral': burnLevel === 3,
                  'bg-red-500': burnLevel === 4,
                  'bg-red-600 animate-pulse': burnLevel === 5,
                }"
              />
            </div>
          </div>

          <!-- Verdict -->
          <div class="flex items-center gap-3 p-3 sm:p-4 bg-bg-deep border border-border-default">
            <span class="text-xl sm:text-2xl">
              {{ burnLevel === 0 ? '😎' : burnLevel <= 2 ? '😬' : burnLevel <= 4 ? '🤯' : '💀' }}
            </span>
            <div>
              <p
                class="font-display font-semibold text-sm sm:text-base"
                :class="verdict.color"
              >
                {{ verdict.text }}
              </p>
              <p class="text-text-dim text-xs mt-0.5">
                {{ totalIssues }} vấn đề phát hiện từ {{ results.length }} rules
              </p>
            </div>
          </div>
        </div>

        <!-- Issue List -->
        <div
          v-for="result in results"
          :key="result.ruleId"
          class="border border-border-default bg-bg-surface overflow-hidden"
        >
          <!-- Rule header -->
          <div class="flex items-center gap-2 px-4 sm:px-5 py-3 bg-bg-elevated/50 border-b border-border-default">
            <span class="text-sm">{{ severityIcon(result.severity) }}</span>
            <span class="font-display font-semibold text-sm text-text-primary">{{ result.ruleName }}</span>
            <span
              class="ml-auto text-xs font-display tracking-wider px-2 py-0.5 border"
              :class="{
                'text-red-400 border-red-400/30 bg-red-400/5': result.severity === 3,
                'text-accent-amber border-accent-amber/30 bg-accent-amber/5': result.severity === 2,
                'text-accent-sky border-accent-sky/30 bg-accent-sky/5': result.severity === 1,
              }"
            >
              {{ severityLabel(result.severity) }}
            </span>
          </div>

          <!-- Matches -->
          <div class="divide-y divide-border-default">
            <div
              v-for="(match, idx) in result.matches"
              :key="idx"
              class="px-4 sm:px-5 py-4"
            >
              <!-- Code snippet -->
              <div class="mb-3 px-3 py-2 bg-bg-deep border border-border-default font-mono text-xs overflow-x-auto">
                <span class="text-text-dim mr-2 select-none">{{ match.line }}│</span>
                <span class="text-text-secondary">{{ match.snippet }}</span>
              </div>

              <!-- Roast message -->
              <p class="text-accent-coral text-sm mb-2 leading-relaxed">
                🎤 {{ match.roast }}
              </p>

              <!-- Fix suggestion -->
              <p class="text-text-dim text-xs flex items-start gap-1.5">
                <span class="text-green-400 mt-0.5 shrink-0">💡</span>
                <span>{{ match.fix }}</span>
              </p>
            </div>
          </div>
        </div>

        <!-- Clean code celebration -->
        <div
          v-if="totalIssues === 0"
          class="border border-green-500/30 bg-green-500/5 p-6 sm:p-8 text-center"
        >
          <div class="text-5xl sm:text-6xl mb-4">🎉</div>
          <h3 class="font-display text-xl sm:text-2xl font-bold text-green-400 mb-2">
            Code Sạch Đẹp!
          </h3>
          <p class="text-text-secondary text-sm max-w-md mx-auto">
            Không tìm thấy anti-pattern nào. Bạn viết code như một senior dev thực thụ!
            Hay code ngắn quá chưa đủ để roast 😏
          </p>
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
          ROAST WITH ❤️ BY HWG — FOR J2TEAM COMMUNITY
        </p>
      </div>
    </div>
  </div>
</template>

<style scoped>
/* Fire particles */
.fire-particle {
  position: absolute;
  bottom: -10px;
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: radial-gradient(circle, #FF6B4A, #FFB830, transparent);
  animation: rise linear infinite;
  opacity: 0;
}

@keyframes rise {
  0% {
    transform: translateY(0) scale(1);
    opacity: 0;
  }
  10% {
    opacity: 0.8;
  }
  50% {
    opacity: 0.4;
  }
  100% {
    transform: translateY(-100vh) scale(0);
    opacity: 0;
  }
}

/* Sparkle particles */
.sparkle-particle {
  position: absolute;
  width: 6px;
  height: 6px;
  background: #FFB830;
  border-radius: 50%;
  animation: sparkle 2s ease-in-out infinite;
  box-shadow: 0 0 10px #FFB830, 0 0 20px #FF6B4A;
}

@keyframes sparkle {
  0%, 100% {
    transform: scale(0);
    opacity: 0;
  }
  50% {
    transform: scale(1);
    opacity: 1;
  }
}

/* Burn bar glow */
.burn-bar {
  box-shadow: 0 0 10px currentColor;
}

/* CTA button glow */
.roast-button:not(:disabled):hover {
  box-shadow: 0 0 20px rgba(255, 107, 74, 0.3), 0 4px 15px rgba(255, 107, 74, 0.2);
}

/* Textarea custom scrollbar */
textarea::-webkit-scrollbar {
  width: 6px;
}

textarea::-webkit-scrollbar-track {
  background: transparent;
}

textarea::-webkit-scrollbar-thumb {
  background: #253549;
  border-radius: 3px;
}

textarea::-webkit-scrollbar-thumb:hover {
  background: #4A6180;
}
</style>
