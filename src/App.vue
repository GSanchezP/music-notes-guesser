<template>
  <div class="app">
    <header class="header">
      <h1>Music Notes Guesser</h1>
      <p class="subtitle">Listen, then guess the note</p>
    </header>
    <main class="main">
      <NoteGuesser />
    </main>
  </div>
</template>

<script setup>
import NoteGuesser from './components/NoteGuesser.vue'
</script>
