<script setup>
import { computed, onMounted, ref } from "vue";

const apiBaseUrl = import.meta.env.VITE_API_BASE_URL || "http://localhost:3000";
const loading = ref(true);
const error = ref("");
const images = ref([]);

const imageCount = computed(() => images.value.length);

async function fetchImages() {
  loading.value = true;
  error.value = "";

  try {
    const response = await fetch(`${apiBaseUrl}/api/images`);

    if (!response.ok) {
      throw new Error(`Error HTTP ${response.status}`);
    }

    const data = await response.json();
    images.value = data.images || [];
  } catch (err) {
    error.value =
      err?.message || "No se pudo conectar con la API de imagenes en GCP.";
  } finally {
    loading.value = false;
  }
}

onMounted(fetchImages);
</script>

<template>
  <div class="page-bg">
    <main class="page">
      <header class="hero">
        <p class="eyebrow">DOGS PRETTY</p>
        <h1>Dog Gallery from GCP</h1>
        <p class="subtitle">
          Imagenes servidas desde Google Cloud Storage a traves de tu API en
          Express.
        </p>
        <div class="actions">
          <button type="button" @click="fetchImages">Recargar</button>
          <span>{{ imageCount }} imagenes</span>
        </div>
      </header>

      <section v-if="loading" class="status">Cargando imagenes...</section>
      <section v-else-if="error" class="status error">{{ error }}</section>

      <section v-else class="gallery">
        <article v-for="image in images" :key="image.path" class="card">
          <img :src="image.url" :alt="image.name" loading="lazy" />
          <div class="meta">
            <h2>{{ image.name }}</h2>
            <p>{{ new Date(image.updated).toLocaleString() }}</p>
          </div>
        </article>
      </section>
    </main>
  </div>
</template>
