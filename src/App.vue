<template>
  <div id="app">
    <CanvasBoard> </CanvasBoard>
  </div>
</template>

<script>
import CanvasBoard from './components/CanvasBoard.vue'

export default {
  name: 'App',
  components: {
    CanvasBoard
  }
}
</script>

<style>
#app {
  font-family: Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}

body {
  width: 100vw;
  height: 100vh;
  overflow: hidden;
}
</style>
