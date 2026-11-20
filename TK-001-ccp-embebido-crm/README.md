# TK-001 – CCP de Amazon Connect embebido en un CRM

Reproducción y solución del ticket **TK-001 – Gestión de CCP (Contact Control Panel)**.

## Problema a resolver

Al cargar el CCP de Amazon Connect dentro de un iframe en un CRM corporativo, el panel queda en blanco y la consola del navegador muestra:

> Refused to display in a frame because it set X-Frame-Options to sameorigin

En navegadores actuales el mismo bloqueo aparece como `Framing '...' violates the following Content Security Policy directive: "frame-ancestors 'self'"`.

**Comportamiento esperado:** el CCP debe autenticar e integrarse dentro del iframe del CRM sin bloqueos de seguridad del navegador.

**Causas posibles** (dan el mismo síntoma y se pueden confundir):

1. **Origen sin aprobar:** el dominio del CRM no está en la lista de **Orígenes aprobados** de la instancia de Amazon Connect. Es la causa que plantea el ticket y la que respalda la documentación de AWS (ver *Respaldo en la documentación oficial*). **La reproduje al final de mi prueba:** con la URL ya corregida, quité el origen de la lista y el error volvió a aparecer (ver *Evidencia*, punto 8; sin captura).
2. **URL del CCP mal armada** (sin `/ccp-v2`, o con el alias sin reemplazar). Esta sí la **observé** en mi prueba y produjo el mismo bloqueo en la consola.

## Archivos

| Archivo | Descripción |
|---|---|
| `crm.html` | CRM de prueba autocontenido (HTML + CSS + JS). Versión A: iframe directo (reproduce el error). Versión B: `amazon-connect-streams` con `initCCP` (solución). Incluye botón de alternancia y cuadro de estado. |
| `lib/amazon-connect-streams.min.js` | Librería oficial Amazon Connect Streams (v2.31.0, licencia Apache-2.0). |
| `lib/LICENSE.txt` | Licencia de la librería Streams. |
| `capturas/captura-error.jpg` | Captura del error del ticket. |

## Cómo usarlo

1. En `crm.html`, reemplazá `REEMPLAZAR-INSTANCIA` por el alias de tu instancia de Connect en **dos lugares**: el `src` del iframe de la versión A y la constante `CCP_URL` de la B. La URL debe terminar en `/ccp-v2`:
   `https://TU-ALIAS.my.connect.aws/ccp-v2`
2. La librería Streams ya está en `lib/`. Si querés bajarla vos desde la fuente oficial (ejemplo en PowerShell, sin instalar Node):
   ```powershell
   Invoke-WebRequest https://registry.npmjs.org/amazon-connect-streams/-/amazon-connect-streams-2.31.0.tgz -OutFile streams.tgz
   tar -xzf streams.tgz
   copy package\release\connect-streams-min.js lib\amazon-connect-streams.min.js
   ```
   Fuente oficial: <https://github.com/amazon-connect/amazon-connect-streams>
3. Levantá un servidor local desde esta carpeta (no funciona abriendo el archivo con doble clic):
   ```
   python -m http.server 8000
   ```
   y abrí `http://localhost:8000/crm.html`.

## Resumen de pasos

1. Armé `crm.html`: barra "CRM Conecta Company", 3 clientes de ejemplo, panel de 400x600 para el CCP, botón A/B y cuadro de estado.
2. Descargué Amazon Connect Streams y lo dejé en la carpeta `lib/`, junto al HTML.
3. Levanté el servidor local en el puerto 8000.
4. Creé la instancia de Amazon Connect (usuarios almacenados en Connect, administrador, telefonía entrante y saliente; sin Customer Profiles ni Cases).
5. Reemplacé `REEMPLAZAR-INSTANCIA` por el alias en el iframe y en `CCP_URL`.
6. **Error observado:** la versión A mostró en la consola el bloqueo `frame-ancestors 'self'`. **Aclaración importante:** el primer mensaje que copié lo vi con el iframe apuntando a la raíz del sitio de Connect (sin `/ccp-v2`), es decir, por una URL mal armada. El caso exacto del ticket (URL correcta y origen sin aprobar) lo reproduje más tarde, al quitar el origen de la lista (paso 7 y *Evidencia*, punto 8).
7. **Resultado:** corregí la URL del CCP a `https://conecta-test.my.connect.aws/ccp-v2` y, con `http://localhost:8000` en *Orígenes aprobados* de la instancia, el CCP cargó dentro del CRM tanto con el iframe directo (A) como con Streams (B), **con la sesión de Connect ya iniciada en el navegador** (el popup de Streams me llevó directo a un panel de Connect, sin pedirme usuario ni contraseña). Después hice la prueba de contraste: con la URL correcta, **quité `http://localhost:8000` de la lista y el error volvió a aparecer**, de modo que con la URL bien armada el origen aprobado es lo que decide si carga o no (ver *Evidencia*, punto 8). Streams (`connect.core.initCCP` con `loginPopup: true` y `softphone.allowFramedSoftphone: true`) es la integración recomendada porque maneja el login y la comunicación entre el CCP y el CRM.

