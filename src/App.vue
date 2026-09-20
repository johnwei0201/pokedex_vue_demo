<script setup>
import { computed, onMounted, ref } from 'vue'
import RadarChart from './components/RadarChart.vue'

const STAT_LABELS = {
  hp: '體力',
  attack: '攻擊',
  defense: '防禦',
  'special-attack': '特攻',
  'special-defense': '特防',
  speed: '速度',
}

const TYPE_COLORS = {
  normal: '#A8A77A',
  fire: '#EE8130',
  water: '#6390F0',
  electric: '#F4C430',
  grass: '#7AC74C',
  ice: '#96D9D6',
  fighting: '#C22E28',
  poison: '#A33EA1',
  ground: '#E2BF65',
  flying: '#A98FF3',
  psychic: '#F95587',
  bug: '#A6B91A',
  rock: '#B6A136',
  ghost: '#735797',
  dragon: '#6F35FC',
  dark: '#705746',
  steel: '#B7B7CE',
  fairy: '#D685AD',
}

const TYPE_LABELS = {
  normal: '一般',
  fighting: '格鬥',
  flying: '飛行',
  poison: '毒',
  ground: '地面',
  rock: '岩石',
  bug: '蟲',
  ghost: '幽靈',
  steel: '鋼',
  fire: '火',
  water: '水',
  grass: '草',
  electric: '雷',
  psychic: '超能力',
  ice: '冰',
  dragon: '龍',
  dark: '惡',
  fairy: '妖精',
}

const HIDDEN_TYPES = new Set(['stellar', 'unknown', 'shadow'])
const NAME_CACHE_KEY = 'pokedex-zh-names-v2'
const typePokemonCache = {}
const zhNameCache = loadNameCache()

const types = ref([])
const selectedType = ref('')
const selectedName = ref('')
const pokemonOptions = ref([])
const listLoading = ref(false)
const pokemon = ref(null)
const loading = ref(false)
const error = ref('')
const pageIndex = ref(0)
const typeHintOpen = ref(false)

let swipeStartX = null
let skipSilhouetteUntil = 0
let skipHistoryPush = false
let skipDexHistoryPush = false
let dexInsertFront = false
const mysteryHistory = []
const dexHistory = []
const dexIndex = ref(-1)

const isMystery = computed(() => !!(pokemon.value && pageIndex.value === 0))

const typeSelectValue = computed({
  get() {
    if (isMystery.value && !typeHintOpen.value) return ''
    return selectedType.value
  },
  set(value) {
    selectedType.value = value
    if (isMystery.value) typeHintOpen.value = true
  },
})

const typeChips = computed(() => {
  const list = pokemonOptions.value
  if (!list.length) return []

  const chips = []
  const current = list.find((item) => item.name === selectedName.value)
  if (current) chips.push(current)

  for (const item of list) {
    if (chips.length >= 5) break
    if (!chips.some((chip) => chip.name === item.name)) {
      chips.push(item)
    }
  }

  return chips
})

function loadNameCache() {
  try {
    return JSON.parse(sessionStorage.getItem(NAME_CACHE_KEY) || '{}')
  } catch {
    return {}
  }
}

function saveNameCache() {
  sessionStorage.setItem(NAME_CACHE_KEY, JSON.stringify(zhNameCache))
}

function capitalize(name) {
  return name.charAt(0).toUpperCase() + name.slice(1)
}

function displayName(name) {
  return name.split('-').map(capitalize).join(' ')
}

function barWidth(value) {
  return `${Math.min(value, 100)}%`
}

function pokemonIdFromUrl(url) {
  const parts = url.replace(/\/$/, '').split('/')
  return Number(parts[parts.length - 1])
}

async function getZhName(id, fallback) {
  const key = String(id)
  if (zhNameCache[key]) return zhNameCache[key]

  try {
    const res = await fetch(`https://pokeapi.co/api/v2/pokemon-species/${id}`)
    if (!res.ok) throw new Error('species failed')
    const data = await res.json()
    const zh =
      data.names.find((item) => item.language.name === 'zh-hant') ||
      data.names.find((item) => item.language.name === 'zh-hans')
    const name = zh?.name
    if (!name) return fallback
    zhNameCache[key] = name
    return name
  } catch {
    return fallback
  }
}

