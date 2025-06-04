<template>
  <div class="column items-center text-white text-weight-black q-py-md">
    <div class="column items-center" v-if="gameState === 'playing'">
      <!-- Botão para alternar entre modos de imagem -->
      <div class="q-mb-md">
        <q-btn-toggle
          unelevated=""
          rounded
          v-model="imageMode"
          toggle-color="primary"
          style="background-color: #cb0300"
          :options="[
            { label: 'Borrar', value: 'blur' },
            { label: 'Silhueta', value: 'silhouette' },
          ]"
        />
      </div>

      <div class="pokemon-container">
        <!-- Imagem de fundo -->
        <img src="Explosion.png" class="background-image" />

        <!-- Imagem do Pokémon -->
        <q-img
          :src="currentImage"
          spinner-color="primary"
          style="max-width: 300px; max-height: 300px"
          :ratio="1"
          :class="{ unblurred: revealed }"
        />
      </div>

      <div class="row q-gutter-sm justify-center items-center q-pb-md">
        <div class="text-subtitle1">Rodada {{ round }}/{{ maxRounds }}</div>
        <div class="text-subtitle1">|</div>
        <div class="text-subtitle1">
          <q-icon
            v-for="(life, index) in lives"
            :key="index"
            name="favorite"
            color="primary"
            size="sm"
          />
        </div>
        <div class="text-subtitle1">|</div>
        <div class="text-subtitle1">Pontos: {{ score }}</div>
      </div>

      <q-select
        v-model="guess"
        standout="bg-primary text-white"
        hide-dropdown-icon
        label="Digite o nome do Pokémon"
        :options="filteredOptions"
        use-input
        fill-input
        hide-selected
        input-debounce="0"
        :disable="loading || revealed"
        @filter="filterFn"
        @keyup.enter="guess ? checkAnswer : ''"
        autofocus
        :loading="loadingNames"
        style="max-width: 400px; width: 90vw"
      >
        <template v-slot:append>
          <q-icon name="cancel" v-if="!guess && !revealed" />
          <q-icon name="check_circle" v-if="guess && !revealed" color="positive" />
        </template>
      </q-select>

      <div v-if="revealed" class="text-h6 text-center q-mt-md">
        {{ message }}
      </div>
    </div>

    <div class="row q-ma-md q-gutter-x-md" v-if="gameState === 'playing'">
      <q-btn @click="revealAnswer" label="Pular" :disable="revealed" color="warning" flat />
      <q-btn
        unelevated
        rounded
        @click="checkAnswer"
        label="Verificar"
        :disable="loading || !guess || revealed"
        color="primary"
      />
    </div>

    <div v-if="gameState === 'gameOver'" class="column items-center">
      <div class="text-h4 q-mb-md">Fim de Jogo!</div>
      <div class="text-h5">Pontuação final: {{ score }}</div>
      <div class="text-h6 q-mt-md">{{ finalMessage }}</div>
      <q-btn label="Jogar Novamente" @click="startGame" color="primary" class="q-mt-lg" />
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, watch } from 'vue'
import { useQuasar } from 'quasar'
import axios from 'axios'

const $q = useQuasar()

// Estados do jogo
const gameState = ref('playing') // playing, gameOver
const pokemonList = ref([])
const currentPokemon = ref('')
const currentImage = ref('')
const blurredImage = ref('')
const normalImage = ref('')
const silhouetteImage = ref('')
const guess = ref('')
const round = ref(1)
const maxRounds = 10
const score = ref(0)
const lives = ref(3)
const revealed = ref(false)
const message = ref('')
const filteredOptions = ref([])
const imageMode = ref('silhouette') // 'blur' ou 'silhouette'

// Estados de carregamento
const loading = ref(true)
const loadingNames = ref(true)

// Filtro para autocomplete
function filterFn(val, update) {
  update(() => {
    if (val === '') {
      filteredOptions.value = []
    } else {
      const needle = val.toLowerCase()
      filteredOptions.value = pokemonList.value.filter((v) => v.toLowerCase().includes(needle))
    }
  })
}

// Mensagem final baseada na pontuação
const finalMessage = computed(() => {
  if (score.value >= maxRounds - 1) return '🎉 Incrível! Você é um verdadeiro Mestre Pokémon!'
  if (score.value >= maxRounds / 2) return '👍 Bom trabalho! Continue treinando!'
  return '💪 Continue tentando! Você vai melhorar!'
})

// Carrega lista de Pokémon
async function loadPokemonList() {
  loadingNames.value = true
  try {
    const res = await axios.get('https://pokeapi.co/api/v2/pokemon?limit=1025')
    pokemonList.value = res.data.results.map((p) => p.name)
  } catch (err) {
    console.error(err)
    showError('Erro ao carregar lista de Pokémon.')
  } finally {
    loadingNames.value = false
  }
}

