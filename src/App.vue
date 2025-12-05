<script setup>
import { computed, reactive, ref } from 'vue'

const MAX_PLAYERS = 4
const MAX_ROUNDS = 8
const THROWS_PER_ROUND = 3

const playerCount = ref(1)
const playerInputs = reactive(
  Array.from({ length: MAX_PLAYERS }, (_, index) => ({
    name: '',
    id: `input-${index}`,
  })),
)

const gameStatus = ref('setting')
const players = ref([])
const currentRoundIndex = ref(0)
const currentPlayerIndex = ref(0)
const selectedMultiplier = ref('S')

const multiplierOptions = [
  { label: 'S', value: 'S' },
  { label: 'D', value: 'D' },
  { label: 'T', value: 'T' },
]

const bullOptions = [
  { label: 'OB', value: 'OB', description: '25 点' },
  { label: 'IB', value: 'IB', description: '50 点' },
  { label: 'MISS', value: 'MISS', description: '0 点' },
]

const numberButtons = Array.from({ length: 20 }, (_, index) => index + 1)

const currentPlayer = computed(() => players.value[currentPlayerIndex.value])
const currentRound = computed(() => currentPlayer.value?.rounds[currentRoundIndex.value])

const isPlaying = computed(() => gameStatus.value === 'playing')

function defaultName(index) {
  return `PLAYER${index + 1}`
}

function createEmptyRounds() {
  return Array.from({ length: MAX_ROUNDS }, (_, index) => ({
    roundIndex: index + 1,
    throws: [],
    roundScore: 0,
  }))
}

function startGame() {
  players.value = playerInputs
    .slice(0, playerCount.value)
    .map((player, index) => ({
      id: `player-${index}`,
      name: player.name.trim() || defaultName(index),
      rounds: createEmptyRounds(),
      totalScore: 0,
    }))

  gameStatus.value = 'playing'
  currentRoundIndex.value = 0
  currentPlayerIndex.value = 0
  selectedMultiplier.value = 'S'
}

function resetScoresForReplay() {
  players.value = players.value.map((player, index) => ({
    ...player,
    name: player.name || defaultName(index),
    rounds: createEmptyRounds(),
    totalScore: 0,
  }))

  gameStatus.value = 'playing'
  currentRoundIndex.value = 0
  currentPlayerIndex.value = 0
  selectedMultiplier.value = 'S'
}

function resetToNewGame() {
  gameStatus.value = 'setting'
  players.value = []
  currentRoundIndex.value = 0
  currentPlayerIndex.value = 0
  selectedMultiplier.value = 'S'
  playerCount.value = 1
  playerInputs.forEach((player) => {
    player.name = ''
  })
}

function scoreForThrow(type, number) {
  if (type === 'MISS') return 0
  if (type === 'OB') return 25
  if (type === 'IB') return 50

  const base = Number(number) || 0
  if (type === 'D') return base * 2
  if (type === 'T') return base * 3
  return base
}

function addThrow(type, number = null) {
  if (!isPlaying.value) return
  if (!currentPlayer.value || !currentRound.value) return
  if (currentRound.value.throws.length >= THROWS_PER_ROUND) return

  const throwScore = scoreForThrow(type, number)
  const dartThrow = {
    type,
    number: number ?? null,
    score: throwScore,
  }

  currentRound.value.throws.push(dartThrow)
  recomputeScores()
  moveToNextTurnIfNeeded()
}

function moveToNextTurnIfNeeded() {
  const roundDone = currentRound.value?.throws.length === THROWS_PER_ROUND
  if (!roundDone) return

  if (
    currentRoundIndex.value === MAX_ROUNDS - 1 &&
    currentPlayerIndex.value === players.value.length - 1
  ) {
    gameStatus.value = 'finished'
    return
  }

  if (currentPlayerIndex.value < players.value.length - 1) {
    currentPlayerIndex.value += 1
  } else {
    currentPlayerIndex.value = 0
    currentRoundIndex.value += 1
  }
}

