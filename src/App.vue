<script setup>
import { onMounted, ref } from 'vue'

const STAT_LABELS = {
  hp: '體力',
  attack: '攻擊力',
  defense: '防禦力',
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

const QUICK_PICKS = ['Bulbasaur', 'Charizard', 'Squirtle', 'Pikachu', 'Snorlax']

const query = ref('pikachu')
const pokemon = ref(null)
const loading = ref(false)
const error = ref('')

function capitalize(name) {
  return name.charAt(0).toUpperCase() + name.slice(1)
}

function barWidth(value) {
  return `${Math.min(value, 100)}%`
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
    pokemon.value = {
      id: data.id,
      name: capitalize(data.name),
      image:
        data.sprites.other?.['official-artwork']?.front_default ||
        data.sprites.front_default,
      types: data.types.map((item) => ({
        name: capitalize(item.type.name),
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
    error.value = '找不到這隻寶可夢，試試英文名或編號'
  } finally {
    loading.value = false
  }
}

function onSearch() {
  fetchPokemon(query.value)
}

function onQuickPick(name) {
  query.value = name.toLowerCase()
  fetchPokemon(name)
}

onMounted(() => {
  fetchPokemon(query.value)
})
</script>

<template>
  <main class="page">
    <section class="panel search-panel">
      <form class="search-row" @submit.prevent="onSearch">
        <input
          v-model="query"
          class="search-input"
          type="text"
          placeholder="輸入名字或編號，例如 pikachu"
          autocomplete="off"
        />
        <button class="search-btn" type="submit">查詢</button>
      </form>

      <div class="chips">
        <button
          v-for="name in QUICK_PICKS"
          :key="name"
          class="chip"
          type="button"
          @click="onQuickPick(name)"
        >
          {{ name }}
        </button>
      </div>
    </section>

    <section class="panel result-panel">
      <p v-if="loading" class="status">查詢中...</p>
      <p v-else-if="error" class="status error">{{ error }}</p>

      <article v-else-if="pokemon" class="pokemon">
        <img
          v-if="pokemon.image"
          class="poke-img"
          :src="pokemon.image"
          :alt="pokemon.name"
        />
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

        <ul class="stats">
          <li v-for="stat in pokemon.stats" :key="stat.key" class="stat-row">
            <span class="stat-label">{{ stat.label }}</span>
            <div class="stat-track">
              <div class="stat-fill" :style="{ width: barWidth(stat.value) }" />
            </div>
            <span class="stat-value">{{ stat.value }}</span>
          </li>
        </ul>
      </article>
    </section>
  </main>
</template>
