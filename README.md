# 🤖 Control de Cuentas Mensuales — n8n Workflow

> Automatización completa del ciclo contable mensual: desde Google Drive hasta Airtable, con extracción inteligente de facturas mediante IA.

![Estado](https://img.shields.io/badge/estado-producción-brightgreen)
![Plataforma](https://img.shields.io/badge/plataforma-n8n-orange)
![IA](https://img.shields.io/badge/IA-Claude%20API-blueviolet)
![Licencia](https://img.shields.io/badge/licencia-MIT-blue)

---

## 📋 Descripción

Workflow para **n8n** que procesa automáticamente los movimientos financieros de un período, clasificando ingresos y gastos, y centralizando toda la información en un tablero de **Airtable**. Al finalizar cada ciclo envía un resumen consolidado por **Gmail**.

El flujo se ejecuta de forma autónoma cada 15 días sin intervención manual, reduciendo a cero el tiempo operativo de consolidación contable.

---

## ✨ Funcionalidades

- 📥 **Lectura de ingresos** desde archivos Excel en Google Drive
- 🧾 **Procesamiento de facturas** PDF e imagen con extracción de texto automática
- 🧠 **Extracción inteligente de datos** mediante Claude API (Anthropic) — proveedor, monto, fecha, número de factura, categoría
- 🔀 **Normalización y unificación** de ingresos y gastos en un único flujo
- 📊 **Registro automático** en Airtable con todos los campos estructurados
- 📧 **Resumen ejecutivo por email** con totales, balance del período y detalle de movimientos

---

## 🗺️ Arquitectura del Workflow

```
Schedule Trigger (cada 15 días)
        │
        ├──── RAMA INGRESOS ────────────────────────────────────────┐
        │     List Ingresos (Drive) → Download → Read Excel         │
        │     → Tag as Ingreso (normaliza campos)                   │
        │                                                           ▼
        │                                              Merge Ingresos + Gastos
        │                                                           │
        └──── RAMA GASTOS ──────────────────────────────────────────┘
              List Gastos (Drive) → Loop (1 x 1)
              → Download Factura → Extract Text (PDF/Imagen)
              → Claude API (extrae JSON estructurado)
              → Parse Claude Response

                                          │
                               Airtable Create Record
                                          │
                                    Armar Resumen
                                          │
                                      Send Gmail
```

---

## 🧩 Nodos del Workflow

| Nodo | Tipo | Función |
|------|------|---------|
| Schedule Trigger | Disparador | Ejecución automática cada 15 días |
| List Ingresos | Google Drive | Lista archivos Excel de ingresos |
| Download Ingresos | Google Drive | Descarga cada archivo Excel |
| Read Excel | Spreadsheet File | Parsea el contenido del Excel |
| Tag as Ingreso | Set (v3) | Normaliza campos y etiqueta como "Ingreso" |
| List Gastos | Google Drive | Lista facturas PDF/imagen de gastos |
| Loop Gastos | SplitInBatches | Itera de a una factura |
| Download Factura | Google Drive | Descarga el archivo binario |
| Extract Text (PDF/Image) | Extract From File | Extrae texto crudo del documento |
| Claude – Extraer Datos | HTTP Request | Llama a Claude API y obtiene JSON estructurado |
| Parse Claude Response | Code (JS) | Parsea y valida la respuesta de Claude |
| Merge Ingresos + Gastos | Merge (append) | Unifica ambas ramas |
| Airtable Create Record | Airtable (v2) | Registra cada movimiento en el tablero |
| Armar Resumen | Code (JS) | Calcula totales y genera el cuerpo del email |
| Send Gmail | Gmail | Envía el resumen consolidado |

---

## ⚙️ Requisitos previos

- [n8n](https://n8n.io/) v1.0 o superior (self-hosted o cloud)
- Cuenta de **Google** con acceso a Drive y Gmail
- Base de datos en **Airtable** con la tabla configurada (ver estructura más abajo)
- API Key de **Anthropic** (Claude API) — [obtenerla aquí](https://console.anthropic.com/)

---

## 🚀 Instalación y configuración

### 1. Importar el workflow

1. En n8n ir a **Workflows → Import from file**
2. Seleccionar el archivo `Control_de_Cuentas_Mensuales.json`

### 2. Configurar credenciales

#### Google Drive / Gmail
En n8n → **Credentials → New → Google OAuth2 API** y autorizar el acceso.

#### Anthropic (Claude API) — Header Auth
En n8n → **Credentials → New → Header Auth**:
| Campo | Valor |
|-------|-------|
| **Name** | `x-api-key` |
| **Value** | `sk-ant-api...` (tu API key) |

> ⚠️ Asegurarse de que el campo **Name** sea exactamente `x-api-key`. Un valor incorrecto aquí impide toda llamada a la API.

#### Airtable
En n8n → **Credentials → New → Airtable Token API** e ingresar el Personal Access Token de Airtable.

### 3. Completar los parámetros en el JSON (o en el editor visual)

| Parámetro | Dónde | Descripción |
|-----------|-------|-------------|
| `ID_DE_CARPETA_INGRESOS` | Nodo *List Ingresos* | ID de la carpeta de Drive con los Excel |
| `ID_DE_CARPETA_GASTOS` | Nodo *List Gastos* | ID de la carpeta de Drive con las facturas |
| `ID_BASE_AIRTABLE` | Nodo *Airtable Create Record* | ID de la base (formato `appXXXXXXXXXXXXXX`) |
| `NOMBRE_TABLA_AIRTABLE` | Nodo *Airtable Create Record* | Nombre exacto de la tabla, ej: `Tablero de Control` |
| `TU_CORREO@gmail.com` | Nodo *Send Gmail* | Dirección de destino del resumen |

---

## 🗄️ Estructura de la tabla en Airtable

Crear una tabla con los siguientes campos:

| Campo | Tipo |
|-------|------|
| Tipo | Single line text |
| Concepto | Single line text |
| Nombre o Razón Social | Single line text |
| Clasificación | Single select |
| Número de factura | Single line text |
| Fecha de vencimiento | Date |
| Forma de Pago | Single line text |
| Monto | Currency / Number |

---

## 📁 Estructura del repositorio

```
├── Control_de_Cuentas_Mensuales.json   # Workflow n8n listo para importar
├── README.md                           # Este archivo
```

---

## 🔄 Lógica de categorización (Claude)

El nodo Claude clasifica automáticamente cada factura en una de estas categorías:

`Servicio` · `Supermercado` · `Salud` · `Transporte` · `Impuesto` · `Alquiler` · `Comunicaciones` · `Entretenimiento` · `Otro`

---

## 🐛 Problemas conocidos y soluciones

| Síntoma | Causa | Solución |
|---------|-------|----------|
| Workflow no ejecuta — *"has issues"* | Header `x-api-key` mal configurado en la credencial | Verificar que el campo Name de la credencial sea `x-api-key` |
| Registros en Airtable llegan vacíos | Nodo Set sin mapeo o referencias de campo incorrectas | Verificar `Tag as Ingreso` y el mapeo de columnas de Airtable |
| Error 401 en Claude API | API key incorrecta o credencial mal nombrada | Revisar la credencial Header Auth |
| `Extract Text` no encuentra el binario | `binaryPropertyName` no configurado | Confirmar que el parámetro sea `data` |

---

## 📄 Licencia

MIT © 2026 — libre para usar, modificar y distribuir con atribución.