async function mapInChunks(items, size, mapper) {
  const result = []
  for (let i = 0; i < items.length; i += size) {
    const chunk = items.slice(i, i + size)
    result.push(...(await Promise.all(chunk.map(mapper))))
  }
  return result
}

async function loadTypes() {
  const res = await fetch('https://pokeapi.co/api/v2/type?limit=30')
  if (!res.ok) throw new Error('types failed')
  const data = await res.json()
  types.value = data.results
    .filter((item) => !HIDDEN_TYPES.has(item.name))
    .map((item) => ({
      name: item.name,
      label: TYPE_LABELS[item.name] || displayName(item.name),
    }))
}

async function loadPokemonByType(typeName) {
  if (!typeName) {
    pokemonOptions.value = []
    return
  }

  if (typePokemonCache[typeName]) {
    pokemonOptions.value = typePokemonCache[typeName]
    return
  }

  listLoading.value = true
  try {
    const res = await fetch(`https://pokeapi.co/api/v2/type/${typeName}`)
    if (!res.ok) throw new Error('type list failed')
    const data = await res.json()
    const seen = new Set()
    const rawList = data.pokemon
      .map((item) => ({
        id: pokemonIdFromUrl(item.pokemon.url),
        name: item.pokemon.name,
      }))
      .filter((item) => item.id < 10000 && !seen.has(item.name) && seen.add(item.name))
      .sort((a, b) => a.id - b.id)

    const list = await mapInChunks(rawList, 12, async (item) => ({
      ...item,
      label: await getZhName(item.id, displayName(item.name)),
    }))

    saveNameCache()
    typePokemonCache[typeName] = list
    pokemonOptions.value = list
  } catch {
    pokemonOptions.value = []
  } finally {
    listLoading.value = false
  }
}

async function fetchPokemon(nameOrId, { mystery = false } = {}) {
  const q = String(nameOrId).trim().toLowerCase()
  if (!q) return

  loading.value = true
  error.value = ''
  pageIndex.value = mystery ? 0 : 1

  try {
    const res = await fetch(`https://pokeapi.co/api/v2/pokemon/${q}`)
    if (!res.ok) {
      throw new Error('not found')
    }

    const data = await res.json()
    const speciesId = pokemonIdFromUrl(data.species.url)
    const zhName = await getZhName(speciesId, displayName(data.name))
    saveNameCache()

    pokemon.value = {
      id: data.id,
      name: zhName,
      apiName: data.name,
      image:
        data.sprites.other?.['official-artwork']?.front_default ||
        data.sprites.front_default,
      typeKeys: data.types.map((item) => item.type.name),
      types: data.types.map((item) => ({
        name: TYPE_LABELS[item.type.name] || capitalize(item.type.name),
        color: TYPE_COLORS[item.type.name] || '#888',
      })),
      stats: data.stats.map((item) => ({
        key: item.stat.name,
        label: STAT_LABELS[item.stat.name] || item.stat.name,
        value: item.base_stat,
      })),
    }

    if (mystery) {
      const typeKeys = data.types.map((item) => item.type.name)
      const primaryType = typeKeys[0]
      if (primaryType && !typeKeys.includes(selectedType.value)) {
        selectedType.value = primaryType
      }
      if (primaryType) typeHintOpen.value = true
    }
  } catch {
    pokemon.value = null
    error.value = '找不到這隻寶可夢，請再選一次'
  } finally {
    loading.value = false
    if (!mystery) rememberDexPokemon()
  }
}

async function onTypeChange() {
  selectedName.value = ''
  await loadPokemonByType(selectedType.value)
  if (isMystery.value && selectedType.value) {
    await pickMysteryFromCurrentType()
  }
}

function onNameChange() {
  if (selectedName.value) {
    fetchPokemon(selectedName.value)
  }
}