function findLastThrowLocation() {
  for (let roundIndex = MAX_ROUNDS - 1; roundIndex >= 0; roundIndex -= 1) {
    for (let playerIndex = players.value.length - 1; playerIndex >= 0; playerIndex -= 1) {
      const candidateRound = players.value[playerIndex]?.rounds?.[roundIndex]
      if (candidateRound && candidateRound.throws.length > 0) {
        return { playerIndex, roundIndex }
      }
    }
  }
  return null
}

function undoLastThrow() {
  if (!players.value.length) return

  const lastLocation = findLastThrowLocation()
  if (!lastLocation) return

  const { playerIndex, roundIndex } = lastLocation
  const round = players.value[playerIndex].rounds[roundIndex]
  round.throws.pop()

  recomputeScores()
  repositionCursorAfterUndo()
}

function recomputeScores() {
  players.value = players.value.map((player) => {
    let total = 0

    const updatedRounds = player.rounds.map((round) => {
      const roundScore = round.throws.reduce((sum, dart) => sum + dart.score, 0)
      total += roundScore
      return { ...round, roundScore }
    })

    return {
      ...player,
      rounds: updatedRounds,
      totalScore: total,
    }
  })
}

function repositionCursorAfterUndo() {
  const nextPosition = findFirstAvailableSlot()
  currentPlayerIndex.value = nextPosition.playerIndex
  currentRoundIndex.value = nextPosition.roundIndex
  gameStatus.value = nextPosition.status
}

function findFirstAvailableSlot() {
  for (let roundIndex = 0; roundIndex < MAX_ROUNDS; roundIndex += 1) {
    for (let playerIndex = 0; playerIndex < players.value.length; playerIndex += 1) {
      const candidateRound = players.value[playerIndex]?.rounds?.[roundIndex]
      if (candidateRound && candidateRound.throws.length < THROWS_PER_ROUND) {
        return { playerIndex, roundIndex, status: 'playing' }
      }
    }
  }

  const finalPlayerIndex = Math.max(players.value.length - 1, 0)
  const finalRoundIndex = MAX_ROUNDS - 1
  return { playerIndex: finalPlayerIndex, roundIndex: finalRoundIndex, status: 'finished' }
}

function selectMultiplier(value) {
  selectedMultiplier.value = value
}

function handleNumberInput(number) {
  addThrow(selectedMultiplier.value, number)
}

function handleBullInput(type) {
  addThrow(type, null)
}

function throwDisplayText(dartThrow) {
  if (!dartThrow) return '-'

  if (dartThrow.type === 'OB' || dartThrow.type === 'IB') {
    return `${dartThrow.type} = ${dartThrow.score}`
  }

  if (dartThrow.type === 'MISS') return 'MISS = 0'

  return `${dartThrow.type}${dartThrow.number} = ${dartThrow.score}`
}
</script>

