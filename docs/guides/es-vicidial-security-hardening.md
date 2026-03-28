# Hardening de Seguridad de VICIdial: CVEs, Firewalls y Control de Acceso

*Desplegaste VICIdial. Esta marcando. Los agentes estan contentos. Entonces tu proveedor te envia una carta enojada sobre fraude de peaje acumulando $14,000 en llamadas internacionales durante un fin de semana. O peor — te despiertas el lunes a un panel admin desfigurado y un dump de MySQL flotando en un canal de Telegram.*

*Esta es [la guia](/blog/es-open-source-call-center-software/) de seguridad que deberia haber existido desde el primer dia.*

Hemos auditado, endurecido, y respondido a incidentes en mas de 100 despliegues de VICIdial. Hemos leido cada filing de CVE, rastreado cada exploit divulgado en los foros y en avisos de seguridad, y visto de primera mano lo que pasa cuando un centro de contacto trata la seguridad como algo secundario.

---

## La Historia de CVEs Que Necesitas Conocer

VICIdial tiene una historia documentada de vulnerabilidades criticas. Algunas han sido parcheadas upstream. Algunas requieren intervencion manual. Todas estan siendo activamente escaneadas por kits de exploit automatizados.

### CVE-2024-8503 — SQL Injection en Autenticacion (Critica)

Esta llego a los titulares. Descubierta por el equipo de Offensive Security de Kali Linux en 2024, esta vulnerabilidad de inyeccion SQL existe en el endpoint `/vicidial/user_stats.php`. Un atacante no autenticado puede inyectar SQL arbitrario a traves del parametro `user`, extraer toda la tabla `vicidial_users` — incluyendo contrasenas en texto plano o debilmente hasheadas — y obtener acceso administrativo completo.

**Como parchear:** Actualiza a VICIdial SVN revision 3848 o posterior.

### CVE-2024-8504 — Ejecucion Remota de Codigo Autenticada (Critica)

Emparejada con CVE-2024-8503, esta vulnerabilidad permite a un usuario admin autenticado inyectar comandos OS arbitrarios a traves de la interfaz admin. Combinadas, estas dos CVEs forman una cadena de ataque completa.

**Como parchear:** Mismo fix — revision SVN 3848+. Pero esta CVE es un recordatorio de que el panel admin de VICIdial **nunca** deberia estar expuesto al internet publico.

### El Patron Es Claro

Cada CVE mayor de VICIdial sigue la misma plantilla: **acceso no autenticado a un script PHP web-facing que acepta entrada no sanitizada.**

Si no haces nada mas de esta guia, haz estas dos cosas:

1. **Manten tu codigo VICIdial actualizado.** Ejecuta `svn update`.
2. **Nunca expongas la interfaz web de VICIdial directamente al internet sin controles de acceso.**

```bash
# Verifica tu revision SVN actual
cd /usr/share/astguiclient
svn info | grep "Revision"

# Actualiza a lo mas reciente
svn update
```

---

## Configuracion de Firewall: La Base de Todo

Un firewall correctamente configurado es la medida de seguridad de mayor impacto que puedes implementar.

### El Principio de Denegacion por Defecto

Empieza desde cero. Bloquea todo. Abre solo lo necesario.

Aqui esta la configuracion [completa de](/blog/es-stir-shaken-vicidial-guide/) firewalld para una instalacion VICIdial [de servidor](/blog/es-vicidial-setup-guide/) unico:

```bash
# Resetear a denegacion por defecto
firewall-cmd --set-default-zone=drop

# Crear una zona dedicada para VICIdial
firewall-cmd --permanent --new-zone=vicidial
firewall-cmd --permanent --zone=vicidial --set-target=DROP

# SSH — restringir a tus IPs de gestion solamente
firewall-cmd --permanent --zone=vicidial --add-rich-rule='rule family="ipv4" source address="TU.IP.OFICINA/32" port port="22" protocol="tcp" accept'

# HTTP/HTTPS — acceso de agentes
firewall-cmd --permanent --zone=vicidial --add-rich-rule='rule family="ipv4" source address="TU.SUBRED.AGENTES/24" port port="443" protocol="tcp" accept'

# SIP solo de carriers
firewall-cmd --permanent --zone=vicidial --add-rich-rule='rule family="ipv4" source address="CARRIER.SIP.IP/32" port port="5060" protocol="udp" accept'

# Puertos de media RTP — solo IPs de carrier
firewall-cmd --permanent --zone=vicidial --add-rich-rule='rule family="ipv4" source address="CARRIER.SIP.IP/32" port port="10000-20000" protocol="udp" accept'

# Aplicar la zona a tu interfaz publica
firewall-cmd --permanent --zone=vicidial --change-interface=eth0
firewall-cmd --reload
```

**La regla critica que la gente olvida:** El puerto 5038 (AMI) debe ser **solo localhost.** El [Asterisk Manager Interface](/blog/asterisk-manager-interface-guide/) otorga control total de Asterisk — originar llamadas, colgar canales, leer configuracion. Si AMI es accesible desde la red, un atacante con las credenciales puede hacer llamadas internacionales, borrar buzon de voz, o crashear el PBX.

---

## Seguridad SIP de Asterisk

Un servidor Asterisk mal configurado expuesto al internet sera encontrado y explotado en horas — no dias, horas. Los scanners SIP corren 24/7 a traves de todo el espacio IPv4.

### Asegurar el Registro SIP

```ini
[general]
; Solo permitir conexiones de IPs conocidas
allowguest=no
alwaysauthreject=yes

; Bind a interfaz especifica, no 0.0.0.0
bindaddr=TU.IP.SERVIDOR

; Deshabilitar lookups DNS SRV
srvlookup=no

; Configuracion NAT
externip=TU.IP.PUBLICA
localnet=192.168.1.0/255.255.255.0
directmedia=no
```

