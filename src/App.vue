<script setup>
import { computed, onMounted, ref } from 'vue'
import RadarChart from './components/RadarChart.vue'
import ZH_NAMES from './data/zh-names.json'

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
const typePokemonCache = {}
const flavorCache = {}
const DEX_NOS = Object.keys(ZH_NAMES)
  .map(Number)
  .filter((id) => Number.isFinite(id) && id > 0)
  .sort((a, b) => a - b)

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
let skipDexHistoryPush = false
let dexInsertFront = false
let skipMysteryHistoryPush = false
let mysteryInsertFront = false
const mysteryHistory = []
const mysteryIndex = ref(-1)
const dexHistory = []
const dexIndex = ref(-1)

const isEnglishFlavor = computed(() => {
  const text = pokemon.value?.flavor || ''
  return /[A-Za-z]/.test(text) && !/[\u4e00-\u9fff]/.test(text)
})

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

function capitalize(name) {
  return name.charAt(0).toUpperCase() + name.slice(1)
}

function displayName(name) {
  return name.split('-').map(capitalize).join(' ')
}

function padDexNo(id) {
  return String(id).padStart(3, '0')
}

function barWidth(value) {
  return `${Math.min(value, 100)}%`
}

function pokemonIdFromUrl(url) {
  const parts = url.replace(/\/$/, '').split('/')
  return Number(parts[parts.length - 1])
}

function getZhName(id, fallback) {
  return ZH_NAMES[id] || ZH_NAMES[String(id)] || fallback
}

function cleanFlavorText(text) {
  return String(text || '')
    .replace(/\f/g, ' ')
    .replace(/\s+/g, ' ')
    .replace(/([\u4e00-\u9fff，。！？、：；])\s+(?=[\u4e00-\u9fff])/g, '$1')
    .trim()
}

function pickFlavorText(entries, lang) {
  const list = (entries || []).filter((item) => item.language?.name === lang)
  return list[list.length - 1]?.flavor_text || ''
}