async function pickMysteryFromCurrentType(mystery = true) {
  if (mystery && !skipHistoryPush && pokemon.value?.apiName) {
    mysteryHistory.push({
      name: pokemon.value.apiName,
      type: selectedType.value,
    })
    if (mysteryHistory.length > 40) mysteryHistory.shift()
  }
  skipHistoryPush = false

  const list = pokemonOptions.value
  if (!list.length) {
    error.value = '這個屬性目前沒有可隨機的角色'
    return
  }

  const pool =
    list.length > 1
      ? list.filter((item) => item.name !== selectedName.value)
      : list
  const pick = pool[Math.floor(Math.random() * pool.length)]
  selectedName.value = pick.name
  await fetchPokemon(pick.name, { mystery })
}

async function onRandom() {
  if (!types.value.length) return

  const typeName = types.value[Math.floor(Math.random() * types.value.length)].name
  selectedType.value = typeName

  await loadPokemonByType(typeName)
  await pickMysteryFromCurrentType(true)
}

async function onRandomAnswer() {
  if (!types.value.length) return

  const typeName = types.value[Math.floor(Math.random() * types.value.length)].name
  selectedType.value = typeName

  await loadPokemonByType(typeName)
  await pickMysteryFromCurrentType(false)
}

function rememberDexPokemon() {
  if (skipDexHistoryPush) {
    skipDexHistoryPush = false
    return
  }

  const name = pokemon.value?.apiName
  if (!name) {
    dexInsertFront = false
    return
  }

  const entry = {
    name,
    type: selectedType.value || pokemon.value.typeKeys?.[0] || '',
  }
  const current = dexHistory[dexIndex.value]
  if (current?.name === entry.name) {
    dexInsertFront = false
    return
  }

  if (dexInsertFront) {
    dexInsertFront = false
    dexHistory.unshift(entry)
    if (dexHistory.length > 40) dexHistory.pop()
    dexIndex.value = 0
    return
  }

  if (dexIndex.value >= 0 && dexIndex.value < dexHistory.length - 1) {
    dexHistory.splice(dexIndex.value + 1)
  }

  dexHistory.push(entry)
  if (dexHistory.length > 40) dexHistory.shift()
  dexIndex.value = dexHistory.length - 1
}

async function showDexEntry(entry) {
  skipDexHistoryPush = true
  selectedType.value = entry.type
  selectedName.value = entry.name
  await loadPokemonByType(entry.type)
  await fetchPokemon(entry.name, { mystery: false })
}

async function onDexNext() {
  rememberDexPokemon()
  if (dexIndex.value < dexHistory.length - 1) {
    dexIndex.value += 1
    await showDexEntry(dexHistory[dexIndex.value])
    return
  }

  await onRandomAnswer()
}

async function onDexPrev() {
  rememberDexPokemon()
  if (dexIndex.value > 0) {
    dexIndex.value -= 1
    await showDexEntry(dexHistory[dexIndex.value])
    return
  }

  dexInsertFront = true
  await onRandomAnswer()
}

function mysteryArt(id, fallback) {
  return id ? `/pokemon/${id}.png` : fallback
}

function onMysteryArtError(event) {
  const fallback = pokemon.value?.image
  if (fallback && event.target.src !== fallback) {
    event.target.src = fallback
  }
}

function goToPage(index) {
  pageIndex.value = index === 1 ? 1 : 0
  if (pageIndex.value === 1) rememberDexPokemon()
}

function onSwipeLeft() {
  if (loading.value || listLoading.value) return
  skipSilhouetteUntil = Date.now() + 700
  if (pageIndex.value === 1) {
    onDexNext()
    return
  }
  onRandom()
}

async function onSwipeRight() {
  if (loading.value || listLoading.value) return
  skipSilhouetteUntil = Date.now() + 700

  if (pageIndex.value === 1) {
    onDexPrev()
    return
  }

  const prev = mysteryHistory.pop()
  if (!prev) {
    onRandom()
    return
  }

  skipHistoryPush = true
  selectedType.value = prev.type
  typeHintOpen.value = true
  selectedName.value = prev.name
  await loadPokemonByType(prev.type)
  await fetchPokemon(prev.name, { mystery: true })
}

function onPointerDown(event) {
  swipeStartX = event.clientX
}

