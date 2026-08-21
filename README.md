**English version:** [README.en.md](./README.en.md)

# 📄 Sheets JSON Publisher

Este proyecto genera archivos JSON desde Google Sheets usando Google Apps Script y los publica automáticamente en un repositorio de GitHub.

El objetivo es evitar que el frontend consulte directamente Google Sheets en cada carga. En su lugar, la web consume un JSON estático, liviano y rápido de leer.

---

## ✨ Descripción

`Sheets JSON Publisher` convierte datos editables desde Google Sheets en archivos JSON versionados en GitHub.

El flujo general es:

```text
Google Sheets → Apps Script → JSON → GitHub → Frontend
```

De esta forma, Google Sheets funciona como fuente de datos editable, mientras que GitHub actúa como capa de publicación/entrega del JSON.

---

## 🧠 ¿Por qué existe este proyecto?

Inicialmente, el frontend podía hacer una petición a Google Apps Script para generar el JSON en el momento.

Ese enfoque tenía algunos problemas:

- El usuario final debía esperar la generación del JSON.
- Google Sheets no está pensado como base de datos de alta concurrencia.
- La respuesta podía ser lenta.
- Se podían consumir cuotas o límites de Google Apps Script.
- Cada visita podía disparar una lectura innecesaria a la hoja.

Por eso se cambió el modelo:

```text
Usuario final
   ↓
Frontend
   ↓
JSON estático en GitHub
   ↑
Apps Script publica el JSON
   ↑
Google Sheets como fuente editable
```

Así, el usuario no espera la generación de datos, sino que lee un archivo ya preparado.

---

## 🏗 Arquitectura

```text
┌──────────────────┐
│  Google Sheets   │
└────────┬─────────┘
         │ edición de datos
         ▼
┌──────────────────┐
│  Apps Script     │
│  - lee datos     │
│  - arma JSON     │
│  - hace commit   │
└────────┬─────────┘
         │ GitHub API
         ▼
┌──────────────────┐
│  Repositorio     │
│  GitHub          │
│  data/*.json     │
└────────┬─────────┘
         │ lectura pública
         ▼
┌──────────────────┐
│  Frontend / Web  │
└──────────────────┘
```

---

## 📦 ¿Qué hace?

- Lee datos desde una o varias hojas de Google Sheets.
- Transforma esos datos en JSON.
- Publica el JSON en este repositorio mediante la API de GitHub.
- Permite actualizaciones:
  - manuales mediante botón o menú en Google Sheets,
  - programadas cada cierto tiempo,
  - automáticas al editar la hoja,
  - o por caso específico.
- El frontend consume el JSON desde `raw.githubusercontent.com`.

---

## 📁 Estructura del repositorio

```text
/
├── data/
│   ├── cache.json
│   ├── caso-a.json
│   ├── caso-b.json
│   └── caso-c.json
├── README.md
└── LICENSE
```

Los archivos JSON generados se guardan principalmente dentro de la carpeta:

```text
data/
```

---

## 🔗 Consumo del JSON

Actualmente el frontend consume los datos desde GitHub Raw:

```text
https://raw.githubusercontent.com/TU_USUARIO/TU_REPO/main/data/cache.json
```

Ejemplo en JavaScript:

```js
const JSON_URL =
  'https://raw.githubusercontent.com/TU_USUARIO/TU_REPO/main/data/cache.json';

fetch(JSON_URL)
  .then(response => response.json())
  .then(data => {
    console.log(data);
  })
  .catch(error => {
    console.error('Error cargando el JSON:', error);
  });
```

Ejemplo con `async/await`:

```js
const JSON_URL =
  'https://raw.githubusercontent.com/TU_USUARIO/TU_REPO/main/data/cache.json';

async function loadData() {
  try {
    const response = await fetch(JSON_URL);

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }

    const data = await response.json();

    console.log('Actualizado:', data.generatedAt);
    console.log('Elementos:', data.count);

    return data.items;
  } catch (error) {
    console.error('No se pudo cargar el JSON:', error);
    return [];
  }
}

loadData();
```

---

## 🧾 Formato esperado del JSON

Cada archivo JSON puede incluir metadatos básicos y los datos principales.

Ejemplo:

