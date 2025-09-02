# agente_onboarding_contable
Agente Onboarding Contable
# Onboarding Copilot (Microsoft 365) – Proyecto Final

> **Objetivo:** Diseñar y demostrar un agente de onboarding para el estudio (texto→texto y texto→imagen) usando **Microsoft Copilot Studio**, **Power Automate**, **SharePoint** y **Azure OpenAI**. El agente responde preguntas sobre procedimientos internos y genera recursos visuales (diagramas de flujo, organigramas y pósters de buenas prácticas), guardando todo en una biblioteca de SharePoint.

---

## 1) Arquitectura (visión general)

```
Usuario (Teams/Web) ──▶ Copilot (Copilot Studio)
                            │
                            ├─ Generative Answers (Conocimiento)
                            │     └─ SharePoint: /Procedimientos del Estudio
                            │
                            └─ Acciones (Actions)
                                  ├─ Power Automate: «GenerateImageFromPrompt» → Azure OpenAI (DALL·E)
                                  └─ Power Automate: «SaveToSharePoint» (crear archivo + metadatos)

Resultado: Respuesta + (opcional) imagen en SharePoint (Visuales/)
```

---

## 2) Estructura del repositorio

```
/onboarding-copilot/
├─ README.md                          ← Este archivo
├─ prompts/
│  ├─ system/agent-persona.md         ← Rol del agente y guardrails
│  ├─ text_to_text/
│  │  ├─ zero-shot.md                 ← Plantillas para QA sin ejemplos
│  │  ├─ one-shot.md                  ← Plantillas con 1 ejemplo interno
│  │  └─ few-shot.md                  ← Plantillas con 3–5 ejemplos
│  └─ text_to_image/
│     ├─ flowchart-prompt.md          ← Plantilla para diagrama de flujo
│     ├─ orgchart-prompt.md           ← Plantilla para organigrama
│     └─ poster-prompt.md             ← Plantilla para póster de buenas prácticas
├─ copilot-studio/
│  ├─ solution-readme.md              ← Cómo exportar/importar la solución del agente
│  └─ topics/
│     ├─ bienvenida.yml               ← Topic de bienvenida + menú rápido
│     ├─ procedimientos.yml           ← Topic de QA con fallback a conocimiento
│     └─ escalamiento.yml             ← Derivación a humano/IT
├─ flows/
│  ├─ power-automate-notes.md         ← Pasos para crear los flujos
│  ├─ GenerateImageFromPrompt.http    ← Ejemplo de llamada HTTP (referencia)
│  └─ SaveToSharePoint.md             ← Paso a paso para crear archivo + metadatos
├─ scripts/
│  ├─ prompt_eval.py                  ← Experimentos de prompting (Python)
│  └─ requirements.txt                ← azure-ai, python-dotenv, requests
├─ data/
│  ├─ sample_procedures/
│  │  ├─ 01-Onboarding-Usuarios-SharePoint.md
│  │  ├─ 02-Estructura-Carpetas-Clientes.md
│  │  └─ 03-Control-IVA-IIBB.md
│  └─ grounding/sharepoint-urls.txt   ← URLs de sitios/bibliotecas de conocimiento
├─ .env.example                       ← Variables (ENDPOINT, API_KEY, SITE_URL, LIBRARY)
├─ SECURITY.md                        ← Consideraciones de seguridad y PII
└─ LICENSE
```

---

## 3) Prerrequisitos

* **Microsoft 365** con acceso a **Copilot Studio** y **Power Automate**.
* **SharePoint** (sitio del estudio) con biblioteca: `Procedimientos` y `Visuales`.
* **Azure AI Foundry / Azure OpenAI** habilitado (modelo de imágenes: DALL·E).
* Permisos para crear **conectores/acciones** y flujos en el entorno.

---


## 4) Conocimiento (texto→texto) en Copilot Studio

1. Crear **Agente** nuevo en Copilot Studio (entorno del estudio).
2. En **Topics**, agregar **Bienvenida** (saludo + menú: “Consultar procedimientos”, “Generar visual”).
3. En **Procedimientos**, insertar un **Generative Answers node** apuntando a **SharePoint** del estudio.

   * Biblioteca sugerida: `Procedimientos` (subcarpetas por área: IVA, IIBB, Sueldos, Clientes, Onboarding IT, etc.).
   * Recomendación editorial: documentos cortos, títulos claros, glosario y “frases claves” por cada artículo.
4. Políticas del agente (system prompt/guardrails) – ver `prompts/system/agent-persona.md`.
5. Fallback: si no hay respuesta, sugerir búsqueda por palabras clave y ofrecer **escalamiento** a humano.

### Plantillas de prompting (QA)