function onPointerUp(event) {
  if (swipeStartX == null) return
  const delta = event.clientX - swipeStartX
  swipeStartX = null
  if (loading.value || listLoading.value) return

  if (delta < -40) {
    onSwipeLeft()
    return
  }

  if (delta > 40) {
    onSwipeRight()
    return
  }

  if (pageIndex.value === 0 && event.target.closest?.('.silhouette-board')) {
    goToPage(1)
  }
}

function onSilhouetteClick(event) {
  event.preventDefault()
  if (Date.now() < skipSilhouetteUntil) return
  goToPage(1)
}

async function onQuickPick(name) {
  selectedName.value = name
  await fetchPokemon(name)
}

onMounted(async () => {
  try {
    await loadTypes()
    selectedType.value = 'electric'
    await loadPokemonByType('electric')
    selectedName.value = 'pikachu'
    await fetchPokemon('pikachu', { mystery: true })
  } catch {
    error.value = '屬性列表載入失敗，請重新整理'
  }
})
</script>

<template>
  <main class="page">
    <h1 class="site-title">
      <svg
        class="site-logo"
        viewBox="0 0 64 64"
        role="img"
        aria-label="精靈球"
      >
        <circle cx="32" cy="32" r="30" fill="#ef3340" />
        <path d="M2 32h60" stroke="#1a1a1a" stroke-width="8" />
        <path d="M2 32a30 30 0 0 1 60 0" fill="#f4f4f4" />
        <circle cx="32" cy="32" r="12" fill="#1a1a1a" />
        <circle cx="32" cy="32" r="7" fill="#f4f4f4" />
        <circle cx="32" cy="32" r="30" fill="none" stroke="#1a1a1a" stroke-width="3" />
      </svg>
      寶可夢圖鑑
    </h1>
    <section class="panel search-panel">
      <form class="search-row" :class="{ 'is-mystery': isMystery }" @submit.prevent="onRandom">
        <template v-if="isMystery">
          <span class="hint-label">提示</span>
          <select
            v-model="typeSelectValue"
            class="search-select"
            aria-label="選擇屬性"
            @change="onTypeChange"
          >
            <option value="" disabled>屬性</option>
            <option v-for="type in types" :key="type.name" :value="type.name">
              {{ type.label }}
            </option>
          </select>
        </template>
        <template v-else>
          <select
            v-model="selectedType"
            class="search-select"
            aria-label="選擇屬性"
            @change="onTypeChange"
          >
            <option value="" disabled>選擇屬性</option>
            <option v-for="type in types" :key="type.name" :value="type.name">
              {{ type.label }}
            </option>
          </select>

          <select
            v-model="selectedName"
            class="search-select"
            aria-label="選擇角色"
            :disabled="!selectedType || listLoading"
            @change="onNameChange"
          >
            <option value="" disabled>
              {{ listLoading ? '載入中...' : '選擇角色' }}
            </option>
            <option
              v-for="poke in pokemonOptions"
              :key="poke.name"
              :value="poke.name"
            >
              {{ poke.label }}
            </option>
          </select>
        </template>

        <button class="search-btn" type="submit" :disabled="listLoading">
          {{ pageIndex === 1 ? '隨機猜' : '隨機' }}
        </button>
      </form>

      <div
        v-if="typeChips.length && pageIndex === 1"
        class="chips"
      >
        <button
          v-for="item in typeChips"
          :key="item.name"
          class="chip"
          :class="{ 'is-active': item.name === selectedName }"
          type="button"
          @click="onQuickPick(item.name)"
        >
          {{ item.label }}
        </button>
      </div>
    </section>

    <section
      class="panel result-panel"
      :class="{ 'is-mystery': isMystery }"
    >
      <p v-if="loading" class="status">查詢中...</p>
      <p v-else-if="error && !pokemon" class="status error">{{ error }}</p>

      <div
        v-else-if="pokemon"
        class="deck"
        @pointerdown="onPointerDown"
        @pointerup="onPointerUp"
        @pointercancel="swipeStartX = null"
      >
        <div
          class="deck-track"
          :class="{ 'is-dex': pageIndex === 1 }"
        >
          <article class="deck-page pokemon mystery-page">
            <h2 class="mystery-title">
              <span class="mystery-marks">????</span>
              <span class="mystery-who">我是誰</span>
            </h2>
            <div class="silhouette-wrap">
              <button
                class="swipe-arrow"
                type="button"
                aria-label="下一題"
                @pointerdown.stop
                @click.stop="onSwipeLeft"
              >
                <svg viewBox="0 0 24 48" aria-hidden="true">
                  <path
                    d="M16 6L6 24l10 18"
                    fill="none"
                    stroke="currentColor"
                    stroke-width="4"
                    stroke-linecap="round"
                    stroke-linejoin="round"
                  />
                </svg>
              </button>
              <button
                class="silhouette-board"
                type="button"
                aria-label="查看圖鑑"
                @click="onSilhouetteClick"
              >
                <img
                  class="silhouette-frame"
                  src="/pokemon/mystery-frame.png"
                  alt=""
                  draggable="false"
                />
                <img
                  v-if="pokemon.image"
                  class="poke-img is-silhouette"
                  :src="mysteryArt(pokemon.id, pokemon.image)"
                  alt="寶可夢剪影"
                  draggable="false"
                  @error="onMysteryArtError"
                />
              </button>
              <button
                class="swipe-arrow"
                type="button"
                aria-label="上一題"
                @pointerdown.stop
                @click.stop="onSwipeRight"
              >
                <svg viewBox="0 0 24 48" aria-hidden="true">
                  <path
                    d="M8 6l10 18L8 42"
                    fill="none"
                    stroke="currentColor"
                    stroke-width="4"
                    stroke-linecap="round"
                    stroke-linejoin="round"
                  />
                </svg>
              </button>
            </div>
            <p class="reveal-hint">點黑影看答案，往左滑出下一題</p>
          </article>

          <article class="deck-page pokemon">
            <h2 class="poke-name">{{ pokemon.name }}</h2>
            <div class="types">
              <span
                v-for="type in pokemon.types"
                :key="type.name"
                class="type-badge"
                :style="{ backgroundColor: type.color }"
              >
                {{ type.name }}
              </span>
            </div>
            <div class="silhouette-wrap">
              <button
                class="swipe-arrow is-muted"
                type="button"
                aria-label="下一隻"
                @pointerdown.stop
                @click.stop="onSwipeLeft"
              >
                <svg viewBox="0 0 24 48" aria-hidden="true">
                  <path
                    d="M16 6L6 24l10 18"
                    fill="none"
                    stroke="currentColor"
                    stroke-width="4"
                    stroke-linecap="round"
                    stroke-linejoin="round"
                  />
                </svg>
              </button>
              <img
                v-if="pokemon.image"
                class="poke-img"
                :src="pokemon.image"
                :alt="pokemon.name"
                draggable="false"
              />
              <button
                class="swipe-arrow is-muted"
                type="button"
                aria-label="上一隻"
                @pointerdown.stop
                @click.stop="onSwipeRight"
              >
                <svg viewBox="0 0 24 48" aria-hidden="true">
                  <path
                    d="M8 6l10 18L8 42"
                    fill="none"
                    stroke="currentColor"
                    stroke-width="4"
                    stroke-linecap="round"
                    stroke-linejoin="round"
                  />
                </svg>
              </button>
            </div>
            <div class="stats-layout">
              <ul class="stats">
                <li v-for="stat in pokemon.stats" :key="stat.key" class="stat-row">
                  <span class="stat-label">{{ stat.label }}</span>
                  <div class="stat-track">
                    <div class="stat-fill" :style="{ width: barWidth(stat.value) }" />
                  </div>
                  <span class="stat-value">{{ stat.value }}</span>
                </li>
              </ul>
              <RadarChart :stats="pokemon.stats" />
            </div>
            <p class="reveal-hint">點左看下一隻，點右看上一隻</p>
          </article>
        </div>

        <div class="deck-dots">
          <button
            class="deck-dot"
            :class="{ 'is-on': pageIndex === 0 }"
            type="button"
            aria-label="我是誰"
            @click="goToPage(0)"
          />
          <button
            class="deck-dot"
            :class="{ 'is-on': pageIndex === 1 }"
            type="button"
            aria-label="圖鑑"
            @click="goToPage(1)"
          />
        </div>
      </div>
    </section>
  </main>
</template>
