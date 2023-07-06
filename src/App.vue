<template>
  <MatterSceneEditor />
</template>

<script>
import MatterSceneEditor from './components/MatterSceneEditor.vue'

export default {
  name: 'App',
  components: {
    MatterSceneEditor
  }
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
