<script setup>
import { computed, reactive, ref } from 'vue'

const MAX_ROUNDS = 8
const THROWS_PER_ROUND = 3
const MULTIPLIERS = ['S', 'D', 'T']

const playerCount = ref(2)
const playerInputs = reactive([
  { name: 'PLAYER1' },
  { name: 'PLAYER2' },
  { name: 'PLAYER3' },
  { name: 'PLAYER4' },
])

const game = reactive({
  status: 'setting',
  players: [],
  currentRoundIndex: 0,
  currentPlayerIndex: 0,
  maxRounds: MAX_ROUNDS,
})

const selectedMultiplier = ref('S')
const throwHistory = ref([])

const numberButtons = computed(() => Array.from({ length: 20 }, (_, i) => i + 1))

const currentPlayer = computed(() => game.players[game.currentPlayerIndex])
const currentRound = computed(() => currentPlayer.value?.rounds[game.currentRoundIndex])
const isRoundComplete = computed(() => (currentRound.value?.throws.length || 0) >= THROWS_PER_ROUND)
const hasHistory = computed(() => throwHistory.value.length > 0)

function createEmptyRounds() {
  return Array.from({ length: MAX_ROUNDS }, (_, i) => ({
    roundIndex: i + 1,
    throws: [],
    roundScore: 0,
  }))
}

function sanitizeName(name, index) {
  const trimmed = name?.trim()
  return trimmed ? trimmed.slice(0, 20) : `PLAYER${index + 1}`
}

function startGame() {
  game.players = Array.from({ length: playerCount.value }, (_, i) => ({
    id: `player-${i + 1}`,
    name: sanitizeName(playerInputs[i].name, i),
    rounds: createEmptyRounds(),
    totalScore: 0,
  }))

  game.status = 'playing'
  game.currentPlayerIndex = 0
  game.currentRoundIndex = 0
  selectedMultiplier.value = 'S'
  throwHistory.value = []
}

function replayWithSamePlayers() {
  game.players = game.players.map((player, index) => ({
    ...player,
    name: sanitizeName(playerInputs[index]?.name ?? player.name, index),
    rounds: createEmptyRounds(),
    totalScore: 0,
  }))

  game.status = 'playing'
  game.currentPlayerIndex = 0
  game.currentRoundIndex = 0
  selectedMultiplier.value = 'S'
  throwHistory.value = []
}

function resetToSettings() {
  game.status = 'setting'
  game.players = []
  game.currentPlayerIndex = 0
  game.currentRoundIndex = 0
  selectedMultiplier.value = 'S'
  throwHistory.value = []
}

function calculateScore(type, number) {
  if (type === 'MISS') return 0
  if (type === 'OB') return 25
  if (type === 'IB') return 50
  const base = Number(number) || 0
  if (type === 'D') return base * 2
  if (type === 'T') return base * 3
  return base
}

function updateScores(player) {
  player.totalScore = player.rounds.reduce((sum, round) => sum + round.roundScore, 0)
}

function addThrow(type, number = null) {
  if (game.status !== 'playing' || !currentRound.value) return
  if (currentRound.value.throws.length >= THROWS_PER_ROUND) return

  const score = calculateScore(type, number)
  const dartThrow = {
    type,
    number: number ?? null,
    score,
  }

  currentRound.value.throws.push(dartThrow)
  currentRound.value.roundScore = currentRound.value.throws.reduce((sum, dart) => sum + dart.score, 0)
  updateScores(currentPlayer.value)
  throwHistory.value.push({
    playerIndex: game.currentPlayerIndex,
    roundIndex: game.currentRoundIndex,
  })

  if (currentRound.value.throws.length >= THROWS_PER_ROUND) {
    advanceTurn()
  }
}

function advanceTurn() {
  if (game.currentPlayerIndex < game.players.length - 1) {
    game.currentPlayerIndex += 1
    return
  }

  game.currentPlayerIndex = 0
  game.currentRoundIndex += 1

  if (game.currentRoundIndex >= game.maxRounds) {
    game.status = 'finished'
    game.currentRoundIndex = game.maxRounds - 1
    game.currentPlayerIndex = game.players.length - 1
  }
}