```json
{
  "generatedAt": "2026-01-01T12:00:00.000Z",
  "source": "Google Sheets",
  "count": 2,
  "items": [
    {
      "id": 1,
      "nombre": "Ejemplo A",
      "estado": "activo"
    },
    {
      "id": 2,
      "nombre": "Ejemplo B",
      "estado": "inactivo"
    }
  ]
}
```

Campos principales:

| Campo | Descripción |
|---|---|
| `generatedAt` | Fecha y hora de generación del JSON. |
| `source` | Origen de los datos. |
| `count` | Cantidad de elementos generados. |
| `items` | Arreglo con los datos principales. |

---

## ⚙️ Configuración

### 1. Repositorio de GitHub

Crea un repositorio donde se publicarán los JSON.

Por ejemplo:

```text
https://github.com/TU_USUARIO/TU_REPO
```

Dentro del repositorio se espera una carpeta:

```text
data/
```

---

### 2. Google Sheets

La hoja de cálculo es la fuente de datos.

Ejemplo de estructura:

| id | nombre | estado |
|----|--------|--------|
| 1  | Ejemplo A | activo |
| 2  | Ejemplo B | inactivo |

La primera fila se usa como encabezados.

Cada fila se convierte en un objeto JSON.

Ejemplo:

```json
{
  "id": 1,
  "nombre": "Ejemplo A",
  "estado": "activo"
}
```

---

### 3. Google Apps Script

El script debe estar vinculado a Google Sheets o tener acceso a la hoja correspondiente.

Apps Script se encarga de:

1. Leer la hoja.
2. Convertir los datos a JSON.
3. Enviar el archivo a GitHub usando la API de Contents.

---

### 4. Token de GitHub

Se requiere un token de GitHub con permisos para escribir en este repositorio.

Recomendación:

- Usar un token de acceso fino/granular.
- Dar permiso solo al repositorio necesario.
- Dar permiso de lectura/escritura sobre contenidos.
- No exponer el token en el frontend.
- No subir el token al repositorio.

---

### 5. Script Properties

En Apps Script, guarda el token como una propiedad del script.

Ruta:

```text
Apps Script → Project Settings → Script Properties
```

Agrega una propiedad:

| Nombre | Valor |
|---|---|
| `GITHUB_TOKEN` | token de GitHub |

Ejemplo:

```text
GITHUB_TOKEN = ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

> ⚠️ No coloques el token directamente en el código.

---

## 🔐 Seguridad

- El token de GitHub se guarda en Apps Script como Script Property.
- El token no debe exponerse en el frontend.
- El token no debe publicarse en el repositorio.
- Si el JSON contiene información sensible, no debería publicarse en un repositorio público.

Para datos privados, considerar:

- repositorio privado con backend autenticado,
- Cloudflare Access,
- Supabase,
- Firebase,
- API propia,
- signed URLs,
- u otro sistema de control de acceso.

---

## 🕒 Actualización automática

La publicación puede ejecutarse mediante triggers de Google Apps Script.

### Actualización cada X horas

Se puede programar un trigger para ejecutar la publicación cada cierto tiempo.

Ejemplos:

```text
cada 1 hora
cada 3 horas
cada 6 horas
cada 12 horas
cada 24 horas
```

---

### Actualización al editar la hoja

También se puede usar un trigger instalable `onEdit`.

Para evitar publicaciones excesivas, se recomienda usar debounce.

Ejemplo conceptual:

```text
Usuario edita hoja
   ↓
Apps Script detecta cambio
   ↓
Espera 30-60 segundos
   ↓
Si no hay más cambios, publica JSON
```

Esto evita publicar por cada tecla presionada.

---

## 🖱 Publicación manual

La publicación también puede ejecutarse manualmente desde Google Sheets.

Opciones:

- menú personalizado,
- botón insertado como imagen o dibujo,
- funciones separadas por caso.

Ejemplo de menú:

```text
Publicar JSON
├── Publicar todo
├── Publicar caso A
├── Publicar caso B
└── Publicar caso C
```

Esto permite generar archivos específicos sin actualizar todo el contenido.

---

## 📄 Archivos por caso

Si existen varios casos o categorías, se pueden generar archivos separados:

```text
data/cache.json
data/caso-a.json
data/caso-b.json
data/caso-c.json
```

Ejemplo de consumo:

```js
fetch('https://raw.githubusercontent.com/TU_USUARIO/TU_REPO/main/data/caso-a.json')
  .then(response => response.json())
  .then(console.log);
