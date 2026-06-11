# Blueprint funcional — Plataforma jurídica (versión: Drive-focused)

> Entrega: blueprint funcional detallado para reorganizar la plataforma por ROLES y dejar el flujo de Google Drive como molde funcional. No hay código — sólo especificación y diagramas.

## Resumen ejecutivo
- Problema actual: la UI expone un webhook n8n directamente; arquitectura centrada en workflows técnicos.
- Objetivo: reorganizar por roles (ADMINISTRADOR, ABOGADO, ASISTENTE LEGAL, CLIENTE), ocultar workflows n8n, y dejar operativo el Flow de Google Drive (subida de PDF) como plantilla.

---

## Hallazgos principales (auditoría resumida)
- Webhook n8n expuesto en `puerbas.html` — peligro de seguridad y mala UX.
- Falta de capa API que abstraiga workflows.
- UX no orientada a procesos legales (expedientes, actuaciones, trazabilidad).
- Ausencia de RBAC/ACL y logging de auditoría.

---

**Decisión arquitectural clave**: n8n y sus workflows quedan como infraestructura interna. El frontend sólo llama a la API propia (Backend/API Gateway). El Backend encola/llama a n8n internamente y devuelve un `jobId` para seguimiento.

---

## Roles (detallado)

### ADMINISTRADOR
- Objetivo: gestionar entorno, integraciones y auditoría.
- Permisos: CRUD en todos los módulos, gestionar usuarios, ver logs, reintentar jobs.
- Restricciones: no editar documentos legales como representante.
- Menú lateral: Dashboard, Clientes, Expedientes, Documentos, IA Jurídica, Agenda, Integraciones, Administración.
- Dashboard: KPIs operativos, errores integraciones.
- Acciones permitidas / prohibidas: ver resumen arriba.

### ABOGADO
- Objetivo: dirigir casos, aprobar estrategias y manejar expedientes.
- Permisos: CRUD en expedientes asignados, ejecutar IA, compartir documentos con clientes.
- Restricciones: no gestionar integraciones.
- Menú: Dashboard, Clientes, Mis Expedientes, Documentos, IA Jurídica, Agenda.
- Dashboard: próximos plazos, tareas.

### ASISTENTE LEGAL
- Objetivo: operaciones: subir evidencias, preparar documentos.
- Permisos: subir/etiquetar documentos, programar actuaciones, organizar fichas.
- Restricciones: no aprobar estrategias finales.
- Menú: Clientes, Expedientes asignados, Documentos, Agenda.

### CLIENTE
- Objetivo: colaborar y aportar documentos; ver estado.
- Permisos: ver expedientes compartidos, subir documentos limitados.
- Restricciones: no ver IA ni logs.
- Menú: Mis Casos, Subir Documento, Agenda, Mensajes.

---

## Módulos visibles y mapeo de workflows (catálogo interno)

1. Clientes
   - WF01 Registro Cliente
   - WF02 Validación Cliente

2. Expedientes
   - WF03 Crear Expediente
   - WF04 Notificar Equipo

3. Documentos y Evidencias (Drive-focused)
   - WF_GD_01 Registro de subida (entrada interna)
   - WF_GD_02 Guardar en Google Drive (upload)
   - WF_GD_03 Vincular a Expediente (metadata en DB)
   - WF_GD_04 Notificación y auditoría

4. IA Jurídica
   - WF10 Diagnóstico
   - WF11 Vectorización
   - WF15 Estrategia Legal

5. Agenda y Actuaciones
   - WF20 Crear Actuación
   - WF21 Recordatorios y follow-ups

6. Administración
   - WF30 Integraciones (OAuth)
   - WF31 Auditoría y Logs

> Nota: Los WFs n8n están numerados y documentados en un catálogo interno; NUNCA se exponen al frontend.

---

## Flujo funcional: Subir PDF a Google Drive (molde operativo)

Objetivo: definir contrato API, lifecycle y responsabilidades de cada capa (sin código).

### Comportamiento UI
- Usuario (Cliente o Asistente) abre `Expediente > Documentos > Subir`. Ve un modal con:
  - Campo `Nombre del documento` (texto)
  - Zona de arrastre / botón `Seleccionar PDF`
  - Botón `Enviar al Drive`
- Al enviar, UI realiza POST a `/api/documents/upload` y muestra estado inmediato: `202 Accepted (En cola)` con `jobId`.
- UI subscribe a actualizaciones por WebSocket o realiza polling al endpoint de status.
- Estados mostrados al usuario: `En cola`, `Subiendo`, `Completado`, `Error`.

### Contrato API (sugerido)
- Endpoint: POST `/api/documents/upload` (multipart/form-data)
  - Campos: `file` (binary), `fileName` (string), `expedienteId` (string), `uploaderId` (string, opcional si está en token)
  - Respuesta 202 Accepted: `{ jobId: string, status: 'queued' }`
- Endpoint: GET `/api/jobs/{jobId}`
  - Respuesta 200: `{ jobId, status: 'queued'|'processing'|'completed'|'failed', progress?, result?, error? }`

