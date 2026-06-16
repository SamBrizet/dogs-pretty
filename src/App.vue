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
const commentText = ref("");
const submittingComment = ref(false);
const authToken = ref(localStorage.getItem("dogsPrettyAuthToken") || "");
const currentUser = ref(null);
const authLoading = ref(false);
const authError = ref("");
const authMode = ref("login");
const loginForm = ref({ username: "", password: "" });
const registerForm = ref({
  displayName: "",
  username: "",
  password: "",
  confirmPassword: "",
});
const authModalOpen = ref(false);

const skeletonItems = Array.from({ length: 6 });

const imageCount = computed(() => images.value.length);
const authInitials = computed(() => {
  const displayName = currentUser.value?.displayName || "Cuenta";
  return displayName
    .split(/\s+/)
    .filter(Boolean)
    .slice(0, 2)
    .map((word) => word[0]?.toUpperCase() || "")
    .join("");
});
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
    const headers = authToken.value
      ? { Authorization: `Bearer ${authToken.value}` }
      : {};

    const response = await fetch(`${apiBaseUrl}/api/images`, { headers });

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

async function fetchMe() {
  if (!authToken.value) {
    currentUser.value = null;
    return;
  }

  try {
    const response = await fetch(`${apiBaseUrl}/api/auth/me`, {
      headers: {
        Authorization: `Bearer ${authToken.value}`,
      },
    });

    const data = await response.json();

    if (!response.ok) {
      throw new Error(data.message || `Error HTTP ${response.status}`);
    }

    currentUser.value = data.user;
  } catch {
    authToken.value = "";
    currentUser.value = null;
    localStorage.removeItem("dogsPrettyAuthToken");
  }
}

async function loginUser() {
  authError.value = "";
  authLoading.value = true;

  try {
    const response = await fetch(`${apiBaseUrl}/api/auth/login`, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        username: loginForm.value.username,
        password: loginForm.value.password,
      }),
    });

    const data = await response.json();

    if (!response.ok) {
      throw new Error(data.message || "No se pudo iniciar sesion.");
    }

    authToken.value = data.token;
    currentUser.value = data.user;
    localStorage.setItem("dogsPrettyAuthToken", data.token);
    loginForm.value.password = "";
    addToast(`Hola ${data.user.displayName}`, "success");
    await fetchImages();
  } catch (err) {
    authError.value = err?.message || "No se pudo iniciar sesion.";
  } finally {
    authLoading.value = false;
  }
}

function logoutUser() {
  authToken.value = "";
  currentUser.value = null;
  localStorage.removeItem("dogsPrettyAuthToken");
  addToast("Sesion cerrada.", "success");
  fetchImages();
  authModalOpen.value = false;
}

function toggleAuthModal() {
  authModalOpen.value = !authModalOpen.value;
}

function closeAuthModal() {
  authModalOpen.value = false;
}

function setAuthMode(mode) {
  authMode.value = mode;
  authError.value = "";
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

async function registerUser() {
  authError.value = "";
  authLoading.value = true;

  if (registerForm.value.password !== registerForm.value.confirmPassword) {
    authError.value = "Las contrasenas no coinciden.";
    authLoading.value = false;
    return;
  }

  try {
    const response = await fetch(`${apiBaseUrl}/api/auth/register`, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        displayName: registerForm.value.displayName,
        username: registerForm.value.username,
        password: registerForm.value.password,
      }),
    });

    const data = await response.json();

    if (!response.ok) {
      throw new Error(data.message || "No se pudo registrar la cuenta.");
    }

    authToken.value = data.token;
    currentUser.value = data.user;
    localStorage.setItem("dogsPrettyAuthToken", data.token);
    registerForm.value = {
      displayName: "",
      username: "",
      password: "",
      confirmPassword: "",
    };
    setAuthMode("login");
    addToast(`Cuenta creada para ${data.user.displayName}`, "success");
    await fetchImages();
  } catch (err) {
    authError.value = err?.message || "No se pudo registrar la cuenta.";
  } finally {
    authLoading.value = false;
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
          likedByUser: interaction.likedByUser,
        }
      : image,
  );

  if (previewImage.value?.path === imagePath) {
    previewImage.value = {
      ...previewImage.value,
      likes: interaction.likes,
      comments: interaction.comments,
      likedByUser: interaction.likedByUser,
    };
  }
}

