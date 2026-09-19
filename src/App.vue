<script setup>
import { onMounted, ref } from 'vue'
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
const QUICK_PICKS = [
  { name: 'bulbasaur', label: '妙蛙種子' },
  { name: 'squirtle', label: '傑尼龜' },
  { name: 'pikachu', label: '皮卡丘' },
]
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

async function fetchPokemon(nameOrId) {
  const q = String(nameOrId).trim().toLowerCase()
  if (!q) return

  loading.value = true
  error.value = ''

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
  } catch {
    pokemon.value = null
    error.value = '找不到這隻寶可夢，請再選一次'
  } finally {
    loading.value = false
  }
}

async function onTypeChange() {
  selectedName.value = ''
  await loadPokemonByType(selectedType.value)
}

function onNameChange() {
  if (selectedName.value) {
    fetchPokemon(selectedName.value)
  }
}

function onSearch() {
  if (!selectedType.value || !selectedName.value) {
    error.value = '請先選擇屬性和角色'
    return
  }
  fetchPokemon(selectedName.value)
}

async function syncSelectsFromPokemon(apiName) {
  if (!pokemon.value?.typeKeys?.length) return
  const typeName = pokemon.value.typeKeys[0]
  selectedType.value = typeName
  await loadPokemonByType(typeName)
  selectedName.value = apiName
}

async function onQuickPick(name) {
  await fetchPokemon(name)
  await syncSelectsFromPokemon(name.toLowerCase())
}

onMounted(async () => {
  try {
    await loadTypes()
    selectedType.value = 'electric'
    await loadPokemonByType('electric')
    selectedName.value = 'pikachu'
    await fetchPokemon('pikachu')
  } catch {
    error.value = '屬性列表載入失敗，請重新整理'
  }
})
</script>

<template>
  <main class="page">
    <section class="panel search-panel">
      <form class="search-row" @submit.prevent="onSearch">
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

        <button class="search-btn" type="submit">查詢</button>
      </form>

      <div class="chips">
        <button
          v-for="item in QUICK_PICKS"
          :key="item.name"
          class="chip"
          type="button"
          @click="onQuickPick(item.name)"
        >
          {{ item.label }}
        </button>
      </div>
    </section>

    <section class="panel result-panel">
      <p v-if="loading" class="status">查詢中...</p>
      <p v-else-if="error && !pokemon" class="status error">{{ error }}</p>

      <article v-else-if="pokemon" class="pokemon">
        <h1 class="poke-name">{{ pokemon.name }}</h1>
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
        <img
          v-if="pokemon.image"
          class="poke-img"
          :src="pokemon.image"
          :alt="pokemon.name"
        />

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
      </article>
    </section>
  </main>
</template>
