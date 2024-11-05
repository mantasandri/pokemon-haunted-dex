<template>
  <div class="min-h-screen bg-gradient-to-b from-gray-900 via-purple-900 to-orange-900 flex items-center justify-center p-4 relative overflow-hidden">
    <!-- Floating Pokémon background -->
    <div class="absolute inset-0 z-0 overflow-hidden">
      <div v-for="(pokemon, index) in backgroundPokemon" :key="index" 
           class="absolute animate-float"
           :style="{ 
             left: `${Math.random() * 100}%`, 
             top: `${Math.random() * 100}%`, 
             animationDelay: `${Math.random() * 5}s` 
           }">
        <img :src="pokemon.sprite" :alt="pokemon.name" class="w-52 h-52 opacity-20" />
      </div>
    </div>
    
    <!-- Main content -->
    <div class="w-full max-w-4xl flex gap-4">
      <!-- PokéDex Card -->
      <div class="w-full max-w-md bg-gray-800 rounded-lg shadow-xl overflow-hidden z-10 relative">
        <!-- Cobweb decorations -->
        <div class="absolute top-0 left-0 w-16 h-16 border-t-4 border-l-4 border-orange-500 rounded-tl-full"></div>
        <div class="absolute top-0 right-0 w-16 h-16 border-t-4 border-r-4 border-orange-500 rounded-tr-full"></div>
        
        <div class="p-6 bg-gradient-to-r from-orange-600 to-purple-700 rounded-t-lg relative">
          <h1 class="text-3xl font-bold text-white mb-4 text-center flex items-center justify-center">
            <Skull class="w-8 h-8 mr-2" />
            Spooky PokéDex
            <Skull class="w-8 h-8 ml-2" />
          </h1>
          <div class="relative">
            <input
              v-model="searchQuery"
              @input="debouncedSearch"
              type="text"
              placeholder="Search Pokémon"
              class="w-full px-4 py-2 rounded-full focus:outline-none focus:ring-2 focus:ring-orange-500 text-gray-800 bg-orange-100"
            />
            <ul v-if="suggestions.length > 0" class="absolute z-20 w-full mt-1 bg-gray-800 rounded-md shadow-lg max-h-60 overflow-auto">
              <li
                v-for="pokemon in suggestions"
                :key="pokemon.name"
                @click="selectPokemon(pokemon.name)"
                class="px-4 py-2 hover:bg-gray-700 cursor-pointer text-orange-100"
              >
                {{ pokemon.name }}
              </li>
            </ul>
          </div>
        </div>
        <div class="p-6 bg-gray-800 rounded-b-lg relative">
          <!-- Tombstone shape -->
          <div class="absolute inset-0 bg-gray-700 rounded-t-full rounded-b-lg transform scale-y-75 translate-y-10 z-0"></div>
          
          <div class="relative z-10">
            <div v-if="loading" class="text-center text-orange-200">
              <Loader class="animate-spin h-8 w-8 mx-auto mb-2" />
              Loading...
            </div>
            <div v-else-if="error" class="text-center text-red-400">
              {{ error }}
            </div>
            <div v-else-if="selectedPokemon" class="text-center">
              <img
                :src="selectedPokemon.sprites.front_default"
                :alt="selectedPokemon.name"
                class="w-32 h-32 mx-auto mb-4 drop-shadow-[0_0_10px_rgba(255,165,0,0.5)]"
              />
              <h2 class="text-2xl font-bold text-orange-300 mb-2 capitalize">{{ selectedPokemon.name }}</h2>
              <div class="flex justify-center space-x-2 mb-4">
                <span
                  v-for="type in selectedPokemon.types"
                  :key="type.type.name"
                  class="px-2 py-1 rounded-full text-xs font-semibold"
                  :class="getTypeColor(type.type.name)"
                >
                  {{ type.type.name }}
                </span>
              </div>
              <div class="grid grid-cols-2 gap-4 text-sm text-orange-100 mb-4">
                <div>
                  <p class="font-semibold">Height</p>
                  <p>{{ selectedPokemon.height / 10 }} m</p>
                </div>
                <div>
                  <p class="font-semibold">Weight</p>
                  <p>{{ selectedPokemon.weight / 10 }} kg</p>
                </div>
                <div>
                  <p class="font-semibold">Abilities</p>
                  <p>{{ selectedPokemon.abilities.map(a => a.ability.name).join(', ') }}</p>
                </div>
                <div>
                  <p class="font-semibold">Base Experience</p>
                  <p>{{ selectedPokemon.base_experience }}</p>
                </div>
              </div>
              <button
                @click="toggleFavorite(selectedPokemon)"
                class="px-4 py-2 bg-orange-500 text-white rounded-full hover:bg-orange-600 transition-colors"
              >
                {{ isFavorite(selectedPokemon) ? 'Remove from Favorites' : 'Add to Favorites' }}
              </button>
            </div>
            <div v-else class="text-center text-orange-200">
              <Ghost class="h-12 w-12 mx-auto mb-2" />
              <p>Search for a Pokémon to see its spooky details</p>
            </div>
          </div>
        </div>
      </div>

      <!-- Favorites Section -->
      <div v-if="favorites.length > 0" class="w-full max-w-sm bg-gray-800 rounded-lg shadow-xl overflow-hidden z-10 p-6">
        <h2 class="text-2xl font-bold text-orange-300 mb-4">Favorite Pokémon</h2>
        <ul class="space-y-2">
          <li 
            v-for="favorite in favorites" 
            :key="favorite.id" 
            class="flex items-center justify-between bg-gray-700 rounded-lg p-2 hover:bg-gray-600 transition-colors cursor-pointer"
            @click="selectPokemon(favorite.name)"
          >
            <div class="flex items-center">
              <img :src="favorite.sprites.front_default" :alt="favorite.name" class="w-12 h-12 mr-2" />
              <span class="text-orange-100 capitalize">{{ favorite.name }}</span>
            </div>
            <button
              @click.stop="toggleFavorite(favorite)"
              class="text-red-400 hover:text-red-500"
            >
              <X class="h-5 w-5" />
            </button>
          </li>
        </ul>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'
