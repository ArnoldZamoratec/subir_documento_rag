README — Configurar flujo "Enviar PDF a Drive" (sin código)

Objetivo
- Documentar pasos operativos para dejar funcional el flujo de subida a Google Drive, manteniendo n8n como infraestructura interna.

Archivos creados
- `OPENAPI.yaml`: contrato para `/api/documents/upload` y `/api/jobs/{jobId}`
- `puerbas.html`: formulario de ejemplo (debe integrarse como modal en `Expediente > Documentos`)
- `BLUEPRINT.md`: blueprint funcional y catálogo de workflows

Requisitos previos
- Proyecto Google Cloud con Drive API habilitada y credenciales OAuth (Client ID/Secret)
- Servidor backend (API Gateway) que valide tokens JWT y cree jobs en una tabla `jobs`
- n8n privado (webhooks protegidos) con WF_GD_01..WF_GD_04 desplegados
- Almacenamiento temporal (S3/GCS) recomendado para archivos mayores

Pasos operativos (alto nivel)
1) Registrar credenciales Google Drive
   - En Google Cloud Console: habilitar Drive API, crear credenciales OAuth (Desktop o Web)
   - Copiar `CLIENT_ID` y `CLIENT_SECRET` al secret manager del entorno

2) Configurar n8n (privado)
   - Crear workflows internos: WF_GD_01..WF_GD_04 (ver `BLUEPRINT.md`)
   - Asegurar webhooks con tokens/signature y restringir acceso (IP/VPC si es posible)

3) Diseñar tabla `jobs`
   - Campos mínimos: jobId, uploaderId, expedienteId, status, payloadPointer, result, createdAt, updatedAt

4) Implementar contrato API (referirse a `OPENAPI.yaml`)
   - POST `/api/documents/upload`: validar permisos, persistir job queued, retornar `jobId`
   - GET `/api/jobs/{jobId}`: devuelve estado y resultado

5) Flujo de archivo
   - Backend recibe multipart; para archivos grandes, subir temporalmente a storage y pasar pointer al webhook n8n
   - n8n realiza upload final a Google Drive y retorna fileId
   - Backend actualiza job y notifica UI por WebSocket/Push

6) Pruebas manuales (sugerencia)
   - Paso 1: obtener token de autenticación (JWT) para un usuario con permisos
   - Paso 2: enviar `curl` de prueba (ejemplo):

```bash
curl -X POST "https://api.tu-dominio.com/api/documents/upload" \
  -H "Authorization: Bearer $TOKEN" \
  -F "expedienteId=EXP123" \
  -F "file=@/ruta/a/ejemplo.pdf" \
  -F "fileName=Mi_Documento.pdf"
```

- Esperar `202 Accepted` y recibir `jobId`.
- Consultar estado:

```bash
curl -X GET "https://api.tu-dominio.com/api/jobs/<jobId>" -H "Authorization: Bearer $TOKEN"
```

7) Buenas prácticas
   - No exponer URL de n8n en frontend.
   - Limitar tamaño de archivo y tipos en backend.
   - Registrar auditoría de accesos y descargas.
   - Implementar reintentos con backoff en n8n para uploads fallidos.

Próximos pasos que puedo ejecutar (sin generar código aún)
- Generar un OpenAPI más detallado (ej.: seguridad, ejemplos y responses).
- Crear `README_ENV.md` con variables de entorno y comandos de configuración para n8n y Google OAuth.
- Preparar checklist de pruebas E2E.

Dime cuál de los anteriores ejecutar ahora o si quieres que genere `README_ENV.md` inmediatamente.