```

---

## 🧪 Ejemplo de uso en frontend

```js
const DATA_URL =
  'https://raw.githubusercontent.com/TU_USUARIO/TU_REPO/main/data/cache.json';

async function fetchData() {
  try {
    const response = await fetch(DATA_URL);

    if (!response.ok) {
      throw new Error(`Error HTTP: ${response.status}`);
    }

    const data = await response.json();

    renderData(data.items);
  } catch (error) {
    console.error('Error al obtener datos:', error);
  }
}

function renderData(items) {
  const container = document.getElementById('app');

  if (!container) return;

  container.innerHTML = '';

  items.forEach(item => {
    const card = document.createElement('div');
    card.className = 'card';
    card.textContent = item.nombre || 'Sin nombre';
    container.appendChild(card);
  });
}

fetchData();
```

---

## 🚀 Estado actual

Actualmente el proyecto consume los JSON directamente desde:

```text
raw.githubusercontent.com
```

Esta solución es suficiente por ahora porque los archivos JSON son livianos y la lectura es rápida.

Más adelante se podría evaluar el uso de un CDN como:

- Cloudflare Workers,
- Cloudflare Pages,
- jsDelivr,
- Vercel Edge Functions,
- Netlify Edge Functions,
- u otra plataforma de cacheo.

Esto permitiría mejorar:

- velocidad,
- control de caché,
- invalidación,
- disponibilidad,
- headers HTTP,
- y estrategias como `stale-while-revalidate`.

---

## 📌 Limitaciones actuales

- `raw.githubusercontent.com` puede aplicar caché durante algunos minutos.
- La actualización no es necesariamente instantánea.
- Google Apps Script tiene límites de ejecución y cuotas.
- Google Sheets no es ideal como base de datos de alta concurrencia.
- Si el volumen de datos crece mucho, conviene migrar a otra solución.

---

## 🧰 Stack

- Google Sheets
- Google Apps Script
- GitHub API
- GitHub Raw
- JavaScript
- JSON

---

## 🧯 Troubleshooting

### El JSON no aparece

Verifica que Apps Script haya hecho commit correctamente.

Revisa:

```text
data/cache.json
```

También revisa el historial de commits del repositorio.

---

### Error de autenticación con GitHub

Posibles causas:

- token inválido,
- token vencido,
- token sin permisos sobre el repositorio,
- token sin permiso de escritura,
- nombre de usuario, repo o rama incorrectos.

Revisa la propiedad:

```text
GITHUB_TOKEN
```

---

### El JSON no se actualiza

Verifica:

- si el trigger está instalado,
- si la función de publicación se ejecuta,
- si hay errores en las ejecuciones de Apps Script,
- si la hoja tiene datos válidos,
- si GitHub está recibiendo el commit.

---

### El frontend muestra datos antiguos

Puede ser caché de:

- navegador,
- GitHub Raw,
- service worker,
- CDN,
- caché HTTP local.

Para pruebas, puedes usar DevTools con caché deshabilitada.

Más adelante, si se usa un CDN, se podrá implementar invalidación o versionado por commit.

---

## 🚀 Roadmap

Próximas mejoras posibles:

- [ ] Usar jsDelivr como CDN.
- [ ] Usar Cloudflare Workers para cacheo inteligente.
- [ ] Implementar `stale-while-revalidate`.
- [ ] Publicar un endpoint de estado.
- [ ] Generar versiones por commit para invalidar caché.
- [ ] Agregar logs de publicación.
- [ ] Agregar validación de datos antes de publicar.
- [ ] Agregar soporte para múltiples hojas.
- [ ] Agregar soporte para múltiples archivos.
- [ ] Migrar a Supabase o Firebase si se necesita lectura en tiempo real.
- [ ] Usar GitHub Actions como pipeline alternativo.

---

## 📄 Licencia

Este proyecto está bajo licencia MIT, salvo que se indique lo contrario.

---

## 🙌 Créditos

Este proyecto usa Google Sheets como fuente editable, Google Apps Script como generador de JSON y GitHub como almacenamiento/entrega de archivos estáticos.
