# Aplicación Glacier Quest

[Presentación](../README.md) · [Infraestructura](infraestructura.md)

## Ejecución local

Requiere Node.js 22 o superior. Desde la raíz del repositorio:

```powershell
cd glacierquest
npm ci --ignore-scripts --prefix api
npm start --prefix api
```

Abrir `http://localhost:8080`. En Windows también se puede ejecutar `start-local.cmd`, que usa `local-env.cmd` para aislar la configuración local. No abrir `client/index.html` directamente.

Sin configuración cloud, el servidor usa memoria: el progreso se pierde al detenerlo y el código de clase local queda vacío por defecto. Las cuentas persistentes con correo necesitan Firebase y su configuración; instalar dependencias no las habilita. Para otro puerto, establecer `$env:PORT='8081'` antes de iniciar.

## Funcionalidades

- Cinco zonas conectadas, exploración, misiones y recursos renovables.
- Cuentas con correo y recuperación de progreso en la demo cloud.
- Experiencia, niveles, inventario, equipo y técnicas desbloqueadas por nivel.
- Enemigos móviles, combate por turnos y duelos consentidos entre jugadores.
- Chat y presencia por zona, menús de juego, minimapa y audio generado en el navegador.

El chat y los movimientos remotos se actualizan con consultas periódicas a la API, sin WebSockets ni Pub/Sub. La API valida acciones y persiste cambios. La música usa Web Audio, sin un servicio de IA.

## Código principal

| Ruta | Responsabilidad |
| --- | --- |
| `api/src/server.js` | Inicia Express. |
| `api/src/app.js` | Rutas HTTP, sesiones, límites, Storage y registros. |
| `api/src/config.js` | Valida configuración. |
| `api/src/realtime.js` | Firebase REST, concurrencia con ETags y limpieza. |
| `api/src/accounts.js` | Registro, acceso y recuperación con Firebase Authentication. |
| `api/src/maintenance.js` | Verifica OIDC de Scheduler. |
| `api/src/game.js`, `arena.js`, `encounters.js` | Progreso, combate y encuentros. |
| `api/src/zones.js`, `expedition.js` | Mundo, recursos y actividades. |
| `api/src/techniques.js`, `wardrobe.js` | Técnicas y equipo. |
| `client/src/game-client.js` | Interfaz e interacción. |
| `client/src/api.js` | Solicitudes y sincronización. |
| `client/src/world.js`, `zone-scenery.js` | Representación del mundo. |
| `client/src/soundscape.js` | Audio generado en el navegador. |

`GET /health` muestra el estado del servicio. Las rutas de juego protegidas requieren sesión. `/internal/cleanup` exige identidad OIDC de mantenimiento.

## Contenedor y alternativas

Cloud Build usa `glacierquest/Dockerfile` con esta carpeta como contexto. Incluye servidor, cliente y recursos.

Se conservan los adaptadores de memoria, Firestore y PostgreSQL, esquemas SQL e infraestructura histórica porque son código y configuración. Su presencia no implica que se utilicen en la demo: la selección activa es `STORE=realtime`.

No guardar credenciales, contraseñas, código de clase, estados de Terraform ni evidencias privadas en Git.
