<script setup>
import { computed, onMounted, onUnmounted, ref } from "vue";

const apiBaseUrl = import.meta.env.VITE_API_BASE_URL || "http://localhost:3000";
const loading = ref(true);
const error = ref("");
const images = ref([]);
const previewImage = ref(null);
const activeScreen = ref("gallery");
const selectedFile = ref(null);
const uploading = ref(false);
const uploadProgress = ref(0);
const uploadError = ref("");
const isDragging = ref(false);
const toasts = ref([]);
const fileInputRef = ref(null);

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

function openPreview(image) {
  previewImage.value = image;
}

function closePreview() {
  previewImage.value = null;
}

function handleKeydown(event) {
  if (event.key === "Escape") {
    closePreview();
  }
}

function onFileSelected(event) {
  const file = event.target.files?.[0] || null;
  selectedFile.value = file;
  uploadError.value = "";
}

function addToast(message, type = "success") {
  const id = Date.now() + Math.random();
  toasts.value.push({ id, message, type });

  setTimeout(() => {
    toasts.value = toasts.value.filter((toast) => toast.id !== id);
  }, 3500);
}

function removeToast(id) {
  toasts.value = toasts.value.filter((toast) => toast.id !== id);
}

function parseJsonSafely(text) {
  try {
    return text ? JSON.parse(text) : {};
  } catch {
    return {};
  }
}

function openFileDialog() {
  fileInputRef.value?.click();
}

function onDragOver(event) {
  event.preventDefault();
  isDragging.value = true;
}

function onDragLeave(event) {
  event.preventDefault();
  isDragging.value = false;
}

function onDrop(event) {
  event.preventDefault();
  isDragging.value = false;

  const file = event.dataTransfer?.files?.[0] || null;

  if (file) {
    selectedFile.value = file;
    uploadError.value = "";
  }
}

async function uploadImage() {
  uploadError.value = "";
  uploadProgress.value = 0;

  if (!selectedFile.value) {
    uploadError.value = "Selecciona una imagen antes de subir.";
    addToast(uploadError.value, "error");
    return;
  }

  const formData = new FormData();
  formData.append("image", selectedFile.value);

  uploading.value = true;

  try {
    await new Promise((resolve, reject) => {
      const xhr = new XMLHttpRequest();

      xhr.open("POST", `${apiBaseUrl}/api/images/upload`);

      xhr.upload.onprogress = (event) => {
        if (!event.lengthComputable) {
          return;
        }

        uploadProgress.value = Math.round((event.loaded * 100) / event.total);
      };

      xhr.onload = () => {
        const data = parseJsonSafely(xhr.responseText);

        if (xhr.status >= 200 && xhr.status < 300) {
          resolve(data);
          return;
        }

        reject(new Error(data.message || `Error HTTP ${xhr.status}`));
      };

      xhr.onerror = () => {
        reject(new Error("No se pudo conectar con la API de subida."));
      };

      xhr.send(formData);
    });

    addToast("Imagen subida correctamente.", "success");
    selectedFile.value = null;
    if (fileInputRef.value) {
      fileInputRef.value.value = "";
    }
    await fetchImages();
    activeScreen.value = "gallery";
  } catch (err) {
    uploadError.value = err?.message || "No se pudo subir la imagen.";
    addToast(uploadError.value, "error");
  } finally {
    uploading.value = false;
    uploadProgress.value = 0;
  }
}

onMounted(fetchImages);
onMounted(() => window.addEventListener("keydown", handleKeydown));
onUnmounted(() => window.removeEventListener("keydown", handleKeydown));
</script>

<template>
  <div class="app-root">
    <div class="page-bg"></div>
    <main class="page">
      <header class="hero">
        <p class="eyebrow">DOGS PRETTY</p>
        <h1>Dog Gallery from GCP</h1>
        <p class="subtitle">
          Imagenes servidas desde Google Cloud Storage a traves de tu API en
          Express.
        </p>
        <div class="screen-switch">
          <button
            type="button"
            class="screen-btn"
            :class="{ active: activeScreen === 'gallery' }"
            @click="activeScreen = 'gallery'"
          >
            Gallery
          </button>
          <button
            type="button"
            class="screen-btn"
            :class="{ active: activeScreen === 'upload' }"
            @click="activeScreen = 'upload'"
          >
            Upload
          </button>
        </div>
        <div class="actions">
          <button type="button" @click="fetchImages">Recargar</button>
          <span>{{ imageCount }} imagenes</span>
        </div>
      </header>

      <section v-if="activeScreen === 'upload'" class="upload-panel">
        <h2>Sube tus imagenes de perritos bonitos</h2>
        <p class="upload-help">
          Arrastra y suelta una imagen o selecciona un archivo desde tu equipo.
        </p>

        <div
          class="dropzone"
          :class="{ dragging: isDragging }"
          @dragenter="onDragOver"
          @dragover="onDragOver"
          @dragleave="onDragLeave"
          @drop="onDrop"
          @click="openFileDialog"
        >
          <p>Drop your dog image here</p>
          <small>PNG, JPG, WEBP, GIF, BMP, AVIF - Max 10MB</small>
        </div>

        <label class="file-input-wrap">
          <span>Seleccionar imagen</span>
          <input
            ref="fileInputRef"
            type="file"
            accept="image/*"
            @change="onFileSelected"
          />
        </label>

        <p v-if="selectedFile" class="file-selected">
          Archivo: {{ selectedFile.name }}
        </p>

        <button type="button" :disabled="uploading" @click="uploadImage">
          {{ uploading ? "Subiendo..." : "Subir imagen" }}
        </button>

        <div v-if="uploading" class="progress-wrap">
          <div class="progress-bar" :style="{ width: `${uploadProgress}%` }"></div>
          <span>{{ uploadProgress }}%</span>
        </div>

        <p v-if="uploadError" class="upload-message error">{{ uploadError }}</p>
      </section>

      <section v-else-if="loading" class="status">Cargando imagenes...</section>
      <section v-else-if="error" class="status error">{{ error }}</section>

      <section v-else class="gallery">
        <article
          v-for="image in images"
          :key="image.path"
          class="card previewable"
          @click="openPreview(image)"
        >
          <img :src="image.url" :alt="image.name" loading="lazy" />
          <div class="meta">
            <h2>{{ image.name }}</h2>
            <p>{{ new Date(image.updated).toLocaleString() }}</p>
          </div>
        </article>
      </section>
    </main>

    <section
      v-if="previewImage"
      class="preview-overlay"
      role="dialog"
      aria-modal="true"
      @click.self="closePreview"
    >
      <div class="preview-content">
        <button class="preview-close" type="button" @click="closePreview">
          Cerrar
        </button>
        <img
          :src="previewImage.url"
          :alt="previewImage.name"
          class="preview-image"
        />
        <p class="preview-title">{{ previewImage.name }}</p>
      </div>
    </section>

    <section class="toast-container" aria-live="polite" aria-atomic="true">
      <article
        v-for="toast in toasts"
        :key="toast.id"
        class="toast"
        :class="toast.type"
        @click="removeToast(toast.id)"
      >
        {{ toast.message }}
      </article>
    </section>
  </div>
</template>