async function likeImage(imagePath) {
  if (!authToken.value) {
    addToast("Debes iniciar sesion para dar like.", "error");
    return;
  }

  try {
    const response = await fetch(`${apiBaseUrl}/api/images/like`, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        Authorization: `Bearer ${authToken.value}`,
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
  if (!authToken.value) {
    addToast("Debes iniciar sesion para comentar.", "error");
    return;
  }

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
        Authorization: `Bearer ${authToken.value}`,
      },
      body: JSON.stringify({
        imagePath,
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
onMounted(fetchMe);
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
          <button
            type="button"
            class="screen-btn"
            :class="{ active: authModalOpen }"
            @click="toggleAuthModal"
          >
            {{ currentUser ? currentUser.displayName : 'Cuenta' }}
          </button>
        </div>
        <div class="actions">
          <button type="button" @click="fetchImages">Recargar</button>
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
                    :class="{ liked: image.likedByUser }"
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

    <Transition name="zoom-fade">
      <section
        v-if="authModalOpen"
        class="auth-overlay"
        role="dialog"
        aria-modal="true"
        @click.self="closeAuthModal"
      >
        <div class="auth-modal">
          <button class="preview-close" type="button" @click="closeAuthModal">
            Cerrar
          </button>

          <div class="auth-modal-header">
            <div class="auth-avatar">{{ authInitials }}</div>
            <div>
              <h2>Cuenta</h2>
              <p class="auth-modal-copy">
                Inicia sesion para dar likes y comentar fotos.
              </p>
            </div>
          </div>

          <template v-if="currentUser">
            <div class="auth-user-card">
              <p>Conectado como</p>
              <h3>{{ currentUser.displayName }}</h3>
              <small>@{{ currentUser.username }}</small>
            </div>
            <button type="button" class="tiny-btn" @click="logoutUser">
              Cerrar sesion
            </button>
          </template>

          <template v-else>
            <div class="auth-tabs">
              <button
                type="button"
                class="screen-btn"
                :class="{ active: authMode === 'login' }"
                @click="setAuthMode('login')"
              >
                Iniciar sesion
              </button>
              <button
                type="button"
                class="screen-btn"
                :class="{ active: authMode === 'register' }"
                @click="setAuthMode('register')"
              >
                Registrarse
              </button>
            </div>

            <form
              v-if="authMode === 'login'"
              class="login-form modal-login"
              @submit.prevent="loginUser"
            >
              <input
                v-model="loginForm.username"
                type="text"
                placeholder="Usuario"
                required
              />
              <input
                v-model="loginForm.password"
                type="password"
                placeholder="Contrasena"
                required
              />
              <button type="submit" :disabled="authLoading">
                {{ authLoading ? "Entrando..." : "Iniciar sesion" }}
              </button>
            </form>

            <form
              v-else
              class="login-form modal-login"
              @submit.prevent="registerUser"
            >
              <input
                v-model="registerForm.displayName"
                type="text"
                placeholder="Nombre visible"
                required
              />
              <input
                v-model="registerForm.username"
                type="text"
                placeholder="Usuario"
                required
              />
              <input
                v-model="registerForm.password"
                type="password"
                placeholder="Contrasena"
                required
              />
              <input
                v-model="registerForm.confirmPassword"
                type="password"
                placeholder="Confirmar contrasena"
                required
              />
              <button type="submit" :disabled="authLoading">
                {{ authLoading ? "Creando cuenta..." : "Crear cuenta" }}
              </button>
            </form>

            <p v-if="authMode === 'login'" class="auth-hint modal-hint">
              Demo: usuario demo / contrasena demo123
            </p>
          </template>

          <p v-if="authError" class="upload-message error">{{ authError }}</p>
        </div>
      </section>
    </Transition>

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
              <button
                type="button"
                class="tiny-btn"
                :class="{ liked: previewImage.likedByUser }"
                @click="likeImage(previewImage.path)"
              >
                ❤️ {{ previewImage.likes || 0 }}
              </button>
              <span>{{ previewImage.comments?.length || 0 }} comentarios</span>
            </div>
          </section>

          <section class="comments-panel">
            <h3>Comentarios</h3>

            <form class="comment-form" @submit.prevent="submitComment(previewImage.path)">
              <textarea
                v-model="commentText"
                placeholder="Escribe un comentario bonito..."
                maxlength="300"
                :disabled="!currentUser"
              ></textarea>
              <button type="submit" :disabled="submittingComment || !currentUser">
                {{ submittingComment ? "Enviando..." : "Comentar" }}
              </button>
              <p v-if="!currentUser" class="auth-hint">
                Inicia sesion para comentar.
              </p>
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
