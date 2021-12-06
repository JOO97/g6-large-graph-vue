<template>
  <div id="app">
    <button @click="showOriginal = !showOriginal">切换</button>
    <LargeGraph v-if="!showOriginal" @on-click-menu="handleGraphMenuClick" />
    <LargeGraphOriginal v-else />
  </div>
</template>

<script>
import LargeGraph from "./components/LargeGraph/index.vue";
import LargeGraphOriginal from "./components/LargeGraphCopy/index.vue";
import CanvasMenu from "./components/CanvasMenu/index.vue";

export default {
  name: "app",
  components: { LargeGraph, CanvasMenu, LargeGraphOriginal },
  data() {
    return {
      showOriginal: false
    };
  },
  methods: {
    handleGraphMenuClick({ type }) {
      switch (type) {
        case "createModel":
          console.log("createModel");
          break;
        default:
          break;
      }
    }
  }
};
</script>

<style lang="scss">
#app {
  font-family: "Avenir", Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  background: #fff;
  overflow: hidden;
  height: 100vh;
  width: 100vw;
}
</style>
