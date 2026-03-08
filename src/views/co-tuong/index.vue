<script setup lang="ts">
import { ref, computed, watch, onMounted, onUnmounted, nextTick } from 'vue'
import { RouterLink } from 'vue-router'

// ═══════════════════════════════════════════════════════════════════════════
// TYPES
// ═══════════════════════════════════════════════════════════════════════════

type PieceType = 'K' | 'A' | 'E' | 'H' | 'R' | 'C' | 'P'
type PieceColor = 'red' | 'black'
type GameMode = 'menu' | 'local' | 'online-setup' | 'online-play' | 'spectator'
type Turn = 'red' | 'black'
type ConnectionState = 'idle' | 'creating-offer' | 'waiting-answer' | 'joining' | 'connected'

interface Piece { type: PieceType; color: PieceColor }
type Board = (Piece | null)[][]
interface Move { from: [number, number]; to: [number, number]; piece: Piece; captured: Piece | null }

const PIECE_CHAR: Record<PieceType, { red: string; black: string }> = {
  K: { red: '帥', black: '將' }, A: { red: '仕', black: '士' },
  E: { red: '相', black: '象' }, H: { red: '馬', black: '馬' },
  R: { red: '車', black: '車' }, C: { red: '炮', black: '砲' },
  P: { red: '兵', black: '卒' },
}
const PIECE_VN: Record<PieceType, string> = { K: 'Tướng', A: 'Sĩ', E: 'Tượng', H: 'Mã', R: 'Xe', C: 'Pháo', P: 'Tốt' }
const COL_NAMES = ['1', '2', '3', '4', '5', '6', '7', '8', '9']

// ═══════════════════════════════════════════════════════════════════════════
// GAME ENGINE
// ═══════════════════════════════════════════════════════════════════════════

function P(type: PieceType, color: PieceColor): Piece { return { type, color } }

function initBoard(): Board {
  const B: PieceColor = 'black', R: PieceColor = 'red'
  return [
    [P('R',B), P('H',B), P('E',B), P('A',B), P('K',B), P('A',B), P('E',B), P('H',B), P('R',B)],
    Array(9).fill(null),
    [null, P('C',B), null, null, null, null, null, P('C',B), null],
    [P('P',B), null, P('P',B), null, P('P',B), null, P('P',B), null, P('P',B)],
    Array(9).fill(null),
    Array(9).fill(null),
    [P('P',R), null, P('P',R), null, P('P',R), null, P('P',R), null, P('P',R)],
    [null, P('C',R), null, null, null, null, null, P('C',R), null],
    Array(9).fill(null),
    [P('R',R), P('H',R), P('E',R), P('A',R), P('K',R), P('A',R), P('E',R), P('H',R), P('R',R)],
  ]
}

function cloneBoard(b: Board): Board { return b.map(row => row.map(p => p ? { ...p } : null)) }

function inBounds(r: number, c: number): boolean { return r >= 0 && r <= 9 && c >= 0 && c <= 8 }

function inPalace(r: number, c: number, color: PieceColor): boolean {
  return c >= 3 && c <= 5 && (color === 'black' ? r >= 0 && r <= 2 : r >= 7 && r <= 9)
}

function isOwnSide(r: number, color: PieceColor): boolean {
  return color === 'black' ? r >= 0 && r <= 4 : r >= 5 && r <= 9
}

function getRawMoves(board: Board, r: number, c: number): [number, number][] {
  const piece = board[r]![c]
  if (!piece) return []
  const moves: [number, number][] = []
  const { type, color } = piece
  const enemy = color === 'red' ? 'black' : 'red'

  const canMoveTo = (tr: number, tc: number) => {
    if (!inBounds(tr, tc)) return false
    const target = board[tr]![tc]
    return !target || target.color === enemy
  }

  if (type === 'K') {
    for (const [dr, dc] of [[0,1],[0,-1],[1,0],[-1,0]]) {
      const nr = r+dr!, nc = c+dc!
      if (inPalace(nr, nc, color) && canMoveTo(nr, nc)) moves.push([nr, nc])
    }
  } else if (type === 'A') {
    for (const [dr, dc] of [[1,1],[1,-1],[-1,1],[-1,-1]]) {
      const nr = r+dr!, nc = c+dc!
      if (inPalace(nr, nc, color) && canMoveTo(nr, nc)) moves.push([nr, nc])
    }
  } else if (type === 'E') {
    for (const [dr, dc] of [[2,2],[2,-2],[-2,2],[-2,-2]]) {
      const nr = r+dr!, nc = c+dc!
      const er = r+dr!/2, ec = c+dc!/2 // eye
      if (inBounds(nr, nc) && isOwnSide(nr, color) && !board[er]![ec] && canMoveTo(nr, nc))
        moves.push([nr, nc])
    }
  } else if (type === 'H') {
    const legs: [number, number, number, number][] = [
      [-1,0,-2,-1],[-1,0,-2,1],[1,0,2,-1],[1,0,2,1],
      [0,-1,-1,-2],[0,-1,1,-2],[0,1,-1,2],[0,1,1,2],
    ]
    for (const [lr, lc, dr, dc] of legs) {
      const legR = r+lr!, legC = c+lc!, nr = r+dr!, nc = c+dc!
      if (inBounds(nr, nc) && !board[legR]?.[legC] && canMoveTo(nr, nc))
        moves.push([nr, nc])
    }
  } else if (type === 'R') {
    for (const [dr, dc] of [[0,1],[0,-1],[1,0],[-1,0]]) {
      let nr = r+dr!, nc = c+dc!
      while (inBounds(nr, nc)) {
        if (!board[nr]![nc]) { moves.push([nr, nc]) }
        else { if (board[nr]![nc]!.color === enemy) moves.push([nr, nc]); break }
        nr += dr!; nc += dc!
      }
    }
  } else if (type === 'C') {
    for (const [dr, dc] of [[0,1],[0,-1],[1,0],[-1,0]]) {
      let nr = r+dr!, nc = c+dc!; let jumped = false
      while (inBounds(nr, nc)) {
        if (!jumped) {
          if (!board[nr]![nc]) moves.push([nr, nc])
          else jumped = true
        } else {
          if (board[nr]![nc]) {
            if (board[nr]![nc]!.color === enemy) moves.push([nr, nc])
            break
          }
        }
        nr += dr!; nc += dc!
      }
    }
  } else if (type === 'P') {
    const forward = color === 'red' ? -1 : 1
    const crossed = color === 'red' ? r <= 4 : r >= 5
    if (canMoveTo(r + forward, c)) moves.push([r + forward, c])
    if (crossed) {
      if (canMoveTo(r, c - 1)) moves.push([r, c - 1])
      if (canMoveTo(r, c + 1)) moves.push([r, c + 1])
    }
  }
  return moves
}

