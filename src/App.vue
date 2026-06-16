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
const localPreviewUrl = ref("");
const commentAuthor = ref("");
const commentText = ref("");
const submittingComment = ref(false);

const skeletonItems = Array.from({ length: 6 });

const imageCount = computed(() => images.value.length);
const latestUploadText = computed(() => {
  if (!images.value.length) {
    return "No uploads yet";
  }

  return new Date(images.value[0].updated).toLocaleString();
});

const totalStorageText = computed(() => {
  const totalBytes = images.value.reduce((acc, image) => acc + (image.size || 0), 0);

  if (totalBytes < 1024 * 1024) {
    return `${(totalBytes / 1024).toFixed(1)} KB`;
  }

  return `${(totalBytes / (1024 * 1024)).toFixed(2)} MB`;
});

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
  commentText.value = "";
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
  updateSelectedFile(file);
  uploadError.value = "";
}

function updateSelectedFile(file) {
  if (localPreviewUrl.value) {
    URL.revokeObjectURL(localPreviewUrl.value);
    localPreviewUrl.value = "";
  }

  selectedFile.value = file;

  if (file) {
    localPreviewUrl.value = URL.createObjectURL(file);
  }
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
    updateSelectedFile(file);
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
    updateSelectedFile(null);
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

function updateImageInteraction(imagePath, interaction) {
  images.value = images.value.map((image) =>
    image.path === imagePath
      ? {
          ...image,
          likes: interaction.likes,
          comments: interaction.comments,
        }
      : image,
  );

  if (previewImage.value?.path === imagePath) {
    previewImage.value = {
      ...previewImage.value,
      likes: interaction.likes,
      comments: interaction.comments,
    };
  }
}

async function likeImage(imagePath) {
  try {
    const response = await fetch(`${apiBaseUrl}/api/images/like`, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ imagePath }),
    });

    const data = await response.json();

    if (!response.ok) {
      throw new Error(data.message || `Error HTTP ${response.status}`);
    }

    updateImageInteraction(imagePath, data.interaction);
  } catch (err) {
    addToast(err?.message || "No se pudo registrar el like.", "error");
  }
}

async function submitComment(imagePath) {
  const text = commentText.value.trim();

  if (!text) {
    addToast("Escribe un comentario antes de enviar.", "error");
    return;
  }

  submittingComment.value = true;

  try {
    const response = await fetch(`${apiBaseUrl}/api/images/comment`, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        imagePath,
        author: commentAuthor.value.trim(),
        text,
      }),
    });

    const data = await response.json();

    if (!response.ok) {
      throw new Error(data.message || `Error HTTP ${response.status}`);
    }

    updateImageInteraction(imagePath, data.interaction);
    commentText.value = "";
    addToast("Comentario agregado.", "success");
  } catch (err) {
    addToast(err?.message || "No se pudo guardar el comentario.", "error");
  } finally {
    submittingComment.value = false;
  }
}

onMounted(fetchImages);
onMounted(() => window.addEventListener("keydown", handleKeydown));
onUnmounted(() => {
  window.removeEventListener("keydown", handleKeydown);
  if (localPreviewUrl.value) {
    URL.revokeObjectURL(localPreviewUrl.value);
  }
});
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
        <div class="stats-strip">
          <article class="stat-card">
            <p>Total Images</p>
            <h3>{{ imageCount }}</h3>
          </article>
          <article class="stat-card">
            <p>Storage</p>
            <h3>{{ totalStorageText }}</h3>
          </article>
          <article class="stat-card">
            <p>Latest Upload</p>
            <h3>{{ latestUploadText }}</h3>
          </article>
        </div>
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

      <Transition name="fade-slide" mode="out-in">
        <section v-if="activeScreen === 'upload'" class="upload-panel">
          <h2>Sube tus imagenes de perritos bonitos</h2>
          <p class="upload-help">
            Arrastra y suelta una imagen o selecciona un archivo desde tu equipo.
          </p>

          <div class="upload-layout">
            <div>
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
                <div
                  class="progress-bar"
                  :style="{ width: `${uploadProgress}%` }"
                ></div>
                <span>{{ uploadProgress }}%</span>
              </div>

              <p v-if="uploadError" class="upload-message error">{{ uploadError }}</p>
            </div>

            <aside class="local-preview-card">
              <p>Local Preview</p>
              <img
                v-if="localPreviewUrl"
                :src="localPreviewUrl"
                alt="Selected preview"
                class="local-preview-image"
              />
              <div v-else class="local-preview-empty">No image selected yet</div>
            </aside>
          </div>
        </section>

        <section v-else>
          <section v-if="loading" class="gallery skeleton-grid">
            <article v-for="(_, idx) in skeletonItems" :key="idx" class="card skeleton-card">
              <div class="skeleton-image"></div>
              <div class="meta">
                <div class="skeleton-line"></div>
                <div class="skeleton-line short"></div>
              </div>
            </article>
          </section>

          <section v-else-if="error" class="status error">{{ error }}</section>

          <section v-else-if="!images.length" class="empty-state">
            <h2>No dog photos yet</h2>
            <p>Sube tu primera imagen para empezar a llenar esta galeria.</p>
            <button type="button" @click="activeScreen = 'upload'">Ir a Upload</button>
          </section>

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
                <div class="card-stats">
                  <button
                    type="button"
                    class="tiny-btn"
                    @click.stop="likeImage(image.path)"
                  >
                    ❤️ {{ image.likes || 0 }}
                  </button>
                  <span>💬 {{ image.comments?.length || 0 }}</span>
                </div>
              </div>
              <div class="card-overlay">Click to preview</div>
            </article>
          </section>
        </section>
      </Transition>
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
        <div class="preview-grid">
          <section class="preview-media">
            <img
              :src="previewImage.url"
              :alt="previewImage.name"
              class="preview-image"
            />
            <p class="preview-title">{{ previewImage.name }}</p>
            <div class="preview-actions">
              <button type="button" class="tiny-btn" @click="likeImage(previewImage.path)">
                ❤️ {{ previewImage.likes || 0 }}
              </button>
              <span>{{ previewImage.comments?.length || 0 }} comentarios</span>
            </div>
          </section>

          <section class="comments-panel">
            <h3>Comentarios</h3>

            <form class="comment-form" @submit.prevent="submitComment(previewImage.path)">
              <input
                v-model="commentAuthor"
                type="text"
                placeholder="Tu nombre (opcional)"
                maxlength="40"
              />
              <textarea
                v-model="commentText"
                placeholder="Escribe un comentario bonito..."
                maxlength="300"
              ></textarea>
              <button type="submit" :disabled="submittingComment">
                {{ submittingComment ? "Enviando..." : "Comentar" }}
              </button>
            </form>

            <ul v-if="previewImage.comments?.length" class="comment-list">
              <li v-for="comment in previewImage.comments" :key="comment.id">
                <p class="comment-author">{{ comment.author || "Anonimo" }}</p>
                <p class="comment-text">{{ comment.text }}</p>
                <p class="comment-date">{{ new Date(comment.createdAt).toLocaleString() }}</p>
              </li>
            </ul>
            <p v-else class="no-comments">Aun no hay comentarios.</p>
          </section>
        </div>
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