function handleNumberPress(number) {
  addThrow(selectedMultiplier.value, number)
}

function selectMultiplier(multiplier) {
  selectedMultiplier.value = multiplier
}

function handleBull(type) {
  addThrow(type, null)
}

function undoLastThrow() {
  if (!hasHistory.value) return
  const lastAction = throwHistory.value.pop()
  const player = game.players[lastAction.playerIndex]
  const round = player.rounds[lastAction.roundIndex]
  round.throws.pop()
  round.roundScore = round.throws.reduce((sum, dart) => sum + dart.score, 0)
  updateScores(player)

  game.currentPlayerIndex = lastAction.playerIndex
  game.currentRoundIndex = lastAction.roundIndex
  game.status = 'playing'
}

const finishedPlayers = computed(() =>
  game.players.map((player) => ({ ...player })).sort((a, b) => a.id.localeCompare(b.id))
)

function displayThrowText(dartThrow) {
  if (!dartThrow) return '-'
  if (dartThrow.type === 'OB' || dartThrow.type === 'IB' || dartThrow.type === 'MISS') return dartThrow.type
  return `${dartThrow.type}${dartThrow.number}`
}
</script>

<template>
  <div class="app-shell">
    <header class="app-header">
      <div>
        <p class="eyebrow">v0.1 / COUNT-UP ONLY</p>
        <h1>MyDarts</h1>
        <p class="subtitle">カウントアップ専用のダーツ得点計算アプリ</p>
      </div>
      <div class="status-chip" :data-status="game.status">
        {{ game.status === 'setting' ? 'SETTING' : game.status === 'playing' ? 'PLAYING' : 'FINISHED' }}
      </div>
    </header>

    <section v-if="game.status === 'setting'" class="card">
      <h2>プレイヤー設定</h2>
      <p class="helper">1〜4人。未入力は自動で PLAYER 名が入ります。</p>

      <div class="form-group">
        <label for="player-count">プレイヤー数</label>
        <select id="player-count" v-model.number="playerCount">
          <option v-for="count in [1, 2, 3, 4]" :key="count" :value="count">
            {{ count }}人
          </option>
        </select>
      </div>

      <div class="player-inputs">
        <label v-for="(_, index) in playerCount" :key="`player-${index}`" class="player-input">
          <span>PLAYER {{ index + 1 }}</span>
          <input v-model="playerInputs[index].name" :placeholder="`PLAYER${index + 1}`" maxlength="20" />
        </label>
      </div>

      <button class="primary" type="button" @click="startGame">ゲーム開始</button>
    </section>

    <section v-else-if="game.status === 'playing'" class="grid">
      <div class="card scoreboard">
        <div class="scoreboard-header">
          <h2>スコアボード</h2>
          <span class="round-tag">Round {{ game.currentRoundIndex + 1 }} / {{ game.maxRounds }}</span>
        </div>
        <div class="players">
          <div
            v-for="(player, index) in game.players"
            :key="player.id"
            class="player-tile"
            :class="{ active: index === game.currentPlayerIndex }"
          >
            <div class="player-name">{{ player.name }}</div>
            <div class="player-score">{{ player.totalScore }}</div>
          </div>
        </div>
      </div>

      <div class="card detail">
        <div class="detail-header">
          <div>
            <p class="label">現在のプレイヤー</p>
            <h3>{{ currentPlayer?.name }}</h3>
          </div>
          <div class="badge">Round {{ game.currentRoundIndex + 1 }}</div>
        </div>

        <div class="throws">
          <div v-for="i in THROWS_PER_ROUND" :key="`throw-${i}`" class="throw-row">
            <div class="throw-index">{{ i }}投目</div>
            <div class="throw-result">
              <span class="throw-text">
                {{ displayThrowText(currentRound?.throws[i - 1]) }}
              </span>
              <span class="throw-score">
                {{ currentRound?.throws[i - 1]?.score ?? '-' }}
              </span>
            </div>
          </div>
        </div>

        <div class="round-total">
          <span>ラウンド小計</span>
          <strong>{{ currentRound?.roundScore ?? 0 }}</strong>
        </div>
      </div>

      <div class="card controls">
        <div class="controls-top">
          <div class="multiplier">
            <span class="label">種別</span>
            <div class="pill-group">
              <button
                v-for="multiplier in MULTIPLIERS"
                :key="multiplier"
                type="button"
                class="pill"
                :class="{ selected: selectedMultiplier === multiplier }"
                @click="selectMultiplier(multiplier)
                "
              >
                {{ multiplier }}
              </button>
            </div>
          </div>
          <button class="ghost" type="button" :disabled="!hasHistory" @click="undoLastThrow">UNDO</button>
        </div>

        <div class="number-grid">
          <button
            v-for="number in numberButtons"
            :key="`num-${number}`"
            type="button"
            class="number"
            :disabled="isRoundComplete"
            @click="handleNumberPress(number)
            "
          >
            {{ number }}
          </button>
        </div>

        <div class="special-actions">
          <button type="button" class="secondary" :disabled="isRoundComplete" @click="handleBull('OB')">OB</button>
          <button type="button" class="secondary" :disabled="isRoundComplete" @click="handleBull('IB')">IB</button>
          <button type="button" class="danger" :disabled="isRoundComplete" @click="addThrow('MISS')">MISS</button>
        </div>

        <p class="helper">1ラウンド3投。入力後は自動で次へ進みます。</p>
      </div>
    </section>

    <section v-else class="card results">
      <div class="results-header">
        <div>
          <p class="label">ゲーム終了</p>
          <h2>結果一覧</h2>
        </div>
        <div class="round-tag">8 ROUNDS</div>
      </div>

      <div class="result-list">
        <div v-for="player in finishedPlayers" :key="player.id" class="result-row">
          <span class="result-name">{{ player.name }}</span>
          <span class="result-score">{{ player.totalScore }}</span>
        </div>
      </div>

      <div class="result-actions">
        <button class="primary" type="button" @click="replayWithSamePlayers">もう一度プレイ</button>
        <button class="ghost" type="button" @click="resetToSettings">新しいゲーム</button>
      </div>
    </section>
  </div>
