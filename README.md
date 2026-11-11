# Cloud Computing 2026 — Proyectos finales

Vitrina de los proyectos finales del curso **Cloud Computing 2026** (Universidad Galileo, FISICC, sexto ciclo, sección A). Cada equipo tomó un proyecto web propio y lo desplegó en la nube. Aquí está la documentación de cada equipo y el video de su demostración.

Los proyectos aparecen de mejor a peor según la evaluación de la entrega final.

## Qué se pedía

Cada equipo debía desplegar un proyecto web en la nube con al menos **7 servicios cloud** de un proveedor grande: Google Cloud, AWS o Azure. El proyecto podía venir de otro curso de la carrera o ser un proyecto de código abierto.

**Servicios que cuentan.** Se aceptaban servicios de cómputo, almacenamiento, bases de datos, redes, seguridad (IAM con roles o cuentas de servicio propias, documentadas), monitoreo, CI/CD e IA. Una función de IA integrada cuenta como un servicio. Los productos de Firebase no cuentan: el almacenamiento y la base de datos debían ser los del proveedor, configurados desde su consola o CLI.

**Entregables:**

- El link al proyecto desplegado.
- La documentación en PDF:
  - el diagrama de arquitectura;
  - los servicios usados y cómo se integran;
  - los pasos de despliegue con su configuración;
  - una estimación de costos mensuales por servicio, hecha con la calculadora del proveedor;
  - las consideraciones de seguridad.
- Un video de 5 a 7 minutos con la funcionalidad, la arquitectura y el uso de cada servicio.
- Opcional: la infraestructura como código (Terraform o equivalente), que daba puntos extra.

**Hitos.**

| Hito | Fecha | Qué se entregaba |
|---|---|---|
| M1, propuesta | 11 de septiembre | El proyecto, el proveedor y los servicios planeados |
| M2, arquitectura | 9 de octubre | El diagrama y al menos 2 servicios ya desplegados |
| Final | 2 de noviembre | La entrega completa y la presentación en clase |

**Rúbrica.**

| Criterio | Peso |
|---|---|
| Integración de servicios cloud | 50 % |
| Diseño de la arquitectura | 20 % |
| Documentación | 10 % |
| Presentación y video | 10 % |
| Hitos | 5 % |
| Usabilidad y desempeño | 5 % |

## Proyectos

### 1. [SIMA — Sistema de inventario distribuido multisucursal](proyectos/sima/README.md)
**Grupo 3 · Google Cloud.** David Alejandro Ortega Reyes y Dilan Jareth Arreaga Tiu.
SIMA es un inventario para una cadena con tres sucursales: cada una corre en su propia VM de Compute Engine y factura aunque se caiga el internet. Una central en Cloud Run coordina los traslados entre sucursales con *two-phase commit* (2PC) sobre Cloud SQL, y Cloud Scheduler recupera las transacciones que quedan a medias. La sincronización viaja por Pub/Sub, y la analítica llega a BigQuery por Datastream, sin cargar la operación. Toda la infraestructura está en Terraform.
[Documentación](proyectos/sima/README.md) · [Video](proyectos/sima/video.mp4)

