<script setup>
import { computed, ref } from 'vue'

const maxRounds = 8
const status = ref('setting')
const playerCount = ref(1)
const nameInputs = ref(['', '', '', ''])
const players = ref([])
const currentPlayerIndex = ref(0)
const currentRoundIndex = ref(0)
const selectedType = ref('S')
const history = ref([])

const currentPlayer = computed(() => players.value[currentPlayerIndex.value])
const currentRoundDisplay = computed(() => currentRoundIndex.value + 1)

const throwTypeLabels = {
  S: 'シングル',
  D: 'ダブル',
  T: 'トリプル',
  OB: 'アウターブル',
  IB: 'インナーブル',
  MISS: 'ミス',
}

const numberButtons = Array.from({ length: 20 }, (_, idx) => idx + 1)

function createPlayers() {
  return Array.from({ length: playerCount.value }, (_, index) => {
    const name = nameInputs.value[index]?.trim()
    return {
      id: `player-${index + 1}`,
      name: name || `PLAYER${index + 1}`,
      rounds: Array.from({ length: maxRounds }, (_, roundIdx) => ({
        roundIndex: roundIdx + 1,
        throws: [],
        roundScore: 0,
      })),
      totalScore: 0,
    }
  })
}

function resetGameState() {
  players.value = createPlayers()
  currentPlayerIndex.value = 0
  currentRoundIndex.value = 0
  selectedType.value = 'S'
  status.value = 'playing'
  history.value = []
}

function startGame() {
  resetGameState()
}

function addThrow({ type, number, score }) {
  if (status.value !== 'playing') return
  if (currentRoundIndex.value >= maxRounds) return

  const player = players.value[currentPlayerIndex.value]
  const round = player.rounds[currentRoundIndex.value]

  if (round.throws.length >= 3) return

  round.throws.push({ type, number, score })
  round.roundScore = round.throws.reduce((total, current) => total + current.score, 0)
  player.totalScore = player.rounds.reduce((total, current) => total + current.roundScore, 0)
  history.value.push({ playerIndex: currentPlayerIndex.value, roundIndex: currentRoundIndex.value })

  if (round.throws.length === 3) {
    advanceTurn()
  }
}

function handleNumberClick(number) {
  const multiplier =
    selectedType.value === 'D' ? 2 : selectedType.value === 'T' ? 3 : 1
  addThrow({
    type: selectedType.value,
    number,
    score: number * multiplier,
  })
}

function handleBull(type) {
  const score = type === 'IB' ? 50 : 25
  addThrow({
    type,
    number: null,
    score,
  })
}

function handleMiss() {
  addThrow({
    type: 'MISS',
    number: null,
    score: 0,
  })
}

function advanceTurn() {
  const isLastPlayer = currentPlayerIndex.value === players.value.length - 1
  const isLastRound = currentRoundIndex.value === maxRounds - 1

  if (isLastPlayer && isLastRound) {
    status.value = 'finished'
    return
  }

  if (isLastPlayer) {
    currentPlayerIndex.value = 0
    currentRoundIndex.value += 1
  } else {
    currentPlayerIndex.value += 1
  }
}

function undoThrow() {
  const last = history.value.pop()
  if (!last) return

  const player = players.value[last.playerIndex]
  const round = player.rounds[last.roundIndex]
  round.throws.pop()
  round.roundScore = round.throws.reduce((total, current) => total + current.score, 0)
  player.totalScore = player.rounds.reduce((total, current) => total + current.roundScore, 0)

  currentPlayerIndex.value = last.playerIndex
  currentRoundIndex.value = last.roundIndex
  status.value = 'playing'
}

function replaySamePlayers() {
  resetGameState()
}

function newGame() {
  players.value = []
  status.value = 'setting'
  currentPlayerIndex.value = 0
  currentRoundIndex.value = 0
  history.value = []
  playerCount.value = 1
  nameInputs.value = ['', '', '', '']
}

const isInputLocked = computed(() => status.value !== 'playing')

