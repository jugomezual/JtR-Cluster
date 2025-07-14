
# JtR-Cluster
Plataforma distribuida para el cracking de contraseñas basada en John the Ripper, con reparto de tareas y gestión multi-nodo y multi-GPU. Desarrollada como parte del Trabajo de Fin de Grado en Ingeniería Informática (Universidad de Almería).
## Descripción general

Este proyecto permite el reparto de tareas de cracking entre múltiples nodos de procesamiento, aprovechando tanto CPUs como GPUs, e integrando módulos de servidor, cliente, base de datos y una interfaz web para la gestión y monitorización de trabajos.

## Estructura del repositorio

- **BD.tgz**: Script SQL para la creación y despliegue de la base de datos (`miweb_db.sql`).
- **Cliente.tgz**: Código fuente, binarios y scripts de los nodos de procesamiento.
- **Server.tgz**: Código fuente y ejecutable del servidor principal de coordinación.
- **Web.tgz**: Archivos de la interfaz web, scripts PHP y logs de gestión.

## Instrucciones de uso

1. **Clonar el repositorio:**

   ```bash
   git clone https://github.com/usuario/JtR-Cluster.git

2. **Descomprimir los módulos según su función:**
  
   ```bash
   tar -xvzf BD.tgz
   tar -xvzf Cliente.tgz
   tar -xvzf Server.tgz
   tar -xvzf Web.tgz

  
3. **Base de datos:**
   ```bash
   mysql -u usuario -p < miweb_db.sql

4. **Cliente y servidor:**
   Compilar el código fuente si es necesario:
   ```bash
   gcc procesamiento.c -o procesamiento
   gcc gpu_report.c -o gpu_report
   gcc cpu_report.c -o cpu_report
   gcc server.c -o server

5.**Web:**
  Copiar el contenido de www/ a tu servidor web Apache/Nginx con soporte PHP.
  Configura la conexión a la base de datos en conexion.php según tus parámetros locales.

  Créditos
  Autor: Juan Rafael Madolell
  TFG dirigido por: Dr. D. Julio Gómez López
  Universidad de Almería, 2025
