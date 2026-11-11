# Estudio 3D

Grupo 1 · Cloud Computing 2026 · Universidad Galileo

- **Integrantes:** Geovanni Alejandro Ordoñez Alvarez, Alan Steven Martínez López
- **Proveedor cloud:** AWS
- **Servicios principales:** Amazon EC2, Amazon S3, Amazon CloudFront, AWS Lambda (además de Supabase para autenticación y base de datos)
- **Repositorio:** https://github.com/AlanMartinezzzz/estudio3d-backend
- **Despliegue:** No se entregó
- **Documentación PDF:** No se entregó
- **Video:** No se entregó video

---

> **Nota:** el grupo no entregó documentación ni README. Esta descripción se redactó a partir del código del repositorio (`app.py`, `templates/`, `static/`, `Dockerfile`, `.github/workflows/deploy.yml` y `puente_camara/`) y describe solo lo que ese código hace.

## Qué es

Estudio 3D es una aplicación web para un negocio de impresión 3D. El cliente sube un modelo `.stl`, el sistema mide el modelo en el navegador (volumen, peso estimado y tiempo de impresión) y calcula el precio con las tarifas de materiales, colores e infill que define el negocio. Después el cliente puede guardar el archivo, agregar piezas aprobadas y productos del catálogo a un carrito y confirmar su pedido.

Tiene dos interfaces sobre el mismo flujo:

- **Cliente:** cotizar archivo, mis archivos (con vista previa 3D del `.stl`), catálogo, carrito, mis pedidos, contacto con el negocio y su cuenta.
- **Negocio:** resumen con indicadores, productos del catálogo, precios y materiales, inventario de filamentos en gramos (se descuenta al aprobar una pieza), revisión y aprobación de archivos de clientes, seguimiento de carritos/pedidos y cámaras.

Una función adicional es la **cámara en vivo**: una webcam en el taller permite que el cliente vea la impresión de su propia pieza.

## Componentes

| Componente | Qué hace |
| --- | --- |
| Backend Flask (`app.py`) | Servido con Gunicorn dentro de un contenedor Docker (Python 3.12). Entrega la página, valida la sesión del usuario con Supabase y expone la API: subida de `.stl` a S3, enlaces temporales de descarga, contenido del `.stl` para la vista 3D y la recepción/entrega de fotos de las cámaras. |
| Frontend (`templates/index.html`, `static/`) | HTML, CSS y JavaScript sin framework. Usa el cliente de Supabase en el navegador para autenticación (correo y contraseña, inicio con Google, recuperación de contraseña) y para leer y escribir las tablas del negocio. La vista previa 3D usa three.js (`STLLoader`, `OrbitControls`). |
| Supabase | Autenticación de usuarios y base de datos PostgreSQL con reglas RLS (tablas de archivos, productos, materiales, colores, filamentos, carritos, cámaras, usuarios y configuración). También guarda las fotos de perfil y de productos en Supabase Storage. El backend consulta Supabase *como el usuario*, de modo que las reglas RLS deciden qué archivo o cámara puede ver cada quien. |
| Puente de cámara (`puente_camara/`) | Script de Python con OpenCV que corre en la PC del taller: toma una foto JPEG cada ~0.5 s y la envía por HTTPS al servidor, sin abrir puertos en el router. El servidor guarda solo la última foto de cada cámara en memoria y la página la va pidiendo. |
| GitHub Actions (`deploy.yml`) | En cada push a `main` se conecta por SSH a la instancia EC2, actualiza el código, reconstruye la imagen Docker y reinicia el contenedor. |

## Servicios de AWS que usa el código

| Servicio | Uso en el código |
| --- | --- |
| **Amazon EC2** | Hospeda el contenedor Docker de la aplicación (puerto 8080). El workflow de GitHub Actions despliega en la instancia por SSH. |
| **Amazon S3** | Guarda los archivos `.stl` de los clientes con cifrado del lado del servidor (AES256). La ruta del objeto usa la huella SHA-256 del contenido, para no guardar copias repetidas. Las descargas se entregan con URL prefirmadas que caducan en 5 minutos. Las credenciales se toman del entorno o del rol de IAM. |
| **Amazon CloudFront** | Distribución delante del servidor: el puente de cámara apunta a un dominio `cloudfront.net` y el backend contempla que CloudFront deje pasar el encabezado `Authorization`. |
| **AWS Lambda** | El formulario «Contactar al negocio» envía el mensaje a una Lambda Function URL (región `us-east-2`). El código de esa función no está en el repositorio. |

## Flujo general

1. El usuario inicia sesión con Supabase (correo o Google); según su rol entra a la vista de cliente o de negocio.
2. El cliente sube un `.stl`: el navegador calcula el precio y el backend guarda el archivo en S3 y devuelve su ruta y huella.
3. El negocio revisa y aprueba la pieza; el cliente la agrega al carrito y confirma el pedido.
4. Para descargar o previsualizar un archivo, el backend verifica con RLS que el usuario pueda verlo y entrega un enlace temporal de S3 (o el contenido del archivo si el navegador no puede leerlo directo).
5. Si la pieza tiene una cámara asignada, el cliente ve en *Mis archivos* el botón **En vivo** con las fotos que envía el puente del taller.

## Tecnologías

Python (Flask, Gunicorn, boto3, supabase-py), PostgreSQL en Supabase, HTML/CSS/JavaScript, three.js, OpenCV, Docker y GitHub Actions.