<template>
  <div class="app-shell">
    <header class="app-header">
      <div>
        <p class="app-subtitle">カウントアップ専用</p>
        <h1>MyDarts v0.1</h1>
      </div>
      <div class="status-chip" :class="gameStatus">{{ gameStatus }}</div>
    </header>

    <main>
      <section v-if="gameStatus === 'setting'" class="card">
        <h2>プレイヤー設定</h2>
        <div class="player-count">
          <p>プレイヤー数</p>
          <div class="count-buttons">
            <button
              v-for="count in MAX_PLAYERS"
              :key="count"
              type="button"
              :class="['pill', { active: playerCount === count } ]"
              @click="playerCount = count"
            >
              {{ count }} 人
            </button>
          </div>
        </div>

        <div class="player-inputs">
          <label v-for="index in playerCount" :key="playerInputs[index - 1].id" class="input-row">
            <span>PLAYER {{ index }}</span>
            <input
              v-model="playerInputs[index - 1].name"
              type="text"
              maxlength="20"
              placeholder="名前を入力 (空欄なら自動設定)"
            />
          </label>
        </div>

        <button class="primary" type="button" @click="startGame">ゲーム開始</button>
      </section>

      <section v-else-if="gameStatus === 'playing'" class="card">
        <div class="scoreboard">
          <div v-for="(player, index) in players" :key="player.id" class="player-score">
            <div class="player-name" :class="{ active: index === currentPlayerIndex }">
              {{ player.name }}
            </div>
            <div class="score">{{ player.totalScore }}</div>
          </div>
        </div>

        <div class="round-info">
          <div class="round-label">Round {{ currentRoundIndex + 1 }} / {{ MAX_ROUNDS }}</div>
          <div class="turn-label">手番: {{ currentPlayer?.name }}</div>
        </div>

        <div class="current-round">
          <h3>{{ currentPlayer?.name }} の投球</h3>
          <div class="throws">
            <div v-for="slot in THROWS_PER_ROUND" :key="slot" class="throw-slot">
              {{ throwDisplayText(currentRound?.throws[slot - 1]) }}
            </div>
          </div>
          <div class="round-total">小計: {{ currentRound?.roundScore ?? 0 }} 点</div>
        </div>

        <div class="controls">
          <div class="multipliers">
            <button
              v-for="option in multiplierOptions"
              :key="option.value"
              type="button"
              :class="['pill', { active: selectedMultiplier === option.value } ]"
              :disabled="!isPlaying"
              @click="selectMultiplier(option.value)"
            >
              {{ option.label }}
            </button>
          </div>

          <div class="numbers">
            <button
              v-for="number in numberButtons"
              :key="number"
              type="button"
              :disabled="!isPlaying || (currentRound?.throws.length ?? 0) >= THROWS_PER_ROUND"
              @click="handleNumberInput(number)"
            >
              {{ number }}
            </button>
          </div>

          <div class="bull-row">
            <button
              v-for="option in bullOptions"
              :key="option.value"
              type="button"
              class="outline"
              :disabled="!isPlaying || (currentRound?.throws.length ?? 0) >= THROWS_PER_ROUND"
              @click="handleBullInput(option.value)"
            >
              {{ option.label }}<span class="sub">{{ option.description }}</span>
            </button>
          </div>

          <button class="secondary" type="button" @click="undoLastThrow">UNDO（1 投戻す）</button>
        </div>
      </section>

      <section v-else class="card result">
        <h2>結果</h2>
        <ul class="result-list">
          <li v-for="player in players" :key="player.id">
            <span class="name">{{ player.name }}</span>
            <span class="value">{{ player.totalScore }} 点</span>
          </li>
        </ul>
        <div class="result-actions">
          <button class="primary" type="button" @click="resetScoresForReplay">もう一度プレイ</button>
          <button class="secondary" type="button" @click="resetToNewGame">新しいゲーム</button>
        </div>
      </section>
    </main>
  </div>
</template>

<style scoped>
.app-shell {
  min-height: 100vh;
  background: linear-gradient(180deg, #0d1117 0%, #121824 60%, #0d1117 100%);
  color: #f7f7f7;
  padding: 16px;
  max-width: 640px;
  margin: 0 auto;
  font-family: 'Noto Sans JP', system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
}

.app-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 12px;
}

.app-subtitle {
  color: #c3c8d4;
  font-size: 12px;
  letter-spacing: 0.04em;
}

h1 {
  font-size: 28px;
  font-weight: 800;
  margin: 2px 0 0;
}

.status-chip {
  text-transform: uppercase;
  padding: 6px 10px;
  border-radius: 999px;
  font-weight: 700;
  font-size: 12px;
  border: 1px solid rgba(255, 255, 255, 0.2);
}

.status-chip.setting {
  background: #14213d;
}

.status-chip.playing {
  background: #1e854a;
}

.status-chip.finished {
  background: #6a1b9a;
}

