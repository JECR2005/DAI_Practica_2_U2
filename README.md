# 🤖 Agente de Soporte con Python, Gemini y GitHub Actions

> **Práctica 2 – DAI**: Automatización del Agente de Soporte en la nube con GitHub Actions y publicación del proyecto en GitHub Pages.

---

## 📋 Descripción General

Este proyecto implementa un **agente autónomo** que monitorea continuamente una hoja de cálculo de Google Sheets, detecta nuevas quejas o comentarios de clientes y, usando inteligencia artificial (Google Gemini 2.5 Flash), los analiza automáticamente para:

1. **Clasificarlos** según el área responsable: `Ventas`, `Soporte Técnico` o `Logística`.
2. **Detectar el sentimiento** del cliente: `Positivo`, `Negativo` o `Neutro`.
3. **Escribir los resultados** directamente en la hoja de cálculo.
4. **Ejecutarse automáticamente** cada hora mediante **GitHub Actions**, sin necesidad de un servidor propio.

---

## 🏗️ Arquitectura del Sistema

```
┌──────────────────────────────────────────────────────────────┐
│                     GitHub Actions (nube)                     │
│                                                              │
│  Trigger: cron 0 * * * *  /  push main  /  manual           │
│                    │                                         │
│                    ▼                                         │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              agente_soporte.py --una-vez             │    │
│  │                                                     │    │
│  │  ┌────────────┐   ┌──────────────┐   ┌──────────┐  │    │
│  │  │  Google    │──▶│ ejecutar_    │──▶│  Gemini  │  │    │
│  │  │  Sheets    │   │ agente()     │   │ 2.5 Flash│  │    │
│  │  │ (gspread)  │◀──│              │◀──│  (IA)    │  │    │
│  │  └────────────┘   └──────────────┘   └──────────┘  │    │
│  └─────────────────────────────────────────────────────┘    │
│       Secrets: GEMINI_API_KEY · GOOGLE_TOKEN_JSON            │
└──────────────────────────────────────────────────────────────┘
```

---

## 📁 Estructura de Archivos

```
DAI_Practica_2_U2/
│
├── agente_soporte.py              # Script principal del agente
├── requirements.txt               # Dependencias Python
├── .env                           # API Key de Gemini (NO subir a GitHub)
├── .env.example                   # Plantilla pública de variables de entorno
├── .gitignore                     # Excluye archivos sensibles de Git
├── README.md                      # Este archivo
│
├── .github/
│   └── workflows/
│       └── agente.yml             # Workflow de GitHub Actions
│
└── docs/                          # GitHub Pages (sitio web del proyecto)
    ├── index.html
    └── style.css
```

> ⚠️ **Importante:** Los archivos `.env` y `client_secrets.json` contienen credenciales privadas y están excluidos del repositorio mediante `.gitignore`. **Nunca los subas a GitHub.** Las credenciales en la nube se gestionan como **Secrets** en GitHub.

---

## ⚙️ Requisitos Previos

### Software
- **Python 3.12** o superior
- **pip** (gestor de paquetes de Python)

### Cuentas y Servicios Externos
- **Google Cloud Console** — para crear credenciales OAuth 2.0
- **Google AI Studio** — para obtener la API Key de Gemini
- **Google Sheets** — hoja de cálculo con el nombre exacto: `Respuestas de Quejas (Base de Datos)`
- **GitHub** — repositorio con GitHub Actions y GitHub Pages habilitados

### Instalar dependencias

```bash
pip install -r requirements.txt
```

O de forma manual:

```bash
pip install google-genai gspread schedule python-dotenv google-auth google-auth-oauthlib google-auth-httplib2
```

---

## 🔐 Configuración de Credenciales

### 1. API Key de Gemini (uso local y GitHub Actions)

