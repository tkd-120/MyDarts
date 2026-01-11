<script setup>
import { computed, reactive, ref } from 'vue'
import DartboardInput from './components/DartboardInput.vue'

const THROWS_PER_ROUND = 3
const throwTypes = ['S', 'D', 'T']
const GAME_MODES = {
  COUNT_UP: 'COUNT_UP',
  ZERO_ONE_301: 'ZERO_ONE_301',
  CRICKET: 'CRICKET',
  CRICKET_PRACTICE: 'CRICKET_PRACTICE'
}
const MODE_MAX_ROUNDS = {
  [GAME_MODES.COUNT_UP]: 8,
  [GAME_MODES.ZERO_ONE_301]: 10,
  [GAME_MODES.CRICKET]: 15,
  [GAME_MODES.CRICKET_PRACTICE]: 1
}
const ZERO_ONE_START_SCORE = 301
const CRICKET_NUMBERS = [15, 16, 17, 18, 19, 20, 'BULL']
const CRICKET_BASE_MARKS = CRICKET_NUMBERS.reduce((acc, key) => {
  acc[key] = 0
  return acc
}, {})
const PRACTICE_TARGETS = [20, 19, 18, 17, 16, 15, 'BULL']
const PRACTICE_MAX_MARKS = 10
const PRACTICE_BASE_MARKS = PRACTICE_TARGETS.reduce((acc, key) => {
  acc[key] = 0
  return acc
}, {})

const playerCount = ref(1)
const playerNames = reactive(['', '', '', ''])
const selectedType = ref('S')
const selectedMode = ref(GAME_MODES.COUNT_UP)
const useDartboardInput = ref(true)

const game = reactive({
  mode: GAME_MODES.COUNT_UP,
  maxRounds: MODE_MAX_ROUNDS[GAME_MODES.COUNT_UP],
  players: [],
  currentRoundIndex: 0,
  currentPlayerIndex: 0,
  status: 'setting',
  practiceCurrentTargetIndex: 0,
  practiceMarks: JSON.parse(JSON.stringify(PRACTICE_BASE_MARKS)),
  practiceThrows: [],
  practiceStartedAt: null,
  practiceEndedAt: null
})

const throwHistory = ref([])

const currentPlayer = computed(() => game.players[game.currentPlayerIndex])
const currentRound = computed(() => currentPlayer.value?.rounds[game.currentRoundIndex])

const hasHistory = computed(() => throwHistory.value.length > 0)
const isPlaying = computed(() => game.status === 'playing')
const isFinished = computed(() => game.status === 'finished')
const isPracticeMode = computed(() => game.mode === GAME_MODES.CRICKET_PRACTICE)

const roundLabel = computed(() => `${game.currentRoundIndex + 1} / ${game.maxRounds}`)

const roundScoreLabel = computed(() => {
  if (game.mode === GAME_MODES.ZERO_ONE_301) return 'このターンの削り'
  if (game.mode === GAME_MODES.CRICKET) return 'このターンの得点'
  return 'ラウンド小計'
})

function displayScore(player) {
  if (game.mode === GAME_MODES.ZERO_ONE_301) return player.remainingScore
  if (game.mode === GAME_MODES.CRICKET) return player.cricketScore
  return player.totalScore
}

function scoreSuffix() {
  if (game.mode === GAME_MODES.ZERO_ONE_301) return '残り'
  if (game.mode === GAME_MODES.CRICKET) return '得点'
  return '合計'
}

const currentPlayerMarks = computed(() => currentPlayer.value?.cricketMarks ?? {})
const practiceTarget = computed(() => PRACTICE_TARGETS[game.practiceCurrentTargetIndex])
const practiceTargetLabel = computed(() => (practiceTarget.value === 'BULL' ? 'BULL' : practiceTarget.value))
const practiceProgress = computed(() => {
  const marks = game.practiceMarks?.[practiceTarget.value] ?? 0
  return `${marks} / ${PRACTICE_MAX_MARKS}`
})
const practiceThrowCount = computed(() => game.practiceThrows?.length ?? 0)

function createEmptyRound(index) {
  return {
    roundIndex: index + 1,
    throws: [],
    roundScore: 0,
    startRemaining: null
  }
}