function formatThrowDisplay(throwData) {
  if (throwData.type === 'OB' || throwData.type === 'IB' || throwData.type === 'MISS') {
    return throwTypeLabels[throwData.type]
  }
  return `${throwData.type}${throwData.number}`
}
</script>

<template>
  <div class="page">
    <header class="hero">
      <div>
        <p class="eyebrow">MyDarts v0.1</p>
        <h1>カウントアップ専用スコア管理</h1>
        <p class="subtitle">数字＋種別ボタンで素早く入力。1〜4人、8ラウンド固定。</p>
      </div>
      <div class="status-chip" :class="status">
        {{ status === 'setting' ? '設定中' : status === 'playing' ? 'プレイ中' : '結果' }}
      </div>
    </header>

    <section v-if="status === 'setting'" class="panel">
      <h2>プレイヤー設定</h2>
      <div class="form-row">
        <label for="player-count">人数</label>
        <select id="player-count" v-model.number="playerCount">
          <option v-for="count in [1, 2, 3, 4]" :key="count" :value="count">{{ count }} 人</option>
        </select>
      </div>
      <div class="player-inputs">
        <div v-for="(_, idx) in playerCount" :key="idx" class="form-row">
          <label :for="`player-${idx}`">プレイヤー{{ idx + 1 }}</label>
          <input
            :id="`player-${idx}`"
            v-model="nameInputs[idx]"
            type="text"
            maxlength="20"
            placeholder="名前 (空欄で自動設定)"
          />
        </div>
      </div>
      <button class="primary" @click="startGame">ゲーム開始</button>
    </section>

    <section v-else-if="status === 'playing'" class="panel">
      <div class="board">
        <div class="board-header">
          <div class="round">Round {{ currentRoundDisplay }} / {{ maxRounds }}</div>
          <button class="ghost" :disabled="!history.length" @click="undoThrow">UNDO</button>
        </div>
        <div class="players">
          <div
            v-for="(player, index) in players"
            :key="player.id"
            class="player-tile"
            :class="{ active: index === currentPlayerIndex }"
          >
            <div class="name">{{ player.name }}</div>
            <div class="score">{{ player.totalScore }}</div>
          </div>
        </div>

        <div class="current">
          <div class="current-header">
            <div>
              <p class="label">現在のプレイヤー</p>
              <h3>{{ currentPlayer?.name }}</h3>
            </div>
            <div class="round-tally">
              <p class="label">このラウンドの小計</p>
              <strong>{{ currentPlayer?.rounds[currentRoundIndex]?.roundScore ?? 0 }}</strong>
            </div>
          </div>
          <div class="throws">
            <div v-for="n in 3" :key="n" class="throw-row">
              <div class="throw-label">{{ n }}投目</div>
              <div class="throw-body">
                <template v-if="currentPlayer?.rounds[currentRoundIndex]?.throws[n - 1]">
                  <span class="pill">
                    {{ formatThrowDisplay(currentPlayer.rounds[currentRoundIndex].throws[n - 1]) }}
                  </span>
                  <span class="throw-score">= {{ currentPlayer.rounds[currentRoundIndex].throws[n - 1].score }}</span>
                </template>
                <span v-else class="placeholder">-</span>
              </div>
            </div>
          </div>
        </div>

        <div class="inputs">
          <div class="toggle-group">
            <button
              v-for="type in ['S', 'D', 'T']"
              :key="type"
              :class="['chip', { selected: selectedType === type }]"
              :disabled="isInputLocked"
              @click="selectedType = type"
            >
              {{ type }}
            </button>
          </div>
          <div class="number-grid">
            <button
              v-for="num in numberButtons"
              :key="num"
              class="num-btn"
              :disabled="isInputLocked"
              @click="handleNumberClick(num)"
            >
              {{ num }}
            </button>
          </div>
          <div class="bull-row">
            <button class="bull" :disabled="isInputLocked" @click="handleBull('OB')">OB (25)</button>
            <button class="bull" :disabled="isInputLocked" @click="handleBull('IB')">IB (50)</button>
            <button class="miss" :disabled="isInputLocked" @click="handleMiss">MISS (0)</button>
          </div>
        </div>
      </div>
    </section>

    <section v-else class="panel">
      <h2>結果</h2>
      <div class="results">
        <div v-for="player in players" :key="player.id" class="result-row">
          <div class="name">{{ player.name }}</div>
          <div class="score">{{ player.totalScore }}</div>
        </div>
      </div>
      <div class="actions">
        <button class="primary" @click="replaySamePlayers">もう一度プレイ</button>
        <button class="ghost" @click="newGame">新しいゲーム</button>
      </div>
    </section>
  </div>
