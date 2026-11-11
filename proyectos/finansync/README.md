# FinanSync

Cloud Computing 2026 · Universidad Galileo

- **Integrantes:** Roberto Javier Molina López, Luis Gabriel Córdova Archila
- **Proveedor cloud:** Google Cloud Platform (GCP)
- **Servicios principales:** Cloud Run, Cloud Build y Artifact Registry, Cloud Storage, Cloud Vision, Vertex AI (Gemini), Cloud Scheduler, Cloud IAM, Cloud Logging y Monitoring (con Supabase como base de datos y autenticación)
- **Repositorio:** https://github.com/RobertoJavier1/FinanSync-v2
- **Despliegue:** https://finansync.me
- **Documentación PDF:** [documentacion.pdf](documentacion.pdf)
- **Video:** [Video de demostración](video.mp4)

![Arquitectura de FinanSync](img/diagrama-arquitectura.png)

---

Aplicación web de finanzas personales construida con **Next.js 15 + React 18 + TypeScript + Tailwind CSS**, conectada a **Supabase (Postgres + Auth)** y desplegada en **Google Cloud Run**. Usa **Gemini 3.8 Flash** (vía Vertex AI) para el chat financiero, las perspectivas personalizadas y la lectura de facturas.

Los datos viven en cinco tablas de Postgres con Row Level Security y el esquema completo está en [`supabase/migrations/`](https://github.com/RobertoJavier1/FinanSync-v2/tree/main/supabase/migrations).

---

## Live Demo

**[https://finansync.me](https://finansync.me)** — desplegada en Google Cloud Run con dominio propio.

---

## Video demo

[Video de demostración](video.mp4)

---

## Servicios de Google Cloud

Ocho servicios del proyecto `finansync-74cd1`, región `us-central1`.

| Servicio | Para qué | Dónde se ve |
|---|---|---|
| **Cloud Run** | Ejecuta la app en un contenedor y escala a cero cuando no hay tráfico | `Dockerfile` |
| **Cloud Build** | Construye la imagen y despliega una revisión nueva en cada push a `main` | `cloudbuild.yaml` |
| **Cloud Storage** | Bucket privado con las imágenes de facturas, servidas con URL firmadas | `lib/gcs.ts` |
| **Cloud Vision** | Extrae el texto de la factura antes de llamar al modelo | `lib/vision.ts` |
| **Vertex AI** | Gemini para el chat, las perspectivas y la lectura de facturas | `lib/vertex.ts` |
| **Cloud Scheduler** | Dispara el estado de cuenta el día 1 de cada mes (`0 8 1 * *`) | `app/api/reportes/mensual/route.ts` |
| **Cloud IAM** | Una cuenta de servicio por tarea, con permisos acotados y ninguna llave en producción | `lib/google-auth.ts` |
| **Cloud Logging y Monitoring** | Registros en JSON y dos políticas de alerta por correo | `monitoring/` |

Cada servicio corre con su propia cuenta y solo con los permisos que necesita:

| Cuenta | La usa | Permisos |
|---|---|---|
| `finansync-runtime` | Cloud Run (la app) | Vertex AI, Storage solo sobre el bucket `finansync-facturas`, escribir logs y métricas |
| `finansync-scheduler` | Cloud Scheduler | Invocar solo el servicio `finansync` |
| `finansync-deployer` | Cloud Build | Desplegar revisiones en Cloud Run |
| `finansync-storage` | Desarrollo local | Storage solo sobre el bucket `finansync-facturas` |

Ningún permiso está dado a nivel de proyecto si se puede dar sobre el recurso. Artifact Registry guarda las imágenes que produce Cloud Build.

---

## Stack tecnológico

| Tecnología | Versión | Uso |
|---|---|---|
| Next.js | 15.5 | Framework principal — App Router, API Routes, SSR |
| React | 18.3 | UI con hooks y Context API |
| TypeScript | 5 | Tipado estático en toda la app |
| Tailwind CSS | 3.4 | Estilos utilitarios |
| TanStack Query | 5 | Caché compartida de datos entre páginas |
| Recharts | 2.15 | AreaChart, PieChart, LineChart, BarChart |
| Lucide React | 0.475 | Iconografía |
| emoji-mart | 5.6 | Selector de emojis para presupuestos y metas (carga diferida) |
| jsPDF + jspdf-autotable | 4 / 5 | PDF de Reportes (navegador) y del estado de cuenta (servidor) |
| react-markdown | 10 | Formato de las respuestas del Chat IA |
| Supabase Auth | 2.111 | Autenticación email + Google (OAuth) |
| Supabase Postgres | 2.111 | Base de datos relacional con Row Level Security |
| @supabase/ssr | 0.12 | Sesión en cookies, compartida entre cliente y servidor |
| Vertex AI (Gemini 3.8 Flash) | — | Chat IA, perspectivas e interpretación de facturas |
| Cloud Vision | 6 | OCR del texto de las facturas |
| Cloud Storage | 8 | Imágenes de facturas en un bucket privado |
| Resend | API HTTP | Envío del estado de cuenta mensual por correo |
| jsDelivr Currency API | pública | Tasas de cambio en tiempo real (caché 24 h) |

---

## Configuración local

### 1. Requisitos previos

| Herramienta | Versión mínima |
|---|---|
| Node.js | 20 |
| npm | 9 |

### 2. Clonar e instalar

```bash
git clone https://github.com/RobertoJavier1/FinanSync-v2
cd FinanSync-v2
npm install
```

### 3. Variables de entorno

Hay una plantilla lista en `.env.local.example`; basta copiarla y rellenar los valores:

```bash
cp .env.local.example .env.local
```

| Variable | Para qué |
|---|---|
| `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Conexión a Supabase (*Project Settings → API*) |
| `GCP_RUNTIME_PROJECT_ID`, `GCP_RUNTIME_CLIENT_EMAIL`, `GCP_RUNTIME_PRIVATE_KEY` | Llave de la cuenta de servicio `finansync-runtime` (Vision y Vertex AI) |
| `VERTEX_LOCATION` | Endpoint de Vertex AI. Vacío = `global` (ver [Integración con Gemini](#-integración-con-gemini)) |
| `GCS_BUCKET_NAME`, `GCS_PROJECT_ID`, `GCS_CLIENT_EMAIL`, `GCS_PRIVATE_KEY` | Bucket de facturas y llave de la cuenta `finansync-storage` |
| `SUPABASE_SERVICE_ROLE_KEY` | Solo para el job del estado de cuenta mensual (nunca con `NEXT_PUBLIC_`) |
| `REPORTE_CRON_TOKEN`, `REPORTE_SERVICE_ACCOUNT` | Autorización del job mensual (token manual y cuenta de Cloud Scheduler) |
| `RESEND_API_KEY`, `CORREO_REMITENTE` | Envío de correo. Sin llave, el job solo registra lo que habría mandado |

**Cuentas de servicio:** en la consola de Google Cloud, *IAM y administración* → *Cuentas de servicio* → la cuenta → *Claves* → *Agregar clave* → JSON. Del archivo se copian `project_id`, `client_email` y `private_key`. `finansync-runtime` necesita las APIs de Cloud Vision y Vertex AI habilitadas y el rol *Usuario de Vertex AI*.

> En Cloud Run las llaves de las cuentas de servicio no se configuran: el servicio se ejecuta con la identidad de `finansync-runtime` y obtiene las credenciales del entorno, así que no hay ninguna llave guardada.

> La clave `anon` es pública por diseño: quien limita el acceso a los datos son las políticas RLS. La `service_role` key nunca debe ponerse en una variable `NEXT_PUBLIC_`.
>
> `.env.local` está en `.gitignore` y nunca se versiona.

### 4. Crear el schema y habilitar los proveedores en Supabase

1. **SQL Editor** → ejecutar en orden los archivos de `supabase/migrations/`:
   - `0001_schema_inicial.sql` — tablas, índices, políticas RLS y el trigger que crea el usuario y sus categorías al registrarse.
   - `0002_agregar_factura_path.sql` — columna para asociar la imagen de la factura a la transacción.
   - `0003_reportes.sql` — funciones SQL del módulo de Reportes.
   - `0004_estado_cuenta_mensual.sql` — preferencia y funciones del estado de cuenta mensual.
2. **Authentication → Providers → Email** ✅ (para desarrollo conviene desactivar *Confirm email*; si no, el registro no devuelve sesión hasta abrir el correo).
3. **Authentication → Providers → Google** ✅ pegando el Client ID y Client Secret de Google Cloud Console.
4. **Authentication → URL Configuration → Redirect URLs**, agregar:
   - `http://localhost:3000/auth/callback`
   - `https://finansync.me/auth/callback`

### 5. Ejecutar en desarrollo

```bash
npm run dev
```

Abrir [http://localhost:3000](http://localhost:3000) en el navegador.

> Local y producción usan la misma base de Supabase: lo que se borre en local se borra de verdad.

### 6. Build de producción

```bash
npm run build
npm start
```

---

## Pantallas implementadas

| Pantalla | Ruta | Descripción |
|---|---|---|
| **Login** | `/` | Email/contraseña + Google Sign-In |
| **Registro** | `/registrarse` | Crea cuenta en Supabase Auth + perfil vía trigger |
| **Panel** | `/dashboard` | Saldo total, gráficas de los últimos 6 meses y por categoría, últimos movimientos |
| **Transacciones** | `/transacciones` | Lista por mes con buscador y filtro por tipo; ver la factura asociada; eliminar con confirmación |
| **Agregar Transacción** | `/transacciones/agregar` | Formulario con conversión de moneda, escaneo de factura con IA y creación de categorías al vuelo |
| **Presupuesto** | `/presupuesto` | Presupuestos por categoría y mes con progreso real; crear categorías desde el modal; cualquier emoji como ícono |
| **Metas** | `/metas` | Metas de ahorro con aportes, fecha límite y cualquier emoji como ícono |
| **Reportes** | `/reportes` | Comparativa mensual/anual y promedio por categoría; exportación a PDF |
| **Perspectivas IA** | `/perspectivas-ia` | Health Score, tendencias, comparativa por categorías y análisis con Gemini |
| **Chat IA** | `/chat-ia` | Asesor financiero conversacional con los datos reales del mes |
| **Configuración** | `/configuracion` | Perfil, moneda, categorías, notificaciones (incluido el estado de cuenta por correo), modo oscuro, eliminar cuenta |

Todas las acciones de borrado (transacciones, presupuestos, metas, categorías y la cuenta) piden confirmación en un modal propio (`context/ConfirmContext.tsx`).

---

## Estructura del proyecto

```
FinanSync-v2/
├── app/
│   ├── layout.tsx                    # Root layout (fuente Inter)
│   ├── providers.tsx                 # QueryClient de TanStack Query
│   ├── page.tsx                      # Login
│   ├── registrarse/page.tsx          # Registro
│   ├── auth/callback/route.ts        # Retorno del login con Google
│   ├── (app)/
│   │   ├── layout.tsx                # Sidebar + providers de la app
│   │   ├── dashboard/page.tsx
│   │   ├── transacciones/page.tsx
│   │   ├── transacciones/agregar/page.tsx
│   │   ├── presupuesto/page.tsx
│   │   ├── metas/page.tsx
│   │   ├── reportes/page.tsx
│   │   ├── perspectivas-ia/page.tsx
│   │   ├── chat-ia/page.tsx
│   │   └── configuracion/page.tsx
│   └── api/
│       ├── chat/route.ts             # Chat IA (Gemini)
│       ├── perspectivas/route.ts     # Perspectivas IA (Gemini)
│       ├── facturas/route.ts         # Lee una factura (Vision + Gemini)
│       ├── facturas/subir/route.ts   # Guarda la imagen en Cloud Storage
│       ├── facturas/ver/route.ts     # URL firmada para ver la imagen
│       ├── reportes/mensual/route.ts # Job del estado de cuenta (Cloud Scheduler)
│       └── tipo-cambio/route.ts      # Tasas de cambio
├── components/
│   ├── layout/Sidebar.tsx            # Navegación lateral / menú móvil
│   ├── SelectorMes.tsx               # Selector del mes que se está viendo
│   ├── SelectorEmoji.tsx             # Sugerencias de emoji + botón "+"
│   └── PanelEmoji.tsx                # Selector completo (emoji-mart), carga diferida
├── context/
│   ├── AuthContext.tsx               # Sesión de Supabase Auth
│   ├── FinanzasContext.tsx           # Moneda, categorías y conversión
│   ├── PeriodoContext.tsx            # Mes y año seleccionados
│   ├── NotifContext.tsx              # Preferencias de notificaciones
│   ├── ConfirmContext.tsx            # Modal de confirmación reutilizable
│   └── ThemeContext.tsx              # Modo oscuro persistido en el navegador
├── hooks/                            # Consultas con TanStack Query
│   ├── useTransacciones.ts
│   ├── usePresupuestos.ts
│   ├── useMetas.ts
│   └── useReportes.ts
├── lib/
│   ├── supabase/
│   │   ├── client.ts                 # Cliente para el navegador
│   │   ├── server.ts                 # Cliente para Server Components y API Routes
│   │   ├── admin.ts                  # Cliente service_role (solo el job mensual)
│   │   └── types.ts                  # Tipos de las tablas
│   ├── categorias.ts                 # CRUD — categorías (traduce nombre ↔ id)
│   ├── transacciones.ts              # CRUD — transacciones
│   ├── presupuestos.ts               # CRUD — presupuestos
│   ├── metas.ts                      # CRUD — metas
│   ├── reportes.ts                   # Funciones SQL de Reportes
│   ├── estado-cuenta.ts              # Datos y PDF del estado de cuenta mensual
│   ├── correo.ts                     # Envío de correo (Resend)
│   ├── vertex.ts                     # Llamadas a Gemini con respaldo y tiempo límite
│   ├── vision.ts                     # OCR con Cloud Vision
│   ├── gcs.ts                        # Cloud Storage (facturas)
│   └── google-auth.ts                # Credenciales de la cuenta de servicio
├── middleware.ts                     # Refresco de sesión + protección de rutas
├── monitoring/                       # Políticas de alerta y guía de observabilidad
├── supabase/migrations/              # Schema, RLS, triggers y funciones SQL
├── Dockerfile                        # Imagen de producción
└── cloudbuild.yaml                   # Build y despliegue a Cloud Run
```

---

## Estructura de datos en Postgres

El modelo sigue el diagrama entidad-relación del proyecto: cinco tablas, todas colgando del usuario con `ON DELETE CASCADE`, así que borrar una cuenta arrastra todos sus datos.

| Tabla | Columnas principales |
|---|---|
| `usuarios` | `id_usuario` (= `auth.users.id`), `nombre`, `email`, `moneda`, `notif_presupuesto`, `notif_metas`, `notif_ia`, `notif_reporte_mensual`, `categorias` (`text[]`, derivada), `creado_en` |
| `categorias` | `id_categoria`, `id_usuario`, `nombre`, `creado_en` — único por (`id_usuario`, `nombre`) |
| `transacciones` | `id_transaccion`, `id_usuario`, `id_categoria`, `descripcion`, `monto`, `tipo` (enum `income`/`expense`), `fecha` (`date`), `moneda_origen`, `factura_path`, `creado_en` |
| `presupuestos` | `id_presupuesto`, `id_usuario`, `id_categoria`, `limite_mensual`, `mes`, `anio`, `color`, `icono`, `moneda_origen`, `creado_en` — único por (`id_usuario`, `id_categoria`, `mes`, `anio`) |
| `metas` | `id_meta`, `id_usuario`, `nombre`, `objetivo`, `monto_actual`, `fecha_limite`, `color`, `icono`, `moneda_origen`, `creado_en` |

`usuarios` no reemplaza a `auth.users` — esa tabla la administra Supabase y no se toca. Es una tabla espejo en el esquema `public`, con el mismo id, donde sí se pueden guardar campos propios.

**`usuarios.categorias` es derivada, no una segunda copia.** El diagrama entidad-relación del proyecto tiene el campo `categorias` dentro de `Usuarios` *y además* la tabla `Categorias`. La misma información en dos lugares suele acabar en datos que no coinciden, así que aquí la copia no la escribe la aplicación: dos triggers la mantienen. `sincronizar_categorias_usuario()` rearma el arreglo cada vez que se inserta, actualiza o borra una fila de `categorias`, y `usuarios_categorias_solo_lectura()` lo recalcula antes de cualquier `update` sobre `usuarios`, de modo que el campo no se puede dejar mal ni a propósito. La fuente de la verdad sigue siendo la tabla `categorias`, porque es la que `transacciones` y `presupuestos` referencian por llave foránea.

**`metas` no tiene columna `completada`**, también como en el diagrama. Una meta está cumplida cuando `monto_actual >= objetivo`, y eso ya se sabe con los datos guardados; una columna aparte podría quedar en `false` con la meta ya alcanzada. `lib/metas.ts` la calcula al leer, así que la interfaz `Meta` que usan las páginas sigue teniendo `completada: boolean`.

**`icono` es texto libre.** Por eso presupuestos y metas aceptan cualquier emoji sin cambiar el schema; las sugerencias de la interfaz son solo atajos.

**Qué pasa al borrar una categoría.** Las dos llaves foráneas hacia `categorias` se comportan distinto a propósito: en `transacciones` es `ON DELETE SET NULL`, así que la transacción se conserva y solo queda sin clasificar (un `CASCADE` ahí borraría el historial financiero del usuario); en `presupuestos` es `ON DELETE CASCADE`, porque un presupuesto de una categoría que ya no existe no significa nada. La pantalla de Configuración pide confirmación antes de quitar una categoría.

Las páginas siguen trabajando con nombres de categoría, no con ids: la traducción entre ambos la hace `lib/categorias.ts` y es invisible para los componentes.

---

## Conversión de monedas

El endpoint `/api/tipo-cambio` consulta la API pública de jsDelivr y cachea las tasas en memoria por 24 horas. Cada transacción guarda su `monedaOrigen` y se convierte al renderizar — el histórico permanece fiel y las tasas frescas se aplican siempre.

**Monedas soportadas:** GTQ, MXN, USD, EUR, COP, ARS.

---

## Integración con Gemini

Las tres funciones de IA (`/api/chat`, `/api/perspectivas` y `/api/facturas`) llaman a Gemini por **Vertex AI**, no por la API key de Google AI Studio. La autenticación la hace la cuenta de servicio `finansync-runtime` con un token OAuth de corta duración, de modo que el acceso se controla con IAM y puede revocarse sin tocar el código. Todo pasa por `lib/vertex.ts`.

**Modelos y respaldo.** El modelo principal es `gemini-3.8-flash` y el de respaldo `gemini-3.5-flash` (lista `MODELOS_GEMINI`). Vertex reparte la capacidad de cada modelo entre todos sus clientes, así que un modelo recién salido puede responder `429` aunque el proyecto casi no lo use. `llamarVertexConRespaldo()` prueba los modelos en orden:

- un `5xx` se reintenta una vez con el mismo modelo;
- un `429`, un `404` o pasar el **tiempo límite de 45 s** saltan directo al siguiente modelo.

El respaldo es 3.5 Flash, y no 3.6 o 3.7, porque esas versiones se retiran pronto (noviembre 2026 y enero 2027), mientras que 3.5 tiene soporte hasta mayo de 2027 o después. Cada respuesta deja en el log el modelo, el tiempo y los tokens de entrada, razonamiento y salida.

**Endpoint `global`.** Los modelos 3.x no están publicados en regiones sueltas como `us-central1`, así que por defecto se usa el endpoint `global`, que además reparte la carga entre regiones cuando hay alta demanda. La contrapartida es que no garantiza en qué región se procesan los datos. La alternativa `us` mantiene el procesamiento en EE. UU. con un costo ~10 % mayor, y se activa con `VERTEX_LOCATION=us`.

**Chat IA** — `/api/chat` adjunta el contexto financiero real del mes (ingresos, gastos, balance, gasto por categoría, presupuestos y metas) con los números ya calculados. El system prompt restringe el chat a temas financieros: no responde nada fuera de tema ni siquiera de forma breve, en peticiones mixtas atiende solo la parte financiera y no cambia sus reglas aunque se le pida. Usa `thinkingLevel: 'low'`, porque con los datos ya calculados no necesita razonar a fondo; así responde en la mitad del tiempo.

**Perspectivas IA** — `/api/perspectivas` recibe ingresos, gastos, categorías, presupuestos y metas del mes y devuelve tres perspectivas en JSON (alerta, positivo o sugerencia, con ahorro estimado). Respeta la preferencia de análisis automático de Configuración.

**Lectura de facturas** — en Agregar Transacción, la imagen pasa primero por **Cloud Vision**, que extrae el texto, y solo ese texto se manda a Gemini para obtener monto, fecha, comercio y categoría sugerida; así se ahorran tokens. Si Vision saca poco texto, se manda la imagen completa. La imagen se guarda en un bucket privado de **Cloud Storage** y se muestra con URLs firmadas de 15 minutos.

**Health Score (0–100)** — cálculo local en Perspectivas IA: `40 + tasaAhorro × 100`, limitado entre 40 y 100.

---

## Reportes y estado de cuenta mensual

**Reportes** (`/reportes`) usa dos funciones SQL de `0003_reportes.sql` (`reporte_comparativa_mensual` y `reporte_promedio_categoria`) que agrupan y suman en Postgres en vez de traer todas las filas. Son `security invoker` con `auth.uid()`, así que las mismas políticas RLS las protegen. El PDF se dibuja en el navegador con jsPDF a partir de los datos ya calculados, no como captura de pantalla.

**Estado de cuenta por correo.** El día 1 de cada mes, **Cloud Scheduler** llama a `/api/reportes/mensual`, que arma el PDF del mes de cada usuario con la preferencia activada y lo manda por correo con Resend. El endpoint solo acepta el token OIDC firmado por la cuenta de servicio configurada en `REPORTE_SERVICE_ACCOUNT` (o el token manual `REPORTE_CRON_TOKEN`) y registra los rechazos. Como corre sin sesión, usa las funciones `SECURITY DEFINER` de `0004_estado_cuenta_mensual.sql`, que solo puede ejecutar la `service_role` key. Un fallo con un usuario no detiene a los demás.

---

## Despliegue y operación

- **Cloud Build** — el trigger `deploy-main` construye la imagen (`Dockerfile`) con cada push a `main`, la sube a Artifact Registry y despliega en **Cloud Run** (`us-central1`), según `cloudbuild.yaml`.
- **Dominio** — `finansync.me` mapeado al servicio de Cloud Run.
- **Observabilidad** — la app escribe logs en JSON con `severity` para que Cloud Logging los clasifique. La carpeta [`monitoring/`](https://github.com/RobertoJavier1/FinanSync-v2/tree/main/monitoring) tiene las políticas de alerta (errores 5xx y fallos del reporte mensual) y la guía para consultarlas.

---

## Seguridad — Row Level Security

Las políticas RLS las aplica Postgres dentro de la propia base de datos: no importa qué consulta mande el cliente, solo puede alcanzar sus propias filas.

Las cinco tablas tienen RLS activo y una política por operación con la forma:

```sql
create policy "transacciones: leer las propias"
  on public.transacciones for select using (auth.uid() = id_usuario);
```

Son 20 políticas en total (select / insert / update / delete por tabla).

Cuatro funciones completan el esquema base:

| Función | Qué hace |
|---|---|
| `handle_new_user()` | Trigger `after insert on auth.users`. Crea la fila de `usuarios` con el `nombre` que viene en los metadatos (o el `full_name` que manda Google) y las categorías por defecto como filas de `categorias`. |
| `sincronizar_categorias_usuario()` | Trigger `after insert or update or delete on categorias`. Vuelve a armar `usuarios.categorias` para el usuario afectado. |
| `usuarios_categorias_solo_lectura()` | Trigger `before update on usuarios`. Recalcula `usuarios.categorias` desde la tabla `categorias`, para que el campo no se pueda escribir a mano. |
| `eliminar_mi_cuenta()` | `security definer`, así que puede borrar de `auth.users` sin exponer la `service_role` key en el navegador. Solo borra al usuario de la sesión actual (`auth.uid()`), y el `on delete cascade` arrastra categorías, transacciones, metas y presupuestos. Se llama desde Configuración con `supabase.rpc('eliminar_mi_cuenta')`. |

Además de RLS, las rutas de IA vuelven a validar la sesión en el servidor (no dependen solo del middleware), y en Cloud Run ninguna llave de cuenta de servicio queda guardada.

---

## Documentación adicional

- [`docs/monitoring.md`](docs/monitoring.md) — eventos registrados, alertas y consultas de Cloud Logging.
- [Documentación cloud (PDF)](documentacion.pdf) — arquitectura, servicios, despliegue, seguridad y costos.
