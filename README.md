# Bajas y Mala Rotación

Aplicación web para **registrar bajas de inventario** (productos vencidos, dañados o con mala rotación) por sede de restaurante y **consultar el historial** con métricas y exportaciones. Los datos viven en un Google Sheet y se sirven mediante un backend de Google Apps Script.

---

## Arquitectura

```
┌───────────────┐    gviz/CSV (lectura)       ┌─────────────────┐
│  index.html   │ ────────────────────────▶  │                 │
│  (Frontend    │                             │  Google Sheets  │
│   SPA: HTML/  │    POST JSON (escritura)    │  (base de datos)│
│   CSS/JS)     │ ──┐    ┌──────────────┐ ──▶ │                 │
└───────────────┘   └──▶│  Code.gs     │      └─────────────────┘
                         │ (Apps Script │
                         │  Web App)    │
                         └──────────────┘
```

- **Frontend** — `index.html`: una sola página, sin dependencias de build. Lee catálogos/sedes/historial directamente del Sheet por el endpoint público `gviz/CSV`, y escribe (guardar reporte / actualizar "Numero de Bajas") por `POST` al Apps Script.
- **Backend** — `Code.gs`: Web App de Apps Script. Recibe los `POST`, aplica la lógica anti-duplicados y escribe en la hoja `Reportes`. También expone lecturas vía `GET` (`?action=catalogo|historial`).
- **Datos** — Google Sheets.

---

## Estructura de archivos

| Archivo          | Descripción                                                        |
|------------------|--------------------------------------------------------------------|
| `index.html`     | Aplicación completa (producción). **Este es el archivo vivo.**      |
| `Code.gs`        | Backend de Google Apps Script.                                     |
| `README.md`      | Este documento.                                                   |

---

## Hojas del Google Sheet

| Hoja            | Uso                                                                 |
|-----------------|---------------------------------------------------------------------|
| `SEDE`          | Sedes/restaurantes y su dirección.                                  |
| `MOTIVOS `      | Lista de motivos de baja. ⚠️ El nombre tiene un **espacio final**.   |
| `ROCOTO`        | Catálogo de productos (con precio) para sedes tipo ROCOTO.          |
| `MALANGA`       | Catálogo de productos (con precio) para sedes que incluyen "MALANGA".|
| `INSUMOS`       | Catálogo usado por el endpoint `GET ?action=catalogo` del backend.  |
| `Reportes`      | Historial de bajas registradas (destino de los `POST`).             |

### Columnas de la hoja `Reportes` (15 columnas)

| Col | Campo            | Quién la escribe                          |
|-----|------------------|-------------------------------------------|
| A   | Fecha Reporte    | Guardar reporte                           |
| B   | Restaurante      | Guardar reporte                           |
| C   | Encargado        | Guardar reporte                           |
| D   | Producto         | Guardar reporte                           |
| E   | Codigo           | Guardar reporte                           |
| F   | Familia          | Guardar reporte                           |
| G   | Subfamilia       | Guardar reporte                           |
| H   | Unidad de medida | Guardar reporte                           |
| I   | Cantidad         | Guardar reporte                           |
| J   | Vencimiento      | Guardar reporte                           |
| K   | Motivo           | Guardar reporte                           |
| L   | Notas            | Guardar reporte                           |
| M   | Fecha Registro   | Guardar reporte (timestamp del servidor)  |
| N   | Numero de Bajas  | Acción `updateNB` (edición en el historial)|
| O   | **ReporteID**    | Guardar reporte (clave de idempotencia)   |

---

## API del backend (`Code.gs`)

Todas las respuestas son JSON: éxito `{ ok:true, ... }`, error `{ ok:false, error }`.