function recalcTotals() {
  game.players.forEach((player) => {
    if (game.mode === GAME_MODES.COUNT_UP) {
      player.totalScore = player.rounds.reduce((sum, round) => sum + round.roundScore, 0)
    } else if (game.mode === GAME_MODES.ZERO_ONE_301) {
      player.totalScore = ZERO_ONE_START_SCORE - player.remainingScore
    } else {
      player.totalScore = player.cricketScore
    }
  })
}

function resetPracticeState() {
  game.practiceCurrentTargetIndex = 0
  game.practiceMarks = JSON.parse(JSON.stringify(PRACTICE_BASE_MARKS))
  game.practiceThrows = []
  game.practiceStartedAt = Date.now()
  game.practiceEndedAt = null
}

function initializeGame() {
  game.mode = selectedMode.value
  game.maxRounds = MODE_MAX_ROUNDS[game.mode]
  game.players = Array.from({ length: playerCount.value }, (_, index) => ({
    id: `player-${index + 1}`,
    name: playerNames[index]?.trim() || `PLAYER${index + 1}`,
    rounds: Array.from({ length: game.maxRounds }, (_, roundIndex) => createEmptyRound(roundIndex)),
    totalScore: 0,
    remainingScore: game.mode === GAME_MODES.ZERO_ONE_301 ? ZERO_ONE_START_SCORE : 0,
    cricketScore: 0,
    cricketMarks: JSON.parse(JSON.stringify(CRICKET_BASE_MARKS))
  }))
  game.currentRoundIndex = 0
  game.currentPlayerIndex = 0
  game.status = 'playing'
  selectedType.value = 'S'
  useDartboardInput.value = true
  throwHistory.value = []
  if (game.mode === GAME_MODES.CRICKET_PRACTICE) {
    resetPracticeState()
  }
}

function resetScores() {
  game.maxRounds = MODE_MAX_ROUNDS[game.mode]
  game.players.forEach((player) => {
    player.rounds = Array.from({ length: game.maxRounds }, (_, roundIndex) => createEmptyRound(roundIndex))
    player.totalScore = 0
    player.remainingScore = game.mode === GAME_MODES.ZERO_ONE_301 ? ZERO_ONE_START_SCORE : 0
    player.cricketScore = 0
    player.cricketMarks = JSON.parse(JSON.stringify(CRICKET_BASE_MARKS))
  })
  game.currentRoundIndex = 0
  game.currentPlayerIndex = 0
  game.status = 'playing'
  selectedType.value = 'S'
  useDartboardInput.value = true
  throwHistory.value = []
  if (game.mode === GAME_MODES.CRICKET_PRACTICE) {
    resetPracticeState()
  }
}

function startGame() {
  if (playerCount.value < 1 || playerCount.value > 4) return
  if (selectedMode.value === GAME_MODES.CRICKET_PRACTICE) {
    playerCount.value = 1
  }
  initializeGame()
}

function newGame() {
  playerCount.value = 1
  playerNames.splice(0, playerNames.length, '', '', '', '')
  game.players = []
  game.status = 'setting'
  game.currentRoundIndex = 0
  game.currentPlayerIndex = 0
  throwHistory.value = []
  selectedType.value = 'S'
}

function selectMode(mode) {
  selectedMode.value = mode
  if (mode === GAME_MODES.CRICKET_PRACTICE) {
    playerCount.value = 1
  }
}

function computeScore(type, number) {
  if (type === 'OB') return 25
  if (type === 'IB') return 50
  if (type === 'MISS') return 0
  if (!number) return 0
  if (type === 'D') return number * 2
  if (type === 'T') return number * 3
  return number
}

function computeCricketMarks(type) {
  if (type === 'MISS') return 0
  if (type === 'OB') return 1
  if (type === 'IB') return 2
  if (type === 'T') return 3
  if (type === 'D') return 2
  return 1
}

function isCricketTarget(numberOrBull) {
  return CRICKET_NUMBERS.includes(numberOrBull)
}

function computePracticeMarks(type, isBull) {
  if (type === 'MISS') return 0
  if (isBull) {
    if (type === 'OB') return 1
    if (type === 'IB') return 2
    return 0
  }
  if (type === 'T') return 3
  if (type === 'D') return 2
  if (type === 'S') return 1
  return 0
}