## Solución de problemas: tres causas que dan síntomas parecidos

Las tres aparecieron en mi prueba. La tercera, que es la del ticket, la reproduje al final, sin captura.

| Qué estaba mal | Cómo se veía |
|---|---|
| Alias sin reemplazar (`REEMPLAZAR-INSTANCIA` en `CCP_URL`) | El popup de login mostraba `DNS_PROBE_FINISHED_NXDOMAIN`. |
| Ruta incompleta (URL sin `/ccp-v2`) | El iframe iba a la página principal de Connect (`/home`), con el bloqueo `frame-ancestors 'self'`. Se parece al error del ticket, pero la causa no es el permiso. |
| Origen sin aprobar | Es la causa que describe el ticket y la que respalda la documentación de AWS (ver más abajo). La reproduje quitando el origen de la lista con la URL ya corregida (sin captura). |

Los dos primeros imitan o tapan al tercero: si la URL está mal, aprobar el origen no arregla nada. Orden de diagnóstico: primero verificar que la URL tenga el alias real y termine en `/ccp-v2` (abrirla a mano en una pestaña); si anda sola pero no dentro del CRM, revisar el origen aprobado.

## Alcance de la prueba

- **Comprobado:** con la URL correcta y el origen aprobado, el CCP carga dentro del CRM (versiones A y B), y Streams abre el popup de login. También comprobé que una URL mal armada (sin `/ccp-v2` o con el alias sin reemplazar) produce errores que se parecen al del ticket.
- **Contraste con el origen:** con la URL correcta, el CCP cargó con el origen aprobado y el error volvió al quitarlo. Lo constaté yo en la prueba, pero no conservé captura ni el texto exacto del mensaje.
- **No comprobado:** que Amazon Connect envíe exactamente el encabezado `X-Frame-Options` (no pude ver los encabezados de respuesta); el login desde cero (la ventana pidiendo usuario y contraseña con la sesión cerrada); ni las funciones de Streams para reaccionar a llamadas (ficha emergente, aceptar o marcar desde el CRM). Streams las habilita, pero no las probé.

## Evidencia de la prueba (texto copiado tal cual de mi consola y de la herramienta)

No conservé capturas de pantalla de la instancia (se eliminó al terminar). Esto es el texto que copié durante la prueba, en el orden en que ocurrió:

1. **Cuadro de estado, primera carga de la versión B (el archivo de Streams todavía no se encontraba):**
   ```
   [22:51:04] Mostrando versión B.
   [22:51:04] ERROR: no se cargó amazon-connect-streams.min.js. Verificá que esté subido junto al HTML y que el nombre coincida.
   ```
2. **Consola, versión A, con el iframe apuntando a la raíz del sitio de Connect (sin `/ccp-v2`):**
   ```
   Framing 'https://conecta-test.my.connect.aws/' violates the following Content Security Policy directive: "frame-ancestors 'self'". The request has been blocked.
   ```
3. **Consola, versión B (misma URL incorrecta) con los avisos de Streams:**
   ```
   [WARN] [null]: ACK_TIMEOUT occurred. Attempting to authenticate.
   [WARN] [null]: Attempting to pop the login page if not already open.
   ```
4. **Popup de login con el alias sin reemplazar en `CCP_URL`:**
   ```
   https://reemplazar-instancia.my.connect.aws/ccp-v2
   DNS_PROBE_FINISHED_NXDOMAIN
   ```
5. **Pestaña Network del navegador, versión A:** la petición del iframe (`crm.html:92`) devolvió un `302` que terminó en `.../home` con estado `(blocked:other)`. Esa petición iba a la raíz del sitio, no a `/ccp-v2`.
6. **Cuadro de estado, versión B, después de corregir el alias pero antes de corregir la ruta:**
   ```
   [23:26:10] Mostrando versión B.
   [23:26:10] initCCP ejecutado. Esperando respuesta del CCP...
   [23:26:25] AVISO: pasaron 15 s sin inicializar. Revisá: 1) origen aprobado en Amazon Connect, 2) HTTPS, 3) popup bloqueado por el navegador.
   ```