main {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.card {
  background: rgba(255, 255, 255, 0.04);
  border: 1px solid rgba(255, 255, 255, 0.08);
  border-radius: 16px;
  padding: 16px;
  box-shadow: 0 12px 30px rgba(0, 0, 0, 0.28);
}

h2 {
  font-size: 20px;
  margin-bottom: 12px;
}

.player-count {
  display: flex;
  flex-direction: column;
  gap: 8px;
  margin-bottom: 12px;
}

.count-buttons {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 8px;
}

.pill {
  background: rgba(255, 255, 255, 0.06);
  border: 1px solid rgba(255, 255, 255, 0.14);
  border-radius: 999px;
  padding: 10px 12px;
  color: #f7f7f7;
  font-weight: 700;
  letter-spacing: 0.02em;
}

.pill.active {
  background: #f08a24;
  border-color: #f08a24;
  color: #0d1117;
}

.player-inputs {
  display: flex;
  flex-direction: column;
  gap: 10px;
  margin: 14px 0;
}

.input-row {
  display: flex;
  flex-direction: column;
  gap: 6px;
  font-size: 14px;
}

.input-row input {
  padding: 12px;
  border-radius: 10px;
  border: 1px solid rgba(255, 255, 255, 0.16);
  background: rgba(255, 255, 255, 0.03);
  color: #f7f7f7;
}

button {
  cursor: pointer;
  border: none;
  border-radius: 12px;
  padding: 12px;
  font-weight: 700;
  font-size: 16px;
  background: #1d2938;
  color: #f7f7f7;
  transition: transform 0.08s ease, opacity 0.2s ease;
}

button:active {
  transform: translateY(1px);
}

button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

button.primary {
  background: linear-gradient(135deg, #f08a24, #ffb347);
  color: #0d1117;
}

button.secondary {
  background: rgba(255, 255, 255, 0.06);
  border: 1px solid rgba(255, 255, 255, 0.16);
}

button.outline {
  background: rgba(255, 255, 255, 0.02);
  border: 1px solid rgba(255, 255, 255, 0.24);
  display: flex;
  flex-direction: column;
  align-items: center;
}

.scoreboard {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(120px, 1fr));
  gap: 8px;
  margin-bottom: 12px;
}

.player-score {
  background: rgba(255, 255, 255, 0.06);
  border-radius: 12px;
  padding: 10px;
  border: 1px solid rgba(255, 255, 255, 0.1);
}

.player-name {
  font-size: 14px;
  color: #c3c8d4;
}

.player-name.active {
  color: #ffe7b3;
}

.score {
  font-size: 22px;
  font-weight: 800;
}

.round-info {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin: 8px 0 14px;
  color: #c3c8d4;
}

.current-round {
  background: rgba(255, 255, 255, 0.05);
  border-radius: 12px;
  padding: 12px;
  border: 1px solid rgba(255, 255, 255, 0.08);
  margin-bottom: 14px;
}

.current-round h3 {
  margin-bottom: 8px;
}

.throws {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 8px;
  margin-bottom: 8px;
}

.throw-slot {
  background: rgba(255, 255, 255, 0.06);
  border-radius: 10px;
  padding: 10px;
  text-align: center;
  font-weight: 700;
}

.round-total {
  text-align: right;
  font-size: 14px;
  color: #ffe7b3;
}

.controls {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.multipliers {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 8px;
}

.numbers {
  display: grid;
  grid-template-columns: repeat(5, minmax(0, 1fr));
  gap: 8px;
}

.numbers button {
  padding: 14px 0;
  font-weight: 800;
  background: rgba(255, 255, 255, 0.08);
  border: 1px solid rgba(255, 255, 255, 0.1);
}

.bull-row {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 8px;
}

.bull-row .sub {
  display: block;
  font-size: 11px;
  color: #c3c8d4;
  margin-top: 2px;
}

.result-list {
  list-style: none;
  padding: 0;
  margin: 0 0 12px;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.result-list li {
  display: flex;
  justify-content: space-between;
  padding: 12px;
  border-radius: 12px;
  background: rgba(255, 255, 255, 0.06);
  border: 1px solid rgba(255, 255, 255, 0.1);
}

.result-actions {
  display: flex;
  gap: 8px;
}

.result-actions button {
  flex: 1;
}

@media (min-width: 720px) {
  .app-shell {
    padding: 24px;
  }

  .numbers {
    grid-template-columns: repeat(5, 80px);
    justify-content: center;
  }
}
</style>