// Carrega imagem do Pokémon
async function loadPokemonImage(name) {
  try {
    const res = await axios.get(`https://pokeapi.co/api/v2/pokemon/${name}`)
    const imageUrl = res.data.sprites.other['official-artwork'].front_default

    // Guarda imagem normal
    normalImage.value = imageUrl

    // Cria versão borrada
    const img = await fetch(imageUrl).then((r) => r.blob())
    const bitmap = await createImageBitmap(img)

    // Cria canvas para processamento de imagem
    const canvas = document.createElement('canvas')
    canvas.width = bitmap.width
    canvas.height = bitmap.height
    const ctx = canvas.getContext('2d')

    // Gera imagem borrada
    ctx.filter = 'blur(12px)'
    ctx.drawImage(bitmap, 0, 0)
    blurredImage.value = canvas.toDataURL()
    ctx.filter = 'none'

    // Gera silhueta preta
    ctx.clearRect(0, 0, canvas.width, canvas.height)
    ctx.drawImage(bitmap, 0, 0)

    // Aplica efeito de silhueta
    const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height)
    const data = imageData.data
    for (let i = 0; i < data.length; i += 4) {
      // Mantém apenas pixels não transparentes
      if (data[i + 3] > 0) {
        // Define como preto
        data[i] = 0 // R
        data[i + 1] = 0 // G
        data[i + 2] = 0 // B
      }
    }
    ctx.putImageData(imageData, 0, 0)
    silhouetteImage.value = canvas.toDataURL()

    // Atualiza imagem inicial
    updateCurrentImage()
  } catch (err) {
    console.error(err)
    showError('Erro ao carregar imagem.')
  }
}

// Atualiza imagem baseada no modo selecionado
function updateCurrentImage() {
  if (revealed.value) {
    currentImage.value = normalImage.value
  } else {
    if (imageMode.value === 'blur') {
      currentImage.value = blurredImage.value
    } else {
      currentImage.value = silhouetteImage.value
    }
  }
}

// Observa mudanças no modo de imagem
watch(imageMode, () => {
  if (!revealed.value) {
    updateCurrentImage()
  }
})

// Atualiza imagem quando revelada
watch(revealed, () => {
  updateCurrentImage()
})

// Inicia nova rodada
function newRound() {
  if (round.value >= maxRounds || lives.value <= 0) {
    endGame()
    return
  }

  loading.value = true
  revealed.value = false
  message.value = ''
  guess.value = ''
  filteredOptions.value = []
  round.value++

  currentPokemon.value = pokemonList.value[Math.floor(Math.random() * pokemonList.value.length)]
  loadPokemonImage(currentPokemon.value).then(() => {
    loading.value = false
  })
}

// Verifica resposta
function checkAnswer() {
  if (revealed.value) return

  const correct = currentPokemon.value.toLowerCase()
  const userGuess = guess.value.trim().toLowerCase()

  revealed.value = true
  currentImage.value = normalImage.value

  if (userGuess === correct) {
    score.value++
    message.value = `✅ Correto! Era ${capitalize(correct)}!`
  } else {
    lives.value--
    message.value = `❌ Errado! Era ${capitalize(correct)}!`

    if (lives.value <= 0) {
      message.value += ' Sem vidas restantes!'
    }
  }

  // Avança automaticamente após 2 segundos
  setTimeout(() => {
    if (lives.value > 0) newRound()
    else endGame()
  }, 2000)
}

// Revela resposta
function revealAnswer() {
  if (revealed.value) return

  revealed.value = true
  lives.value--
  currentImage.value = normalImage.value
  message.value = `👀 Revelado! Era ${capitalize(currentPokemon.value)}!`

  // Avança automaticamente após 2 segundos
  setTimeout(() => {
    if (lives.value > 0) newRound()
    else endGame()
  }, 2000)
}

// Finaliza o jogo
function endGame() {
  gameState.value = 'gameOver'
}

// Inicia novo jogo
function startGame() {
  gameState.value = 'playing'
  round.value = 0
  score.value = 0
  lives.value = 3
  newRound()
}

// Mostra erros
function showError(msg) {
  $q.notify({
    type: 'negative',
    message: msg,
    timeout: 3000,
  })
}

// Capitaliza nomes
function capitalize(str) {
  return str.charAt(0).toUpperCase() + str.slice(1)
}

// Inicialização
onMounted(() => {
  loadPokemonList().then(startGame)
})
</script>

<style scoped>
.pokemon-container {
  position: relative;
  width: 80vw; /* Largura responsiva */
  max-width: 400px; /* Máximo para telas grandes */
  aspect-ratio: 1/1; /* Mantém proporção quadrada */
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: hidden; /* Esconde partes excedentes */
}

.background-image {
  position: absolute;
  width: 100%; /* Cobre todo o container */
  height: 100%;
  object-fit: contain; /* Mantém proporção sem distorcer */
  z-index: 0;
}

.q-img {
  position: relative;
  width: 100%; /* Preenche o container */
  height: 100%;
  z-index: 1;
  transition: filter 0.5s ease;
  object-fit: contain; /* Garante proporção correta */
}

.unblurred {
  filter: none !important;
}

.text-subtitle1 {
  font-weight: 500;
}
</style>
