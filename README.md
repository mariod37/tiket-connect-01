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
7. **Solución:** agregué `http://localhost:8000` en *Orígenes aprobados* de la instancia y usé la versión B (`connect.core.initCCP` con `loginPopup: true` y `softphone.allowFramedSoftphone: true`). El CCP cargó autenticado dentro del panel.

## Notas

- En producción el origen aprobado sería el dominio **HTTPS** del CRM (por ejemplo S3 + CloudFront), no `localhost`.
- Un error de configuración que me costó tiempo: dejar la URL sin `/ccp-v2`. Sin esa ruta el iframe va a la página principal de Connect, que nunca se puede embeber.
- No subas a este repo datos reales de tu instancia ni credenciales.
- Al terminar las pruebas, eliminá la instancia de Connect si no la vas a usar, para evitar costos.