function recordPracticeHit({ type, number }) {
  if (!isPlaying.value) return

  pushSnapshot()

  const target = PRACTICE_TARGETS[game.practiceCurrentTargetIndex]
  const isBull = target === 'BULL'
  let normalizedType = type
  let isValid = false

  if (isBull) {
    isValid = ['OB', 'IB'].includes(type)
  } else {
    isValid = ['S', 'D', 'T'].includes(type) && number === target
  }

  if (!isValid) {
    normalizedType = 'MISS'
  }

  const markGain = computePracticeMarks(normalizedType, isBull)
  const beforeMarks = game.practiceMarks[target] ?? 0
  const afterMarks = Math.min(PRACTICE_MAX_MARKS, beforeMarks + markGain)

  game.practiceMarks[target] = afterMarks
  game.practiceThrows.push({ target, type: normalizedType })

  if (beforeMarks < PRACTICE_MAX_MARKS && afterMarks >= PRACTICE_MAX_MARKS) {
    if (game.practiceCurrentTargetIndex < PRACTICE_TARGETS.length - 1) {
      game.practiceCurrentTargetIndex += 1
    } else {
      game.practiceEndedAt = Date.now()
      game.status = 'finished'
    }
  }
}

function recordThrow({ type, number }) {
  if (game.mode === GAME_MODES.CRICKET_PRACTICE) {
    recordPracticeHit({ type, number })
    return
  }

  if (!isPlaying.value) return
  const player = game.players[game.currentPlayerIndex]
  const round = player?.rounds[game.currentRoundIndex]

  if (!player || !round || round.throws.length >= THROWS_PER_ROUND) return

  pushSnapshot()

  if (game.mode === GAME_MODES.ZERO_ONE_301 && round.startRemaining == null) {
    round.startRemaining = player.remainingScore
  }

  const score = computeScore(type, number)

  if (game.mode === GAME_MODES.ZERO_ONE_301) {
    const startRemaining = round.startRemaining ?? player.remainingScore
    const updatedRoundScore = round.roundScore + score
    const remainingAfter = startRemaining - updatedRoundScore

    if (remainingAfter < 0) {
      round.throws = []
      round.roundScore = 0
      player.remainingScore = startRemaining
      recalcTotals()
      advanceTurn()
      return
    }

    round.throws.push({ type, number: number ?? null, score })
    round.roundScore = updatedRoundScore
    player.remainingScore = remainingAfter
    recalcTotals()

    if (remainingAfter === 0) {
      game.status = 'finished'
      return
    }

    if (round.throws.length === THROWS_PER_ROUND) {
      advanceTurn()
    }

    return
  }

  if (game.mode === GAME_MODES.CRICKET) {
    const target = type === 'OB' || type === 'IB' ? 'BULL' : number
    const marks = computeCricketMarks(type)
    const isValidTarget = isCricketTarget(target)
    let scoreGain = 0

    if (isValidTarget && marks > 0) {
      const beforeMarks = player.cricketMarks[target]
      const afterMarks = beforeMarks + marks
      const newOverflow = Math.max(0, afterMarks - 3)
      const previousOverflow = Math.max(0, beforeMarks - 3)
      const overflowFromThisThrow = Math.max(0, newOverflow - previousOverflow)

      const opponentOpenExists = game.players.some(
        (p) => p.id !== player.id && (p.cricketMarks[target] ?? 0) < 3
      )

      if (overflowFromThisThrow > 0 && opponentOpenExists) {
        const baseValue = target === 'BULL' ? 25 : Number(target)
        scoreGain = baseValue * overflowFromThisThrow
      }

      player.cricketMarks[target] = afterMarks
      player.cricketScore += scoreGain
    }

    round.throws.push({ type, number: number ?? null, score: scoreGain })
    round.roundScore += scoreGain
    recalcTotals()

    const allClosed = CRICKET_NUMBERS.every((num) => (player.cricketMarks[num] ?? 0) >= 3)
    const leading = game.players.every(
      (p) => p.id === player.id || player.cricketScore >= (p.cricketScore ?? 0)
    )

    if (allClosed && leading) {
      game.status = 'finished'
      return
    }

    if (round.throws.length === THROWS_PER_ROUND) {
      advanceTurn()
    }

    return
  }

  round.throws.push({ type, number: number ?? null, score })
  round.roundScore = round.throws.reduce((sum, item) => sum + item.score, 0)
  recalcTotals()

  if (round.throws.length === THROWS_PER_ROUND) {
    advanceTurn()
  }
}

