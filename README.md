<img width="2752" height="1536" alt="Automatización_inteligente_de_cuentas_mensuales" src="https://github.com/user-attachments/assets/99f01f4e-7ee0-4c75-8a6d-029eedcccb1a" />

# 🤖 Control de Cuentas Mensuales — n8n Workflow

[![Estado](https://img.shields.io/badge/estado-producción-brightgreen?style=for-the-badge)](https://github.com/coderhouse2025-droid/n8n_Workflows)
[![Plataforma](https://img.shields.io/badge/plataforma-n8n-orange?style=for-the-badge)](https://n8n.io)
[![IA](https://img.shields.io/badge/IA-Claude%20API-blueviolet?style=for-the-badge)](https://anthropic.com)
[![Licencia](https://img.shields.io/badge/licencia-MIT-blue?style=for-the-badge)](LICENSE)

> Automatización completa del ciclo contable mensual: desde Google Drive hasta Airtable, con extracción inteligente de facturas mediante IA (Claude API). Cero intervención manual.

---

## 📋 Índice

- [Descripción](#-descripción)
- [Caso de negocio](#-caso-de-negocio)
- [Decisiones técnicas y su justificación](#-decisiones-técnicas-y-su-justificación)
- [Arquitectura del workflow](#️-arquitectura-del-workflow)
- [Pipeline de datos: de documentos heterogéneos a registros estructurados](#-pipeline-de-datos-de-documentos-heterogéneos-a-registros-estructurados)
- [El rol de la IA: por qué Claude y qué hace exactamente](#-el-rol-de-la-ia-por-qué-claude-y-qué-hace-exactamente)
- [¿Por qué este camino y no otro?](#-por-qué-este-camino-y-no-otro)
- [Nodos del workflow](#-nodos-del-workflow)
- [Configuración](#️-configuración)
- [Estructura del repositorio](#-estructura-del-repositorio)
- [Roadmap](#-roadmap)

---

## 📋 Descripción

Workflow para **n8n** que procesa automáticamente los movimientos financieros de un período, clasificando ingresos y gastos, y centralizando toda la información en un tablero de **Airtable**. Al finalizar cada ciclo envía un resumen consolidado por **Gmail**.

El flujo se ejecuta de forma autónoma cada 15 días sin intervención manual, reduciendo a cero el tiempo operativo de consolidación contable.

---

## 💼 Caso de negocio

### El problema que resuelve

Una persona o pequeña empresa que lleva el control de sus finanzas enfrenta un proceso repetitivo al cierre de cada período: reunir comprobantes de distintas fuentes (Excel de ingresos, facturas PDF de proveedores, tickets escaneados), extraer los datos relevantes de cada uno, unificarlos en un formato consistente, y calcular el balance del período.

Este proceso tiene tres problemas concretos:

**1. Fragmentación de fuentes:** los ingresos pueden venir en una planilla Excel, los gastos son facturas PDF recibidas por email o imágenes de tickets escaneados. No hay un formato común.

**2. Extracción manual tedioso y propenso a errores:** leer cada factura, copiar proveedor, monto, fecha y número de comprobante manualmente consume tiempo y genera errores de tipeo. Una factura con IVA discriminado requiere hacer el cálculo para obtener el neto.

**3. Consolidación manual periódica:** cada dos semanas o cada mes hay que repetir el mismo proceso. Es trabajo que no genera valor — es pura operación mecánica reemplazable por automatización.

### La propuesta de valor

El workflow elimina completamente el trabajo manual del ciclo contable periódico:

- **Ingresos**: se leen automáticamente desde un Excel en Google Drive
- **Gastos**: se extraen automáticamente de facturas PDF e imágenes usando IA
- **Consolidación**: se unifican en Airtable con todos los campos estructurados
- **Reporte**: se envía por email un resumen ejecutivo con balance del período

El tiempo operativo pasa de 2-4 horas cada 15 días a **cero**. El workflow corre solo, a las 8:00 del primer día hábil de cada quincena.

### El contexto de automatización sin código

Este workflow es un ejemplo del paradigma **no-code / low-code para automatización**: no requiere escribir un backend, no requiere un servidor propio corriendo código. n8n orquesta APIs existentes (Google Drive, Claude, Airtable, Gmail) mediante una interfaz visual. Esto lo hace replicable por cualquier profesional sin background de desarrollo, que es exactamente el perfil de usuario que lo necesita.

---

## 🧠 Decisiones técnicas y su justificación

### 1. n8n — no Zapier, no Make (Integromat), no código Python

**¿Por qué n8n?**

n8n es la plataforma de automatización de workflows open-source más madura del mercado. La decisión frente a las alternativas:

| Plataforma | Modelo | Costo | Self-hosted | Lógica compleja |
|------------|--------|-------|-------------|-----------------|
| **n8n** | Open source / Cloud | Gratuito self-hosted | ✅ | ✅ Loop, condiciones, transformaciones JS |
| Zapier | SaaS | $19-$69/mes | ❌ | ⚠️ Limitado en el tier básico |
| Make (Integromat) | SaaS | $9-$16/mes | ❌ | ✅ Bueno pero más caro que n8n |
| Python script | Código | Gratis | ✅ | ✅ Total | 

**¿Por qué no Zapier?**

Zapier es la herramienta más conocida pero tiene limitaciones que afectan directamente este workflow: el nodo Loop (necesario para procesar cada factura individualmente) solo está disponible a partir del plan Teams (~$69/mes). El workflow necesita iterar sobre N facturas por período — sin Loop, no es posible.

**¿Por qué no Make?**

Make es una alternativa sólida y más económica que Zapier. Se descartó por una razón práctica: n8n tiene integración nativa con la API de Anthropic (Claude) como nodo oficial desde 2024, mientras que en Make la integración requiere un nodo HTTP genérico con configuración manual. Para un workflow que usa Claude como componente central, tener el nodo oficial simplifica el mantenimiento y las actualizaciones de versión del modelo.

**¿Por qué no un script Python?**

Un script Python daría control total sobre la lógica. El trade-off: requiere un servidor para ejecutarse en schedule (un EC2, un VPS, o un cron en servidor propio), gestión de dependencias, manejo de errores y reintentos, y actualización de credenciales de API manualmente. n8n resuelve todo eso con una interfaz visual, reintentos automáticos configurable por nodo, y gestión centralizada de credenciales. Para un workflow de automatización sin lógica algorítmica compleja, n8n es el nivel correcto de abstracción.

---

### 2. Claude API (Anthropic) — no GPT-4o, no Gemini, no OCR tradicional

**¿Por qué un LLM para extracción de datos de facturas?**

La alternativa "clásica" para extraer datos de facturas PDF es OCR + regex: extraer texto con una librería de OCR y luego aplicar expresiones regulares para capturar el número de factura, el monto, el proveedor, etc.

El problema es que las facturas no tienen un formato estándar. Cada proveedor emite facturas con distinta estructura, distintos nombres para los campos, y distinta ubicación del monto total. Una regex que funciona para las facturas de un proveedor falla para las del siguiente.

Un LLM entiende semánticamente el contenido del documento. Puede encontrar el monto total de una factura independientemente de si el campo se llama "Total", "Importe", "Monto a pagar", "TOTAL FACTURA" o está simplemente en el pie del documento en negrita sin etiqueta explícita.

**¿Por qué Claude y no GPT-4o?**

Tres razones concretas para este caso de uso:

1. **Ventana de contexto larga**: Claude Sonnet tiene 200K tokens de contexto. Una factura PDF convertida a texto puede ser larga si tiene tablas de ítems detalladas. GPT-4o tiene 128K tokens — suficiente para la mayoría de los casos, pero Claude da más margen.

2. **Consistencia en outputs estructurados**: Claude sigue instrucciones de formato JSON con alta fidelidad. Para un workflow donde el siguiente nodo depende de parsear el JSON devuelto, la consistencia del formato es crítica. Un JSON malformado rompe el workflow.

3. **Costo por token**: Claude Sonnet es competitivo con GPT-4o en precio y en la mayoría de benchmarks de extracción de información supera a GPT-4o Mini (la alternativa económica de OpenAI) en tareas de comprensión de documentos.

---

### 3. Google Drive — no Dropbox, no OneDrive, no servidor FTP

**¿Por qué Google Drive como fuente de documentos?**

Google Drive es la herramienta de almacenamiento de archivos más adoptada en el segmento de pequeñas empresas y profesionales independientes en Argentina. La probabilidad de que el usuario objetivo ya tenga sus facturas y planillas en Drive es alta.

n8n tiene integración nativa con Google Drive via OAuth2 — listar archivos, descargar contenido, y filtrar por carpeta o nombre se configura en el nodo sin escribir código. Dropbox y OneDrive tienen integraciones equivalentes en n8n, pero Drive es la elección por adopción del usuario objetivo, no por ventaja técnica.

**Estructura de carpetas en Drive:**

```
📁 Control Cuentas/
    📁 Ingresos/
    │   └── ingresos_enero_2025.xlsx
    │   └── ingresos_febrero_2025.xlsx
    │
    └── 📁 Gastos/
        └── factura_proveedor_A_001.pdf
        └── factura_proveedor_B_045.jpg
        └── ticket_supermercado.png
```

El workflow lee la carpeta `Ingresos/` para el Excel y la carpeta `Gastos/` para las facturas. La separación es intencional: simplifica el trigger del workflow y evita procesar archivos equivocados.

---

### 4. Airtable — no Google Sheets, no Notion, no base de datos SQL

**¿Por qué Airtable como destino de datos?**

Airtable es una base de datos con interfaz de spreadsheet: tiene la familiaridad visual de una planilla pero la estructura relacional de una base de datos. Para el caso de uso del control contable, esto importa por dos razones:

**Tipos de datos nativos**: Airtable tiene campos de tipo Moneda, Fecha, Selección múltiple (para categorías de gasto), y Fórmula. Un registro de movimiento financiero puede tener el campo `Monto` como tipo Currency (con formato $1.234,56 automático), `Fecha` como tipo Date (con filtros de calendario), y `Categoría` como tipo Single Select con colores asignados. En Google Sheets, todo esto requiere formateo manual.

**Vistas sin modificar datos**: Airtable permite crear vistas (galería, kanban, calendario, formulario) sobre los mismos datos sin alterar la estructura. El dueño puede ver los gastos en una vista calendario sin riesgo de romper la estructura de datos que el workflow usa para insertar registros.

**¿Por qué no Google Sheets?**

Google Sheets es la alternativa más obvia y la más fácil de configurar. Se descartó por un problema operativo: cuando el workflow inserta una fila nueva, Google Sheets la agrega al final de la hoja. Si el usuario ha aplicado filtros o reorganizado las columnas manualmente, la inserción puede ir a una posición incorrecta. Airtable inserta siempre en el modelo de datos subyacente, independientemente de la vista activa.

**¿Por qué no una base de datos SQL (PostgreSQL, SQLite)?**

Una base SQL es técnicamente superior para queries complejas y grandes volúmenes. El trade-off: requiere un servidor de base de datos, un cliente SQL para consultar los datos, y conocimiento técnico para operar. Para el usuario objetivo del workflow, Airtable ofrece el 90% de las capacidades de una base de datos relacional con una interfaz que puede usar sin formación técnica.

---

### 5. Gmail — no SendGrid, no Mailchimp, no Slack

**¿Por qué Gmail para el resumen ejecutivo?**

El resumen quincenal es una notificación personal al dueño de las cuentas — no es un email de marketing ni una notificación a múltiples usuarios. Gmail es la solución más directa: el workflow se autentica con la misma cuenta Google que usa Drive, sin configuración adicional de un servidor SMTP externo ni credenciales adicionales.

El email generado incluye:
- Total de ingresos del período
- Total de gastos del período  
- Balance (ingresos - gastos)
- Tabla de movimientos con proveedor, monto y categoría
- Alerta si hay facturas que no pudieron procesarse automáticamente

---

## 🏗️ Arquitectura del workflow

```
Schedule Trigger (cada 15 días — 8:00 hs, 1° día hábil)
        │
        ├──── RAMA INGRESOS ─────────────────────────────────────────┐
        │     List Ingresos (Drive)                                  │
        │       └── Download Excel                                   │
        │             └── Read Binary Data                           │
        │                   └── Parse Excel → Array de objetos       │
        │                         └── Tag as "ingreso"               │
        │                               └── Normalizar campos ───────┤
        │                                                            ▼
        │                                          Merge Node (combina ambas ramas)
        │                                                            │
        └──── RAMA GASTOS ───────────────────────────────────────────┘
              List Gastos (Drive) → Loop Over Items (1 x 1)
                    │
                    ├── Download Factura (PDF o imagen)
                    │
                    ├── IF es PDF → Extract Text (PDF)
                    │   IF es imagen → Extract Text (OCR)
                    │
                    └── Claude API
                          Prompt: "Extraé los datos de esta factura y devolvé JSON"
                          └── Parse JSON response
                                └── Tag as "gasto"
                                      └── Normalizar campos ─────────┘

                                            │
                               ┌────────────┴─────────────┐
                               │                          │
                        Airtable Create              IF hay errores
                          Record (loop)               └── Agregar a lista
                               │                          de facturas fallidas
                        Calcular Totales
                               │
                          Armar HTML email
                               │
                           Send Gmail
```

**¿Por qué Loop Over Items y no procesar todas las facturas en paralelo?**

La API de Claude tiene rate limits: una cantidad máxima de requests por minuto y tokens por minuto. Si el workflow envía 20 facturas en paralelo (20 requests simultáneos a Claude), es probable que supere el rate limit y reciba errores `429 Too Many Requests`.

El nodo Loop procesa una factura a la vez (batch size = 1), esperando la respuesta de Claude antes de procesar la siguiente. Es más lento en tiempo total pero 100% confiable. Con el volumen esperado (10-30 facturas por período), la diferencia de velocidad es de minutos — irrelevante para un proceso que corre desatendido en segundo plano.

---

## 🔄 Pipeline de datos: de documentos heterogéneos a registros estructurados

Este es el núcleo técnico del workflow. Los datos de entrada son documentos de formatos y estructuras completamente distintos; el output es un conjunto de registros uniformes en Airtable. Se documenta cada transformación.

---

### Problema 1: Ingresos en Excel con estructura variable

El archivo de ingresos es una planilla Excel que puede venir con distintos formatos según quién lo preparó. Las columnas pueden llamarse `Concepto` o `Descripción`, `Importe` o `Monto`, `Fecha` o `Fecha de cobro`.

**Transformación en el nodo "Normalizar Ingresos":**

```javascript
// Nodo Code en n8n — JavaScript
const item = $input.item.json;

// Mapeo flexible de columnas posibles
const monto = item['Monto'] || item['Importe'] || item['Total'] || item['Amount'] || 0;
const concepto = item['Concepto'] || item['Descripción'] || item['Descripcion'] || item['Detalle'] || '';
const fecha = item['Fecha'] || item['Fecha de cobro'] || item['Date'] || new Date().toISOString();

// Normalizar monto: puede venir como string "$1.234,56" o number 1234.56
function parsearMonto(valor) {
  if (typeof valor === 'number') return valor;
  return parseFloat(
    valor.toString()
      .replace(/[^0-9,\.]/g, '')
      .replace(/\.(?=\d{3})/g, '')
      .replace(',', '.')
  ) || 0;
}

return {
  tipo: 'ingreso',
  concepto: concepto.toString().trim(),
  monto: parsearMonto(monto),
  fecha: new Date(fecha).toISOString().split('T')[0],
  fuente: 'Excel Drive',
  categoria: 'Ingreso'
};
```

**¿Por qué normalizar en un nodo Code y no usar el nodo Set de n8n?**

El nodo Set de n8n asigna campos con valores fijos o expresiones simples. Para la lógica de "usar la columna A si existe, sino B, sino C", más el parseo de monto con formato argentino, se necesita JavaScript. El nodo Code permite exactamente eso — es el nodo de "escape hatch" cuando la lógica supera lo que los nodos visuales pueden expresar.

---

### Problema 2: Facturas PDF con estructura no estándar

Cada proveedor emite facturas con distinta estructura visual y campos con distintos nombres. Algunas tienen una tabla de ítems con subtotales; otras tienen un único monto total. Algunas tienen IVA discriminado; otras son monotributistas y no tienen IVA.

**Solución: delegar la comprensión semántica a Claude.**

El nodo de extracción de texto (para PDF) o OCR (para imagen) convierte el documento a texto plano. Ese texto se envía a Claude con un prompt estructurado:

```
Sos un asistente especializado en extracción de datos de facturas.
A continuación te paso el contenido de texto de una factura.
Extraé los siguientes campos y devolvé ÚNICAMENTE un JSON válido, sin texto adicional:

{
  "proveedor": "nombre del emisor de la factura",
  "numero_factura": "número de comprobante (ej: 0001-00012345)",
  "fecha": "fecha en formato YYYY-MM-DD",
  "monto_total": número sin símbolo de moneda,
  "monto_neto": número sin IVA (si no está discriminado, igual al total),
  "iva": número del IVA si está discriminado, 0 si no,
  "categoria": una de estas opciones: [Servicios, Insumos, Tecnología, Logística, Marketing, Personal, Impuestos, Otros],
  "moneda": "ARS" o "USD",
  "notas": cualquier información relevante adicional o "" si no hay
}

CONTENIDO DE LA FACTURA:
{{texto_de_la_factura}}
```

**¿Por qué el prompt especifica las categorías disponibles?**

Si el prompt dijera "clasificá el gasto en una categoría", Claude devolvería categorías inventadas y distintas en cada ejecución: "Servicios de internet", "Telefonía", "Conectividad", dependiendo de cómo interpretara la factura. Al forzar la elección entre opciones predefinidas, el output es siempre uno de los valores válidos en el campo Single Select de Airtable — sin errores de inserción por valores no reconocidos.

---

### Problema 3: Parse del JSON devuelto por Claude

Claude devuelve un string de texto que contiene el JSON. Aunque el prompt pide "solo JSON", los modelos de lenguaje ocasionalmente añaden frases como "Aquí está el JSON:" o encierran el JSON en backticks de markdown.

**Transformación en el nodo "Parse Claude Response":**

```javascript
const respuesta = $input.item.json.message.content[0].text;

// Limpiar posibles envoltorios de markdown
const jsonLimpio = respuesta
  .replace(/```json\n?/g, '')
  .replace(/```\n?/g, '')
  .trim();

let datos;
try {
  datos = JSON.parse(jsonLimpio);
} catch (error) {
  // Si el parse falla, marcar como error para revisión manual
  datos = {
    error: true,
    mensaje_error: `JSON inválido: ${error.message}`,
    respuesta_cruda: respuesta
  };
}

return datos;
```

**¿Por qué `try/catch` y no dejar que el error corte el workflow?**

Si el parse falla y el error se propaga, el workflow entero se detiene y el resto de las facturas no se procesan. Con el `try/catch`, una factura que genera un JSON malformado se marca con `error: true` y el workflow continúa con las siguientes. Al final, el email de resumen incluye la lista de facturas que requieren revisión manual — visibilidad del problema sin detener el proceso.

---

### Problema 4: Unificación de ingresos y gastos en un único esquema

Ingresos y gastos tienen distintos campos relevantes. Los ingresos tienen `concepto` y `categoria: Ingreso`; los gastos tienen `proveedor`, `numero_factura`, `iva`, y una categoría de gasto. Para insertar ambos en la misma tabla de Airtable, deben tener el mismo esquema.

**Esquema unificado:**

```javascript
{
  tipo:            "ingreso" | "gasto",
  concepto:        string,  // concepto del ingreso o nombre del proveedor
  monto_total:     number,  // monto total del movimiento
  monto_neto:      number,  // neto sin IVA (igual a total para ingresos)
  iva:             number,  // IVA del gasto, 0 para ingresos
  fecha:           string,  // YYYY-MM-DD
  categoria:       string,  // categoría del movimiento
  numero_ref:      string,  // número de factura o referencia
  moneda:          "ARS" | "USD",
  fuente:          string,  // "Excel Drive" | "PDF" | "Imagen"
  requiere_revision: boolean // true si el procesamiento tuvo algún problema
}
```

El nodo Merge combina los arrays de ingresos y gastos normalizados. Todos los campos están presentes en todos los registros — los que no aplican tienen valor vacío o 0, lo que garantiza que Airtable nunca recibe un campo `undefined`.

---

### Problema 5: Facturas en dólares con tipo de cambio

Algunos proveedores (servicios de software, hosting internacional) facturan en USD. El tablero de Airtable necesita los montos en una moneda uniforme para que los totales y el balance tengan sentido.

**Transformación aplicada:**

```javascript
// Si la moneda es USD, convertir a ARS usando el tipo de cambio del día
if (datos.moneda === 'USD') {
  // Consulta al nodo HTTP Request → API de tipo de cambio (dolarapi.com)
  const tipoCambio = $('Get Tipo de Cambio').item.json.blue.value; // Dólar blue
  datos.monto_total_ars = datos.monto_total * tipoCambio;
  datos.tipo_cambio_usado = tipoCambio;
  datos.nota_cambio = `USD ${datos.monto_total} × $${tipoCambio} = ARS ${datos.monto_total_ars}`;
} else {
  datos.monto_total_ars = datos.monto_total;
}
```

**¿Por qué dólar blue y no dólar oficial?**

Para una pequeña empresa o profesional independiente en Argentina, el costo real de un servicio internacional pagado con tarjeta o transferencia incluye el tipo de cambio con impuestos (dólar tarjeta / MEP), no el oficial. El dólar blue es el proxy más cercano al costo económico real. Esta decisión está documentada en el email de resumen para que el usuario sepa qué tipo de cambio se usó.

---

## 🎯 El rol de la IA: por qué Claude y qué hace exactamente

Vale la pena separar específicamente qué hace y qué no hace la IA en este workflow, para entender por qué es la herramienta correcta para esta tarea específica.

**Claude hace:**
- Leer texto no estructurado de una factura (resultado del OCR o extracción de PDF)
- Identificar semánticamente qué dato es el proveedor, el monto total, la fecha, el número de factura — independientemente del formato visual del documento
- Clasificar el gasto en una de las categorías predefinidas según el contexto del documento
- Devolver los datos en un formato JSON estrictamente estructurado

**Claude no hace:**
- Leer el archivo PDF directamente — eso lo hace un nodo de extracción de texto previo
- Acceder a ninguna base de datos ni API externa — solo procesa el texto que le pasa el workflow
- Tomar decisiones de negocio — solo extrae y clasifica datos

Esta delimitación es importante: Claude es un componente de transformación de datos en el pipeline, no un agente autónomo. El workflow mantiene el control del flujo; Claude resuelve el paso específico donde la comprensión semántica es necesaria.

---

## 🤔 ¿Por qué este camino y no otro?

### Alternativa descartada: OCR + regex para extracción de facturas

Regex sobre texto de OCR es la solución clásica para extracción de datos de documentos. Es determinística, rápida, y sin costo de API. El problema para este caso: requiere escribir y mantener una regex por cada proveedor diferente. Con 10 proveedores distintos, son 10 conjuntos de regex. Si un proveedor cambia el formato de su factura (actualización de sistema de facturación, cambio de plantilla), la regex correspondiente rompe silenciosamente — el campo queda vacío sin que el workflow lo sepa.

Claude resuelve esto con un único prompt que funciona para cualquier proveedor, cualquier formato, y se adapta automáticamente a cambios de plantilla.

### Alternativa descartada: Google Apps Script

Google Apps Script permite automatizar tareas sobre el ecosistema Google (Sheets, Drive, Gmail) con JavaScript. Hubiera resuelto la parte de ingresos (Excel en Drive → Sheets) y el envío de email. El límite: no tiene integración nativa con Claude API ni con Airtable — habría que hacer llamadas HTTP manuales con gestión de errores propia. n8n resuelve esas integraciones con nodos oficiales ya construidos y mantenidos.

### Alternativa descartada: Automatización con Python + Airflow / Prefect

Un pipeline de datos con Python y un orquestador como Airflow daría control total sobre la lógica, mejor manejo de errores, y facilidad de testing unitario. El trade-off: requiere un servidor, configuración de Airflow (que tiene una curva de setup significativa), y es una solución de ingeniería de datos para un problema de automatización personal. El nivel de complejidad no es proporcional al caso de uso.

### Por qué la arquitectura de dos ramas paralelas

El workflow tiene dos ramas separadas (Ingresos y Gastos) que se fusionan en el nodo Merge antes de insertar en Airtable. La alternativa sería un flujo secuencial: primero procesar todos los ingresos, luego todos los gastos.

La arquitectura de dos ramas es más clara conceptualmente — cada rama tiene su propia lógica de procesamiento sin interferir con la otra. También habilita en el futuro ejecutar ambas ramas en paralelo (con n8n Cloud o self-hosted con workers) para reducir el tiempo total de ejecución.

---

## 🧩 Nodos del workflow

| Nodo | Tipo | Función |
|------|------|---------|
| Schedule Trigger | Disparador | Ejecución automática cada 15 días |
| List Ingresos | Google Drive | Lista archivos Excel en carpeta Ingresos/ |
| Download Excel | Google Drive | Descarga el archivo binario |
| Read Binary → Excel | n8n nativo | Parsea Excel a array de objetos JSON |
| Normalizar Ingresos | Code (JS) | Mapeo flexible de columnas + parseo de monto |
| List Gastos | Google Drive | Lista PDFs e imágenes en carpeta Gastos/ |
| Loop Over Items | n8n nativo | Itera de a una factura (batch=1, rate limit) |
| Download Factura | Google Drive | Descarga el archivo de factura |
| IF PDF / Imagen | IF | Enruta según extensión del archivo |
| Extract Text PDF | n8n nativo | Extrae texto de PDF |
| Extract Text OCR | HTTP Request | OCR via API externa para imágenes |
| Claude API | Anthropic | Extracción semántica → JSON estructurado |
| Parse Claude Response | Code (JS) | Limpieza y parse del JSON con try/catch |
| Get Tipo de Cambio | HTTP Request | dolarapi.com → tipo de cambio del día |
| Normalizar Gastos | Code (JS) | Unificar esquema con rama Ingresos |
| Merge | n8n nativo | Une ingresos + gastos normalizados |
| Airtable Create Record | Airtable | Inserta cada movimiento en la tabla |
| Calcular Totales | Code (JS) | Suma ingresos, gastos, balance |
| Armar Email HTML | Code (JS) | Genera HTML del resumen ejecutivo |
| Send Gmail | Gmail | Envía el resumen al destinatario configurado |

---

## ⚙️ Configuración

### Credenciales requeridas

```
ANTHROPIC_API_KEY        → Claude API (Anthropic Console)
GOOGLE_OAUTH_CREDENTIALS → Google Drive + Gmail (Google Cloud Console)
AIRTABLE_API_KEY         → Airtable (perfil → API)
```

### Variables del workflow

```
DRIVE_FOLDER_INGRESOS    → ID de la carpeta Ingresos en Drive
DRIVE_FOLDER_GASTOS      → ID de la carpeta Gastos en Drive
AIRTABLE_BASE_ID         → ID de la base en Airtable
AIRTABLE_TABLE_NAME      → Nombre de la tabla (ej: "Movimientos")
EMAIL_DESTINATARIO       → Email donde llega el resumen quincenal
```

### Importar el workflow en n8n

1. En n8n, ir a **Workflows → Import from file**
2. Seleccionar el archivo `Control de Cuentas/workflow.json`
3. Configurar las credenciales en cada nodo (Drive, Claude, Airtable, Gmail)
4. Actualizar las variables con los IDs de tus carpetas y base de Airtable
5. Activar el workflow con el toggle de la esquina superior derecha

---

## 📁 Estructura del repositorio

```
/
├── Control de Cuentas/
│   └── workflow.json      # Exportación del workflow n8n (importable directamente)
└── README.md
```

**¿Por qué el workflow se exporta como JSON y no como código?**

n8n almacena los workflows en formato JSON — es el formato nativo de la plataforma. El JSON contiene la definición de todos los nodos, sus conexiones, sus parámetros, y las referencias a credenciales (sin las credenciales en sí). Importar el JSON en cualquier instancia de n8n reconstruye el workflow exacto, del mismo modo que un `package.json` reconstruye las dependencias de un proyecto Node.js.

---

## 🧩 Roadmap

| Prioridad | Mejora | Justificación |
|-----------|--------|---------------|
| Alta | 📁 Trigger por nuevo archivo en Drive | Actualmente el workflow corre en schedule fijo; triggear al detectar una factura nueva en la carpeta permitiría procesamiento en tiempo real |
| Alta | ✅ Validación de duplicados | Si una factura ya fue procesada en el período anterior, no debe insertarse dos veces en Airtable |
| Media | 📊 Dashboard en Airtable con vistas automáticas | Crear la vista de calendario y galería programáticamente via Airtable API |
| Media | 🔔 Alerta de factura anómala | Si Claude detecta un monto inusualmente alto comparado con el histórico del proveedor, notificar antes de insertar |
| Baja | 🌐 Soporte multi-moneda con histórico de tipo de cambio | Guardar el tipo de cambio de cada día para comparativas históricas correctas |

---

## 👨‍💻 Autor

**Juan Manuel Orellana**
Data Science · Analytics · Automatización

---

## 📄 Licencia

MIT License — libre para uso, adaptación y distribución.