| Método | Parámetros                                   | Acción                                              |
|--------|----------------------------------------------|-----------------------------------------------------|
| GET    | `?action=catalogo`                           | Devuelve los insumos de la hoja `INSUMOS`.          |
| GET    | `?action=historial`                          | Devuelve las filas de la hoja `Reportes`.           |
| GET    | (sin `action`)                               | Ping: `{ ok:true, msg:'API activa' }`.              |
| POST   | `{ action:'updateNB', fechaRegistro, producto, codigo, numeroBajas }` | Actualiza la columna **Numero de Bajas** de una fila. |
| POST   | `{ reporteId, fechaReporte, sede/restaurante, encargado, productos:[...] }` | **Guarda un reporte** (idempotente).                |

---

## Protección anti-duplicados

El sistema garantiza que un mismo reporte **se guarde como máximo una vez**, incluso con red inestable o doble clic. Tres capas se combinan:

**Frontend (`saveReport` en `index.html`)**
1. **Candado `isSaving`** — ignora cualquier disparo extra mientras hay un guardado en curso.
2. **`reporteId` único por intento** — se genera con `crypto.randomUUID()` y viaja en el `POST`. Si hay que reintentar, se reenvía el **mismo** id.
3. **`resetForm()` tras éxito** — vacía los productos en pantalla, así es imposible reenviar el mismo reporte con un segundo clic.

**Backend (`doPost` en `Code.gs`)**
4. **`LockService`** — serializa todas las escrituras; dos peticiones casi simultáneas hacen fila en lugar de escribir a la vez.
5. **Guardia de idempotencia** — antes de escribir, comprueba si el `reporteId` ya existe en la columna `O`. Si existe, responde `{ ok:true, duplicate:true }` **sin volver a escribir**.

> Las filas antiguas sin `ReporteID` (columna `O` vacía) no generan falsos positivos: la comparación tolera celdas vacías.

---

## Despliegue

### Backend (`Code.gs`)
1. Abre el proyecto en [Apps Script](https://script.google.com) y pega el contenido de `Code.gs`. Guarda.
2. Confirma que `SHEET_ID` apunta a tu Google Sheet.
3. **Implementar → Administrar implementaciones → editar la activa (lápiz) → Versión: _Nueva versión_ → Implementar.** Esto conserva la **misma URL `/exec`**, así no tienes que cambiar `SCRIPT_URL` en el frontend.
4. En "Quién tiene acceso" selecciona **Cualquiera**, para que el frontend pueda llamar la API.

> Si en lugar de editar la implementación creas una **nueva**, obtendrás una URL distinta y deberás actualizar `SCRIPT_URL` en `index.html`.

### Frontend (`index.html`)
1. Verifica las dos constantes al inicio del bloque de script: `SCRIPT_URL` (la `/exec` del Web App) y `SHEET_ID`.
2. Publica el archivo donde lo tengas hospedado (GitHub Pages, Netlify, Drive, etc.).

---

## Uso

1. **Selecciona la sede** → se carga su catálogo y dirección automáticamente.
2. **Agrega productos**: nombre (autocompletado desde el catálogo), cantidad, motivo y notas. El código y la unidad se completan solos.
3. **Guardar en Google Sheets** → el reporte queda registrado. El formulario se limpia para evitar duplicados.
4. **Historial**: filtra por sede/motivo, edita el campo **Numero de Bajas**, revisa métricas y exporta a **CSV / Excel / PDF**.

---

## Sobre `index (3).html` (obsoleto — se puede borrar)

Es un **prototipo anterior** y no forma parte de la app en producción. Diferencias confirmadas:

- Apunta a un **deployment antiguo y distinto** del Apps Script.
- Guarda el historial solo en **`localStorage`** (se pierde al limpiar el navegador; no es multiusuario).
- **No** incluye: "Numero de Bajas" (`updateNB`), historial leído desde Sheets, exportar PDF/Excel, métricas ni gráfico.
- Ningún archivo del proyecto lo referencia.

**Conclusión:** se puede eliminar sin afectar la aplicación. *(Opcionalmente, si su deployment viejo ya no se usa, puedes archivarlo desde el editor de Apps Script.)*