</template>

<style scoped>
.app-shell {
  max-width: 960px;
  margin: 0 auto;
  padding: 2rem 1.25rem 3rem;
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.app-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  gap: 1rem;
}

.eyebrow {
  font-size: 0.8rem;
  letter-spacing: 0.1em;
  color: #7a7a7a;
}

h1 {
  font-size: 2.25rem;
  margin-top: 0.1rem;
  color: #0f172a;
}

.subtitle {
  color: #4a5568;
}

.status-chip {
  align-self: center;
  padding: 0.4rem 0.9rem;
  border-radius: 999px;
  font-weight: 700;
  font-size: 0.9rem;
  color: #1e293b;
  background: #e2e8f0;
}

.card {
  background: #ffffff;
  border: 1px solid #e5e7eb;
  border-radius: 16px;
  padding: 1.25rem;
  box-shadow: 0 12px 35px rgba(0, 0, 0, 0.06);
}

.grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: 1rem;
}

.scoreboard-header,
.detail-header,
.results-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 0.5rem;
  margin-bottom: 0.75rem;
}

.round-tag {
  background: #0ea5e9;
  color: white;
  border-radius: 999px;
  padding: 0.35rem 0.85rem;
  font-size: 0.9rem;
  font-weight: 700;
}

.players {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
  gap: 0.75rem;
}

.player-tile {
  border: 1px solid #e2e8f0;
  border-radius: 14px;
  padding: 0.75rem 0.9rem;
  display: flex;
  justify-content: space-between;
  align-items: center;
  transition: border-color 0.2s, box-shadow 0.2s;
}

.player-tile.active {
  border-color: #0ea5e9;
  box-shadow: 0 8px 20px rgba(14, 165, 233, 0.15);
}

.player-name {
  font-weight: 600;
  color: #111827;
}

.player-score {
  font-weight: 800;
  color: #0f172a;
  font-size: 1.2rem;
}

