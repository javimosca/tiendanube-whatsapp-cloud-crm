# Tiendanube WhatsApp Cloud CRM & Marketing Engine

> Automatización de campañas, recupero de clientes y bandeja de atención omnicanal con Ficha 360° conectada directo a Meta Cloud API.

Plataforma de CRM y WhatsApp Marketing diseñada para comercios de **Tiendanube** y puntos de venta omnicanal. Sincroniza clientes, órdenes e interacciones presenciales (WiFi / sucursales), segmenta audiencias por comportamiento de compra y ejecuta campañas salientes de WhatsApp mediante la **API Oficial de Meta (Cloud API)** sin intermediarios ni recargos por mensaje. Incluye bandeja de atención multiagente y vista unificada (Ficha 360°) de cada comprador.

---

## Módulos Principales

| Módulo | Funcionalidad |
|---|---|
| **Ingesta Omnicanal & SCV** | Unifica clientes y órdenes de Tiendanube junto con captación en sucursales físicas dentro de una vista única de contacto (`unified_contacts`). |
| **Segmentación & Cross-Sell** | Cruce automático por historial de compras, recurrencia, ticket promedio y estado crediticio para campañas de recompra y venta cruzada. |
| **Campañas Outbound** | Envío masivo en segundo plano (`background tasks`) de plantillas oficiales aprobadas por Meta, con barra de progreso en tiempo real y control de lotes. |
| **Gestor de Plantillas Meta** | Creación, consulta de estado de aprobación y baja de plantillas directo en Meta Graph API, incluyendo subida de imágenes de encabezado y botones de acción. |
| **Redirector & Tracker `/go/{codigo}`** | Acortador propio para botones de plantillas que registra clics reales (filtrando bots por hash de IP) y redirige al chat con atribución de campaña (`CAMP-{CODIGO}`). |
| **Bandeja de Mensajes Multiagente** | Inbox unificado por cliente (`phone10`) con validación automática de ventana de 24 h (texto libre vs. plantilla de seguimiento), clasificación automática de intenciones y cierre tipificado. |
| **Ficha 360° (`Ctrl+K`)** | Panel lateral global en tiempo real con el historial de órdenes en Tiendanube, estado de pagos/envíos, campañas recibidas y conversaciones previas. |
| **Notificaciones Web Push** | Alertas nativas en escritorio vía Service Worker (VAPID) ante nuevos mensajes entrantes, incluso con el navegador cerrado. |

---

## Arquitectura Técnica

```text
[Tiendanube Webhooks / API]           [ERP / POS Connector]         [Meta Cloud API (v23.0)]
 POST /webhook/tiendanube              POST /whatsapp/crosssell/     Webhooks de mensajes
 (Órdenes y Clientes)                  ingesta (por DNI / Email)     y estados de entrega
        │                                     │                             │
        ▼                                     ▼                             ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│  Backend FastAPI (Python 3.11+ · asyncpg · httpx)                                │
│  main.py · whatsapp_routes.py · whatsapp_provider.py · wa_entrantes · bandeja.py │
│  ficha.py · web_push.py · db_ddl.py                                              │
│                                                                                  │
│  Sincronización: Matching de cliente → Historial de órdenes → Etapa del funnel   │
│  Campañas:       Segmento → Exclusión (bajas/cadencia) → Envío async → Progreso  │
│  Entrantes:      Opt-out automático → Clasificación → Bandeja → Aviso Web Push   │
└──────────────────────────────────────────────────────────────────────────────────┘
        │                                                                   │
        ▼                                                                   ▼
   PostgreSQL                                                        Brevo API (Email)
                                                                     Listas y atributos

┌──────────────────────────────────────────────────────────────────────────────────┐
│  Panel Administrativo Next.js (Vercel)                                           │
│  NextAuth + RBAC → Proxy seguro /api/proxy/[...path] → Backend con X-Admin-Key   │
│  Las credenciales de administración nunca se exponen en el cliente.              │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### Stack Tecnológico

- **Backend:** Python 3.11+, FastAPI, `databases` + `asyncpg`, `httpx`
- **Base de datos:** PostgreSQL (con migraciones automáticas no bloqueantes mediante `lock_timeout`)
- **Frontend:** Next.js, NextAuth (control de roles `admin`, `agente`, `viewer`), Tailwind CSS
- **E-commerce:** Tiendanube REST API / Webhooks
- **Mensajería:** WhatsApp Cloud API (Meta Graph API directo, sin BSPs intermediarios)
- **Email Marketing:** Brevo API
- **Notificaciones:** Web Push nativo (VAPID + Service Worker)

---

## Estructura del Proyecto

```text
main.py                    # App FastAPI: webhooks, sincronización, SCV y redirector /go/
whatsapp_routes.py         # Router /whatsapp: configuración, plantillas, campañas, progreso, clics y cross-sell
whatsapp_provider.py       # MetaCloudProvider, MetaTemplateManager y MetaMediaUploader
bandeja.py                 # Bandeja de entrada multiagente y motor de clasificación por reglas
ficha.py                   # Endpoint GET /clientes/ficha (Ficha 360° unificada)
web_push.py                # Suscripciones y despacho de notificaciones Web Push (VAPID)
db_ddl.py                  # Verificación e inicialización de schema con control de locks
requirements.txt

pages/
  index.js                 # Dashboard operativo: últimos contactos e indicadores de campaña
  scv.js                   # Gestión de contactos unificados (Single Customer View)
  campanas.js              # Lanzador y monitor de campañas de WhatsApp
  whatsapp.js              # ABM de plantillas conectadas a Meta
  respuestas.js            # Bandeja de atención al cliente en tiempo real
  admin.js · login.js      # Configuración y acceso
  api/auth/[...nextauth].js
  api/proxy/[...path].js   # Proxy autenticado hacia el backend FastAPI