function findKing(board: Board, color: PieceColor): [number, number] | null {
  for (let r = 0; r < 10; r++) for (let c = 0; c < 9; c++)
    if (board[r]![c]?.type === 'K' && board[r]![c]?.color === color) return [r, c]
  return null
}

function isKingFacing(board: Board): boolean {
  const rk = findKing(board, 'red'), bk = findKing(board, 'black')
  if (!rk || !bk || rk[1] !== bk[1]) return false
  for (let r = Math.min(rk[0], bk[0]) + 1; r < Math.max(rk[0], bk[0]); r++)
    if (board[r]![rk[1]]) return false
  return true
}

function isInCheck(board: Board, color: PieceColor): boolean {
  const king = findKing(board, color)
  if (!king) return true
  const enemy = color === 'red' ? 'black' : 'red'
  for (let r = 0; r < 10; r++) for (let c = 0; c < 9; c++) {
    if (board[r]![c]?.color === enemy) {
      const moves = getRawMoves(board, r, c)
      if (moves.some(([mr, mc]) => mr === king[0] && mc === king[1])) return true
    }
  }
  return false
}

function getLegalMoves(board: Board, r: number, c: number): [number, number][] {
  const piece = board[r]![c]
  if (!piece) return []
  return getRawMoves(board, r, c).filter(([tr, tc]) => {
    const test = cloneBoard(board)
    test[tr]![tc] = test[r]![c]
    test[r]![c] = null
    return !isInCheck(test, piece.color) && !isKingFacing(test)
  })
}

function isCheckmate(board: Board, color: PieceColor): boolean {
  for (let r = 0; r < 10; r++) for (let c = 0; c < 9; c++)
    if (board[r]![c]?.color === color && getLegalMoves(board, r, c).length > 0) return false
  return true
}

// ═══════════════════════════════════════════════════════════════════════════
// CANVAS RENDERER
// ═══════════════════════════════════════════════════════════════════════════

const canvasRef = ref<HTMLCanvasElement | null>(null)
const BOARD_PAD = 30
let cellSize = 50

function getCanvasSize() {
  const maxW = Math.min(460, window.innerWidth - 32)
  cellSize = Math.floor((maxW - BOARD_PAD * 2) / 8)
  return { w: cellSize * 8 + BOARD_PAD * 2, h: cellSize * 9 + BOARD_PAD * 2 }
}