function advanceTurn() {
  if (game.currentRoundIndex === game.maxRounds - 1 && game.currentPlayerIndex === game.players.length - 1) {
    game.status = 'finished'
    return
  }

  if (game.currentPlayerIndex < game.players.length - 1) {
    game.currentPlayerIndex += 1
  } else {
    game.currentPlayerIndex = 0
    game.currentRoundIndex += 1
  }

  selectedType.value = 'S'
}

function handleNumberClick(number) {
  if (!throwTypes.includes(selectedType.value)) return
  recordThrow({ type: selectedType.value, number })
}

function handleBull(type) {
  recordThrow({ type, number: null })
}

function handleMiss() {
  recordThrow({ type: 'MISS', number: null })
}

function handlePracticeThrow(type) {
  const target = PRACTICE_TARGETS[game.practiceCurrentTargetIndex]
  const number = target === 'BULL' ? null : target
  recordPracticeHit({ type, number })
}

function resetPracticeSession() {
  const shouldReset = window.confirm('練習セッションを最初からやり直しますか？')
  if (!shouldReset) return
  resetPracticeState()
  game.status = 'playing'
  throwHistory.value = []
}

function undoThrow() {
  if (!hasHistory.value) return
  const last = throwHistory.value.pop()
  if (!last) return

  Object.assign(game, last.game)
  selectedType.value = last.selectedType
}

function pushSnapshot() {
  throwHistory.value.push({
    game: JSON.parse(JSON.stringify(game)),
    selectedType: selectedType.value
  })
}

const currentRoundDisplayThrows = computed(() => {
  return currentRound.value?.throws ?? []
})

const isInputDisabled = computed(() => game.status !== 'playing' || !game.players.length)

const resultPlayers = computed(() => {
  if (!isFinished.value) return []
  const players = [...game.players]

  if (game.mode === GAME_MODES.COUNT_UP) {
    return players.sort((a, b) => b.totalScore - a.totalScore)
  }

  if (game.mode === GAME_MODES.ZERO_ONE_301) {
    return players.sort((a, b) => a.remainingScore - b.remainingScore)
  }

  const closedPlayers = players.filter((player) =>
    CRICKET_NUMBERS.every((num) => (player.cricketMarks?.[num] ?? 0) >= 3)
  )

  if (closedPlayers.length) {
    return closedPlayers.sort((a, b) => b.cricketScore - a.cricketScore)
  }

  return players.sort((a, b) => b.cricketScore - a.cricketScore)
})

function formatDuration(ms) {
  if (!ms || ms < 0) return '0:00'
  const totalSeconds = Math.floor(ms / 1000)
  const minutes = Math.floor(totalSeconds / 60)
  const seconds = String(totalSeconds % 60).padStart(2, '0')
  return `${minutes}:${seconds}`
}

const practiceElapsed = computed(() => {
  const start = game.practiceStartedAt
  if (!start) return '0:00'
  const end = game.practiceEndedAt ?? Date.now()
  return formatDuration(end - start)
})

const practiceTargetStats = computed(() => {
  const stats = PRACTICE_TARGETS.reduce((acc, target) => {
    acc[target] = {
      S: 0,
      D: 0,
      T: 0,
      OB: 0,
      IB: 0,
      MISS: 0
    }
    return acc
  }, {})

  game.practiceThrows.forEach((entry) => {
    if (!stats[entry.target]) return
    stats[entry.target][entry.type] = (stats[entry.target][entry.type] ?? 0) + 1
  })

  return stats
})
</script>

