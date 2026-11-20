# Connect

Portafolio de mis avances con **Amazon Connect**: cada carpeta es un ticket con el problema, los archivos usados y cómo lo resolví.

## Tickets

| Ticket | Problema | Solución | Carpeta |
|---|---|---|---|
| TK-001 | El CCP embebido en un CRM se bloquea (`X-Frame-Options` / `frame-ancestors`) | Aprobar el origen del CRM en Amazon Connect y usar Amazon Connect Streams | [`TK-001-ccp-embebido-crm/`](TK-001-ccp-embebido-crm/) |

## Cómo está organizado cada ticket

```
TK-00X-nombre/
├── README.md     ← problema, causa, solución, alcance de la prueba
├── (archivos del ticket)
├── lib/          ← librerías de terceros (con su licencia)
└── capturas/     ← evidencia
```

## Notas

- No se suben credenciales ni datos reales de instancias. Los alias se dejan como `REEMPLAZAR-INSTANCIA`.
- La librería Amazon Connect Streams es de Amazon y se incluye con su licencia (Apache-2.0).