async function getFlavorText(speciesId) {
  const key = String(speciesId)
  if (flavorCache[key] !== undefined) return flavorCache[key]

  try {
    const res = await fetch(`https://pokeapi.co/api/v2/pokemon-species/${speciesId}`)
    if (!res.ok) throw new Error('species failed')
    const data = await res.json()
    const entries = data.flavor_text_entries || []
    const text = cleanFlavorText(
      pickFlavorText(entries, 'zh-hant') ||
        pickFlavorText(entries, 'zh-hans') ||
        pickFlavorText(entries, 'en'),
    )
    flavorCache[key] = text
    return text
  } catch {
    flavorCache[key] = ''
    return ''
  }
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

    const list = rawList.map((item) => ({
      ...item,
      label: getZhName(item.id, displayName(item.name)),
    }))

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
    const zhName = getZhName(speciesId, displayName(data.name))
    const flavor = await getFlavorText(speciesId)

    pokemon.value = {
      id: data.id,
      name: zhName,
      apiName: data.name,
      flavor,
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
    if (mystery) rememberMysteryPokemon()
    else rememberDexPokemon()
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

async function onDexNoChange(event) {
  const id = event.target.value
  if (!id) return

  await fetchPokemon(id)
  const current = pokemon.value
  if (!current) return

  selectedName.value = current.apiName
  const typeName = current.typeKeys?.[0]
  if (typeName) {
    selectedType.value = typeName
    await loadPokemonByType(typeName)
  }
}

async function pickMysteryFromCurrentType(mystery = true) {
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

function rememberMysteryPokemon() {
  if (skipMysteryHistoryPush) {
    skipMysteryHistoryPush = false
    return
  }

  const name = pokemon.value?.apiName
  if (!name) {
    mysteryInsertFront = false
    return
  }

  const entry = {
    name,
    type: selectedType.value || pokemon.value.typeKeys?.[0] || '',
  }
  const current = mysteryHistory[mysteryIndex.value]
  if (current?.name === entry.name) {
    mysteryInsertFront = false
    return
  }

  if (mysteryInsertFront) {
    mysteryInsertFront = false
    mysteryHistory.unshift(entry)
    if (mysteryHistory.length > 40) mysteryHistory.pop()
    mysteryIndex.value = 0
    return
  }

  if (mysteryIndex.value >= 0 && mysteryIndex.value < mysteryHistory.length - 1) {
    mysteryHistory.splice(mysteryIndex.value + 1)
  }

  mysteryHistory.push(entry)
  if (mysteryHistory.length > 40) mysteryHistory.shift()
  mysteryIndex.value = mysteryHistory.length - 1
}

async function showMysteryEntry(entry) {
  skipMysteryHistoryPush = true
  selectedType.value = entry.type
  typeHintOpen.value = true
  selectedName.value = entry.name
  await loadPokemonByType(entry.type)
  await fetchPokemon(entry.name, { mystery: true })
}

async function onMysteryNext() {
  rememberMysteryPokemon()
  if (mysteryIndex.value < mysteryHistory.length - 1) {
    mysteryIndex.value += 1
    await showMysteryEntry(mysteryHistory[mysteryIndex.value])
    return
  }

  await onRandom()
}

async function onMysteryPrev() {
  rememberMysteryPokemon()
  if (mysteryIndex.value <= 0) return

  mysteryIndex.value -= 1
  await showMysteryEntry(mysteryHistory[mysteryIndex.value])
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
  else rememberMysteryPokemon()
}

function onSwipeLeft() {
  if (loading.value || listLoading.value) return
  skipSilhouetteUntil = Date.now() + 700
  if (pageIndex.value === 1) {
    onDexNext()
    return
  }
  onMysteryNext()
}

async function onSwipeRight() {
  if (loading.value || listLoading.value) return
  skipSilhouetteUntil = Date.now() + 700

  if (pageIndex.value === 1) {
    onDexPrev()
    return
  }

  onMysteryPrev()
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
    selectedType.value = 'electric'
    selectedName.value = 'pikachu'
    await Promise.all([
      loadTypes(),
      loadPokemonByType('electric'),
      fetchPokemon('pikachu', { mystery: true }),
    ])
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
    <section class="panel search-panel" :class="{ 'is-dex': pageIndex === 1 }">
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
            class="search-select search-select-no"
            aria-label="選擇編號"
            :value="pokemon ? String(pokemon.id) : ''"
            :disabled="listLoading"
            @change="onDexNoChange"
          >
            <option value="" disabled>編號</option>
            <option v-for="id in DEX_NOS" :key="id" :value="String(id)">
              {{ padDexNo(id) }}
            </option>
          </select>

          <select
            v-model="selectedType"
            class="search-select search-select-type"
            aria-label="選擇屬性"
            @change="onTypeChange"
          >
            <option value="" disabled>屬性</option>
            <option v-for="type in types" :key="type.name" :value="type.name">
              {{ type.label }}
            </option>
          </select>

          <select
            v-model="selectedName"
            class="search-select search-select-name"
            aria-label="選擇角色"
            :disabled="!selectedType || listLoading"
            @change="onNameChange"
          >
            <option value="" disabled>
              {{ listLoading ? '載入中' : '角色' }}
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

      <div v-if="pageIndex === 1" class="chips">
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
                aria-label="上一題"
                @pointerdown.stop
                @click.stop="onMysteryPrev"
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
                aria-label="下一題"
                @pointerdown.stop
                @click.stop="onMysteryNext"
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
            <p class="reveal-hint">點黑影看答案！</p>
          </article>

          <article class="deck-page pokemon">
            <div class="poke-title">
              <span class="poke-no">No. {{ padDexNo(pokemon.id) }}</span>
              <h2 class="poke-name">{{ pokemon.name }}</h2>
              <p class="poke-en">{{ displayName(pokemon.apiName) }}</p>
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
            <p
              v-if="pokemon.flavor"
              class="flavor-box"
              :class="{ 'is-en': isEnglishFlavor }"
            >{{ pokemon.flavor }}</p>
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
              <div class="radar-side">
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
                <RadarChart :stats="pokemon.stats" />
              </div>
            </div>
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