<template>
  <div class="app-shell">
    <header class="app-header">
      <div>
        <p class="version">v0.1</p>
        <h1>MyDarts</h1>
        <p class="subtitle">カウントアップ / 01 / クリケット対応スコア計算</p>
      </div>
      <div v-if="isPlaying && !isPracticeMode" class="round-info">
        <p class="label">Round</p>
        <p class="value">{{ roundLabel }}</p>
      </div>
    </header>

    <section v-if="game.status === 'setting'" class="panel">
      <h2>プレイヤー設定</h2>
      <div class="form-row">
        <label>モード</label>
        <div class="pill-group">
          <button
            v-for="mode in [
              GAME_MODES.COUNT_UP,
              GAME_MODES.ZERO_ONE_301,
              GAME_MODES.CRICKET,
              GAME_MODES.CRICKET_PRACTICE
            ]"
            :key="mode"
            type="button"
            :class="['pill', { active: selectedMode === mode }]"
            @click="selectMode(mode)"
          >
            {{
              mode === GAME_MODES.COUNT_UP
                ? 'COUNT UP'
                : mode === GAME_MODES.ZERO_ONE_301
                  ? '01 - 301'
                  : mode === GAME_MODES.CRICKET
                    ? 'CRICKET'
                    : 'CRICKET 10 MARKS'
            }}
          </button>
        </div>
      </div>

      <div class="form-row">
        <label>人数</label>
        <div class="pill-group">
          <button
            v-for="count in [1, 2, 3, 4]"
            :key="count"
            type="button"
            :class="['pill', { active: playerCount === count }]"
            :disabled="selectedMode === GAME_MODES.CRICKET_PRACTICE && count !== 1"
            @click="playerCount = count"
          >
            {{ count }}人
          </button>
        </div>
      </div>
      <div class="form-row">
        <label>プレイヤー名</label>
        <div class="player-inputs">
          <div
            v-for="(name, index) in playerNames"
            :key="index"
            class="input-row"
            :class="{ muted: index >= playerCount }"
          >
            <span class="input-label">P{{ index + 1 }}</span>
            <input
              v-model="playerNames[index]"
              :placeholder="`PLAYER${index + 1}`"
              :disabled="index >= playerCount"
              maxlength="20"
            />
          </div>
        </div>
      </div>
      <button class="primary" type="button" @click="startGame">ゲーム開始</button>
    </section>

    <section v-else-if="isPlaying" class="panel">
      <template v-if="game.mode === GAME_MODES.CRICKET_PRACTICE">
        <div class="practice-status">
          <div class="practice-card">
            <p class="label">現在の的</p>
            <p class="value">{{ practiceTargetLabel }}</p>
          </div>
          <div class="practice-card">
            <p class="label">進捗</p>
            <p class="value">{{ practiceProgress }}</p>
          </div>
          <div class="practice-card">
            <p class="label">投数</p>
            <p class="value">{{ practiceThrowCount }}</p>
          </div>
        </div>
        <div class="practice-grid">
          <div v-for="target in PRACTICE_TARGETS" :key="target" class="practice-cell">
            <p class="label">{{ target === 'BULL' ? 'BULL' : target }}</p>
            <p class="value">{{ game.practiceMarks[target] ?? 0 }} / {{ PRACTICE_MAX_MARKS }}</p>
          </div>
        </div>
      </template>
      <template v-else>
        <div class="scoreboard">
          <div
            v-for="(player, index) in game.players"
            :key="player.id"
            :class="['player-card', { active: index === game.currentPlayerIndex } ]"
          >
            <div class="player-header">
              <span class="name">{{ player.name }}</span>
              <span class="score">
                {{ scoreSuffix() }}: {{ displayScore(player) }}
              </span>
            </div>
            <div class="progress">Round {{ game.currentRoundIndex + 1 }} / {{ game.maxRounds }}</div>
          </div>
        </div>

        <div class="round-detail" v-if="currentPlayer">
          <div class="section-header">
            <div>
              <p class="label">手番</p>
              <p class="value">{{ currentPlayer.name }}</p>
            </div>
            <div>
              <p class="label">{{ roundScoreLabel }}</p>
              <p class="value">{{ currentRound?.roundScore ?? 0 }}</p>
            </div>
          </div>
          <div class="throws">
            <div v-for="slot in THROWS_PER_ROUND" :key="slot" class="throw-card">
              <p class="label">{{ slot }}投目</p>
              <p class="value">
                <template v-if="currentRoundDisplayThrows[slot - 1]">
                  {{ currentRoundDisplayThrows[slot - 1].type }}
                  <span v-if="currentRoundDisplayThrows[slot - 1].number">
                    {{ currentRoundDisplayThrows[slot - 1].number }}
                  </span>
                  = {{ currentRoundDisplayThrows[slot - 1].score }}
                </template>
                <template v-else>-</template>
              </p>
            </div>
          </div>
          <div v-if="game.mode === GAME_MODES.CRICKET" class="cricket-grid">
            <div v-for="target in CRICKET_NUMBERS" :key="target" class="cricket-cell">
              <p class="label">{{ target === 'BULL' ? 'BULL' : target }}</p>
              <p class="value">{{ currentPlayerMarks[target] ?? 0 }} マーク</p>
            </div>
          </div>
        </div>
      </template>

      <div class="controls">
        <template v-if="game.mode === GAME_MODES.CRICKET_PRACTICE">
          <div class="control-row control-head">
            <div>
              <p class="label">入力</p>
              <p class="value small">現在の的だけ有効</p>
            </div>
            <div class="button-row">
              <button class="ghost" type="button" :disabled="!hasHistory" @click="undoThrow">UNDO</button>
              <button class="ghost" type="button" @click="resetPracticeSession">RESET</button>
            </div>
          </div>
          <div class="practice-input">
            <template v-if="practiceTarget === 'BULL'">
              <button type="button" class="pill secondary" :disabled="isInputDisabled" @click="handlePracticeThrow('OB')">
                OUTER
              </button>
              <button type="button" class="pill secondary" :disabled="isInputDisabled" @click="handlePracticeThrow('IB')">
                INNER
              </button>
              <button type="button" class="pill danger" :disabled="isInputDisabled" @click="handlePracticeThrow('MISS')">
                MISS
              </button>
            </template>
            <template v-else>
              <button type="button" class="pill secondary" :disabled="isInputDisabled" @click="handlePracticeThrow('S')">
                S
              </button>
              <button type="button" class="pill secondary" :disabled="isInputDisabled" @click="handlePracticeThrow('D')">
                D
              </button>
              <button type="button" class="pill secondary" :disabled="isInputDisabled" @click="handlePracticeThrow('T')">
                T
              </button>
              <button type="button" class="pill danger" :disabled="isInputDisabled" @click="handlePracticeThrow('MISS')">
                MISS
              </button>
            </template>
          </div>
        </template>
        <template v-else>
          <div class="control-row control-head">
            <div>
              <p class="label">入力方式</p>
              <div class="pill-group">
                <button
                  type="button"
                  :class="['pill', { active: !useDartboardInput } ]"
                  @click="useDartboardInput = false"
                >
                  ボタン
                </button>
                <button
                  type="button"
                  :class="['pill', { active: useDartboardInput } ]"
                  @click="useDartboardInput = true"
                >
                  ダーツ盤
                </button>
              </div>
            </div>
            <button class="ghost" type="button" :disabled="!hasHistory" @click="undoThrow">UNDO</button>
          </div>

          <div v-if="useDartboardInput" class="dartboard-block">
            <DartboardInput :disabled="isInputDisabled" @hit="recordThrow" />
            <p class="helper-text">ダーツ盤をタップして入力。外側タップで MISS。</p>
          </div>

          <template v-else>
            <div class="control-row">
              <p class="label">種別</p>
              <div class="pill-group">
                <button
                  v-for="type in throwTypes"
                  :key="type"
                  type="button"
                  :class="['pill', { active: selectedType === type }]"
                  :disabled="isInputDisabled"
                  @click="selectedType = type"
                >
                  {{ type }}
                </button>
              </div>
            </div>

            <div class="control-row">
              <div class="pill-group fill">
                <button type="button" class="pill secondary" :disabled="isInputDisabled" @click="handleBull('OB')">OB</button>
                <button type="button" class="pill secondary" :disabled="isInputDisabled" @click="handleBull('IB')">IB</button>
                <button type="button" class="pill danger" :disabled="isInputDisabled" @click="handleMiss">MISS</button>
              </div>
            </div>

            <div class="numbers-grid">
              <button
                v-for="number in 20"
                :key="number"
                type="button"
                class="number-btn"
                :disabled="isInputDisabled"
                @click="handleNumberClick(number)"
              >
                {{ number }}
              </button>
            </div>
          </template>
        </template>
      </div>
    </section>

    <section v-else-if="isFinished" class="panel">
      <h2>結果</h2>
      <template v-if="game.mode === GAME_MODES.CRICKET_PRACTICE">
        <div class="practice-summary">
          <div class="summary-card">
            <p class="label">合計投数</p>
            <p class="value">{{ practiceThrowCount }}</p>
          </div>
          <div class="summary-card">
            <p class="label">経過時間</p>
            <p class="value">{{ practiceElapsed }}</p>
          </div>
        </div>
        <div class="practice-results">
          <div v-for="target in PRACTICE_TARGETS" :key="target" class="practice-result-card">
            <p class="label">{{ target === 'BULL' ? 'BULL' : target }}</p>
            <div class="result-lines">
              <template v-if="target === 'BULL'">
                <span>OUTER: {{ practiceTargetStats[target].OB }}</span>
                <span>INNER: {{ practiceTargetStats[target].IB }}</span>
                <span>MISS: {{ practiceTargetStats[target].MISS }}</span>
              </template>
              <template v-else>
                <span>S: {{ practiceTargetStats[target].S }}</span>
                <span>D: {{ practiceTargetStats[target].D }}</span>
                <span>T: {{ practiceTargetStats[target].T }}</span>
                <span>MISS: {{ practiceTargetStats[target].MISS }}</span>
              </template>
            </div>
          </div>
        </div>
      </template>
      <ul v-else class="result-list">
        <li v-for="player in resultPlayers" :key="player.id" class="result-item">
          <span class="name">{{ player.name }}</span>
          <span class="score">{{ scoreSuffix() }}: {{ displayScore(player) }}</span>
        </li>
      </ul>
      <div class="actions">
        <button class="ghost" type="button" :disabled="!hasHistory" @click="undoThrow">UNDO</button>
        <button class="primary" type="button" @click="game.mode === GAME_MODES.CRICKET_PRACTICE ? resetPracticeSession() : resetScores()">
          もう一度プレイ
        </button>
        <button class="ghost" type="button" @click="newGame">新しいゲーム</button>
      </div>
    </section>
  </div>
