# README — Integración Hostaway ↔ Hikvision (Apps Script)

> **Propósito general:** Automatizar el flujo completo de **detección y purga de reservas canceladas**.
> 1️⃣ Desde **Hostaway**, obtener reservas que **cruzan el día actual** y estén **canceladas**.
> 2️⃣ En **Hikvision**, eliminar las credenciales de acceso (rostros y usuarios) de dichas reservas en los dispositivos correspondientes a cada edificio.

---

## 🧩 Arquitectura general

```
[Hostaway API] → [Google Sheets: Hostaway_Reservas_Canceladas_Hoy]
                     ↓
            [Apps Script: purgeCanceledReservationsInHikvision()]
                     ↓
       [Gateway Hikvision (Digest Auth)] → [Devices / Rostros / Usuarios]
                     ↓
            [Logs + Alertas por correo]
```

### Flujo global

1. **exportReservasCanceladasQueCruzanHoy()**

   * Consulta Hostaway → obtiene reservas que cruzan HOY.
   * Filtra canceladas y las escribe en `Hostaway_Reservas_Canceladas_Hoy`.
2. **purgeCanceledReservationsInHikvision()**

   * Lee esa hoja, busca devices por edificio, y ejecuta las bajas en Hikvision (rostro y usuario).
   * Marca `status` como `eliminado` o `error`, registra logs y envía alertas por correo.

---

## ⚙️ Configuración principal

### Hostaway

```js
const CLIENT_ID     = '36321';
const CLIENT_SECRET = '...'; // usar PropertiesService
const TZ3           = 'America/Mexico_City';
const PAGE_LIMIT    = 200;
const MAX_RETRIES   = 4;
const CHANNEL_IDS   = []; // opcional: filtrar por canal
const OUT_SHEET     = 'Hostaway_Reservas_Canceladas_Hoy';
```

### Hikvision

```js
const BASE_URL       = 'http://144.126.216.86'; // Gateway (usar HTTPS si es posible)
const GATEWAY_USER   = 'admin';
const GATEWAY_PASS   = '***'; // mover a Script Properties
const TZ             = 'America/Mexico_City';

const SOURCE_SHEET   = 'Hostaway_Reservas_Canceladas_Hoy';
const DEVICES_SHEET  = 'Hikvision_Devices';
const LOG_SHEET      = 'Hikvision_Logs';

const REQUEST_SLEEP_MS = 350;
const MAX_ATTEMPTS  = 3;
const ALERT_EMAILS  = ['ipadilla@vive.properties','tecnologia@vive.properties','csupport@vive.properties','csuccess@vive.properties'];
const ALERT_SUBJECT = 'Hikvision — Error al borrar reservas canceladas';
```

### Mapeo edificio → devices

```js
const BUILDING_TO_DEVNAMES = {
  'tribu colon': ['tribu colon entrada'],
  'bitloft': ['tribu colon entrada'],
  'tribu san martin': ['tribusm'],
  'punto panamericano': ['panamericano'],
  'tribu chapultepec 67': ['tribu chapultepec 67 entrada'],
  'tribu sonata': ['tribu sonata elevador','tribu sonata ent'],
  'casa 8': ['casa8 madera','casa8 herreria'],
  'tribu coordenada lafayette': ['tribu lafayette']
};
```

---

## 🧠 Script 1 — Exportación de canceladas (Hostaway)

### Función principal

**`exportReservasCanceladasQueCruzanHoy()`**

* Autentica con `client_credentials` (Bearer token).
* Llama a `/v1/reservations` filtrando `arrival<=today` y `departure>=today`.
* Filtra las canceladas (`canceled`, `cancelled`, `cancelado`, `cancelada`).
* Obtiene `internalListingName` de `/v1/listings/{id}` y deriva el **edificio**.
* Escribe la hoja `Hostaway_Reservas_Canceladas_Hoy` con:

  ```
  reservationId | status | status_hostaway | guestName | email | phone | listingId | channel | edificio | arrivalDate | departureDate | nights | totalPrice | currency | createdAt | updatedAt
  ```

### Derivación de edificio

* Extrae nombre tras guion o entre paréntesis.
* Ejemplo: *Colmena – Tribu Chapultepec (Depto 301)* → `Tribu Chapultepec`.

### Seguridad

```js
function getSecret_(key) {
  return PropertiesService.getScriptProperties().getProperty(key);
}
```

Usar `getSecret_('HOSTAWAY_CLIENT_SECRET')` en lugar de hardcodear el secreto.

---

## 🧠 Script 2 — Purga de canceladas (Hikvision)

### Función principal

**`purgeCanceledReservationsInHikvision()`**