1. Ve a [Google AI Studio](https://aistudio.google.com/app/api-keys)
2. Crea una nueva API Key
3. **Localmente**: crea el archivo `.env`:
   ```env
   GEMINI_API_KEY=tu_api_key_aqui
   ```
4. **En GitHub**: ve a *Settings → Secrets → Actions* y crea el secret `GEMINI_API_KEY`.

### 2. Credenciales OAuth 2.0 de Google Sheets (uso local)

1. Ve a [Google Cloud Console](https://console.cloud.google.com/)
2. Activa las APIs: **Google Sheets API** y **Google Drive API**
3. Crea una credencial **ID de cliente OAuth 2.0** (tipo: Aplicación de escritorio)
4. Descarga el JSON y renómbralo `client_secrets.json`
5. Colócalo junto a `agente_soporte.py` y ejecuta el agente una vez para autorizar

### 3. GOOGLE_TOKEN_JSON — Secret para GitHub Actions (uso en la nube)

Después de autorizar localmente, gspread guarda el token en:
```
~/.config/gspread/authorized_user.json    # Linux/Mac
%APPDATA%\gspread\authorized_user.json    # Windows
```

Copia ese contenido (es un JSON) y créalo como secret en GitHub con el nombre **`GOOGLE_TOKEN_JSON`**.

---

## 📊 Estructura de la Hoja de Google Sheets

La hoja debe llamarse exactamente **`Respuestas de Quejas (Base de Datos)`**:

| Columna | Letra | Contenido | Quién lo escribe |
|---|---|---|---|
| 1 | A | Marca de tiempo / ID | Usuario / Formulario |
| 2 | B | Nombre del cliente | Usuario / Formulario |
| 3 | C | Comentario o queja | Usuario / Formulario |
| 4 | **D** | **Clasificación** | **El agente (IA)** |
| 5 | **E** | **Sentimiento** | **El agente (IA)** |
| 6 | **F** | **Ejecutado (SI/NO)** | **El agente (IA)** |

> La **fila 1** se trata como encabezado y es ignorada. El procesamiento comienza desde la **fila 2**.

---

## 🚀 Cómo Ejecutar el Agente

### Modo local (bucle continuo)

```bash
python agente_soporte.py
```

El agente revisará la hoja cada `INTERVALO_MINUTOS` (1 min por defecto).

### Modo una sola ejecución (GitHub Actions)

```bash
python agente_soporte.py --una-vez
```

Procesa todas las filas pendientes y termina. Retorna código de salida `0` (éxito) o `1` (error de conexión).

---

## ⚙️ GitHub Actions Workflow

El archivo `.github/workflows/agente.yml` ejecuta el agente automáticamente:

| Trigger | Cuándo ocurre |
|---|---|
| `schedule: cron 0 * * * *` | Cada hora en punto |
| `push: branches: main` | Al subir cambios a `main` |
| `workflow_dispatch` | Manualmente desde GitHub |

**Pasos del workflow:**
1. Clona el repositorio (`actions/checkout@v4`)
2. Configura Python 3.12 (`actions/setup-python@v5`)
3. Instala dependencias (`pip install -r requirements.txt`)
4. Ejecuta el agente con `--una-vez`, inyectando los secrets como variables de entorno

---

## 🔄 Flujo de Ejecución Detallado

```
INICIO
  │
  ▼
¿GOOGLE_TOKEN_JSON en env? ── SÍ ──▶ Autenticar con token JSON (modo nube)
  │ NO
  ▼
Autenticar con client_secrets.json (modo local, abre navegador 1ª vez)
  │
  ▼
hoja.get_all_values() → lee todas las filas
  │
  ▼
Para cada fila (desde fila 2):
  ├─ ¿Columna F == "SI"? ──▶ OMITIR (ya procesada)
  ├─ ¿Comentario vacío?  ──▶ OMITIR
  └─ analizar_con_gemini(comentario)
        │
        ▼
     Gemini devuelve: {"clasificacion": "...", "sentimiento": "..."}
        │
        ▼
     Escribe en Sheets: Col D ← clasificación | Col E ← sentimiento | Col F ← "SI"
  │
  ▼
FIN → retorna True/False
```

---

## 🤖 Comportamiento de la IA (Gemini 2.5 Flash)

**Prompt de sistema:**
```
Eres un analista experto de soporte al cliente.
Tu única tarea es analizar el comentario recibido y devolver
una respuesta EXCLUSIVAMENTE en formato JSON puro.
El JSON debe contener exactamente dos claves:
  "clasificacion" → "Ventas" | "Soporte Tecnico" | "Logistica"
  "sentimiento"   → "Positivo" | "Negativo" | "Neutro"
```

**Ejemplo:**
```
Entrada:  "Mi pedido llegó roto y nadie me ha dado solución."
Salida:   {"clasificacion": "Logistica", "sentimiento": "Negativo"}
```

---

## 🛡️ Manejo de Errores

| Error | Comportamiento |
|---|---|
| `GEMINI_API_KEY` no definida | El programa se detiene con mensaje claro |
| `client_secrets.json` ausente (modo local) | El programa se detiene con mensaje claro |
| Error de conexión con Google Sheets | Se aborta el ciclo; GitHub Actions lo reintentará en 1 hora |
| `JSONDecodeError` en respuesta de Gemini | Se omite la fila y se continúa |
| Error al escribir en Sheets | Se muestra el error y se continúa |

---

## 🔧 Parámetros de Configuración

| Constante | Valor | Descripción |
|---|---|---|
| `ARCHIVO_CREDENCIALES` | `"client_secrets.json"` | Credenciales OAuth locales |
| `NOMBRE_HOJA` | `"Respuestas de Quejas (Base de Datos)"` | Nombre exacto de la hoja |
| `INTERVALO_MINUTOS` | `1` | Frecuencia del bucle local (minutos) |
| `COL_CLASIFICACION` | `4` (col D) | Columna de clasificación |
| `COL_SENTIMIENTO` | `5` (col E) | Columna de sentimiento |
| `COL_EJECUTADO` | `6` (col F) | Columna de control |

---

## 🌐 GitHub Pages

El directorio `docs/` contiene el sitio web del proyecto, publicado en GitHub Pages.

Para activarlo: *Settings → Pages → Source: Deploy from branch → Branch: main → Folder: /docs*

---

## 🔒 Seguridad y Buenas Prácticas

- **`.env`** — API Key local. Excluido de Git con `.gitignore`.
- **`client_secrets.json`** — Credenciales OAuth locales. Excluido de Git.
- **`.env.example`** — Plantilla pública sin valores reales.
- **GitHub Secrets** — `GEMINI_API_KEY` y `GOOGLE_TOKEN_JSON` para la nube.
- **`python-dotenv`** — Carga variables en tiempo de ejecución, sin valores hardcodeados.

---

## 📝 Licencia

Proyecto de práctica educativa — Desarrollo de Aplicaciones con Inteligencia Artificial (DAI) · Práctica 2.


---

## 🏗️ Arquitectura del Sistema

```
┌─────────────────────────────────────────────────────────────┐
│                      agente_soporte.py                       │
│                                                             │
│  ┌─────────────┐     ┌──────────────────┐    ┌──────────┐  │
│  │ Google      │────▶│  ejecutar_agente  │───▶│ Gemini   │  │
│  │ Sheets      │     │  (orquestador)    │    │ 2.5 Flash│  │
│  │ (gspread)   │◀────│                  │◀───│ (IA)     │  │
│  └─────────────┘     └──────────────────┘    └──────────┘  │
│         ▲                                                   │
│         │  OAuth 2.0                                        │
│  client_secrets.json                                        │
└─────────────────────────────────────────────────────────────┘
              │
         Loop cada 15 min (schedule)
```

---

## 📁 Estructura de Archivos

```
Practica_DAI_Agente_Python/
│
├── agente_soporte.py       # Script principal del agente
├── client_secrets.json     # Credenciales OAuth 2.0 de Google (NO subir a GitHub)
├── .env                    # API Key de Gemini (NO subir a GitHub)
├── .env.example            # Plantilla pública de variables de entorno
├── .gitignore              # Excluye archivos sensibles de Git
└── README.md               # Este archivo
```

> ⚠️ **Importante:** Los archivos `.env` y `client_secrets.json` contienen credenciales privadas y están excluidos del repositorio mediante `.gitignore`. **Nunca los subas a GitHub.**

---

## ⚙️ Requisitos Previos

### Software
- **Python 3.11** o superior
- **pip** (gestor de paquetes de Python)

### Cuentas y Servicios Externos
- **Google Cloud Console** — para crear credenciales OAuth 2.0
- **Google AI Studio** — para obtener la API Key de Gemini
- **Google Sheets** — hoja de cálculo con el nombre exacto: `Respuestas de Quejas (Base de Datos)`

### Librerías Python

| Librería | Función |
|---|---|
| `google-genai` | Cliente oficial de la API de Google Gemini |
| `gspread` | Lectura y escritura en Google Sheets |
| `schedule` | Ejecución periódica de tareas (cada 15 min) |
| `python-dotenv` | Carga de variables de entorno desde `.env` |

Instálalas con:

```bash
pip install google-genai gspread schedule python-dotenv
```

---

## 🔐 Configuración de Credenciales

### 1. API Key de Gemini

1. Ve a [Google AI Studio](https://aistudio.google.com/app/api-keys)
2. Crea una nueva API Key
3. Crea el archivo `.env` en la carpeta del proyecto:

```env
GEMINI_API_KEY=tu_api_key_aqui
```

### 2. Credenciales OAuth 2.0 de Google Sheets

1. Ve a [Google Cloud Console](https://console.cloud.google.com/)
2. Crea un proyecto (o usa uno existente)
3. Activa las APIs: **Google Sheets API** y **Google Drive API**
4. Ve a **APIs y servicios > Credenciales**
5. Crea una credencial de tipo **ID de cliente OAuth 2.0** (tipo: Aplicación de escritorio)
6. Descarga el archivo JSON y renómbralo como `client_secrets.json`
7. Colócalo en la misma carpeta que `agente_soporte.py`

---

## 📊 Estructura de la Hoja de Google Sheets

El agente espera una hoja llamada exactamente **`Respuestas de Quejas (Base de Datos)`** con la siguiente estructura de columnas:

| Columna | Letra | Contenido | Quién lo escribe |
|---|---|---|---|
| 1 | A | Marca de tiempo / ID | Usuario / Formulario |
| 2 | B | Nombre del cliente | Usuario / Formulario |
| 3 | C | Comentario o queja | Usuario / Formulario |
| 4 | **D** | **Clasificación** | **El agente (IA)** |
| 5 | **E** | **Sentimiento** | **El agente (IA)** |
| 6 | **F** | **Ejecutado (SI/NO)** | **El agente (IA)** |

> La **fila 1** se trata como encabezado y es ignorada por el agente. El procesamiento comienza desde la **fila 2**.

---

## 🚀 Cómo Ejecutar el Agente

```bash
python agente_soporte.py
```

Al arrancar, el agente realiza las siguientes validaciones:

1. ✅ Verifica que la variable `GEMINI_API_KEY` esté definida en `.env`
2. ✅ Verifica que el archivo `client_secrets.json` exista
3. 🔑 En la primera ejecución, abrirá el navegador para autorizar el acceso a Google Sheets mediante OAuth 2.0

Una vez autorizado, el agente iniciará su ciclo de trabajo de forma continua.

---

## 🔄 Flujo de Ejecución Detallado

### Ciclo principal (`ejecutar_agente`)

```
INICIO DEL CICLO
       │
       ▼
1. conectar_sheets()
   └─ Autentica con Google vía OAuth 2.0
   └─ Abre la hoja "Respuestas de Quejas (Base de Datos)"
       │
       ▼
2. hoja.get_all_values()
   └─ Lee TODAS las filas de la hoja
   └─ Omite la fila 1 (encabezado)
       │
       ▼
3. Para cada fila de datos:
   │
   ├─ ¿Columna F == "SI"? ──▶ OMITIR (ya procesada)
   │
   ├─ ¿Comentario vacío?  ──▶ OMITIR
   │
   └─ analizar_con_gemini(comentario)
         │
         ▼
      Envía a Gemini 2.5 Flash con instrucción de sistema:
      "Devuelve JSON con 'clasificacion' y 'sentimiento'"
         │
         ▼
      Recibe: {"clasificacion": "Ventas", "sentimiento": "Negativo"}
         │
         ▼
      Escribe en Google Sheets:
        - Col D ← clasificacion
        - Col E ← sentimiento
        - Col F ← "SI"
       │
       ▼
FIN DEL CICLO → Esperar 15 minutos → REPETIR
```

### Bucle continuo

```python
# Primera ejecución inmediata al arrancar
ejecutar_agente()

# Luego, se repite automáticamente cada 15 minutos
schedule.every(15).minutes.do(ejecutar_agente)

while True:
    schedule.run_pending()
    time.sleep(30)   # revisa cada 30 segundos si hay tareas pendientes
```

---

## 🤖 Comportamiento de la IA (Gemini 2.5 Flash)

El agente usa un **prompt de sistema** específicamente diseñado para obtener respuestas estructuradas y predecibles:

```
Eres un analista experto de soporte al cliente.
Tu única tarea es analizar el comentario recibido y devolver
una respuesta EXCLUSIVAMENTE en formato JSON puro, sin texto adicional,
sin explicaciones y sin bloques de código markdown.
El JSON debe contener exactamente dos claves:
  "clasificacion" (valores posibles: "Ventas", "Soporte Tecnico" o "Logistica")
  "sentimiento"   (valores posibles: "Positivo", "Negativo" o "Neutro")
```

**Ejemplo de entrada:**
```
"Mi pedido llegó roto y nadie me ha dado solución después de 3 llamadas."
```

**Ejemplo de respuesta de Gemini:**
```json
{"clasificacion": "Logistica", "sentimiento": "Negativo"}
```

El agente también incluye una limpieza defensiva para eliminar posibles bloques de código Markdown (` ```json `) que Gemini podría devolver accidentalmente.

---

## 🛡️ Manejo de Errores

El agente está diseñado para ser **resiliente**: si una fila falla, continúa con las demás.

| Error | Comportamiento |
|---|---|
| `GEMINI_API_KEY` no definida | El programa se detiene con mensaje claro al arrancar |
| `client_secrets.json` no encontrado | El programa se detiene con mensaje claro al arrancar |
| Error de conexión con Google Sheets | Se aborta el ciclo actual; se reintenta en 15 min |
| `JSONDecodeError` en respuesta de Gemini | Se omite la fila y se continúa con la siguiente |
| Error al escribir en Sheets | Se muestra el error y se continúa con la siguiente fila |

---

## 🔧 Parámetros de Configuración

Todos los parámetros configurables se encuentran al inicio de `agente_soporte.py`:

| Constante | Valor por defecto | Descripción |
|---|---|---|
| `ARCHIVO_CREDENCIALES` | `"client_secrets.json"` | Nombre del archivo de credenciales OAuth |
| `NOMBRE_HOJA` | `"Respuestas de Quejas (Base de Datos)"` | Nombre exacto de la hoja en Google Sheets |
| `INTERVALO_MINUTOS` | `15` | Frecuencia de revisión en minutos |
| `COL_CLASIFICACION` | `4` (columna D) | Columna donde se escribe la clasificación |
| `COL_SENTIMIENTO` | `5` (columna E) | Columna donde se escribe el sentimiento |
| `COL_EJECUTADO` | `6` (columna F) | Columna de control (marca `SI` al procesar) |

---

## 🔒 Seguridad y Buenas Prácticas

Este proyecto sigue las mejores prácticas para proteger las credenciales:

- **`.env`** — Almacena la API Key de Gemini. Nunca se sube a GitHub (incluido en `.gitignore`).
- **`client_secrets.json`** — Almacena las credenciales OAuth de Google. Nunca se sube a GitHub (incluido en `.gitignore`).
- **`.env.example`** — Plantilla pública sin valores reales, que sirve de guía para otros desarrolladores.
- **`python-dotenv`** — Carga las variables de entorno en tiempo de ejecución, evitando valores hardcodeados en el código.

---

## 📝 Licencia

Proyecto de práctica educativa — Desarrollo de Aplicaciones con Inteligencia Artificial (DAI).
