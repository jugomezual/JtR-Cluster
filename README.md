
# JtR-Cluster
Distributed platform for password cracking based on John the Ripper, featuring task distribution and multi-node, multi-GPU management. Developed as part of the Final Degree Project in Computer Engineering (University of Almería).

## Overview

This project enables the distribution of cracking tasks among multiple processing nodes, leveraging both CPUs and GPUs. It integrates server, client, and database modules, along with a web interface for job management and monitoring.

## Repository Structure

- **BD.tgz**: SQL script for creating and deploying the database (`miweb_db.sql`).
- **Cliente.tgz**: Source code, binaries, and scripts for the processing nodes.
- **Server.tgz**: Source code and executable for the main coordination server.
- **Web.tgz**: Web interface files, PHP scripts, and management logs.

## Usage Instructions

1.  **Clone the repository:**

    ```bash
    git clone https://github.com/jugomezual/JtR-Cluster 
    ```

2.  **Decompress the modules according to their function:**

    ```bash
    tar -xvzf BD.tgz
    tar -xvzf Cliente.tgz
    tar -xvzf Server.tgz
    tar -xvzf Web.tgz
    ```

3.  **Database:**

    ```bash
    mysql -u username -p < miweb_db.sql
    ```
    *Note: Replace `usuario` with your actual MySQL username.*

4.  **Client and Server:**
    Compile the source code if necessary:

    ```bash
    gcc procesamiento.c -o procesamiento
    gcc gpu_report.c -o gpu_report
    gcc cpu_report.c -o cpu_report
    gcc server.c -o server
    ```

5.  **Web:**
    Copy the contents of `www/` to your Apache/Nginx web server with PHP support.
    Configure the database connection in `conexion.php` according to your local parameters.

### Credits
Authors: Juan Rafael Madolell, Dr. Julio Gómez López and Dr. Raúl Baños Navarro

Final Degree Project Advisor: Dr. D. Julio Gómez López
University of Almería, 2025
