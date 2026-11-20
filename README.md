# TK-001 – CCP de Amazon Connect embebido en un CRM

Reproducción y solución del ticket **TK-004 – Gestión de CCP (Contact Control Panel)**.

## Problema a resolver

Al cargar el CCP de Amazon Connect dentro de un iframe en un CRM corporativo, el panel queda en blanco y la consola del navegador muestra:

> Refused to display in a frame because it set X-Frame-Options to sameorigin

En navegadores actuales el mismo bloqueo aparece como `Framing '...' violates the following Content Security Policy directive: "frame-ancestors 'self'"`.

**Comportamiento esperado:** el CCP debe autenticar e integrarse dentro del iframe del CRM sin bloqueos de seguridad del navegador.

**Causa raíz:** el dominio del CRM no está en la lista de **Orígenes aprobados** de la instancia de Amazon Connect, y un iframe directo no maneja el login ni la comunicación con el CCP.

## Archivos

| Archivo | Descripción |
|---|---|
| `crm.html` | CRM de prueba autocontenido (HTML + CSS + JS). Versión A: iframe directo (reproduce el error). Versión B: `amazon-connect-streams` con `initCCP` (solución). Incluye botón de alternancia y cuadro de estado. |
| `amazon-connect-streams.min.js` | **No incluido en el repo.** Librería oficial de Amazon, hay que descargarla (ver abajo). |

## Cómo usarlo

1. En `crm.html`, reemplazá `REEMPLAZAR-INSTANCIA` por el alias de tu instancia de Connect en **dos lugares**: el `src` del iframe de la versión A y la constante `CCP_URL` de la B. La URL debe terminar en `/ccp-v2`:
   `https://TU-ALIAS.my.connect.aws/ccp-v2`
2. Descargá la librería Streams (ejemplo en PowerShell, sin instalar Node):
   ```powershell
   Invoke-WebRequest https://registry.npmjs.org/amazon-connect-streams/-/amazon-connect-streams-2.31.0.tgz -OutFile streams.tgz
   tar -xzf streams.tgz
   copy package\release\connect-streams-min.js amazon-connect-streams.min.js
   ```
   Fuente oficial: <https://github.com/amazon-connect/amazon-connect-streams>
3. Levantá un servidor local desde la carpeta (no funciona abriendo el archivo con doble clic):
   ```
   python -m http.server 8000
   ```
   y abrí `http://localhost:8000/crm.html`.

## Resumen de pasos

1. Armé `crm.html`: barra "CRM Conecta Company", 3 clientes de ejemplo, panel de 400x600 para el CCP, botón A/B y cuadro de estado.
2. Descargué Amazon Connect Streams y lo dejé junto al HTML.
3. Levanté el servidor local en el puerto 8000.
4. Creé la instancia de Amazon Connect (usuarios almacenados en Connect, administrador, telefonía entrante y saliente; sin Customer Profiles ni Cases).
5. Reemplacé `REEMPLAZAR-INSTANCIA` por el alias en el iframe y en `CCP_URL`.
6. **Reproducción:** con el origen sin aprobar, la versión A muestra el bloqueo `frame-ancestors 'self'` / `X-Frame-Options` en la consola.
7. **Solución:** agregué `http://localhost:8000` en *Orígenes aprobados* de la instancia. Con solo eso el CCP cargó dentro del CRM tanto con el iframe directo (A) como con Streams (B), **con la sesión de Connect ya iniciada en el navegador** (el popup de Streams me llevó directo a un panel de Connect, sin pedirme usuario ni contraseña). Streams (`connect.core.initCCP` con `loginPopup: true` y `softphone.allowFramedSoftphone: true`) es la integración recomendada porque maneja el login y la comunicación entre el CCP y el CRM.

## Solución de problemas: tres errores de configuración que aparecieron

| Qué estaba mal | Cómo se veía |
|---|---|
| Alias sin reemplazar (`REEMPLAZAR-INSTANCIA` en `CCP_URL`) | El popup de login mostraba `DNS_PROBE_FINISHED_NXDOMAIN`. |
| Ruta incompleta (URL sin `/ccp-v2`) | El iframe iba a la página principal de Connect (`/home`), con el bloqueo `frame-ancestors 'self'`. Se parece al error del ticket, pero la causa no es el permiso. |
| Origen sin aprobar | El bloqueo del ticket, con la URL correcta. |

Los dos primeros imitan o tapan al tercero: si la URL está mal, aprobar el origen no arregla nada. Orden de diagnóstico: primero verificar que la URL tenga el alias real y termine en `/ccp-v2` (abrirla a mano en una pestaña); si anda sola pero no dentro del CRM, revisar el origen aprobado.

## Alcance de la prueba

- **Comprobado:** el CCP carga dentro del CRM una vez aprobado el origen (versiones A y B), y Streams abre el popup de login.
- **No comprobado:** el login desde cero (la ventana pidiendo usuario y contraseña con la sesión cerrada) ni las funciones de Streams para reaccionar a llamadas (ficha emergente, aceptar o marcar desde el CRM). Streams las habilita, pero no las probé.

## Notas

- En producción el origen aprobado sería el dominio **HTTPS** del CRM (por ejemplo S3 + CloudFront), no `localhost`.
- El error de la consola puede verse como `X-Frame-Options: sameorigin` (el del ticket) o como `frame-ancestors 'self'` (navegadores actuales). `SAMEORIGIN` equivale a `frame-ancestors 'self'`, y los navegadores modernos priorizan la CSP. Referencias: MDN (CSP `frame-ancestors` y `X-Frame-Options`).
- No subas a este repo datos reales de tu instancia ni credenciales.
- Al terminar las pruebas, eliminá la instancia de Connect si no la vas a usar, para evitar costos.