**`allowguest=no`** previene que Asterisk acepte llamadas de endpoints no registrados.

**`alwaysauthreject=yes`** retorna el mismo rechazo sin importar si el usuario existe o no, eliminando la fuga de informacion.

### Fail2ban para SIP: No Negociable

```ini
# /etc/fail2ban/jail.local

[asterisk]
enabled  = true
filter   = asterisk
action   = iptables-allports[name=ASTERISK, protocol=all]
logpath  = /var/log/asterisk/messages
maxretry = 3
findtime = 600
bantime  = 86400
ignoreip = 127.0.0.1/8 TU.IP.OFICINA/32
```

**Establece `bantime` en al menos 86400 (24 horas).** El default de 600 segundos es un chiste — los scanners automatizados simplemente esperan 10 minutos y continuan. Nosotros corremos bans de 7 dias en sistemas de produccion.

```bash
# Verificar estado de fail2ban
fail2ban-client status asterisk

# Ver IPs actualmente baneadas
fail2ban-client status asterisk | grep "Banned IP"
```

### Credenciales SIP Fuertes

```bash
# Generar una contrasena SIP fuerte
openssl rand -base64 24
```

Cada endpoint SIP — cada telefono, cada trunk, cada extension de agente — necesita una contrasena unica, generada aleatoriamente de al menos 20 caracteres.

---

## Control de Acceso MySQL

La base de datos MySQL de VICIdial contiene todo: credenciales de agentes, metadata de grabaciones de llamadas, datos de leads (nombres, numeros de telefono, direcciones), configuraciones de campanas, y claves API. Una base de datos comprometida es un compromiso total.

### Restringir MySQL Binding

```ini
# /etc/my.cnf
[mysqld]
bind-address = 127.0.0.1
```

En un entorno de cluster, bind a la interfaz privada solamente:

```ini
bind-address = 192.168.1.10
```

### Eliminar Usuarios por Defecto y Sobre-Privilegiados

```sql
-- Listar todos los usuarios y sus hosts
SELECT user, host FROM mysql.user;

-- Verificar hosts comodin (peligroso)
SELECT user, host FROM mysql.user WHERE host = '%';

-- Remover acceso comodin
DROP USER 'cron'@'%';
CREATE USER 'cron'@'192.168.1.%' IDENTIFIED BY 'CONTRASENA_ALEATORIA_FUERTE';
GRANT SELECT, INSERT, UPDATE, DELETE ON asterisk.* TO 'cron'@'192.168.1.%';

-- Asegurar root
DROP USER 'root'@'%';
ALTER USER 'root'@'localhost' IDENTIFIED BY 'CONTRASENA_MUY_FUERTE';

FLUSH PRIVILEGES;
```

### Permisos de Archivos de Credenciales VICIdial

```bash
# astguiclient.conf — legible solo por root y asterisk
chmod 640 /etc/astguiclient.conf
chown root:asterisk /etc/astguiclient.conf

# Archivos de configuracion web
chmod 640 /var/www/html/vicidial/dbconnect_mysqli.php
chown root:apache /var/www/html/vicidial/dbconnect_mysqli.php
```

---

## Aplicacion de HTTPS

Correr VICIdial sobre HTTP plano en 2026 es negligente. Credenciales de agentes, datos de leads, y tokens de sesion atraviesan la red en texto claro.

```bash
# Instalar certbot
yum install -y certbot python3-certbot-apache

# Obtener certificado
certbot --apache -d dialer.tudominio.com

# Verificar auto-renovacion
certbot renew --dry-run
```

```apache
<VirtualHost *:443>
    ServerName dialer.tudominio.com
    DocumentRoot /var/www/html

    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/dialer.tudominio.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/dialer.tudominio.com/privkey.pem

    # Configuracion TLS moderna
    SSLProtocol all -SSLv3 -TLSv1 -TLSv1.1
    SSLHonorCipherOrder on

    # Headers de seguridad
    Header always set Strict-Transport-Security "max-age=63072000; includeSubDomains"
    Header always set X-Content-Type-Options "nosniff"
    Header always set X-Frame-Options "SAMEORIGIN"
</VirtualHost>
```

---

## Checklist de Hardening de 22 Puntos

1. Firewall configurado con denegacion por defecto
2. Solo puertos requeridos abiertos
3. SSH restringido a IPs de gestion
4. Puerto SSH cambiado del 22
5. Autenticacion SSH basada en llaves
6. Login root SSH deshabilitado
7. fail2ban habilitado para SSH
8. fail2ban habilitado para SIP/Asterisk
9. SIP allowguest=no
10. SIP alwaysauthreject=yes
11. AMI restringido a localhost
12. Contrasenas SIP de 20+ caracteres, unicas
13. ACLs de Asterisk configuradas para carriers
14. MySQL bind a localhost o interfaz privada
15. Usuarios MySQL wildcard eliminados
16. Contrasena root MySQL establecida
17. Archivos de credenciales con permisos restrictivos
18. HTTPS habilitado con TLS 1.2+
19. Codigo VICIdial actualizado a SVN mas reciente
20. Panel admin detras de VPN o whitelist de IP
21. Backups automaticos configurados
22. Monitoreo y alertas activas

> **Tu Firewall No Deberia Ser Adivinanza.**
> Los clusters ViciStack vienen con aislamiento dual-NIC, reglas SIP especificas por carrier, y cero puertos de gestion expuestos al internet. [Obtener Tu Evaluacion de Seguridad Gratuita →](/free-audit/)

---

*Esta guia de seguridad es mantenida por ViciStack y actualizada a medida que se divulgan nuevas vulnerabilidades. Ultima actualizacion: Marzo 2026.*

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/es-vicidial-security-hardening).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