</template>

<style scoped>
.app-shell {
  max-width: 720px;
  margin: 0 auto;
  padding: 16px;
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.app-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.version {
  font-size: 12px;
  color: #6b7280;
}

h1 {
  font-size: 28px;
  font-weight: 700;
  color: #111827;
}

.subtitle {
  color: #4b5563;
}

.round-info {
  text-align: right;
  background: #eef2ff;
  padding: 8px 12px;
  border-radius: 12px;
  min-width: 120px;
}

.round-info .label {
  font-size: 12px;
  color: #4b5563;
}

.round-info .value {
  font-size: 18px;
  font-weight: 700;
  color: #4338ca;
}

.panel {
  background: #ffffff;
  border: 1px solid #e5e7eb;
  border-radius: 16px;
  padding: 16px;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.04);
}

.form-row {
  margin-bottom: 16px;
}

.form-row label {
  display: block;
  font-weight: 600;
  margin-bottom: 8px;
  color: #111827;
}

.player-inputs {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.input-row {
  display: flex;
  align-items: center;
  gap: 8px;
}

.input-row input {
  flex: 1;
  padding: 10px 12px;
  border-radius: 12px;
  border: 1px solid #d1d5db;
  background: #f9fafb;
}

.input-row.muted input {
  opacity: 0.5;
}

.input-label {
  width: 44px;
  font-weight: 600;
}

.primary {
  background: linear-gradient(90deg, #6366f1, #4338ca);
  color: #fff;
  border: none;
  padding: 12px;
  border-radius: 12px;
  width: 100%;
  font-weight: 700;
  cursor: pointer;
}

.primary:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.scoreboard {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(150px, 1fr));
  gap: 12px;
}

.player-card {
  border: 1px solid #e5e7eb;
  border-radius: 12px;
  padding: 12px;
  background: #f9fafb;
}

.player-card.active {
  border-color: #818cf8;
  background: #eef2ff;
  box-shadow: 0 0 0 2px rgba(99, 102, 241, 0.15);
}

.player-header {
  display: flex;
  justify-content: space-between;
  align-items: baseline;
}

.player-header .name {
  font-weight: 700;
}

.player-header .score {
  font-size: 20px;
  font-weight: 800;
  color: #111827;
}

.progress {
  margin-top: 6px;
  font-size: 12px;
  color: #4b5563;
}

.round-detail {
  margin-top: 16px;
  border: 1px solid #e5e7eb;
  border-radius: 12px;
  padding: 12px;
  background: #fff;
}

.section-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 12px;
}

