<template>
  <div class="game" @keydown="onKeyDown" tabindex="0" ref="gameEl">
    <header class="game__header">
      <h1 class="game__title">2048</h1>
      <div class="game__scores">
        <div class="game__score-box">
          <div class="game__score-box-label">Score</div>
          <div class="game__score-box-value">{{ score }}</div>
        </div>
        <div class="game__score-box">
          <div class="game__score-box-label">Best</div>
          <div class="game__score-box-value">{{ best }}</div>
        </div>
      </div>
    </header>

    <div class="game__actions">
      <button class="game__btn" @click="newGame">New Game</button>
    </div>

    <div
      class="grid"
      @touchstart.prevent="onTouchStart"
      @touchend.prevent="onTouchEnd"
    >
      <div v-for="i in 16" :key="i" class="grid__cell" />

      <div class="tiles">
        <TransitionGroup name="tile">
          <div
            v-for="tile in tiles"
            :key="tile.id"
            class="tile"
            :class="[
              `tile--${tile.value}`,
              tile.value >= 1000 ? 'tile--big' : '',
              tile.isNew ? 'tile--new' : '',
              tile.isMerged ? 'tile--merged' : '',
            ]"
            :style="tileStyle(tile)"
          >
            {{ tile.value }}
          </div>
        </TransitionGroup>
      </div>

      <Transition name="overlay">
        <div v-if="gameOver || won" class="overlay">
          <div class="overlay__title">{{ won ? 'You Win!' : 'Game Over' }}</div>
          <button class="overlay__btn" @click="newGame">Try Again</button>
        </div>
      </Transition>
    </div>

    <p class="hint">Use arrow keys or swipe to move tiles</p>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, nextTick } from 'vue'

const SIZE = 4
let idCounter = 0

const grid = ref([])
const tiles = ref([])
const score = ref(0)
const best = ref(Number(localStorage.getItem('2048-best') || 0))
const gameOver = ref(false)
const won = ref(false)
const gameEl = ref(null)

let touchStartX = 0
let touchStartY = 0

function newGame() {
  grid.value = Array.from({ length: SIZE }, () => Array(SIZE).fill(0))
  tiles.value = []
  score.value = 0
  gameOver.value = false
  won.value = false
  idCounter = 0
  addRandomTile()
  addRandomTile()
  nextTick(() => gameEl.value?.focus())
}

function addRandomTile() {
  const empty = []
  for (let r = 0; r < SIZE; r++)
    for (let c = 0; c < SIZE; c++)
      if (grid.value[r][c] === 0) empty.push([r, c])
  if (!empty.length) return
  const [r, c] = empty[Math.floor(Math.random() * empty.length)]
  const value = Math.random() < 0.9 ? 2 : 4
  grid.value[r][c] = ++idCounter
  tiles.value.push({ id: idCounter, value, row: r, col: c, isNew: true, isMerged: false })
}

function tileStyle(tile) {
  const gap = 10
  const cellSize = `calc((100% - ${gap * (SIZE - 1)}px) / ${SIZE})`
  return {
    width: cellSize,
    height: cellSize,
    left: `calc(${tile.col} * (${cellSize} + ${gap}px))`,
    top: `calc(${tile.row} * (${cellSize} + ${gap}px))`,
    position: 'absolute',
    transition: 'left 0.1s ease, top 0.1s ease',
  }
}

function move(direction) {
  if (gameOver.value || won.value) return

  tiles.value.forEach(t => { t.isNew = false; t.isMerged = false })

  let moved = false
  const newGrid = Array.from({ length: SIZE }, () => Array(SIZE).fill(0))
  const toRemove = []
  let earned = 0

  const getLine = (i, j, dir) => {
    if (dir === 'left')  return { r: i, c: j }
    if (dir === 'right') return { r: i, c: SIZE - 1 - j }
    if (dir === 'up')    return { r: j, c: i }
    if (dir === 'down')  return { r: SIZE - 1 - j, c: i }
  }

  for (let i = 0; i < SIZE; i++) {
    const line = []
    for (let j = 0; j < SIZE; j++) {
      const { r, c } = getLine(i, j, direction)
      const id = grid.value[r][c]
      if (id !== 0) {
        const tile = tiles.value.find(t => t.id === id)
        if (tile) line.push(tile)
      }
    }

    const merged = []
    let k = 0
    while (k < line.length) {
      if (k + 1 < line.length && line[k].value === line[k + 1].value) {
        const newVal = line[k].value * 2
        earned += newVal
        const winner = line[k]
        winner.value = newVal
        winner.isMerged = true
        toRemove.push(line[k + 1].id)
        merged.push(winner)
        k += 2
      } else {
        merged.push(line[k])
        k++
      }
    }

    for (let j = 0; j < SIZE; j++) {
      const { r, c } = getLine(i, j, direction)
      if (j < merged.length) {
        const tile = merged[j]
        if (tile.row !== r || tile.col !== c) moved = true
        tile.row = r
        tile.col = c
        newGrid[r][c] = tile.id
      } else {
        newGrid[r][c] = 0
      }
    }
    if (merged.length !== line.length) moved = true
  }

  if (!moved) return

  tiles.value = tiles.value.filter(t => !toRemove.includes(t.id))
  grid.value = newGrid
  score.value += earned
  if (score.value > best.value) {
    best.value = score.value
    localStorage.setItem('2048-best', best.value)
  }

  if (tiles.value.some(t => t.value === 2048)) {
    won.value = true
    return
  }

  addRandomTile()
  checkGameOver()
}

function checkGameOver() {
  for (let r = 0; r < SIZE; r++) {
    for (let c = 0; c < SIZE; c++) {
      if (grid.value[r][c] === 0) return
      const id = grid.value[r][c]
      const tile = tiles.value.find(t => t.id === id)
      if (!tile) continue
      if (c + 1 < SIZE) {
        const right = tiles.value.find(t => t.id === grid.value[r][c + 1])
        if (right && right.value === tile.value) return
      }
      if (r + 1 < SIZE) {
        const down = tiles.value.find(t => t.id === grid.value[r + 1][c])
        if (down && down.value === tile.value) return
      }
    }
  }
  gameOver.value = true
}

function onKeyDown(e) {
  const map = {
    ArrowLeft: 'left', ArrowRight: 'right',
    ArrowUp: 'up', ArrowDown: 'down',
  }
  if (map[e.key]) {
    e.preventDefault()
    move(map[e.key])
  }
}

function onTouchStart(e) {
  touchStartX = e.touches[0].clientX
  touchStartY = e.touches[0].clientY
}

function onTouchEnd(e) {
  const dx = e.changedTouches[0].clientX - touchStartX
  const dy = e.changedTouches[0].clientY - touchStartY
  if (Math.abs(dx) < 10 && Math.abs(dy) < 10) return
  if (Math.abs(dx) > Math.abs(dy)) {
    move(dx > 0 ? 'right' : 'left')
  } else {
    move(dy > 0 ? 'down' : 'up')
  }
}

onMounted(() => {
  newGame()
  window.addEventListener('keydown', onKeyDown)
})
onUnmounted(() => window.removeEventListener('keydown', onKeyDown))
</script>
