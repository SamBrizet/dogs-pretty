# Dogs Pretty Frontend

Frontend en Vue 3 + Vite para galeria y subida de imagenes de perritos.

## Stack

- Vue 3
- Vite
- CSS puro

## Requisitos

- Node.js 18+

## Variables de entorno

Crear un archivo `.env` en esta carpeta:

```env
VITE_API_BASE_URL=http://localhost:3000
```

## Scripts

- `npm run dev`: levanta frontend local
- `npm run build`: compila para produccion
- `npm run preview`: vista local del build

## Funcionalidades

- Galeria de imagenes desde GCP
- Preview tipo lightbox
- Pantalla Upload con:
	- Drag and drop
	- Barra de progreso real
	- Toasts de exito/error