.section-header .label {
  font-size: 12px;
  color: #6b7280;
}

.section-header .value {
  font-size: 20px;
  font-weight: 700;
}

.throws {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(120px, 1fr));
  gap: 8px;
}

.throw-card {
  border: 1px dashed #d1d5db;
  border-radius: 12px;
  padding: 10px;
  background: #f9fafb;
}

.throw-card .label {
  font-size: 12px;
  color: #6b7280;
}

.throw-card .value {
  font-weight: 700;
  color: #111827;
}

.cricket-grid {
  margin-top: 12px;
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(100px, 1fr));
  gap: 8px;
}

.cricket-cell {
  border: 1px solid #e5e7eb;
  border-radius: 10px;
  padding: 8px;
  background: #f3f4f6;
}

.cricket-cell .label {
  font-size: 12px;
  color: #6b7280;
}

.cricket-cell .value {
  font-weight: 700;
  color: #111827;
}

.practice-status {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(140px, 1fr));
  gap: 12px;
}

.practice-card {
  border: 1px solid #e5e7eb;
  border-radius: 12px;
  padding: 12px;
  background: #f9fafb;
}

.practice-card .label {
  font-size: 12px;
  color: #6b7280;
}

.practice-card .value {
  font-size: 20px;
  font-weight: 700;
}

