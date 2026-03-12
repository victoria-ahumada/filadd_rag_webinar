# Filadd - **Inteligencia sobre tus Datos con RAG**

## Ejercicio Práctico — RAG con n8n + Supabase

> **Objetivo:** Construir un sistema de preguntas y respuestas sobre un documento PDF usando RAG.  
> **Herramientas:** n8n Cloud, Supabase (free tier), OpenAI API  
> **Tiempo estimado para replicar:** 30–45 minutos  
> **Nivel:** No se requiere experiencia en programación

---

## Requisitos Previos

Antes de comenzar, asegúrate de tener:

- **Cuenta en n8n Cloud**: [https://n8n.io](https://n8n.io) — registrarse con plan starter o trial
- **Cuenta en Supabase**: [https://supabase.com](https://supabase.com) — registrarse con cuenta gratuita
- **API Key de OpenAI**: [https://platform.openai.com/api-keys](https://platform.openai.com/api-keys) — necesitas créditos activos
- **Un documento PDF** para indexar (puede ser un manual, FAQ, política interna, etc.)

---

## Parte 1: Configurar Supabase (10 min)

### Paso 1.1 — Crear un proyecto nuevo

1. Inicia sesión en [supabase.com](https://supabase.com)
2. Haz clic en **"New Project"**
3. Configura:
  - **Name:** `rag-documentos` (o el nombre que prefieras)
  - **Database Password:** elige una contraseña segura y **guárdala** (la necesitarás después)
  - **Region:** selecciona la más cercana a tu ubicación
4. Haz clic en **"Create new project"**
5. Espera 1-2 minutos mientras se aprovisiona

### Paso 1.2 — Habilitar pgvector y crear la tabla

1. En el menú lateral, ve a **SQL Editor**
2. Haz clic en **"New query"**
3. Copia y pega el siguiente script SQL completo:

```sql
-- ============================================
-- SCRIPT DE CONFIGURACIÓN PARA RAG + SUPABASE
-- ============================================

-- Paso 1: Habilitar la extensión pgvector
CREATE EXTENSION IF NOT EXISTS vector;

-- Paso 2: Crear la tabla para almacenar los documentos
CREATE TABLE IF NOT EXISTS documents (
  id BIGSERIAL PRIMARY KEY,
  content TEXT,                          -- El texto del chunk
  metadata JSONB,                        -- Metadatos (archivo, página, etc.)
  embedding VECTOR(1536)                 -- El vector de embedding (1536 dims para text-embedding-3-small)
);

-- Paso 3: Crear un índice para búsquedas rápidas
-- (Usar ivfflat para mejor rendimiento con muchos registros)
CREATE INDEX ON documents 
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);

-- Paso 4: Crear la función de búsqueda por similitud
CREATE OR REPLACE FUNCTION match_documents (
  query_embedding VECTOR(1536),
  match_count INT DEFAULT 4,
  filter JSONB DEFAULT '{}'
) RETURNS TABLE (
  id BIGINT,
  content TEXT,
  metadata JSONB,
  similarity FLOAT
)
LANGUAGE plpgsql
AS $$
#variable_conflict use_column
BEGIN
  RETURN QUERY
  SELECT
    id,
    content,
    metadata,
    1 - (documents.embedding <=> query_embedding) AS similarity
  FROM documents
  WHERE metadata @> filter
  ORDER BY documents.embedding <=> query_embedding
  LIMIT match_count;
END;
$$;
```

1. Haz clic en **"Run"** (o presiona Ctrl+Enter)
2. Deberías ver: `Success. No rows returned` — esto es correcto

### Paso 1.3 — Obtener las credenciales de conexión

1. Ve a **Project Settings** (icono de engranaje en el menú lateral)
2. Luego a **API**
3. Copia y guarda estos valores (los necesitarás en n8n):
  - **Project URL** (algo como `https://xxxxx.supabase.co`)
  - **anon/public key** (empieza con `eyJ...`)

> **Nota importante:** Guarda también el **service_role key** (en la sección inferior). Este tiene permisos completos y es el que usaremos en n8n para insertar datos.

---

## Parte 2: Configurar las Credenciales en n8n (5 min)

### Paso 2.1 — Credencial de OpenAI

1. En n8n, ve a **Settings** → **Credentials**
2. Haz clic en **"Add Credential"**
3. Busca **"OpenAI"**
4. Pega tu **API Key** de OpenAI
5. Haz clic en **"Save"**

### Paso 2.2 — Credencial de Supabase

1. Haz clic en **"Add Credential"**
2. Busca **"Supabase"**
3. Configura:
  - **Host:** tu Project URL (ej: `https://xxxxx.supabase.co`)
  - **Service Role Key:** el service_role key que guardaste
4. Haz clic en **"Save"**

---

## Parte 3: Workflow de Indexación (15 min)

> Este workflow lee un PDF, lo divide en chunks, genera embeddings y los guarda en Supabase.

### Paso 3.1 — Importar el workflow

1. Descarga el archivo `workflow_indexacion.json` (proporcionado por la instructora)
2. En n8n, haz clic en **"..."** (menú) → **"Import from file"**
3. Selecciona el archivo JSON
4. El workflow aparecerá con todos los nodos ya conectados

### Paso 3.2 — Entender los nodos

El workflow tiene esta estructura:

```
[Manual Trigger] → [Read Binary File] → [Extract Document Text] → [Recursive Character Text Splitter] → [Embeddings OpenAI] → [Supabase Vector Store]
```

**Descripción de cada nodo:**


| Nodo                                  | Función                           | Configuración clave                  |
| ------------------------------------- | --------------------------------- | ------------------------------------ |
| **Manual Trigger**                    | Inicia el workflow manualmente    | Sin configuración                    |
| **Read Binary File**                  | Lee el archivo PDF                | Ruta del archivo o upload            |
| **Extract Document Text**             | Extrae texto plano del PDF        | Operación: Extract text              |
| **Recursive Character Text Splitter** | Divide el texto en chunks         | Chunk Size: 1000, Chunk Overlap: 200 |
| **Embeddings OpenAI**                 | Genera el embedding de cada chunk | Model: text-embedding-3-small        |
| **Supabase Vector Store**             | Guarda en la base de datos        | Mode: Insert, Table: documents       |


### Paso 3.3 — Configurar los nodos

**Si importaste el JSON**, solo necesitas:

1. **Asignar las credenciales**: haz clic en cada nodo que requiera credenciales (OpenAI y Supabase) y selecciona las que creaste en la Parte 2
2. **Cargar tu PDF**: en el nodo de lectura de archivo, sube tu documento PDF

**Si estás creando desde cero**, configura cada nodo según la tabla anterior.

### Paso 3.4 — Ejecutar la indexación

1. Haz clic en **"Test Workflow"** (o "Execute Workflow")
2. Observa cómo cada nodo se ilumina conforme procesa
3. El tiempo depende del tamaño del PDF (~1-3 minutos para 50 páginas)
4. Al terminar, verifica en el último nodo que los items se procesaron correctamente

### Paso 3.5 — Verificar en Supabase

1. Ve a Supabase → **Table Editor**
2. Selecciona la tabla **documents**
3. Deberías ver filas con:
  - `content`: el texto de cada chunk
  - `metadata`: información como nombre del archivo
  - `embedding`: el vector (verás una lista larga de números)
4. Si ves filas con datos, la indexación fue exitosa

---

## Parte 4: Workflow de Consulta (10 min)

> Este workflow crea un chatbot que responde preguntas usando los documentos indexados.

### Paso 4.1 — Importar el workflow

1. Descarga el archivo `workflow_consulta.json`
2. Importa en n8n igual que antes

### Paso 4.2 — Entender los nodos

```
[Chat Trigger] → [AI Agent] → [OpenAI Chat Model (GPT-4)] 
                     ↓
              [Vector Store Tool (Supabase)]
                     ↓
              [Embeddings OpenAI]
```

**Descripción de cada nodo:**


| Nodo                  | Función                                  | Configuración clave            |
| --------------------- | ---------------------------------------- | ------------------------------ |
| **Chat Trigger**      | Abre una ventana de chat para el usuario | Sin configuración especial     |
| **AI Agent**          | Orquesta la consulta RAG                 | System Prompt (ver abajo)      |
| **OpenAI Chat Model** | Genera las respuestas                    | Model: gpt-4, Temperature: 0.2 |
| **Vector Store Tool** | Busca chunks similares en Supabase       | Top K: 4                       |
| **Embeddings OpenAI** | Convierte la pregunta en embedding       | Model: text-embedding-3-small  |


### Paso 4.3 — El System Prompt del AI Agent

Este es el prompt que controla el comportamiento del agente. Asegúrate de que contenga algo similar a:

```
Eres un asistente experto en los procedimientos y políticas internas de la empresa. 
Tu función es responder preguntas basándote EXCLUSIVAMENTE en la información 
proporcionada por la herramienta de búsqueda en documentos.

REGLAS ESTRICTAS:
1. SIEMPRE usa la herramienta de búsqueda antes de responder.
2. Responde ÚNICAMENTE con información encontrada en los documentos.
3. Si la información no está disponible en los documentos, indica claramente: 
   "No encontré información sobre este tema en los documentos consultados."
4. Cuando cites información, menciona la fuente o sección relevante si está 
   disponible en los metadatos.
5. Responde en el mismo idioma en que se hizo la pregunta.
6. Sé conciso pero completo en tus respuestas.
```

### Paso 4.4 — Configurar y probar

1. **Asigna las credenciales** de OpenAI y Supabase a cada nodo
2. **Activa el workflow** (toggle en la esquina superior derecha)
3. Haz clic en **"Chat"** (botón en la parte inferior del canvas)
4. Escribe una pregunta sobre tu documento

### Paso 4.5 — Preguntas de prueba sugeridas

Prueba con estas preguntas (adaptadas a tu documento):


| Tipo                    | Ejemplo                               | Resultado esperado                |
| ----------------------- | ------------------------------------- | --------------------------------- |
| Pregunta directa        | "¿Cuál es el procedimiento para [X]?" | Respuesta detallada con pasos     |
| Pregunta conceptual     | "¿Qué políticas aplican para [Y]?"    | Explicación basada en el doc      |
| Pregunta que NO está    | "¿Cuánto cuesta un viaje a Marte?"    | "No encontré información..."      |
| Pregunta en otro idioma | "What is the procedure for [X]?"      | Respuesta en inglés, misma fuente |


---

## Parte 5: Verificación y Troubleshooting

### Todo funciona si:

- Supabase tiene la tabla `documents` con filas de chunks
- El workflow de indexación se ejecuta sin errores
- El chat del workflow de consulta responde con información del documento
- Cuando preguntas algo que NO está en el documento, responde que no encontró info

### Problemas comunes y soluciones:


| Problema                                    | Causa probable                                  | Solución                                                                                    |
| ------------------------------------------- | ----------------------------------------------- | ------------------------------------------------------------------------------------------- |
| "Invalid API key" en OpenAI                 | API key incorrecta o sin créditos               | Verificar la key en platform.openai.com. Verificar que tienes créditos.                     |
| La tabla no tiene datos después de indexar  | Credenciales de Supabase incorrectas            | Verificar que usas el **service_role key** (no el anon key)                                 |
| "Function match_documents not found"        | El SQL no se ejecutó correctamente              | Ir al SQL Editor de Supabase y re-ejecutar el script                                        |
| Respuestas genéricas sin citar el documento | El Vector Store Tool no está conectado al Agent | Verificar que el nodo de Supabase está correctamente enlazado como herramienta del AI Agent |
| "Could not load model"                      | Modelo incorrecto o no disponible               | Verificar que tienes acceso a GPT-4 en tu cuenta de OpenAI (requiere tier de pago)          |
| Chunks demasiado cortos o largos            | Configuración del Text Splitter                 | Ajustar chunk_size (probar con 800-1500) y overlap (150-300)                                |
| Resultados irrelevantes                     | Pocos chunks o PDF mal extraído                 | Verificar en la tabla de Supabase que el contenido de los chunks tiene sentido              |


---

## Extras: Personalización

### Cambiar el modelo de LLM

Si GPT-4 es muy costoso para pruebas, puedes usar **gpt-4o-mini** que es más económico. Cambia el modelo en el nodo "OpenAI Chat Model".

### Agregar más documentos

Simplemente ejecuta el workflow de indexación con un nuevo PDF. Los chunks se agregan a la misma tabla. Asegúrate de incluir el nombre del archivo en los metadatos para poder filtrar después.

### Conectar a otros canales

Reemplaza el **Chat Trigger** por:

- **Slack Trigger**: para un bot en Slack
- **Webhook**: para integrarlo con cualquier aplicación externa
- **Telegram Trigger**: para un bot de Telegram

### Limpiar y re-indexar un documento

Si actualizas un documento, primero elimina los registros anteriores:

```sql
-- Eliminar chunks de un archivo específico
DELETE FROM documents 
WHERE metadata->>'source' = 'nombre_del_archivo.pdf';
```

Luego ejecuta nuevamente el workflow de indexación con el archivo actualizado.

---

## Resumen de Arquitectura

```
┌───────────────────────────────────────────────────────────────┐
│                    FASE DE INDEXACIÓN                         │
│                                                               │
│  📄 PDF ──→ ✂️ Chunking ──→ 🔢 Embeddings ──→ 🗄️ Supabase   │
│            (1000 chars,     (text-embedding-     (pgvector)   │
│             200 overlap)     3-small)                         │
└───────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│                    FASE DE CONSULTA                            │
│                                                                │ 
│  ❓ Pregunta ──→ 🔢 Embedding ──→ 🔍 Búsqueda  ──→ 📋 Prompt │
│                  (mismo modelo)    (top 4 chunks)    + Chunks  │
│                                                        │       │
│                                                        ▼       │
│                                              🤖 GPT-4          │
│                                                        │       │
│                                                        ▼       │
│                                              ✅ Respuesta     │
│                                              fundamentada      │
└────────────────────────────────────────────────────────────────┘
```
