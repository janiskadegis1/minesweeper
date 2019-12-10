<template>
  <div id="app">
    <MineSweeper />
  </div>
</template>

<script lang="ts">
import { Component, Vue } from "vue-property-decorator";
import MineSweeper from "./components/MineSweeper.vue";

@Component({
  components: {
    MineSweeper
  }
})
export default class App extends Vue {}
</script>

<style>
:root {
  --color-white: #fff9fb;
  --color-gray: #d3d4d9;
  --color-blue: #4b88a2;
  --color-red: #bb0a21;
  --color-black: #252627;
}

#app {
  font-family: "Avenir", Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