function drawBoard() {
  const canvas = canvasRef.value
  if (!canvas) return
  const ctx = canvas.getContext('2d')
  if (!ctx) return
  const dpr = window.devicePixelRatio || 1
  const { w, h } = getCanvasSize()
  canvas.width = w * dpr; canvas.height = h * dpr
  canvas.style.width = `${w}px`; canvas.style.height = `${h}px`
  ctx.setTransform(dpr, 0, 0, dpr, 0, 0)

  const flipped = isFlipped.value
  const brd = board.value

  // Background
  const bg = ctx.createLinearGradient(0, 0, w, h)
  bg.addColorStop(0, '#2A1810'); bg.addColorStop(1, '#1A0E08')
  ctx.fillStyle = bg; ctx.fillRect(0, 0, w, h)

  // Board surface
  ctx.fillStyle = '#3D2B1F'
  ctx.fillRect(BOARD_PAD - 8, BOARD_PAD - 8, cellSize * 8 + 16, cellSize * 9 + 16)

  const ox = BOARD_PAD, oy = BOARD_PAD

  // Grid lines
  ctx.strokeStyle = '#8B7355'; ctx.lineWidth = 1
  for (let r = 0; r <= 9; r++) {
    ctx.beginPath(); ctx.moveTo(ox, oy + r * cellSize); ctx.lineTo(ox + 8 * cellSize, oy + r * cellSize); ctx.stroke()
  }
  for (let c = 0; c <= 8; c++) {
    ctx.beginPath()
    ctx.moveTo(ox + c * cellSize, oy); ctx.lineTo(ox + c * cellSize, oy + 4 * cellSize); ctx.stroke()
    ctx.beginPath()
    ctx.moveTo(ox + c * cellSize, oy + 5 * cellSize); ctx.lineTo(ox + c * cellSize, oy + 9 * cellSize); ctx.stroke()
  }

  // Palace diagonals
  ctx.strokeStyle = '#6B5B4F'; ctx.lineWidth = 0.8
  const drawPalace = (topR: number) => {
    const y1 = oy + topR * cellSize, y2 = oy + (topR + 2) * cellSize
    const x1 = ox + 3 * cellSize, x2 = ox + 5 * cellSize
    ctx.beginPath(); ctx.moveTo(x1, y1); ctx.lineTo(x2, y2); ctx.stroke()
    ctx.beginPath(); ctx.moveTo(x2, y1); ctx.lineTo(x1, y2); ctx.stroke()
  }
  drawPalace(flipped ? 7 : 0); drawPalace(flipped ? 0 : 7)

  // River text
  ctx.fillStyle = '#6B5B4F'; ctx.font = `italic ${cellSize * 0.35}px "Anybody", serif`
  ctx.textAlign = 'center'; ctx.textBaseline = 'middle'
  const riverY = oy + 4.5 * cellSize
  ctx.fillText('楚 河', ox + 2 * cellSize, riverY)
  ctx.fillText('漢 界', ox + 6 * cellSize, riverY)

  // Selected highlight
  if (selectedPos.value) {
    const [sr, sc] = selectedPos.value
    const dr = flipped ? 9 - sr : sr, dc = flipped ? 8 - sc : sc
    ctx.fillStyle = 'rgba(255, 184, 48, 0.25)'
    ctx.fillRect(ox + dc * cellSize - cellSize * 0.45, oy + dr * cellSize - cellSize * 0.45, cellSize * 0.9, cellSize * 0.9)
  }

  // Valid move indicators
  for (const [mr, mc] of validMoves.value) {
    const dr = flipped ? 9 - mr : mr, dc = flipped ? 8 - mc : mc
    const tx = ox + dc * cellSize, ty = oy + dr * cellSize
    if (brd[mr]![mc]) {
      ctx.strokeStyle = 'rgba(255, 107, 74, 0.7)'; ctx.lineWidth = 2
      ctx.beginPath(); ctx.arc(tx, ty, cellSize * 0.4, 0, Math.PI * 2); ctx.stroke()
    } else {
      ctx.fillStyle = 'rgba(255, 184, 48, 0.5)'
      ctx.beginPath(); ctx.arc(tx, ty, cellSize * 0.12, 0, Math.PI * 2); ctx.fill()
    }
  }

  // Pieces
  const pieceR = cellSize * 0.4
  for (let r = 0; r < 10; r++) for (let c = 0; c < 9; c++) {
    const p = brd[r]![c]
    if (!p) continue
    const dr = flipped ? 9 - r : r, dc = flipped ? 8 - c : c
    const px = ox + dc * cellSize, py = oy + dr * cellSize

    // Shadow
    ctx.fillStyle = 'rgba(0,0,0,0.3)'
    ctx.beginPath(); ctx.arc(px + 1, py + 2, pieceR, 0, Math.PI * 2); ctx.fill()

    // Piece circle
    const grad = ctx.createRadialGradient(px - pieceR * 0.2, py - pieceR * 0.2, 0, px, py, pieceR)
    if (p.color === 'red') {
      grad.addColorStop(0, '#5C1A0A'); grad.addColorStop(1, '#3A0E05')
    } else {
      grad.addColorStop(0, '#1F2A20'); grad.addColorStop(1, '#0E150F')
    }
    ctx.fillStyle = grad
    ctx.beginPath(); ctx.arc(px, py, pieceR, 0, Math.PI * 2); ctx.fill()

    // Border
    ctx.strokeStyle = p.color === 'red' ? '#D4703A' : '#6B8B5E'; ctx.lineWidth = 1.5
    ctx.beginPath(); ctx.arc(px, py, pieceR - 1, 0, Math.PI * 2); ctx.stroke()
    // Inner ring
    ctx.strokeStyle = p.color === 'red' ? '#D4703A44' : '#6B8B5E44'; ctx.lineWidth = 0.8
    ctx.beginPath(); ctx.arc(px, py, pieceR - 4, 0, Math.PI * 2); ctx.stroke()

    // Character
    const ch = PIECE_CHAR[p.type]![p.color]
    ctx.fillStyle = p.color === 'red' ? '#FF6B4A' : '#A8D8A0'
    ctx.font = `bold ${cellSize * 0.42}px "Noto Serif SC", "SimSun", serif`
    ctx.textAlign = 'center'; ctx.textBaseline = 'middle'
    ctx.fillText(ch, px, py + 1)
  }

  // Check indicator
  if (check.value) {
    const king = findKing(brd, turn.value)
    if (king) {
      const [kr, kc] = king
      const dkr = flipped ? 9 - kr : kr, dkc = flipped ? 8 - kc : kc
      ctx.strokeStyle = '#EF4444'; ctx.lineWidth = 3
      ctx.beginPath(); ctx.arc(ox + dkc * cellSize, oy + dkr * cellSize, pieceR + 3, 0, Math.PI * 2)
      ctx.stroke()
    }
  }
}

