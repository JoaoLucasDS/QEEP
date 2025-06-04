<template>
  <q-layout view="hHh Lpr lFf">
    <q-header>
      <q-toolbar>
        <q-toolbar-title class="q-pb-sm" style="font-family: 'pokemonFont', sans-serif"
          >Quem é esse Pokemon?</q-toolbar-title
        >
        <div class="q-pb-sm" style="font-family: 'pokemonFont', sans-serif">By João Dantas</div>
      </q-toolbar>
    </q-header>

    <q-page-container class="background">
      <div class="shooting-stars">
        <span></span>
        <span></span>
        <span></span>
        <span></span>
        <span></span>
        <span></span>

        <!-- Adicione mais spans conforme necessário -->
      </div>
      <router-view />
    </q-page-container>
  </q-layout>
</template>

<style scoped>
.background {
  position: relative;
  overflow: hidden;
  min-height: 100vh;
  background: linear-gradient(200deg, #fe0000, #ff6419);
}

.shooting-stars {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
}

.shooting-stars span {
  position: absolute;
  top: 0;
  left: 80%;
  width: calc(2px + 2 * var(--random-size));
  height: calc(2px + 2 * var(--random-size));
  background: white;
  border-radius: 50%;
  box-shadow: 0 0 10px white;
  animation: shoot 3s linear infinite;
}

.shooting-stars span::before {
  content: '';
  position: absolute;
  top: 50%;
  left: 0;
  width: 350px;
  height: 1px;
  background: linear-gradient(90deg, white, transparent);
  transform: translateY(-50%);
}

@keyframes shoot {
  0% {
    transform: rotate(315deg) translateX(0);
    opacity: 1;
  }
  100% {
    transform: rotate(315deg) translateX(-1000px);
    opacity: 0;
  }
}

.shooting-stars span:nth-child(1) {
  top: 20%;
  animation-delay: 0s;
  animation-duration: 2s;
}

.shooting-stars span:nth-child(2) {
  top: 40%;
  animation-delay: 1s;
  animation-duration: 3s;
}

.shooting-stars span:nth-child(3) {
  top: 60%;
  animation-delay: 2s;
  animation-duration: 2.5s;
}
</style>
