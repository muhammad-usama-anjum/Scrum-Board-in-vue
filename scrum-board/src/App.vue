<template>
  <div id="app">
    <ScrumBoard />
  </div>
</template>

<script>
import ScrumBoard from './components/ScrumBoard.vue';

export default {
  name: 'App',
  components: {
    ScrumBoard
  }
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin: 0%;
  padding: 10px;
}
</style>