function canvasToBoard(ex: number, ey: number): [number, number] | null {
  const canvas = canvasRef.value
  if (!canvas) return null
  const rect = canvas.getBoundingClientRect()
  const sx = (ex - rect.left), sy = (ey - rect.top)
  const c = Math.round((sx - BOARD_PAD) / cellSize)
  const r = Math.round((sy - BOARD_PAD) / cellSize)
  if (r < 0 || r > 9 || c < 0 || c > 8) return null
  const flipped = isFlipped.value
  return [flipped ? 9 - r : r, flipped ? 8 - c : c]
}

// ═══════════════════════════════════════════════════════════════════════════
// WEBRTC (adapted from dongnguyenvie:feat/peer-call + DataChannel)
// ═══════════════════════════════════════════════════════════════════════════

const ICE_CFG: RTCConfiguration = {
  iceServers: [
    { urls: 'stun:stun.l.google.com:19302' },
    { urls: 'stun:stun1.l.google.com:19302' },
    { urls: 'turn:openrelay.metered.ca:80', username: 'openrelayproject', credential: 'openrelayproject' },
    { urls: 'turn:openrelay.metered.ca:443', username: 'openrelayproject', credential: 'openrelayproject' },
  ],
}

const connState = ref<ConnectionState>('idle')
const localStream = ref<MediaStream | null>(null)
const remoteStream = ref<MediaStream | null>(null)
const localVideoRef = ref<HTMLVideoElement | null>(null)
const remoteVideoRef = ref<HTMLVideoElement | null>(null)
const offerSdp = ref(''); const answerSdp = ref('')
const pastedOffer = ref(''); const pastedAnswer = ref('')
const isMuted = ref(false); const isCameraOff = ref(false)
const copied = ref(false); const connError = ref('')

let pc: RTCPeerConnection | null = null
let dataChannel: RTCDataChannel | null = null

function sdpEncode(s: string): string { return btoa(s) }
function sdpDecode(s: string): string {
  const t = s.trim(); if (t.startsWith('{')) return t
  try { return atob(t) } catch { return t }
}

async function startMedia() {
  const stream = await navigator.mediaDevices.getUserMedia({ video: true, audio: true })
  localStream.value = stream
  await nextTick()
  if (localVideoRef.value) { localVideoRef.value.srcObject = stream; localVideoRef.value.play() }
  return stream
}

function setupPeerConnection(stream: MediaStream) {
  pc = new RTCPeerConnection(ICE_CFG)
  remoteStream.value = new MediaStream()

  for (const track of stream.getTracks()) pc.addTrack(track, stream)

  pc.ontrack = (e) => {
    const s = e.streams[0]
    if (s) for (const track of s.getTracks()) remoteStream.value?.addTrack(track)
    nextTick(() => { if (remoteVideoRef.value && remoteStream.value) { remoteVideoRef.value.srcObject = remoteStream.value; remoteVideoRef.value.play().catch(() => {}) } })
  }

  pc.oniceconnectionstatechange = () => {
    if (pc?.iceConnectionState === 'connected' && connState.value !== 'joining')
      connState.value = 'connected'
    if (pc?.iceConnectionState === 'failed' && connState.value === 'connected')
      disconnectRTC()
  }

  return pc
}

function setupDataChannel(ch: RTCDataChannel) {
  dataChannel = ch
  ch.onmessage = (e) => {
    try {
      const msg = JSON.parse(e.data)
      if (msg.type === 'move') handleRemoteMove(msg.from, msg.to)
      if (msg.type === 'resign') handleRemoteResign()
      if (msg.type === 'sync') handleRemoteSync(msg)
    } catch { /* ignore */ }
  }
}

function waitIce(conn: RTCPeerConnection): Promise<string> {
  return new Promise((resolve) => {
    const done = () => resolve(JSON.stringify(conn.localDescription))
    if (conn.iceGatheringState === 'complete') { done(); return }
    const t = setTimeout(() => { conn.onicegatheringstatechange = null; done() }, 10000)
    conn.onicegatheringstatechange = () => {
      if (conn.iceGatheringState === 'complete') { clearTimeout(t); done() }
    }
  })
}

async function createRoom() {
  connState.value = 'creating-offer'; connError.value = ''
  try {
    const stream = await startMedia()
    const conn = setupPeerConnection(stream)
    const ch = conn.createDataChannel('chess')
    setupDataChannel(ch)
    const offer = await conn.createOffer()
    await conn.setLocalDescription(offer)
    const sdp = await waitIce(conn)
    offerSdp.value = sdpEncode(sdp)
  } catch (err) {
    connError.value = 'Không thể bật camera/mic. Vui lòng cấp quyền.'
    connState.value = 'idle'
  }
}