1. Inicializa logs y refresca hoja de dispositivos.
2. Lee `Hostaway_Reservas_Canceladas_Hoy` (crea columna `status` si falta).
3. Por cada reserva:

   * Localiza devices asociados al edificio.
   * Ejecuta:

     * `deleteFace_(devIndex, reservationId)`
     * `deleteUser_(devIndex, reservationId)`
   * Si alguna falla → registra en log y envía alerta.
4. Marca cada fila con `status = eliminado | error`.

### Endpoints usados (Digest Auth)

* `PUT /ISAPI/Intelligent/FDLib/FDSearch/Delete`  → rostros
* `PUT /ISAPI/AccessControl/UserInfoDetail/Delete` → usuarios

### Reintentos y alertas

* `_withRetries_()` aplica hasta `MAX_ATTEMPTS` con backoff exponencial.
* `sendAlertEmail_()` envía HTML con detalles del fallo.

### Logs operativos

* Hoja `Hikvision_Logs`: columnas `Timestamp | Nivel | Mensaje`.
* Atajos: `logI_`, `logW_`, `logE_`.

---

## 🔐 Seguridad

* Guardar contraseñas en **Script Properties** (`HOSTAWAY_CLIENT_SECRET`, `HIK_GATEWAY_PASS`).
* Restringir acceso al script y la hoja.
* Usar `https://` y certificados válidos en el gateway si es posible.

---

## ⚡ Recomendaciones

| Categoría               | Sugerencia                                                                                                     |
| ----------------------- | -------------------------------------------------------------------------------------------------------------- |
| **Triggers**            | Programar ejecución diaria/secuencial (Hostaway → Hikvision).                                                  |
| **Errores HTTP**        | Revisar `Hikvision_Logs`; las alertas incluyen detalles.                                                       |
| **Tiempo de ejecución** | Apps Script tiene límite (6 min estándar). Si hay muchas reservas, dividir por edificios o ejecutar por lotes. |
| **Idempotencia**        | Borrar rostros/usuarios es seguro aunque no existan.                                                           |
| **Monitoreo**           | Crear tablero en Sheets/Looker con conteos de OK/error.                                                        |

---

## 🧪 Pruebas sugeridas

1. Crear fila de prueba manual en `Hostaway_Reservas_Canceladas_Hoy`.
2. Ejecutar `purgeCanceledReservationsInHikvision()` y verificar que el status cambie.
3. Introducir error (device inexistente o password incorrecta) para validar alertas.

---

## 🔄 Flujo sugerido con triggers

| Orden | Script                                 | Frecuencia       | Descripción                            |
| ----- | -------------------------------------- | ---------------- | -------------------------------------- |
| 1️⃣   | `exportReservasCanceladasQueCruzanHoy` | Cada hora (día)  | Actualiza la lista de canceladas.      |
| 2️⃣   | `purgeCanceledReservationsInHikvision` | 5–10 min después | Purga rostros y usuarios en Hikvision. |

---

## 🧰 Dependencias y utilidades clave

* `UrlFetchApp.fetch` / `fetchAll`
* `MailApp.sendEmail`
* `Utilities.computeDigest` (MD5)
* `PropertiesService` (manejo seguro de secretos)
* `SpreadsheetApp` (lectura/escritura de hojas)

---

## 📊 Hojas generadas

| Hoja                               | Función                                             |
| ---------------------------------- | --------------------------------------------------- |
| `Hostaway_Reservas_Canceladas_Hoy` | Datos de reservas canceladas (Hostaway).            |
| `Hikvision_Devices`                | Inventario actualizado de dispositivos del gateway. |
| `Hikvision_Logs`                   | Bitácora con timestamp, nivel y mensaje.            |

---

## 🧱 Mejores prácticas

* Mantener nombres consistentes entre `internalListingName` (Hostaway) y `devName` (Hikvision).
* Revisar periódicamente el mapeo `BUILDING_TO_DEVNAMES`.
* No modificar directamente las hojas mientras los scripts estén activos.
* Documentar cada nueva propiedad o cambio de endpoint.

---

## 📘 Créditos y mantenimiento

**Autor:** Equipo de Tecnología — Vive.Properties
**Responsable:** @ipadilla
**Última revisión:** Octubre 2025
**Scripts involucrados:**

* `Hostaway_Export_Canceladas.gs`
* `Hikvision_Purge_Canceladas.gs`

---

### 🚀 Resultado esperado

> Con esta integración, las cancelaciones en Hostaway se reflejan automáticamente en Hikvision, asegurando que los accesos físicos de huéspedes cancelados sean revocados en tiempo real, manteniendo la seguridad operativa y la trazabilidad en las hojas de control.