import { Ghost, Skull, Loader, X } from 'lucide-vue-next'

const searchQuery = ref('')
const allPokemon = ref([])
const selectedPokemon = ref(null)
const loading = ref(false)
const error = ref(null)
const favorites = ref([])

const backgroundPokemon = [
  { name: 'Gastly', sprite: 'https://raw.githubusercontent.com/PokeAPI/sprites/master/sprites/pokemon/92.png' },
  { name: 'Haunter', sprite: 'https://raw.githubusercontent.com/PokeAPI/sprites/master/sprites/pokemon/93.png' },
  { name: 'Gengar', sprite: 'https://raw.githubusercontent.com/PokeAPI/sprites/master/sprites/pokemon/94.png' },
  { name: 'Misdreavus', sprite: 'https://raw.githubusercontent.com/PokeAPI/sprites/master/sprites/pokemon/200.png' },
  { name: 'Shuppet', sprite: 'https://raw.githubusercontent.com/PokeAPI/sprites/master/sprites/pokemon/353.png' },
  { name: 'Duskull', sprite: 'https://raw.githubusercontent.com/PokeAPI/sprites/master/sprites/pokemon/355.png' },
  { name: 'Drifloon', sprite: 'https://raw.githubusercontent.com/PokeAPI/sprites/master/sprites/pokemon/425.png' },
  { name: 'Mismagius', sprite: 'https://raw.githubusercontent.com/PokeAPI/sprites/master/sprites/pokemon/429.png' },
  { name: 'Yamask', sprite: 'https://raw.githubusercontent.com/PokeAPI/sprites/master/sprites/pokemon/562.png' },
  { name: 'Litwick', sprite: 'https://raw.githubusercontent.com/PokeAPI/sprites/master/sprites/pokemon/607.png' },
]

onMounted(() => {
  fetchAllPokemon()
  loadFavorites()
})

const suggestions = computed(() => {
  if (searchQuery.value.length < 2) return []
  return allPokemon.value
    .filter(pokemon => pokemon.name.toLowerCase().includes(searchQuery.value.toLowerCase()))
    .slice(0, 5)
})

const debouncedSearch = (() => {
  let timeout
  return () => {
    clearTimeout(timeout)
    timeout = setTimeout(() => {
      if (searchQuery.value.length >= 2) {
        fetchAllPokemon()
      }
    }, 300)
  }
})()

async function fetchAllPokemon() {
  if (allPokemon.value.length > 0) return
  try {
    loading.value = true
    const response = await fetch('https://pokeapi.co/api/v2/pokemon?limit=1000')
    const data = await response.json()
    allPokemon.value = data.results
  } catch (err) {
    error.value = 'Failed to fetch Pokémon list'
  } finally {
    loading.value = false
  }
}

async function selectPokemon(name) {
  try {
    loading.value = true
    error.value = null
    searchQuery.value = name
    const response = await fetch(`https://pokeapi.co/api/v2/pokemon/${name}`)
    selectedPokemon.value = await response.json()
  } catch (err) {
    error.value = 'Failed to fetch Pokémon details'
    selectedPokemon.value = null
  } finally {
    loading.value = false
  }
}

function getTypeColor(type) {
  const colors = {
    normal: 'bg-gray-400 text-gray-800',
    fire: 'bg-orange-500 text-white',
    water: 'bg-blue-500 text-white',
    electric: 'bg-yellow-400 text-gray-800',
    grass: 'bg-green-500 text-white',
    ice: 'bg-blue-200 text-gray-800',
    fighting: 'bg-red-700 text-white',
    poison: 'bg-purple-500 text-white',
    ground: 'bg-yellow-600 text-white',
    flying: 'bg-indigo-400 text-white',
    psychic: 'bg-pink-500 text-white',
    bug: 'bg-green-400 text-gray-800',
    rock: 'bg-yellow-700 text-white',
    ghost: 'bg-purple-700 text-white',
    dragon: 'bg-indigo-700 text-white',
    dark: 'bg-gray-700 text-white',
    steel: 'bg-gray-400 text-gray-800',
    fairy: 'bg-pink-300 text-gray-800'
  }
  return colors[type] || 'bg-gray-400 text-gray-800'
}

function toggleFavorite(pokemon) {
  const index = favorites.value.findIndex(fav => fav.id === pokemon.id)
  if (index === -1) {
    favorites.value.push(pokemon)
  } else {
    favorites.value.splice(index, 1)
  }
  saveFavorites()
}

function isFavorite(pokemon) {
  return favorites.value.some(fav => fav.id === pokemon.id)
}

function saveFavorites() {
  localStorage.setItem('favorites', JSON.stringify(favorites.value))
}

function loadFavorites() {
  const storedFavorites = localStorage.getItem('favorites')
  if (storedFavorites) {
    favorites.value = JSON.parse(storedFavorites)
  }
}
</script>

<style scoped>
@keyframes float {
  0%, 100% { transform: translateY(0) rotate(0deg); }
  50% { transform: translateY(-20px) rotate(10deg); }
}

.animate-float {
  animation: float 5s ease-in-out infinite;
}
</style>