async function joinRoom() {
  connState.value = 'joining'; connError.value = ''
  try {
    const stream = await startMedia()
    const conn = setupPeerConnection(stream)
    conn.ondatachannel = (e) => setupDataChannel(e.channel)
    const decoded = sdpDecode(pastedOffer.value)
    await conn.setRemoteDescription(JSON.parse(decoded))
    const answer = await conn.createAnswer()
    await conn.setLocalDescription(answer)
    const sdp = await waitIce(conn)
    answerSdp.value = sdpEncode(sdp)
    // Auto copy
    safeCopy(answerSdp.value)
  } catch {
    connError.value = 'Mã mời không hợp lệ.'
    connState.value = 'idle'
  }
}

async function acceptAnswer() {
  if (!pc) return
  try {
    const decoded = sdpDecode(pastedAnswer.value)
    await pc.setRemoteDescription(JSON.parse(decoded))
  } catch { connError.value = 'Mã trả lời không hợp lệ.' }
}

function enterOnlineGame() {
  connState.value = 'connected'
  gameMode.value = 'online-play'
  // Joiner plays black (board flipped), creator plays red
  if (answerSdp.value) { myColor.value = 'black'; isFlipped.value = true }
  else { myColor.value = 'red'; isFlipped.value = false }
  sendSync()
}

function sendMove(from: [number, number], to: [number, number]) {
  dataChannel?.send(JSON.stringify({ type: 'move', from, to }))
}

function sendSync() {
  const state = { type: 'sync', board: board.value, turn: turn.value, moveHistory: moveHistory.value }
  dataChannel?.send(JSON.stringify(state))
}

function sendResign() { dataChannel?.send(JSON.stringify({ type: 'resign' })) }

function disconnectRTC() {
  pc?.close(); pc = null; dataChannel = null
  localStream.value?.getTracks().forEach(t => t.stop())
  localStream.value = null; remoteStream.value = null
  connState.value = 'idle'; offerSdp.value = ''; answerSdp.value = ''
  pastedOffer.value = ''; pastedAnswer.value = ''
}

async function safeCopy(text: string) {
  try { await navigator.clipboard.writeText(text); copied.value = true; setTimeout(() => copied.value = false, 2000); return true }
  catch { return false }
}

function toggleMute() {
  const t = localStream.value?.getAudioTracks()[0]
  if (t) { t.enabled = !t.enabled; isMuted.value = !t.enabled }
}

function toggleCamera() {
  const t = localStream.value?.getVideoTracks()[0]
  if (t) { t.enabled = !t.enabled; isCameraOff.value = !t.enabled }
}

// ═══════════════════════════════════════════════════════════════════════════
// GAME STATE
// ═══════════════════════════════════════════════════════════════════════════

const gameMode = ref<GameMode>('menu')
const board = ref<Board>(initBoard())
const turn = ref<Turn>('red')
const selectedPos = ref<[number, number] | null>(null)
const validMoves = ref<[number, number][]>([])
const moveHistory = ref<Move[]>([])
const check = ref(false)
const gameOver = ref(false)
const winner = ref<PieceColor | null>(null)
const myColor = ref<PieceColor>('red')
const isFlipped = ref(false)

function resetGame() {
  board.value = initBoard()
  turn.value = 'red'; selectedPos.value = null; validMoves.value = []
  moveHistory.value = []; check.value = false; gameOver.value = false; winner.value = null
}

function handleCanvasClick(e: MouseEvent | TouchEvent) {
  if (gameOver.value) return
  if (gameMode.value === 'spectator') return

  // Online: only move on your turn
  if (gameMode.value === 'online-play' && turn.value !== myColor.value) return

  const ev = 'touches' in e ? e.touches[0]! : e
  const pos = canvasToBoard(ev.clientX, ev.clientY)
  if (!pos) return
  const [r, c] = pos

  if (selectedPos.value) {
    const [sr, sc] = selectedPos.value
    // Try to move
    if (validMoves.value.some(([mr, mc]) => mr === r && mc === c)) {
      makeMove(sr, sc, r, c)
      if (gameMode.value === 'online-play') sendMove([sr, sc], [r, c])
      return
    }
    // Select another piece of same color
    const p = board.value[r]![c]
    if (p && p.color === turn.value) { selectPiece(r, c); return }
    // Deselect
    selectedPos.value = null; validMoves.value = []
    nextTick(drawBoard)
    return
  }

  // Select piece
  const p = board.value[r]![c]
  if (p && p.color === turn.value) selectPiece(r, c)
}

function selectPiece(r: number, c: number) {
  selectedPos.value = [r, c]
  validMoves.value = getLegalMoves(board.value, r, c)
  nextTick(drawBoard)
}