* **zero-shot** (rápido, sin ejemplos): ver `prompts/text_to_text/zero-shot.md`.
* **one-shot/few-shot** (mejor precisión en documentos repetitivos): ver `prompts/text_to_text/*`.

> Métrica mínima de calidad: la respuesta debe **citar** el documento/fuente (título o URL interna) y proponer **pasos accionables**.

---

## 5) Acciones (texto→imagen) con Power Automate + Azure OpenAI

### Flujo A: `GenerateImageFromPrompt`

**Objetivo:** generar una imagen basada en un prompt del agente.

**Entradas del flujo** (desde Copilot):

* `prompt`: texto que describe el diagrama/póster.
* `style`: "flowchart" | "orgchart" | "poster" (opcional).
* `size`: "1024x1024" (por defecto) u otro.

**Pasos del flujo (resumen):**

1. **Compose**: normalizar variables (prompt + estilo + tamaño).
2. **HTTP** → Azure OpenAI (DALL·E) para `image generation`.
3. **Parse JSON** de la respuesta (recibir `b64_json` o `url`).
4. **Salida del flujo**: objeto con `image_name`, `image_bytes/base64`, `mime_type`, `style`.

**Ejemplo de body HTTP** (referencia; ajustar a tu endpoint/modelo):

```json
{
  "model": "dall-e-3",
  "prompt": "[TEXTO DEL USUARIO + CONSIGNAS DE ESTILO]",
  "size": "1024x1024",
  "n": 1,
  "response_format": "b64_json"
}
```

> **Tip de prompting para diagramas:** en `prompt` especifica: “diagram-style, high-contrast, labeled nodes, consistent arrows, white background, minimal text”. Para pósters: “clean layout, large headline, 3–5 bullet points, brand colors (gris/naranja), iconography minimal”.

### Flujo B: `SaveToSharePoint`

**Objetivo:** guardar la imagen generada en SharePoint.

**Entradas:** `image_name`, `image_bytes/base64`, `library`, `folder`, `metadata`.

**Pasos:**

1. **Compose** nombre de archivo: `YYYYMMDD-HHMM-<tipo>-<slug>.png`.
2. **Create file** (SharePoint) en biblioteca `Visuales/`
3. **Update file properties**: `Autor`, `Área`, `Cliente`, `Tags`.
4. **Salida:** URL del archivo para devolver al agente.

### Conectar acciones al agente

* En Copilot Studio → **Actions/Tools** → **Add** → referenciar cada flujo (se exponen como **acciones** del agente).
* Crear un **topic** “Generar recursos visuales”: recoge intención del usuario, solicita aclaraciones (tamaño, estilo, idioma), llama a `GenerateImageFromPrompt` y luego a `SaveToSharePoint`. Devuelve la URL y una mini-guía de uso.

---

## 6) Demostración (VS Code + Teams)

**Escenario 1 (QA):**

* Usuario: “¿Cómo armo la carpeta de cliente en SharePoint?”
* Agente: Respuesta en pasos + cita del procedimiento + botón “Ver documento”.

**Escenario 2 (Visual):**

* Usuario: “Crea un organigrama del Estudio (6 equipos de impuestos y 1 de sueldos).”
* Agente → Acción: genera imagen PNG 1024×1024 + guarda en `Visuales/Organigramas/` + devuelve link.

> Para la demo, prepara 3–5 **prompts curados** en `prompts/text_to_image/` y 10–15 documentos cortos en `data/sample_procedures/`.

---

## 8ç7) Experimentos de Prompting (Python)

Archivo: `scripts/prompt_eval.py`

**Objetivo:** comparar **zero/one/few-shot** para preguntas típicas de onboarding y medir: *exactitud percibida*, *acción sugerida*, *fuente citada* (sí/no).

**Esquema:**

* Cargar `.env` con `AZURE_OPENAI_API_KEY` y `ENDPOINT`.
* Definir `tests.json` (preguntas + respuesta esperada/fuentes).
* Ejecutar cada estrategia de prompt (plantillas de `prompts/text_to_text/`).
* Registrar resultados en `out/results.csv` (para análisis rápido en Excel/Power BI).

> **Métrica sugerida**: 5–10 preguntas frecuentes; calificación 1–5 por criterio; promedio general y por técnica.

---

## 8) Guía editorial para documentos de Procedimientos

* **Formato corto (≤ 1.5 páginas)** con: objetivo, alcance, pasos, checklist, responsables, FAQs.
* **Nombrado**: `AAAMM-Área-Tema-vX.md` (ej.: `202509-IT-Onboarding-SharePoint-v1.md`).
* **Glosario**: agrega palabras clave y sinónimos (ayuda a la búsqueda y a las respuestas del agente).
* **Versionado**: Usa PRs para cambios; etiqueta `breaking-change` si alteras pasos críticos.