</template>

<style scoped>
.page {
  max-width: 960px;
  margin: 0 auto;
  padding: 1.5rem;
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
}

.hero {
  display: flex;
  justify-content: space-between;
  align-items: center;
  background: linear-gradient(135deg, #0f172a, #111827);
  color: #e5e7eb;
  padding: 1.25rem 1.5rem;
  border-radius: 16px;
  box-shadow: 0 12px 40px rgba(0, 0, 0, 0.25);
}

.eyebrow {
  font-size: 0.85rem;
  letter-spacing: 0.08em;
  text-transform: uppercase;
  color: #7dd3fc;
  margin-bottom: 0.4rem;
}

.hero h1 {
  font-size: 1.75rem;
  margin-bottom: 0.2rem;
}

.subtitle {
  color: #cbd5e1;
  font-size: 0.95rem;
}

.status-chip {
  padding: 0.5rem 0.85rem;
  border-radius: 12px;
  font-weight: 700;
  color: #0b1021;
  background: #a5b4fc;
}

.status-chip.playing {
  background: #34d399;
}

.status-chip.finished {
  background: #fbbf24;
}

.panel {
  background: #0b1021;
  color: #e5e7eb;
  padding: 1.25rem;
  border-radius: 16px;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.35);
  border: 1px solid rgba(255, 255, 255, 0.06);
}

.panel h2 {
  margin-bottom: 0.75rem;
  font-size: 1.2rem;
}

.form-row {
  display: flex;
  flex-direction: column;
  gap: 0.3rem;
  margin-bottom: 0.75rem;
}

label {
  font-size: 0.95rem;
  color: #cbd5e1;
}

select,
input,
button {
  font: inherit;
}

select,
input {
  background: #0f172a;
  color: #e5e7eb;
  border: 1px solid rgba(255, 255, 255, 0.1);
  border-radius: 10px;
  padding: 0.6rem 0.75rem;
}

.primary,
.ghost {
  padding: 0.75rem 1rem;
  border-radius: 12px;
  border: none;
  cursor: pointer;
  font-weight: 700;
  transition: transform 0.1s ease, box-shadow 0.1s ease, background 0.2s ease;
}

.primary {
  background: linear-gradient(135deg, #22c55e, #16a34a);
  color: #0b1021;
  box-shadow: 0 12px 30px rgba(34, 197, 94, 0.3);
}

.primary:active {
  transform: translateY(1px);
}

.ghost {
  background: rgba(255, 255, 255, 0.07);
  color: #e5e7eb;
  border: 1px solid rgba(255, 255, 255, 0.12);
}

.ghost:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

.player-inputs {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
  gap: 0.75rem;
  margin: 0.5rem 0 1rem;
}

.board-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.75rem;
}

.round {
  padding: 0.4rem 0.8rem;
  background: rgba(255, 255, 255, 0.08);
  border-radius: 12px;
  font-weight: 700;
  letter-spacing: 0.02em;
}

.players {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
  gap: 0.75rem;
}

.player-tile {
  background: #0f172a;
  padding: 0.9rem;
  border-radius: 12px;
  border: 1px solid rgba(255, 255, 255, 0.08);
  transition: border 0.2s ease, box-shadow 0.2s ease;
}

.player-tile.active {
  border-color: #34d399;
  box-shadow: 0 0 0 1px rgba(52, 211, 153, 0.3);
}

