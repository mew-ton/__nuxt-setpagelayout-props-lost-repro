<script setup lang="ts">
setPageLayout("default", { title: "Initial title set by setPageLayout" });

const route = useRoute();
const activeTab = computed(() => route.query.tab ?? "a");
</script>

<template>
  <div>
    <p>Active tab: <strong>{{ activeTab }}</strong></p>

    <nav style="display: flex; gap: 0.5rem;">
      <NuxtLink :to="{ path: '/', query: { tab: 'a' } }" replace>Tab A</NuxtLink>
      <NuxtLink :to="{ path: '/', query: { tab: 'b' } }" replace>Tab B</NuxtLink>
      <NuxtLink :to="{ path: '/', query: { tab: 'c' } }" replace>Tab C</NuxtLink>
    </nav>

    <p style="margin-top: 1rem;">
      Expected: the header above always reads
      <em>"Initial title set by setPageLayout"</em>.
      <br />
      Actual: after the first tab click, the header goes blank
      ("(no title)") because <code>route.meta.layoutProps</code> is
      rebuilt from the static route record and <code>&lt;script setup&gt;</code>
      is not re-run on same-path navigations.
    </p>
  </div>
</template>