.label {
  color: #6b7280;
  font-size: 0.85rem;
}

.badge {
  background: #111827;
  color: white;
  padding: 0.4rem 0.75rem;
  border-radius: 12px;
  font-weight: 700;
}

.throws {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  margin-top: 0.75rem;
}

.throw-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  background: #f8fafc;
  border-radius: 12px;
  padding: 0.6rem 0.8rem;
  border: 1px solid #e5e7eb;
}

.throw-index {
  color: #475569;
  font-weight: 600;
}

.throw-result {
  display: flex;
  align-items: baseline;
  gap: 0.75rem;
  font-weight: 700;
}

.throw-text {
  color: #0f172a;
  min-width: 56px;
  display: inline-block;
}

.throw-score {
  color: #0ea5e9;
  font-size: 1.1rem;
  min-width: 32px;
  text-align: right;
}

.round-total {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 1rem;
  padding-top: 0.75rem;
  border-top: 1px dashed #e2e8f0;
  font-weight: 700;
}

.controls {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.controls-top {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 0.75rem;
}

.multiplier {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.pill-group {
  display: inline-flex;
  background: #e2e8f0;
  border-radius: 999px;
  padding: 0.2rem;
  gap: 0.25rem;
}

.pill {
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 999px;
  background: transparent;
  font-weight: 700;
  color: #111827;
  cursor: pointer;
  transition: background 0.2s, color 0.2s;
}

.pill.selected {
  background: #0ea5e9;
  color: white;
}

.number-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(60px, 1fr));
  gap: 0.45rem;
}

.number {
  background: #0f172a;
  color: white;
  border: none;
  padding: 0.75rem 0.5rem;
  border-radius: 12px;
  font-weight: 700;
  font-size: 1.1rem;
  cursor: pointer;
  transition: transform 0.05s ease, box-shadow 0.1s ease;
}

.number:active {
  transform: translateY(1px);
}

.special-actions {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 0.5rem;
}

button {
  font: inherit;
}

.primary,
.secondary,
.danger,
.ghost {
  border: none;
  padding: 0.9rem 1rem;
  border-radius: 12px;
  font-weight: 800;
  cursor: pointer;
  transition: transform 0.05s ease, box-shadow 0.1s ease, background 0.2s;
}

.primary {
  background: linear-gradient(135deg, #0ea5e9, #0ea5e9);
  color: white;
  width: 100%;
  box-shadow: 0 10px 20px rgba(14, 165, 233, 0.25);
}

.secondary {
  background: #e2e8f0;
  color: #0f172a;
}

.danger {
  background: #f87171;
  color: white;
}

.ghost {
  background: transparent;
  color: #111827;
  border: 1px solid #e5e7eb;
}

button:disabled {
  opacity: 0.4;
  cursor: not-allowed;
  box-shadow: none;
}

.number:disabled {
  background: #cbd5e1;
  color: #475569;
}

button:not(:disabled):active,
.number:not(:disabled):active {
  transform: translateY(1px);
}

.helper {
  color: #6b7280;
  font-size: 0.9rem;
  margin-top: 0.5rem;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
  margin: 1rem 0;
}

.form-group select,
.player-input input {
  padding: 0.7rem 0.85rem;
  border-radius: 12px;
  border: 1px solid #e5e7eb;
  font-weight: 600;
}

.player-inputs {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
  gap: 0.75rem;
  margin-bottom: 1rem;
}

.player-input {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
  font-weight: 700;
}

.results {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.result-list {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.result-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.8rem 1rem;
  border: 1px solid #e5e7eb;
  border-radius: 12px;
  background: #f8fafc;
}

.result-name {
  font-weight: 700;
}

.result-score {
  font-weight: 800;
  color: #0f172a;
  font-size: 1.1rem;
}

.result-actions {
  display: flex;
  flex-wrap: wrap;
  gap: 0.75rem;
}

@media (min-width: 768px) {
  .app-shell {
    padding: 3rem 1.5rem 3.5rem;
  }

  .controls-top {
    align-items: center;
  }
}
</style>