.player-tile .name {
  color: #cbd5e1;
  margin-bottom: 0.2rem;
}

.player-tile .score {
  font-size: 1.6rem;
  font-weight: 800;
}

.current {
  margin-top: 1rem;
  padding: 1rem;
  background: #0f172a;
  border-radius: 12px;
  border: 1px solid rgba(255, 255, 255, 0.08);
  display: flex;
  flex-direction: column;
  gap: 0.8rem;
}

.current-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.label {
  color: #94a3b8;
  font-size: 0.85rem;
}

.round-tally strong {
  font-size: 1.4rem;
}

.throws {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.throw-row {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.65rem 0.75rem;
  background: rgba(255, 255, 255, 0.02);
  border-radius: 10px;
  border: 1px solid rgba(255, 255, 255, 0.06);
}

.throw-label {
  width: 56px;
  color: #cbd5e1;
}

.throw-body {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  flex: 1;
}

.placeholder {
  color: #475569;
}

.pill {
  display: inline-block;
  padding: 0.35rem 0.65rem;
  background: rgba(124, 58, 237, 0.2);
  border: 1px solid rgba(124, 58, 237, 0.4);
  border-radius: 999px;
  color: #e9d5ff;
  font-weight: 700;
  min-width: 64px;
  text-align: center;
}

.throw-score {
  color: #e5e7eb;
  font-weight: 700;
}

.inputs {
  margin-top: 1rem;
  display: flex;
  flex-direction: column;
  gap: 0.8rem;
}

.toggle-group {
  display: inline-flex;
  gap: 0.5rem;
}

.chip {
  padding: 0.55rem 0.9rem;
  border-radius: 12px;
  border: 1px solid rgba(255, 255, 255, 0.12);
  background: rgba(255, 255, 255, 0.06);
  color: #e5e7eb;
  cursor: pointer;
}

.chip.selected {
  background: rgba(52, 211, 153, 0.2);
  border-color: rgba(52, 211, 153, 0.6);
  color: #bbf7d0;
}

.chip:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

.number-grid {
  display: grid;
  grid-template-columns: repeat(5, minmax(0, 1fr));
  gap: 0.5rem;
}

.num-btn {
  padding: 0.65rem;
  border-radius: 12px;
  background: #111827;
  color: #f8fafc;
  border: 1px solid rgba(255, 255, 255, 0.08);
  font-size: 1.05rem;
  cursor: pointer;
  transition: transform 0.1s ease, background 0.2s ease;
}

.num-btn:active {
  transform: translateY(1px);
}

.num-btn:disabled {
  opacity: 0.35;
  cursor: not-allowed;
}

.bull-row {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 0.5rem;
}

.bull,
.miss {
  padding: 0.75rem;
  border-radius: 12px;
  border: 1px solid rgba(255, 255, 255, 0.12);
  background: rgba(255, 255, 255, 0.07);
  color: #e5e7eb;
  font-weight: 700;
  cursor: pointer;
}

.miss {
  background: rgba(239, 68, 68, 0.15);
  border-color: rgba(239, 68, 68, 0.4);
  color: #fecdd3;
}

.bull:disabled,
.miss:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

.results {
  display: grid;
  gap: 0.6rem;
  margin: 1rem 0;
}

.result-row {
  display: flex;
  justify-content: space-between;
  padding: 0.85rem 1rem;
  background: #0f172a;
  border-radius: 12px;
  border: 1px solid rgba(255, 255, 255, 0.08);
}

.actions {
  display: flex;
  gap: 0.75rem;
  flex-wrap: wrap;
}

@media (max-width: 640px) {
  .hero {
    flex-direction: column;
    align-items: flex-start;
    gap: 0.75rem;
  }

  .current-header {
    flex-direction: column;
    align-items: flex-start;
    gap: 0.6rem;
  }

  .board-header {
    flex-direction: column;
    align-items: flex-start;
    gap: 0.5rem;
  }
}
</style>
