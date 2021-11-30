<template>
  <div id="app">
    <LargeGraph />
  </div>
</template>

<script>
import LargeGraph from "./components/LargeGraph/index.vue";

export default {
  name: "app",
  components: { LargeGraph },
  data() {
    return {};
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
