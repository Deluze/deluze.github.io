<template>
  <div class="container text-gray-300">
    <div>
      <h1 class="text-6xl font-extralight title-appear">
        Bart Ramaekers
      </h1>
      <div class="mt-8">
        <button class="mx-4 button-appear">
          About Me
        </button>
        <button class="mx-4 button-appear">
          Projects
        </button>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import {defineComponent, useMeta} from '@nuxtjs/composition-api';

export default defineComponent({
  head() { return {} },
  setup() {
    useMeta({
      title: 'Home Page',
    });
  }
});
</script>

<style>

.container {
  margin: 0 auto;
  min-height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
  text-align: center;
}

.button-appear {
  animation: 1300ms buttonApear ease-out;
}

.title-appear {
  animation: 1s titleAppear ease-out;
}

@keyframes titleAppear {
  0% {
    opacity: 0;
  }
}

@keyframes buttonApear {
  0% {
    transform: translate(0px, 24px);
    opacity: 0;
  }

  25% {
    opacity: 0.2;
  }
}
</style>