### Backend responsibilities
- Validar token y permisos (RBAC + ACL sobre `expedienteId`).
- Persistir metadatos de job en DB con estado `queued`.
- Llamada a webhook privado de n8n (o encolar en broker) con payload seguro: `{ jobId, fileStoragePointer, metadata }`.
- Exponer job status al UI y notificar por WebSocket cuando cambie.

### n8n (workflows internos)
- WF_GD_01: recibe payload interno (no público). Accede al archivo (temporal o desde storage interno).
- WF_GD_02: realiza upload a Google Drive usando OAuth (carpeta por expediente). Retorna Drive fileId y URL.
- WF_GD_03: actualiza metadata en DB: vincula fileId, tamaño, mimetype, owner, timestamps.
- WF_GD_04: registra auditoría y notifica por email o rol correspondiente (ej., notificar al abogado asignado).

### Seguridad y recomendaciones
- Webhooks n8n deben ser privados y autenticados (signed tokens o conexión VPC). Nunca exponer URL n8n en frontend.
- Google Drive OAuth: almacenar credenciales en secret store; usar refresh tokens y rotación.
- Tamaño máximo y tipos: validar en backend (ej. 20 MB por archivo) antes de forward.
- Escalado: usar almacenamiento temporal (S3/GCS) para archivos grandes y pasar pointer a n8n.

### Gestión de errores y reintentos
- n8n debe retornar status a backend (success/failure). Backend actualiza job.
- Implementar reintentos con backoff en WF_GD_02 para fallas transitorias.
- Exponer al ADMIN la lista de jobs fallidos con opción `Reintentar`.

---

## Relación Panel → API → Webhook → Workflow (mapa resumido)

- Panel `Subir Documento` (Expediente) → POST `/api/documents/upload` → Backend valida y crea job → Webhook privado → n8n WF_GD_01..04 → Resultado persisted → Backend notifica UI.

- Panel `IA Jurídica > Diagnóstico` → POST `/api/ia/diagnose` → Backend crea request → Webhook privado WF10..WF15 → Resultado almacenado / notificado.

---

## Pantallas que eliminar/ocultar
- Cualquier página o formulario que apunte directamente a una URL de n8n.
- Consolas técnicas/`Webhook test` expuestas a usuarios no-admin.

## Pantallas que fusionar
- La actual `puerbas.html` (formulario independiente) → fusionar como modal/contextual en `Expediente > Documentos`.
- Fusionar paneles de `Subidas` y `Bandeja de auditoría ligera` en una sola vista para roles no-admin.

## Workflows que deben quedar ocultos
- Todos los WF_GD_*; WF01..WF31; solo ADMIN puede ver trazabilidad simplificada.

---

## Diseño SaaS profesional (UX guidelines)
- Layout tipo HubSpot: lateral navigation, main content, top quick actions.
- Terminología legal y consistencia: `Expediente`, `Actuación`, `Prueba`, `Parte`.
- Indicadores de estado y trazabilidad: jobId visible en historial del documento.
- Control de versiones y permisos por documento.

---

## Diagrama de navegación (mermaid)

```mermaid
flowchart LR
  subgraph UI[Interfaz de Usuario]
    A[Dashboard] --> B[Clientes]
    B --> C[Expedientes]
    C --> D[Documentos]
    D --> E[Subir Documento (modal)]
    C --> F[Agenda]
    UIclick[E] --> API
  end

  subgraph API[Backend / API Gateway]
    API --> JOBDB[Job DB]
    API --> n8nWebhook[Webhook Privado n8n]
    JOBDB --> UI
  end

  subgraph n8n[n8n (infra)]
    n8nWebhook --> WF_GD_01
    WF_GD_01 --> WF_GD_02[Upload a Google Drive]
    WF_GD_02 --> WF_GD_03[Registrar Metadata]
    WF_GD_03 --> WF_GD_04[Notificar/Auditar]
    WF_GD_04 --> API
  end

  API -->|notifica| UI
```

---

## Checklist operativo para dejar funcional el flujo Drive (sin código)
1. Registrar credenciales OAuth de Google Drive (Client ID/Secret) y guardarlas en secrets manager.  
2. Implementar endpoint POST `/api/documents/upload` y GET `/api/jobs/{jobId}`.  
3. Implementar job DB (tabla jobs: jobId, uploaderId, expedienteId, status, createdAt, updatedAt, result).  
4. Configurar n8n privado: crear WF_GD_01..WF_GD_04; asegurar webhooks con token.  
5. Configurar almacenamiento temporal si archivos son grandes.  
6. UI: integrar modal `Subir documento` en `Expediente` y manejar estados (202 Accepted, polling/WebSocket).  
7. Pruebas E2E: subir PDF real, validar que Drive devuelve fileId y que el expediente contiene el enlace.  

---

## Siguientes pasos (recomendado)
- Validar credenciales Google Drive y permisos del proyecto GCP.  
- Implementar la API Gateway y job persistence (puedo documentar endpoints OpenAPI si deseas).  
- Preparar pruebas para WF_GD con archivos de ejemplo.

---

Archivo creado: `BLUEPRINT.md` (en la raíz del workspace). Si quieres, genero ahora un `README.md` con instrucciones de ejecución o un diagrama más detallado por pantalla.