components/                # Navegación, modales, Ficha 360° y componentes de UI
lib/                       # Clientes HTTP, adaptadores y utilidades Web Push
public/sw.js               # Service Worker para notificaciones en segundo plano
middleware.js              # Protección de rutas por sesión y rol
```

---

## Seguridad y Control de Roles (RBAC)

- Todas las peticiones del panel pasan por `/api/proxy/[...path]`, que valida la sesión activa (`401`), verifica los permisos del rol (`403`) y firma la comunicación hacia el backend con `X-Admin-Key`.
- Las claves de administración y tokens de Meta o Tiendanube operan exclusivamente del lado del servidor.

| Rol | Permisos |
|---|---|
| `admin` | Acceso total a configuración, plantillas, campañas, segmentos y administración. |
| `agente` | Atención en Bandeja de Mensajes (respuestas, plantillas de seguimiento, cambio de estado), consulta de Ficha 360° y alertas Web Push. |
| `viewer` | Acceso de solo lectura a métricas, alcance de segmentos y tableros. |

---

## Reglas de Negocio y Protección de Cuenta Meta

### Higiene de Audiencias y Exclusiones Automáticas
Para proteger la calidad del número en Meta (*Quality Rating*) y evitar bloqueos, el motor aplica exclusiones estrictas antes de cada despacho:
- **Opt-out automático (`wa_bajas`):** Procesamiento inmediato cuando un usuario responde `BAJA` o cuando un agente cierra una conversación marcando baja solicitada.
- **Control de cadencia (30 días):** Ningún contacto recibe más de una campaña promocional dentro de una ventana móvil de 30 días.
- **Deduplicación por teléfono normalizado (`phone10`):** Garantiza un único envío por línea aunque el cliente figure en múltiples órdenes o registros.
- **Corte temprano (Fail-fast):** Si Meta rechaza el recurso multimedia del encabezado en el primer envío, la campaña se aborta automáticamente (`abortada`) para no disparar errores en lote.

### Estados del Motor de Campañas

| Estado | Comportamiento |
|---|---|
| `enviando` | Tarea asíncrona en ejecución; el frontend consulta `/whatsapp/campaigns/{id}/progreso` cada 2 s. |
| `parcial` | Lote completado alcanzando el tope operativo configurado por tanda. |
| `finalizada` | Audiencia objetivo procesada en su totalidad. |
| `abortada` | Cancelación preventiva por rechazo de plantilla o multimedia en Meta. |
| `error` | Excepción registrada en el motor con detalle diagnóstico en `error_motivo`. |
| `interrumpida` | Detección automática por monitoreo de *heartbeat* (> 180 s sin actividad). |

### Bandeja de Atención y Ventana de 24 Horas
- **Dentro de las 24 h del último mensaje del cliente:** Habilita respuesta en texto libre.
- **Fuera de la ventana de 24 h:** Bloquea el texto libre y habilita el re-contacto mediante la plantilla transaccional homologada (`seguimiento_consulta`), limitada a un envío cada 7 días por contacto.
- **Clasificación automática de entrantes:** Reglas de enrutamiento por intención (`baja` -> `reclamo` -> `no_interesa` -> `ya_cliente` -> `interes`).

---

## Instalación y Desarrollo Local

**Requisitos:** Python 3.11+, Node.js 18+, PostgreSQL.

```bash
# 1. Clonar el repositorio
git clone [https://github.com/tu-usuario/tiendanube-whatsapp-cloud-crm.git](https://github.com/tu-usuario/tiendanube-whatsapp-cloud-crm.git)
cd tiendanube-whatsapp-cloud-crm

# 2. levantar el Backend
python3 -m venv venv
source venv/bin/activate        # En Windows: venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env            # Configurar variables de entorno
uvicorn main:app --reload --port 10000

# 3. Levantar el Frontend (en otra terminal)
npm install
npm run dev                     # Disponible en http://localhost:3000
```

---

## Variables de Entorno Principales (`.env.example`)

| Variable | Descripción |
|---|---|
| `DATABASE_URL` | Cadena de conexión a PostgreSQL |
| `ADMIN_API_KEY` | Clave de autenticación interna (soporta múltiples claves separadas por coma para rotación sin downtime) |
| `TN_APP_ID`, `TN_CLIENT_SECRET` | Credenciales OAuth de la aplicación en Tiendanube Partners |
| `TN_WEBHOOK_SECRET` | Firma de verificación para webhooks entrantes de Tiendanube |
| `META_ACCESS_TOKEN` | System User Token permanente de Meta Business |
| `META_PHONE_NUMBER_ID` | Identificador del número emisor en WhatsApp Cloud API |
| `META_WABA_ID` | Identificador de la cuenta de WhatsApp Business (gestión de plantillas) |
| `META_WA_APP_ID` | App ID en Meta Developers (subida de imágenes para encabezados) |
| `META_GRAPH_VERSION` | Versión de la API de Meta (default `v23.0`) |
| `WA_PLANTILLA_SEGUIMIENTO` | Nombre de la plantilla homologada para re-contacto post 24 h |
| `VAPID_PRIVATE_KEY` | Clave privada para firma de notificaciones Web Push |
| `BREVO_API_KEY` | API Key para sincronización de listas de correo electrónico |
| `NEXT_PUBLIC_API_URL` | URL pública del backend para el cliente Next.js |
| `NEXTAUTH_SECRET` | Clave de firmado de sesiones para NextAuth |# tiendanube-whatsapp-cloud-crm
WhatsApp Cloud API (Meta Direct) CRM, outbound campaign engine &amp; 360° customer inbox integrated with Tiendanube.