### 2. [FinanSync](proyectos/finansync/README.md)
**Grupo 2 · Google Cloud.** Roberto Javier Molina López y Luis Gabriel Córdova Archila.
FinanSync es una aplicación de finanzas personales hecha en Next.js y desplegada en Cloud Run con dominio propio. Tiene un chat financiero con Gemini en Vertex AI y extrae el texto de las facturas con Cloud Vision. Los reportes mensuales salen programados con Cloud Scheduler.
[Repositorio](https://github.com/RobertoJavier1/FinanSync-v2) · [Documentación](proyectos/finansync/README.md) · [Video](proyectos/finansync/video.mp4) · [Video en YouTube](https://www.youtube.com/watch?v=NpCvoWSodEU)

### 3. [AquaControl 360](proyectos/aquacontrol-360/README.md)
**Grupo 9, ShirmpProgrammers · Google Cloud.** Edgar Andrés Culajay Hernández y Allan Josué Gómez Pérez.
AquaControl 360 gestiona granjas de camarón y tilapia. Un ESP32 mide el oxígeno de cada piscina y envía las lecturas a una API en Cloud Run. Las lecturas pasan por Pub/Sub, se guardan en Cloud SQL y generan alertas. La web, la app Android y la app iOS controlan los aireadores. Además, Gemini en Vertex AI responde preguntas sobre la granja y cuenta ganado en fotos.
[Documentación](proyectos/aquacontrol-360/README.md) · [Video](proyectos/aquacontrol-360/video.mp4)

### 4. [Guíame](proyectos/guiame/README.md)
**Grupo 4 · Google Cloud.** Molly Toledo Vega y María Teresa Rivera López.
Guíame es un ecosistema de salud digital que acompaña el autoexamen de mama. La API corre en Cloud Run detrás de Cloud Load Balancing, con Cloud SQL, Cloud Storage y Secret Manager. Al terminar cada autoexamen, Gemini en Vertex AI redacta un reporte en español, sin dar diagnósticos.
[Repositorio](https://github.com/mtvhua/Guiame) · [Documentación](proyectos/guiame/README.md) · [Video](proyectos/guiame/video.mp4) · [Video en YouTube](https://youtu.be/me_xdoWI5Rs)

### 5. [LEAF](proyectos/leaf/README.md)
**Grupo 10 · AWS.** Pablo Daniel Vásquez Aldana.
LEAF registra y consulta las lecturas de los contadores de agua de un condominio y calcula el consumo de cada vivienda. Corre en EC2 detrás de un Application Load Balancer y CloudFront, con RDS PostgreSQL y S3. Tiene roles de IAM con mínimo privilegio y alarmas de CloudWatch que avisan por SNS.
[Repositorio](https://github.com/Zylohw/Proyecto-Final-Cloud-Computing) · [Documentación](proyectos/leaf/README.md) · [Video en YouTube](https://www.youtube.com/watch?v=jv2t3hB3gt4) (privado)

### 6. [Semilla — asistente de gestión para pequeños negocios](proyectos/semilla/README.md)
**Grupo 7 · Azure.** Lizardo Enmanuel González Méndez y Jessell Andrea Contreras Escobar.
Semilla ayuda a pequeños emprendedores a ordenar las cuentas de su negocio. Corre en Azure Container Apps, con PostgreSQL Flexible Server, Blob Storage para los comprobantes y Key Vault, al que la app entra con identidades administradas. El despliegue es continuo con GitHub Actions y OIDC.
[Repositorio](https://github.com/lizardoenm12/semilla-cloud) · [Documentación](proyectos/semilla/README.md) · [Video](proyectos/semilla/video.mp4)

### 7. [SyncNopsis](proyectos/syncnopsis/README.md)
**Grupo 8 · AWS.** Fernando Josué Alegría Velásquez y Anyelo Arthur Sánchez Naranjo.
SyncNopsis es un "segundo cerebro" que captura notas e ideas, y la IA las organiza y conecta entre sí. Corre en EC2 detrás de CloudFront, con RDS PostgreSQL y pgvector para la búsqueda semántica, y S3 para las imágenes. Cada push se despliega solo con GitHub Actions y CodeBuild.
[Documentación](proyectos/syncnopsis/README.md) · [Video](proyectos/syncnopsis/video.mp4)

### 8. [Nexoria](proyectos/nexoria/README.md)
**Grupo 6 · Google Cloud.** Jean Carlo Pacheco Pérez y Rodrigo Armando Nathanael Gálvez Mazariegos.
Nexoria administra condominios: visitas con código QR, reservas de amenidades, estados de cuenta y paquetería. El backend corre en Cloud Run con CI/CD de Cloud Build. Gemini en Vertex AI analiza las incidencias, y AWS Rekognition lee las placas en la garita mediante federación de identidad, sin llaves fijas.
[Documentación](proyectos/nexoria/README.md)

### 9. [Glacier Quest](proyectos/glacier-quest/README.md)
**Grupo 11 · Google Cloud.** Pablo David Urbina Alvarez y Angel David Camargo Hernández.
Glacier Quest es un juego multijugador académico con misiones, combate, duelos y chat. Corre en Cloud Run, con Secret Manager y Cloud Storage. Despliega con Cloud Build y Artifact Registry, y Cloud Scheduler limpia la presencia y los mensajes temporales.
[Repositorio](https://github.com/angelprogt3985/Cloud-Computing-proyecto) · [Documentación](proyectos/glacier-quest/README.md) · [Video](proyectos/glacier-quest/video.mp4)

### 10. [SmartFlow AI](proyectos/smartflow/README.md)
**Grupo 5 · Google Cloud.** Andrea Sofía Dávila Equité y Wilder Javier Cardoza de la Cruz.
SmartFlow AI genera diagramas de procesos a partir de una descripción en lenguaje natural con Gemini en Vertex AI, y detecta los cuellos de botella del proceso. Corre en Cloud Run con Cloud SQL y Cloud Storage.
[Repositorio](https://github.com/Sistemas-UG/Proyecto-IS) · [Documentación](proyectos/smartflow/README.md) · [Video](proyectos/smartflow/video.mp4)

### 11. [Estudio 3D](proyectos/estudio-3d/README.md)
**Grupo 1 · AWS.** Geovanni Alejandro Ordoñez Alvarez y Alan Steven Martínez López.
Estudio 3D cotiza impresiones 3D a partir de un archivo `.stl` y gestiona los pedidos del taller, con una cámara en vivo de la impresión. Usa EC2, S3 para los modelos, CloudFront y Lambda.
[Repositorio](https://github.com/AlanMartinezzzz/estudio3d-backend) · [Documentación](proyectos/estudio-3d/README.md)

## Sobre este repositorio

- Cada carpeta de `proyectos/` tiene un `README.md` con la documentación del equipo, su PDF si lo entregó y su video recomprimido a 720p.
- Los repositorios aparecen enlazados solo cuando son públicos.
- Se quitaron los números de carné, las credenciales y los datos personales que venían en el material original.
- El contenido técnico es de cada equipo y se conserva como lo entregaron.