function makeMove(fr: number, fc: number, tr: number, tc: number) {
  const piece = board.value[fr]![fc]
  if (!piece) return
  const captured = board.value[tr]![tc]
  board.value[tr]![tc] = piece
  board.value[fr]![fc] = null
  moveHistory.value.push({ from: [fr, fc], to: [tr, tc], piece, captured })

  selectedPos.value = null; validMoves.value = []
  turn.value = turn.value === 'red' ? 'black' : 'red'

  check.value = isInCheck(board.value, turn.value)
  if (isCheckmate(board.value, turn.value)) {
    gameOver.value = true
    winner.value = turn.value === 'red' ? 'black' : 'red'
  }
  nextTick(drawBoard)
}

function handleRemoteMove(from: [number, number], to: [number, number]) {
  makeMove(from[0], from[1], to[0], to[1])
}

function handleRemoteResign() {
  gameOver.value = true
  winner.value = myColor.value
}

function handleRemoteSync(msg: { board: Board; turn: Turn; moveHistory: Move[] }) {
  board.value = msg.board; turn.value = msg.turn; moveHistory.value = msg.moveHistory
  check.value = isInCheck(board.value, turn.value)
  nextTick(drawBoard)
}

function resign() {
  gameOver.value = true
  winner.value = turn.value === 'red' ? 'black' : 'red'
  if (gameMode.value === 'online-play') sendResign()
}

function startLocal() { gameMode.value = 'local'; resetGame(); nextTick(drawBoard) }
function startOnline() { gameMode.value = 'online-setup'; resetGame() }
function backToMenu() { disconnectRTC(); gameMode.value = 'menu'; resetGame() }
function flipBoard() { isFlipped.value = !isFlipped.value; nextTick(drawBoard) }

const lastMove = computed(() => {
  if (moveHistory.value.length === 0) return null
  const m = moveHistory.value[moveHistory.value.length - 1]!
  const ch = PIECE_CHAR[m.piece.type]![m.piece.color]
  const vn = PIECE_VN[m.piece.type]
  return `${ch} ${vn} (${COL_NAMES[m.from[1]]},${m.from[0]+1}) → (${COL_NAMES[m.to[1]]},${m.to[0]+1})${m.captured ? ' ✕' : ''}`
})

const turnLabel = computed(() => turn.value === 'red' ? '🔴 Đỏ' : '⚫ Đen')
const moveCount = computed(() => moveHistory.value.length)

onMounted(() => { window.addEventListener('resize', () => { if (gameMode.value !== 'menu') drawBoard() }) })
onUnmounted(() => { disconnectRTC() })

watch(board, () => { nextTick(drawBoard) }, { deep: true })
</script>