7. **Resultado final:** después de corregir la URL a `https://conecta-test.my.connect.aws/ccp-v2`, con el origen `http://localhost:8000` aprobado, el CCP apareció dentro del CRM en las dos versiones. Esto lo constaté a ojo y no tengo texto de consola ni captura de ese momento.
8. **Prueba de contraste:** con la URL correcta y las dos versiones funcionando, quité `http://localhost:8000` de la lista de Orígenes aprobados y el error volvió a aparecer. Lo constaté yo en la prueba; no conservé captura ni el texto exacto del mensaje.

## Respaldo en la documentación oficial

Cada parte de la solución, con la fuente que la respalda. Verifiqué las fuentes el 20/11/2026.

| Parte de la solución | Qué dice la fuente | Fuente |
|---|---|---|
| **Aprobar el origen del CRM** | Todos los dominios que embeben el CCP de una instancia deben estar permitidos de forma explícita. Pasos: instancia → *Approved origins* → *Add origin* → escribir la URL → *Add*. | [Amazon Connect – Use an allowlist for integrated applications](https://docs.aws.amazon.com/connect/latest/adminguide/app-integration.html) |
| **La URL del CCP debe ser `/ccp-v2`** | Con Streams, la URL de `initCCP()` se actualiza de `/ccp#` a `/ccp-v2`, y el dominio se agrega a la lista de orígenes aprobados. | [Upgrade your CCP when using the Streams API](https://docs.aws.amazon.com/connect/latest/adminguide/upgrade-ccp-streams-api.html) |
| **Streams y la lista de orígenes** | Permitir las páginas es el primer paso para usar Streams. Cada entrada identifica protocolo, host y puerto (por eso se agrega `http://localhost:8000` con su puerto). | [README de Amazon Connect Streams](https://github.com/amazon-connect/amazon-connect-streams), sección *Allowlisting* |
| **Opciones de `initCCP`** | `ccpUrl` es obligatoria. `loginPopup` es opcional y por defecto está activa. `allowFramedSoftphone` es opcional y por defecto está en `false`: sin ella el softphone (micrófono y parlantes) no puede estar dentro de un iframe. | [README de Streams](https://github.com/amazon-connect/amazon-connect-streams), sección `connect.core.initCCP()` |
| **El aviso `ACK_TIMEOUT` en la consola** | El evento `ACK_TIMEOUT` se sigue disparando en el login inicial. Por eso apareció en mi prueba antes de abrirse el popup, y no indica un fallo por sí solo. | [README de Streams](https://github.com/amazon-connect/amazon-connect-streams), historial de versiones (2.18.7) |
| **`X-Frame-Options` vs `frame-ancestors`** | `SAMEORIGIN` permite embeber solo si todos los marcos ancestros tienen el mismo origen. MDN recomienda `frame-ancestors` por tener más opciones y muestra `frame-ancestors 'self'` como el equivalente de `SAMEORIGIN`. | [MDN – X-Frame-Options](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/X-Frame-Options) y [MDN – CSP frame-ancestors](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Security-Policy/frame-ancestors) |
| **Qué gana si están los dos encabezados** | El navegador respeta `frame-ancestors` e ignora `X-Frame-Options`. MDN no lo dice de forma explícita: lo respalda la implementación de Chromium. | [Chromium – CSP 'frame-ancestors' should override 'x-frame-options'](https://codereview.chromium.org/1326823003) |

### Diferencias entre lo documentado y lo que observé

- **HTTPS:** el README de Streams dice que los dominios permitidos deben ser HTTPS. En mi prueba, la consola de Connect aceptó `http://localhost:8000` y el CCP cargó. Esa excepción para `localhost` **no la encontré documentada**: es una observación de mi prueba. En producción hay que usar HTTPS.
- **Nombre del menú:** la guía de AWS lo llama *Approved origins* y el README de Streams, *Application integration*. La ubicación varía según la versión de la consola.
- **Mensaje de error:** el ticket cita `X-Frame-Options: sameorigin` y mi navegador mostró `frame-ancestors 'self'`. Que sean la misma regla en versión vieja y moderna está respaldado por MDN y Chromium (filas anteriores). Que Amazon Connect envíe exactamente ese encabezado no lo pude verificar, porque la instancia ya estaba borrada.

## Notas

- En producción el origen aprobado sería el dominio **HTTPS** del CRM (por ejemplo S3 + CloudFront), no `localhost`.
- El error de la consola puede verse como `X-Frame-Options: sameorigin` (el del ticket) o como `frame-ancestors 'self'` (navegadores actuales). Ver la sección anterior para las fuentes.
- No subas a este repo datos reales de tu instancia ni credenciales.
- Al terminar las pruebas, eliminá la instancia de Connect si no la vas a usar, para evitar costos.
