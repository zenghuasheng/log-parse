<template>
  <div id="app">
    <nav>
      <router-link to="/">Home</router-link>
    </nav>
    <router-view></router-view>
  </div>
</template>

<script setup>
</script>

<style scoped>
nav {
  margin-bottom: 1em;
}
nav a {
  margin-right: 1em;
}
</style>