<template>
  <div class="min-h-screen bg-bg-deep text-text-primary font-body">
    <div class="max-w-5xl mx-auto px-4 py-8">

      <!-- ═══════ MENU ═══════ -->
      <div v-if="gameMode === 'menu'" class="min-h-[80vh] flex flex-col items-center justify-center">
        <div class="text-center animate-fade-up">
          <span class="text-6xl block mb-3">♟️</span>
          <h1 class="font-display text-4xl sm:text-6xl font-bold text-accent-coral mb-2 tracking-tight">Cờ Tướng</h1>
          <p class="text-text-secondary text-base sm:text-lg mb-1">Chinese Chess × WebRTC P2P</p>
          <p class="text-text-dim text-xs font-display tracking-wider mb-10">XIANGQI ENGINE × CANVAS × WEBCAM</p>
        </div>
        <div class="grid gap-4 sm:grid-cols-2 max-w-md w-full animate-fade-up animate-delay-1">
          <button class="border border-border-default bg-bg-surface p-6 text-left transition-all hover:-translate-y-1 hover:border-accent-coral hover:shadow-lg hover:shadow-accent-coral/5" @click="startLocal">
            <p class="font-display text-lg font-semibold mb-1">🎮 Chơi tại chỗ</p>
            <p class="text-sm text-text-dim">2 người, 1 thiết bị</p>
          </button>
          <button class="border border-border-default bg-bg-surface p-6 text-left transition-all hover:-translate-y-1 hover:border-accent-sky hover:shadow-lg hover:shadow-accent-sky/5" @click="startOnline">
            <p class="font-display text-lg font-semibold mb-1">🌐 Chơi online</p>
            <p class="text-sm text-text-dim">Video call P2P + DataChannel</p>
          </button>
        </div>
        <div class="mt-8 animate-fade-up animate-delay-2">
          <RouterLink to="/" class="inline-flex items-center gap-2 border border-border-default bg-bg-surface px-5 py-2 text-sm text-text-secondary hover:border-accent-coral hover:text-text-primary transition">&larr; Về trang chủ</RouterLink>
        </div>
      </div>

      <!-- ═══════ ONLINE SETUP ═══════ -->
      <div v-if="gameMode === 'online-setup'" class="max-w-lg mx-auto py-8 animate-fade-up">
        <div class="flex items-center gap-3 mb-6">
          <button class="text-text-dim hover:text-text-primary transition text-sm" @click="backToMenu">&larr; Menu</button>
          <h2 class="font-display text-2xl font-bold text-accent-coral">🌐 Kết nối P2P</h2>
        </div>

        <div v-if="connError" class="mb-4 border border-accent-coral/40 bg-accent-coral/10 p-3 text-sm text-accent-coral">{{ connError }}</div>

        <!-- IDLE: Choose role -->
        <div v-if="connState === 'idle'" class="space-y-4">
          <div class="grid gap-4 sm:grid-cols-2">
            <button class="border border-border-default bg-bg-surface p-5 text-left transition hover:-translate-y-0.5 hover:border-accent-coral" @click="createRoom">
              <p class="font-display font-semibold mb-1">📡 Tạo phòng</p>
              <p class="text-xs text-text-dim">Tạo mã mời & gửi cho đối thủ</p>
            </button>
            <div class="border border-border-default bg-bg-surface p-5">
              <p class="font-display font-semibold mb-2">📥 Tham gia</p>
              <textarea v-model="pastedOffer" placeholder="Dán mã mời..." class="w-full bg-bg-deep border border-border-default p-2 text-xs resize-none h-16 focus:border-accent-sky focus:outline-none" />
              <button class="mt-2 w-full border border-accent-sky bg-accent-sky/10 px-3 py-1.5 text-xs text-accent-sky transition hover:bg-accent-sky/20 disabled:opacity-30" :disabled="!pastedOffer.trim()" @click="joinRoom">Kết nối</button>
            </div>
          </div>
        </div>

        <!-- CREATING OFFER -->
        <div v-if="connState === 'creating-offer'" class="space-y-4">
          <div class="border border-border-default bg-bg-surface p-4">
            <p class="font-display text-sm font-semibold text-accent-amber mb-2">Bước 1: Copy mã mời</p>
            <textarea :value="offerSdp" readonly class="w-full bg-bg-deep border border-border-default p-2 text-xs resize-none h-20 focus:outline-none" />
            <button class="mt-2 w-full bg-accent-coral px-3 py-2 text-sm font-display font-bold text-bg-deep transition hover:opacity-90" @click="safeCopy(offerSdp)">{{ copied ? '✅ Đã copy!' : '📋 Copy mã mời' }}</button>
          </div>
          <div class="border border-border-default bg-bg-surface p-4">
            <p class="font-display text-sm font-semibold text-accent-amber mb-2">Bước 2: Dán mã trả lời</p>
            <textarea v-model="pastedAnswer" placeholder="Dán mã trả lời từ đối thủ..." class="w-full bg-bg-deep border border-border-default p-2 text-xs resize-none h-20 focus:border-accent-amber focus:outline-none" />
            <button class="mt-2 w-full border border-accent-coral bg-accent-coral/10 px-3 py-2 text-sm text-accent-coral transition disabled:opacity-30" :disabled="!pastedAnswer.trim()" @click="acceptAnswer">Kết nối</button>
          </div>
        </div>

        <!-- JOINING -->
        <div v-if="connState === 'joining'" class="space-y-4">
          <div class="border border-border-default bg-bg-surface p-4">
            <p class="font-display text-sm font-semibold text-accent-sky mb-2">Mã trả lời (gửi lại cho người tạo phòng)</p>
            <textarea :value="answerSdp" readonly class="w-full bg-bg-deep border border-border-default p-2 text-xs resize-none h-20 focus:outline-none" />
            <button class="mt-2 w-full bg-accent-sky px-3 py-2 text-sm font-display font-bold text-bg-deep transition hover:opacity-90" @click="safeCopy(answerSdp)">{{ copied ? '✅ Đã copy!' : '📋 Copy mã trả lời' }}</button>
          </div>
          <button class="w-full bg-green-600 px-4 py-3 font-display font-bold text-white text-sm transition hover:bg-green-500" @click="enterOnlineGame">▶ VÀO GAME</button>
        </div>

        <!-- CONNECTED -->
        <div v-if="connState === 'connected' && gameMode === 'online-setup'">
          <div class="border border-green-500/40 bg-green-500/10 p-4 text-center">
            <p class="text-green-400 font-display font-semibold mb-2">✅ Đã kết nối!</p>
            <button class="bg-green-600 px-6 py-3 font-display font-bold text-white transition hover:bg-green-500" @click="enterOnlineGame">▶ BẮT ĐẦU CHƠI</button>
          </div>
        </div>
      </div>

      <!-- ═══════ GAME BOARD (local + online) ═══════ -->
      <div v-if="gameMode === 'local' || gameMode === 'online-play' || gameMode === 'spectator'" class="animate-fade-up">
        <div class="flex items-center justify-between mb-4">
          <button class="text-text-dim hover:text-text-primary transition text-sm" @click="backToMenu">&larr; Menu</button>
          <div class="flex items-center gap-3">
            <span class="font-display text-sm font-semibold" :class="turn === 'red' ? 'text-accent-coral' : 'text-text-primary'">{{ turnLabel }}</span>
            <span v-if="check && !gameOver" class="text-xs bg-red-600/20 text-red-400 px-2 py-0.5 font-display">CHIẾU!</span>
          </div>
          <span class="text-text-dim text-xs font-display">Nước {{ moveCount }}</span>
        </div>

        <div class="flex flex-col lg:flex-row gap-4 items-start">
          <!-- Webcam panels (online) -->
          <div v-if="gameMode === 'online-play'" class="w-full lg:w-48 flex lg:flex-col gap-2">
            <div class="flex-1 border border-border-default bg-bg-surface overflow-hidden relative">
              <video ref="remoteVideoRef" autoplay playsinline muted class="w-full aspect-video object-cover" style="transform: scaleX(-1)" />
              <span class="absolute bottom-1 left-1 text-xs bg-bg-deep/80 px-1.5 py-0.5 font-display" :class="myColor === 'red' ? 'text-text-primary' : 'text-accent-coral'">{{ myColor === 'red' ? '⚫ Đối thủ' : '🔴 Đối thủ' }}</span>
            </div>
            <div class="flex-1 border border-border-default bg-bg-surface overflow-hidden relative">
              <video ref="localVideoRef" autoplay playsinline muted class="w-full aspect-video object-cover" style="transform: scaleX(-1)" />
              <span class="absolute bottom-1 left-1 text-xs bg-bg-deep/80 px-1.5 py-0.5 font-display" :class="myColor === 'red' ? 'text-accent-coral' : 'text-text-primary'">{{ myColor === 'red' ? '🔴 Bạn' : '⚫ Bạn' }}</span>
            </div>
            <div class="flex lg:flex-col gap-1">
              <button class="flex-1 border border-border-default bg-bg-surface p-1.5 text-xs transition hover:border-accent-coral" @click="toggleMute">{{ isMuted ? '🔇' : '🔊' }}</button>
              <button class="flex-1 border border-border-default bg-bg-surface p-1.5 text-xs transition hover:border-accent-coral" @click="toggleCamera">{{ isCameraOff ? '📷❌' : '📷' }}</button>
            </div>
          </div>

          <!-- Board -->
          <div class="flex-shrink-0">
            <canvas ref="canvasRef" class="block cursor-pointer border border-border-default" @click="handleCanvasClick" @touchstart.prevent="handleCanvasClick" />
          </div>

          <!-- Side panel -->
          <div class="flex-1 min-w-0 lg:max-w-[220px] space-y-3">
            <!-- Controls -->
            <div class="flex flex-wrap gap-2">
              <button class="border border-border-default bg-bg-surface px-3 py-1.5 text-xs transition hover:border-accent-sky" @click="flipBoard">🔄 Lật</button>
              <button v-if="!gameOver" class="border border-border-default bg-bg-surface px-3 py-1.5 text-xs transition hover:border-accent-coral text-accent-coral" @click="resign">🏳️ Xin thua</button>
              <button v-if="gameOver" class="border border-accent-coral bg-accent-coral/10 px-3 py-1.5 text-xs text-accent-coral transition hover:bg-accent-coral/20" @click="resetGame(); nextTick(drawBoard)">🔄 Ván mới</button>
            </div>

            <!-- Game Over -->
            <div v-if="gameOver" class="border border-accent-amber/40 bg-accent-amber/10 p-3 text-center">
              <p class="font-display text-lg font-bold" :class="winner === 'red' ? 'text-accent-coral' : 'text-text-primary'">
                {{ winner === 'red' ? '🔴 ĐỎ THẮNG!' : '⚫ ĐEN THẮNG!' }}
              </p>
              <p class="text-text-dim text-xs mt-1">{{ check ? 'Chiếu bí!' : 'Đối thủ xin thua' }}</p>
            </div>

            <!-- Last move -->
            <div v-if="lastMove" class="border border-border-default bg-bg-surface p-2">
              <p class="text-text-dim text-xs font-display mb-1">NƯỚC ĐI CUỐI</p>
              <p class="text-sm text-text-secondary">{{ lastMove }}</p>
            </div>

            <!-- Move history -->
            <div class="border border-border-default bg-bg-surface p-2 max-h-48 overflow-y-auto">
              <p class="text-text-dim text-xs font-display mb-1">LỊCH SỬ ({{ moveCount }})</p>
              <div v-if="moveHistory.length === 0" class="text-text-dim text-xs italic">Chưa có nước đi</div>
              <div v-for="(m, i) in moveHistory" :key="i" class="text-xs py-0.5" :class="m.piece.color === 'red' ? 'text-accent-coral' : 'text-text-secondary'">
                {{ i + 1 }}. {{ PIECE_CHAR[m.piece.type]![m.piece.color] }} {{ PIECE_VN[m.piece.type] }}
                ({{ COL_NAMES[m.from[1]] }},{{ m.from[0]+1 }}) → ({{ COL_NAMES[m.to[1]] }},{{ m.to[0]+1 }})
                <span v-if="m.captured" class="text-accent-amber">✕{{ PIECE_CHAR[m.captured.type]![m.captured.color] }}</span>
              </div>
            </div>
          </div>
        </div>

        <!-- Footer -->
        <div class="mt-8 text-center">
          <p class="text-text-dim text-xs font-display tracking-wider">CỜ TƯỚNG ONLINE ♟️ XIANGQI × WEBRTC × CANVAS — BY HWG</p>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
canvas { image-rendering: -webkit-optimize-contrast; touch-action: none; }
video { background: #0F1923; }
</style>
