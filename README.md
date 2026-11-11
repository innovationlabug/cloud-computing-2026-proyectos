# Cloud Computing 2026

**[Ver la vitrina de proyectos](https://cloud-computing-2026.web.app)**

Este semestre, en el curso de Cloud Computing de la Universidad Galileo, cada equipo tomó una aplicación web propia y la puso a funcionar en Google Cloud, AWS o Azure. Aquí están los once proyectos finales, con su arquitectura, el video de la demo y la documentación que escribió cada equipo. Gracias a todos por compartir su trabajo.

## Qué se pedía

Le pedí a cada equipo desplegar un proyecto web propio en Google Cloud, AWS o Azure, integrando al menos siete servicios del proveedor entre cómputo, almacenamiento, base de datos, redes, seguridad, monitoreo, CI/CD e inteligencia artificial. Firebase no contaba como servicio. El almacenamiento y la base de datos tenían que configurarse directamente en el proveedor, desde su consola o su CLI.

## Proyectos

### [SIMA — Inventario distribuido para varias sucursales](proyectos/sima/README.md)
**Google Cloud.** David Alejandro Ortega Reyes y Dilan Jareth Arreaga Tiu.
Un sistema de inventario para una cadena con tres sucursales. Cada sucursal corre en su propia VM y sigue vendiendo aunque se caiga el internet. Una central en Cloud Run coordina los traslados con two-phase commit sobre Cloud SQL, Pub/Sub lleva la sincronización y Datastream copia los datos a BigQuery para la analítica. Toda la infraestructura está en Terraform.
[Documentación](proyectos/sima/README.md) · [Video](proyectos/sima/video.mp4)

### [FinanSync](proyectos/finansync/README.md)
**Google Cloud.** Roberto Javier Molina López y Luis Gabriel Córdova Archila.
Una aplicación de finanzas personales hecha en Next.js, con dominio propio. Tiene un chat financiero con Gemini, lee el texto de las facturas con Cloud Vision y envía un reporte mensual que programa Cloud Scheduler.
[Ver la app desplegada](https://finansync.me) · [Repositorio](https://github.com/RobertoJavier1/FinanSync-v2) · [Documentación](proyectos/finansync/README.md) · [Video](proyectos/finansync/video.mp4) · [Video en YouTube](https://www.youtube.com/watch?v=NpCvoWSodEU)

### [AquaControl 360](proyectos/aquacontrol-360/README.md)
**Google Cloud.** Edgar Andrés Culajay Hernández y Allan Josué Gómez Pérez.
Una plataforma para granjas de camarón y tilapia. Un ESP32 mide el oxígeno de cada piscina y manda las lecturas a una API en Cloud Run, que pasa por Pub/Sub, las guarda en Cloud SQL y genera alertas. Desde la web y las apps de Android e iOS se encienden los aireadores, y Gemini cuenta ganado en fotos.
[Ver la app desplegada](https://aquacontrol-web-1000315935309.us-central1.run.app) · [Documentación](proyectos/aquacontrol-360/README.md) · [Video](proyectos/aquacontrol-360/video.mp4)

### [Guíame](proyectos/guiame/README.md)
**Google Cloud.** Molly Toledo Vega y María Teresa Rivera López.
Una app de Android y un portal web que acompañan el autoexamen de mama. La API corre en Cloud Run detrás de un balanceador, y al terminar cada autoexamen Gemini redacta un reporte en español, sin dar diagnósticos.
[Repositorio](https://github.com/mtvhua/Guiame) · [Documentación](proyectos/guiame/README.md) · [Video](proyectos/guiame/video.mp4) · [Video en YouTube](https://youtu.be/me_xdoWI5Rs)

### [LEAF](proyectos/leaf/README.md)
**AWS.** Pablo Daniel Vásquez Aldana.
Una aplicación para registrar las lecturas de los contadores de agua de un condominio y calcular el consumo de cada casa. Corre en EC2 detrás de un balanceador y CloudFront, con RDS y S3, roles de IAM con permisos mínimos y alarmas que avisan por correo.
[Ver la app desplegada](https://d1eik3xsy2bfk4.cloudfront.net) · [Repositorio](https://github.com/Zylohw/Proyecto-Final-Cloud-Computing) · [Documentación](proyectos/leaf/README.md)

### [Semilla — Asistente de gestión para pequeños negocios](proyectos/semilla/README.md)
**Azure.** Lizardo Enmanuel González Méndez y Jessell Andrea Contreras Escobar.
Un asistente para que pequeños emprendedores ordenen las cuentas de su negocio. Corre en Container Apps con PostgreSQL, guarda los comprobantes en Blob Storage y lee sus secretos de Key Vault con identidades administradas. Hace deploy con GitHub Actions, sin guardar secretos en el repositorio.
[Repositorio](https://github.com/lizardoenm12/semilla-cloud) · [Documentación](proyectos/semilla/README.md) · [Video](proyectos/semilla/video.mp4)

### [SyncNopsis](proyectos/syncnopsis/README.md)
**AWS.** Fernando Josué Alegría Velásquez y Anyelo Arthur Sánchez Naranjo.
Un _segundo cerebro_ donde guardas capturas y notas, y la IA las organiza y las conecta. Corre en EC2 detrás de CloudFront, busca por significado con pgvector en RDS, guarda las imágenes en S3 y hace deploy automático con CodeBuild.
[Ver la app desplegada](https://d11s6bjcprs8g7.cloudfront.net) · [Documentación](proyectos/syncnopsis/README.md) · [Video](proyectos/syncnopsis/video.mp4)

### [Nexoria](proyectos/nexoria/README.md)
**Google Cloud.** Jean Carlo Pacheco Pérez y Rodrigo Armando Nathanael Gálvez Mazariegos.
Una plataforma para administrar condominios, con visitas por código QR, reservas de amenidades, estados de cuenta y paquetería. Gemini analiza las incidencias y AWS Rekognition lee las placas en la garita, sin uso de keys, gracias a la federación de identidad entre Google Cloud y AWS.
[Documentación](proyectos/nexoria/README.md)

### [Glacier Quest](proyectos/glacier-quest/README.md)
**Google Cloud.** Pablo David Urbina Alvarez y Angel David Camargo Hernández.
Un juego multijugador con misiones, combates, duelos y chat. Corre en Cloud Run, hace deploy con Cloud Build y Artifact Registry, y Cloud Scheduler limpia cada cierto tiempo la presencia de los jugadores y los mensajes temporales.
[Ver la app desplegada](https://glacierquest-demo-636305098699.us-central1.run.app/) · [Repositorio](https://github.com/angelprogt3985/Cloud-Computing-proyecto) · [Documentación](proyectos/glacier-quest/README.md) · [Video](proyectos/glacier-quest/video.mp4)

### [SmartFlow AI](proyectos/smartflow/README.md)
**Google Cloud.** Andrea Sofía Dávila Equité y Wilder Javier Cardoza de la Cruz.
Describes un proceso de tu negocio con tus palabras y Gemini dibuja el diagrama de flujo, encuentra los cuellos de botella y arma un reporte. El frontend y el backend corren en Cloud Run, con Cloud SQL y Cloud Storage.
[Ver la app desplegada](https://smartflow-frontend-1020540188448.us-central1.run.app) · [Repositorio](https://github.com/Sistemas-UG/Proyecto-IS) · [Documentación](proyectos/smartflow/README.md) · [Video](proyectos/smartflow/video.mp4)

### [Estudio 3D](proyectos/estudio-3d/README.md)
**AWS.** Geovanni Alejandro Ordoñez Alvarez y Alan Steven Martínez López.
Una aplicación para un taller de impresión 3D. Cotiza la pieza a partir del archivo .stl, gestiona los pedidos y tiene una cámara en vivo para ver cómo se imprime tu pieza.
[Repositorio](https://github.com/AlanMartinezzzz/estudio3d-backend) · [Documentación](proyectos/estudio-3d/README.md)

## Sobre este repositorio

Cada carpeta de `proyectos/` tiene un `README.md` con la documentación del equipo, un diagrama de su arquitectura y, cuando lo entregaron, su PDF y su video en 720p. Quité los números de carné, las credenciales y los datos personales que venían en el material original. El contenido técnico de cada proyecto es de su equipo.
