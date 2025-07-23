<template>
  <div id="app">
    <header>
      <h1>Vue Route Animation</h1>
    </header>
    <main>
      <RouteAnimation />
    </main>
  </div>
</template>

<script setup>
import RouteAnimation from './components/RouteAnimation.vue'
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  text-align: center;
  color: #2c3e50;
  margin: 0;
  padding: 0;
}

header {
  padding: 20px;
  background: #f5f5f5;
  margin-bottom: 20px;
}

main {
  padding: 0 20px;
  max-width: 1200px;
  margin: 0 auto;
}
</style>