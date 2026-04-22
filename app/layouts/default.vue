<script setup lang="ts">
defineProps<{ title?: string }>();
</script>

<template>
  <div>
    <header style="padding: 1rem; background: #eee;">
      <h1>{{ title ?? "(no title)" }}</h1>
    </header>
    <main style="padding: 1rem;">
      <slot />
    </main>
  </div>
</template>