.practice-grid {
  margin-top: 12px;
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(100px, 1fr));
  gap: 8px;
}

.practice-cell {
  border: 1px solid #e5e7eb;
  border-radius: 10px;
  padding: 8px;
  background: #f9fafb;
}

.practice-cell .label {
  font-size: 12px;
  color: #6b7280;
}

.practice-cell .value {
  font-weight: 700;
  color: #111827;
}

.controls {
  margin-top: 16px;
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.control-head {
  justify-content: space-between;
  align-items: center;
}

.control-head .pill-group {
  flex-wrap: wrap;
}

.control-head .value.small {
  font-size: 12px;
  color: #6b7280;
}

.button-row {
  display: flex;
  gap: 8px;
}

.dartboard-block {
  background: #0f172a;
  border-radius: 14px;
  padding: 12px;
  border: 1px solid #1f2937;
}

.helper-text {
  margin-top: 8px;
  color: #e5e7eb;
  font-size: 13px;
  text-align: center;
}

.control-row {
  display: flex;
  align-items: center;
  gap: 12px;
  flex-wrap: wrap;
}

.practice-input {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(120px, 1fr));
  gap: 10px;
}

.pill-group {
  display: flex;
  gap: 8px;
}

.pill-group.fill {
  width: 100%;
}

.pill-group.fill .pill {
  flex: 1;
  text-align: center;
}

.pill {
  border-radius: 999px;
  border: 1px solid #e5e7eb;
  padding: 10px 16px;
  background: #fff;
  cursor: pointer;
  font-weight: 700;
}

.pill.secondary {
  background: #eef2ff;
  border-color: #c7d2fe;
  color: #4338ca;
}

.pill.danger {
  background: #fef2f2;
  border-color: #fecdd3;
  color: #b91c1c;
}

.pill.active {
  background: #4338ca;
  color: #fff;
  border-color: #4338ca;
}

.pill:disabled,
.number-btn:disabled,
.ghost:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.ghost {
  border: 1px solid #d1d5db;
  background: #fff;
  border-radius: 12px;
  padding: 10px 14px;
  font-weight: 700;
  cursor: pointer;
}

.numbers-grid {
  display: grid;
  grid-template-columns: repeat(5, 1fr);
  gap: 8px;
}

.number-btn {
  padding: 14px 0;
  border-radius: 12px;
  border: 1px solid #e5e7eb;
  background: #f9fafb;
  font-weight: 700;
  cursor: pointer;
}

.result-list {
  list-style: none;
  padding: 0;
  margin: 0 0 16px;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.result-item {
  display: flex;
  justify-content: space-between;
  padding: 12px;
  border: 1px solid #e5e7eb;
  border-radius: 12px;
  background: #f9fafb;
}

.practice-summary {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(140px, 1fr));
  gap: 12px;
  margin-bottom: 16px;
}

.summary-card {
  border: 1px solid #e5e7eb;
  border-radius: 12px;
  padding: 12px;
  background: #f9fafb;
}

.summary-card .label {
  font-size: 12px;
  color: #6b7280;
}

.summary-card .value {
  font-size: 20px;
  font-weight: 700;
}

.practice-results {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(160px, 1fr));
  gap: 10px;
  margin-bottom: 16px;
}

.practice-result-card {
  border: 1px solid #e5e7eb;
  border-radius: 12px;
  padding: 12px;
  background: #f9fafb;
}

.practice-result-card .label {
  font-size: 12px;
  color: #6b7280;
  margin-bottom: 8px;
}

.result-lines {
  display: grid;
  gap: 4px;
  font-weight: 600;
  color: #111827;
}

.actions {
  display: flex;
  gap: 12px;
  flex-direction: column;
}

@media (min-width: 640px) {
  .app-shell {
    padding: 24px;
  }

  .actions {
    flex-direction: row;
  }

  .actions button {
    flex: 1;
  }
}
</style>
