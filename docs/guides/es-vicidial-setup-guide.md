# La Guia Completa de Instalacion de VICIdial (2026): De Servidor Nuevo a Primera Llamada en Menos de 2 Horas

**Ultima actualizacion: Marzo 2026 | Tiempo de lectura: ~18 minutos**

Ya hiciste las cuentas. [Convoso quiere $150/puesto/mes](/blog/vicidial-vs-convoso/). [Five9 quiere aun mas](/blog/vicidial-vs-five9/). Mientras tanto, VICIdial — el mismo [marcador predictivo](/blog/es-best-predictive-dialer/) open-source que alimenta mas de 14,000 instalaciones en todo el mundo — no cuesta absolutamente nada en licencias.

Solo hay un problema: instalarlo.

Cada guia que vas a encontrar en Google ahora mismo es un PDF de ViciBox 7 del 2018, un hilo de foro con 47 respuestas contradictorias, o un post de blog que parece que fue traducido tres veces. CentOS 7 — el sistema operativo que el 90% de esas guias referencian — llego a su fin de vida en junio de 2024. Si sigues esas instrucciones en 2026, estas construyendo sobre una base muerta.

Esta guia arregla eso. Vamos a cubrir ViciBox 12.0.2 (la version estable actual), instalaciones desde cero en AlmaLinux 9, despliegues de servidor unico, clusters multi-servidor, [configuracion de](/blog/es-vicidial-amd-guide/) troncales SIP con ejemplos reales de carriers, configuracion de WebRTC para agentes remotos, y cada problema comun que te va a arruinar el fin de semana si no lo conoces de antemano.

**O — escuchanos — puedes saltarte todo esto y dejar que ViciStack lo maneje [en una](/blog/es-vicidial-vs-gohighlevel/) noche.** Migramos operadores de VICIdial a infraestructura bare-metal completamente optimizada con precision de [AMD](/blog/vicidial-amd-guide/) llegando al 92-96% desde el primer dia. Sin compilar Asterisk desde el codigo fuente. Sin depurar audio unidireccional a las 2 AM. Pero si quieres hacerlo tu mismo, sigue leyendo. Respetamos el espiritu DIY. Solo queremos que sepas que hay una mejor opcion cuando estes listo.

---

## Que Cambio Realmente en 2024-2026 (Y Por Que Tu Guia Antigua Te Esta Mintiendo)

Saquemos esto del camino primero, porque si te saltas esta seccion y sigues un tutorial desactualizado, vas a perder unas buenas 6-8 horas antes de darte cuenta de que algo esta roto de raiz.

**CentOS 7 esta muerto.** Literalmente muerto. EOL junio 2024. Sin mas parches [de seguridad](/blog/es-vicidial-security-hardening/), sin mas actualizaciones. Cada "guia de instalacion de VICIdial" de striker24x7, cada curso de Udemy, cada hilo de foro que dice `yum install centos-release-scl` — todo eso es ficcion historica ahora.

**ViciBox salto de la v9 a la v12.** Los numeros de version no son un error. ViciBox 12.0.2 salio en enero de 2025, corriendo sobre OpenSuSE Leap 15.6 con Asterisk 18, MariaDB 10.11.9, y PHP 8.2. ViciBox 13.0 ya esta en beta con OpenSuSE 16.0 y soporte para SELinux. Si estas siguiendo una guia que menciona ViciBox 8 o 9, estas leyendo historia antigua.

**Asterisk 18 es ahora el estandar.** El salto de Asterisk 13/16 a 18 trajo soporte PJSIP, mejor manejo de WebRTC, y mejor negociacion de codecs. Matt Florell confirmo oficialmente el soporte completo de Asterisk 18 en septiembre de 2025. Los parches especificos de VICIdial ahora apuntan exclusivamente a Asterisk 18 para nuevas instalaciones.

**PHP 8.2 es estandar.** El codigo de VICIdial con mas de 4 anos tirara advertencias de depreciacion o directamente se rompera en PHP 8.x. Las funciones `mysql_*` que tus viejos scripts de instalacion referencian desaparecieron desde PHP 7.0.

**El trunk SVN esta en la revision 3939+**, version 2.14b0.5, esquema de base de datos 1729. Todavia alojado en `svn://svn.eflo.net:3690/agc_2-X/trunk` porque el proyecto VICIdial no tiene planes de migrar a Git. Algunas cosas nunca cambian.

Esto es lo que significa en la practica: **los unicos dos caminos que valen la pena para una nueva instalacion [de VICIdial en](/blog/es-vicidial-cloud-deployment/) 2026 son ViciBox 12.0.2 (recomendado) o una instalacion desde cero en AlmaLinux 9 / Rocky Linux 9.** Todo lo demas es una perdida de tiempo.

> **Tu Guia Antigua Te Esta Mintiendo. Nosotros No.**
> ViciStack despliega VICIdial completamente optimizado en Asterisk 18, AlmaLinux 9, con cada problema pre-resuelto. [Salta la Instalacion →](/free-audit/)

---

## Hardware: Lo Que Realmente Necesitas (No Lo Que El Foro Te Dijo en 2015)

Hablemos de numeros reales. La documentacion de ViciBox 12 finalmente tiene especificaciones de dimensionamiento adecuadas, y son diferentes de lo que encontraras en posts viejos de foros.

### Servidor Unico (La Configuracion "Tengo 10-25 Agentes")

| Componente | Minimo | Recomendado |
|-----------|---------|-------------|
| CPU | 4 nucleos @ 2.0+ GHz | 6+ nucleos @ 2.0+ GHz |
| RAM | 8 GB | 16 GB ECC |
| Almacenamiento | 160 GB SSD | 500 GB RAID1 SSD |
| Red | 1 Gbps | 1 Gbps dedicado |

**Los SSDs son obligatorios en 2026.** No recomendados — obligatorios. La documentacion de ViciBox establece explicitamente SATA SSD como el minimo. Si alguien intenta venderte un servidor VICIdial con discos mecanicos, o estan atascados en 2016 o no les importa que tus agentes se queden sentados esperando consultas de base de datos.

Un solo servidor Express maneja de manera realista **15-20 agentes outbound** con marcacion predictiva activa, o aproximadamente 50 agentes solo-inbound bajo condiciones ideales. Una vez que pasas de 25 agentes outbound, estas jugando con fuego.

### Cluster Multi-Servidor (La Configuracion "Realmente Estoy Manejando un Negocio")

Cuando superas un solo servidor, VICIdial se divide en cuatro roles. Para [la guia completa](/blog/es-open-source-call-center-software/) sobre [arquitectura de cluster, planificacion de capacidad, y cada detalle de configuracion](/blog/vicidial-cluster-guide/), consulta nuestra guia dedicada de clusters.

**Servidor de base de datos** — El cerebro. Uno por cluster, siempre. Para 150 agentes: 8+ nucleos, 32 GB de RAM ECC, NVMe RAID1.

**Servidores de telefonia/marcador** — Los pulmones. Cada uno maneja aproximadamente 25 agentes outbound con grabacion pesada y ratios de marcacion de 4:1.

**Servidores web** — La cara. 2-4 nucleos, 4-8 GB de RAM. SSL reduce la capacidad aproximadamente a la mitad debido al overhead de TLS.

**Servidor de archivo** — La memoria. Este es el unico lugar donde los discos mecanicos realmente estan bien.

> **Deja de Adivinar las Especificaciones del Servidor.**
> ViciStack provisiona [bare metal](/blog/vicidial-setup-guide/) construido a medida para tu numero exacto de agentes y ratio de marcacion. [Obtener Tu Cotizacion Personalizada →](/pricing/)

---

## Metodo de Instalacion 1: ViciBox ISO (El Camino Sensato)

ViciBox es el ISO pre-construido oficial mantenido por Kumba (el desarrollador de ViciBox). Empaqueta OpenSuSE Leap 15.6, Asterisk 18, MariaDB, Apache, PHP, y VICIdial en una sola imagen booteable. Es el camino de menor resistencia y el que recomendamos para cualquiera que valore su tiempo.

### Descarga e Inicio

Descarga ViciBox 12.0.2 de `download.vicidial.com/iso/vicibox/server/`. Dos sabores: **Standard** (disco unico, RAID por hardware, VMs) y **MD** (RAID1 por software entre dos discos). Graba en USB con Rufus o dd, inicia, y selecciona "Install ViciBox" del menu.

El instalador copia el sistema operativo al disco, inicias sesion como root, y te guia a traves de idioma, teclado, zona horaria, y contrasena de root. Reinicia cuando se te indique. Tiempo total: aproximadamente 10 minutos.

### Configuracion Pre-VICIdial (No Te Saltes Esto)

Antes de tocar VICIdial, asegura tu red. La configuracion de VICIdial esta permanentemente ligada a la direccion IP de tu servidor — cambiarla despues es una cirugia dolorosa en multiples archivos.

**Establece una IP estatica:**
```bash
yast lan
```
Selecciona tu interfaz, elige "Statically assigned IP Address," ingresa la IP/subred/gateway, y establece DNS. Presiona ALT-O para aplicar. Verifica con `ping -4 google.com`.

**Establece la zona horaria (usa el comando de ViciBox, no yast):**
```bash
vicibox-timezone
```
El comando regular de yast para zona horaria no actualiza la zona horaria de PHP. Preguntame como lo se.

**Actualiza el sistema:**
```bash
zypper ref
zypper up
reboot
```

**Advertencia critica:** Siempre `zypper up`, nunca `zypper dup`. El comando `dup` (actualizacion de distribucion) puede degradar MariaDB o romper la compatibilidad con DAHDI. Multiples posts del foro documentan esto destruyendo sistemas de produccion.

### Instala VICIdial (El Milagro de Un Solo Comando)

Para un servidor unico con 20 o menos agentes:
```bash
vicibox-express
```

Escribe Y. Espera. Reinicia. Eso es todo. VICIdial esta corriendo.

Verifica con `screen -ls` — deberias ver 10-12 sesiones de screen. Accede a `http://<IP-de-tu-servidor>/vicidial/welcome.php` con las credenciales por defecto **6666 / 1234** y estaras viendo un marcador funcionando.

Para un cluster, ejecuta `vicibox-install` en cada servidor (base de datos primero, luego web, luego telefonia), selecciona que roles habilitar, y apunta los servidores que no son DB a la IP del servidor de base de datos. Mismo proceso, solo repetido.

### El Unico Bug Que Necesitas Arreglar Inmediatamente

ViciBox 12 viene con una version de MariaDB que deprecio el comportamiento implicito de TIMESTAMP. Esto puede romper tablas silenciosamente. Arreglalo antes de hacer cualquier otra cosa:

```bash
echo "explicit_defaults_for_timestamp = Off" >> /etc/my.cnf.d/general.cnf
systemctl restart mariadb.service
```

> **ViciBox Te Pone en Marcha. ViciStack Te Da Resultados.**
> Vamos mas alla de la instalacion — ajuste de AMD, gestion de DID, optimizacion de carriers, todo incluido. [Ver la Diferencia →](/free-audit/)

---

## Metodo de Instalacion 2: Instalacion Desde Cero en AlmaLinux 9 (El Camino del Controlador)

Algunas personas necesitan Linux de la familia RHEL. Algunas personas quieren entender cada componente. Algunas personas simplemente disfrutan compilar software desde el codigo fuente un viernes por la noche. No juzgamos.

La mejor opcion en 2026 es el **auto-instalador de carpenox** mantenido por Chris en CyburDial/Dialer.one. Es el script comunitario mas activamente mantenido y maneja AlmaLinux 9 + Rocky Linux 9 con Asterisk 18:

```bash
timedatectl set-timezone America/New_York
yum check-update && yum update -y
yum -y install epel-release && yum update -y
yum install git kernel* --exclude=kernel-debug* -y
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
cd /usr/src
git clone https://github.com/carpenox/vicidial-install-scripts.git
reboot
cd /usr/src/vicidial-install-scripts
chmod +x alma-rocky9-ast18.sh
./alma-rocky9-ast18.sh
```

**SELinux debe estar deshabilitado.** Esto no es negociable. Los scripts Perl de VICIdial, las operaciones de archivo de Asterisk, y la configuracion de Apache asumen que SELinux esta desactivado. Cada guia de instalacion desde cero comienza deshabilitandolo.

El script maneja la instalacion de dependencias, la compilacion de Asterisk 18 con los parches de VICIdial, DAHDI, LAME, Jansson, checkout de VICIdial por SVN, configuracion de base de datos, configuracion de crontab, y scripts de inicio. Los cinco parches especificos de VICIdial para Asterisk (estadisticas de AMD, estado de peer IAX, logging de peer SIP, y dos parches de reinicio de timeout) se aplican automaticamente.

---

## Post-Instalacion: De "Esta Corriendo" a "Estamos Haciendo Llamadas"

Aqui es donde cada otra guia en internet se detiene. "Felicitaciones, instalaste VICIdial! Aqui hay una captura de pantalla de la pagina de login. Buena suerte!" No ayuda. Vamos a configurar esto de verdad.

### Asegura los Valores por Defecto (Haz Esto Primero)

VICIdial viene con valores por defecto que son tambien agujeros de seguridad:

1. **Cambia la contrasena de admin** — Admin → Users → Modify user 6666. Las credenciales por defecto 6666/1234 las conoce literalmente todo el que haya buscado en Google "vicidial."
2. **Establece la contrasena root de MySQL** — `mysqladmin -u root password 'ALGO_FUERTE'`
3. **Cambia las contrasenas de registro de telefonos** — El valor por defecto es `test`. Si, en serio.
4. **Mueve SSH del puerto 22** — Cada bot en internet esta golpeando el puerto 22 ahora mismo.

### Configurando Tu Troncal SIP

Aqui es donde la mayoria de las instalaciones DIY se atascan. Necesitas un carrier VoIP para realmente hacer llamadas telefonicas, y la configuracion de carriers de VICIdial tiene algunas particularidades no obvias.

Navega a Admin → Carriers → Add A New Carrier. Para un trunk autenticado por IP (la mayoria de los carriers empresariales):

**Account Entry:**
```
[tu-carrier]
disallow=all
allow=ulaw
allow=g729
type=peer
insecure=port,invite
host=sip.tucarrier.com
dtmfmode=rfc2833
context=trunkinbound
canreinvite=no
```

**Dialplan Entry:**
```
exten => _91NXXNXXXXXX,1,AGI(agi://127.0.0.1:4577/call_log)
exten => _91NXXNXXXXXX,2,Dial(${CARRIER}/${EXTEN:1},60,tTor)
exten => _91NXXNXXXXXX,3,Hangup
```

**Global String:** `CARRIER=SIP/tu-carrier`

El prefijo `9` es una convencion de dial string — cuando tu campana usa el prefijo de marcacion `9`, VICIdial lo antepone a cada numero, y el dialplan lo elimina antes de enviarlo al carrier. Verifica tu trunk con:

```bash
asterisk -rx "sip show registry"
asterisk -rx "sip show peers"
```

**Consejo profesional de operar 100+ centros VICIdial:** La [atestacion STIR/SHAKEN](/blog/stir-shaken-vicidial-guide/) importa enormemente en 2026. Necesitas atestacion de nivel A, lo que requiere que tus DIDs y la terminacion esten en el mismo carrier. El enfoque dual-stack te da redundancia mientras mantienes nivel A en ambos.

> **La Configuracion del Carrier Es Donde Las Instalaciones DIY Van a Morir.**
> Una configuracion incorrecta de STIR/SHAKEN = "Posible Spam" en una semana. ViciStack configura tus carriers correctamente desde el primer dia. [Hazlo Bien →](/free-audit/)

### Creando Tu Primera Campana

Admin → Campaigns → Add a New Campaign. La configuracion critica es el **Dial Method:**

- **RATIO** — Llamadas fijas por agente (ej., 2.0 = dos llamadas simultaneas por agente disponible). Simple, predecible, bueno para equipos pequenos.
- **ADAPT_HARD_LIMIT** — Marcacion predictiva con un techo fijo en la tasa de abandono. Establece esto en 3% para cumplimiento TCPA. Esto es lo que la mayoria de las operaciones outbound deberian usar.
- **ADAPT_TAPERED** — Mas agresivo al principio, mas conservador a medida que avanza el dia. Bueno para operaciones experimentadas que entienden los compromisos.
- **MANUAL** — El agente hace clic para marcar. Para entornos de alto cumplimiento normativo o pruebas.

Configuraciones clave para establecer desde el inicio: **Hopper Level** (100-200 leads pre-cargados), **Dial Timeout** (26-30 segundos — depende del carrier), **Available Only Tally = Y** (solo marcar cuando los agentes estan realmente disponibles), y **Auto Dial Level** (empieza en 1.5 para modos adaptativos y ajusta basado en rendimiento). Para el desglose completo de [cada configuracion del marcador que importa](/blog/vicidial-predictive-dialer-settings/), consulta nuestra guia dedicada.

### Cargando Leads

Lists → Add A New List → asigna a tu campana → Lists → Load New Leads. Sube un CSV con como minimo: `phone_number`, `first_name`, `last_name`, `state`. Siempre prueba con un lote pequeno primero. El cargador de leads de VICIdial es potente pero implacable con problemas de formato.

### Configuracion del Telefono del Agente

Dos opciones en 2026:

**Softphone SIP (MicroSIP, Zoiper, X-Lite):** Crea un telefono en Admin → Phones con una extension (ej., 1001), IP del servidor, y contrasena de registro. El agente configura su softphone con estas credenciales. Funciona, pero requiere software en cada maquina de agente.

**WebRTC/ViciPhone (la forma moderna):** Requiere SSL/TLS en tu servidor web y el puerto 8089 abierto. Configura con `vicibox-ssl` en ViciBox, o certbot en instalaciones desde cero. Habilita plantillas de telefono WebRTC en Admin, establece los telefonos en "As Webphone = Y," y los agentes obtienen un telefono basado en navegador integrado directamente en la interfaz del agente. Sin instalaciones de software, funciona desde cualquier lugar. Asi es como la mayoria de las operaciones remotas funcionan en 2026.

---

## Clustering Multi-Servidor: Las Reglas Que Nadie Escribe

Una vez que creces mas alla de 20-25 agentes outbound, necesitas un [cluster](/blog/vicidial-cluster-guide/). Aqui estan las reglas que te salvaran del cementerio de posts de clusters rotos del foro:

**Regla 1: Un proceso adaptativo, un servidor.** El proceso `AST_VDadapt` (keepalive 5) gestiona el algoritmo predictivo. Corre en **exactamente un servidor** en todo el cluster. Ejecutarlo en dos servidores causa conflictos de nivel de marcacion que parecen abandonos aleatorios. Lo mismo aplica para `AST_VDauto_dial_FILL` (keepalive 7).

**Regla 2: Misma LAN, sin routers.** Todos los servidores del cluster deben estar en la misma red local con latencia de menos de 1ms. Un router entre tus servidores de base de datos y marcador agrega suficiente latencia para romper las sesiones de los agentes. Usa IAX2 (no SIP) para trunks inter-servidor.

**Regla 3: NTP desde una sola fuente.** Todos los servidores sincronizan sus relojes al servidor de base de datos o a una fuente NTP designada. La sincronizacion NTP independiente a servidores externos causa desviacion del reloj que rompe sesiones de agentes, desconecta llamadas, y corrompe reportes.

**Regla 4: Conoce tu techo.** Las tablas MEMORY de VICIdial son de un solo hilo. Un cluster llega al maximo alrededor de 450-500 agentes. Planifica tu crecimiento de acuerdo a esto.

### Cuando Agregar Que

| Llegas a... | Agregas... |
|------------|-----------|
| 20 agentes outbound | Separar DB de telefonia |
| 25 agentes mas | Segundo servidor de marcacion |
| 50+ agentes | Servidor de DB dedicado |
| 70+ agentes | Servidor web dedicado |
| 150+ agentes | Base de datos esclava para reportes |
| 450+ agentes | Segundo cluster |

> **Regla #1 del Clustering: No Lo Aprendas Por Las Malas.**
> Hemos construido mas de 100 clusters. Que nuestras cicatrices salven tu fin de semana. [Habla con un Experto en Clusters →](/free-audit/)

---

## El Salon de la Fama de la Resolucion de Problemas

Estos son los problemas que llenan los mas de 13,400 hilos de soporte del foro de VICIdial. Aprende del dolor de otros:

**Sin audio / audio unidireccional** — 80% de probabilidad de que el firewall este bloqueando UDP 10000-20000 (puertos RTP). 15% de probabilidad de que falta un `externip` en sip.conf. 5% de probabilidad de que sea SIP ALG en un router NAT. Deshabilita temporalmente el firewall y prueba. Si el audio funciona, es el firewall.

**"No available sessions"** — Las extensiones de conferencia no estan pobladas para la IP de tu servidor. Admin → Conferences → Show VICIDIAL Conferences. Cada servidor necesita su propio rango de conferencias.

**Grabaciones faltantes** — Verifica todo el pipeline: esta SOX instalado? La grabacion de campana esta en ALLCALLS? Estan corriendo los cron jobs? Revisa `/var/spool/asterisk/monitor/` para archivos crudos. La configuracion de grabacion a nivel de usuario puede sobreescribir silenciosamente la configuracion de campana — verifica ambas.

**Advertencia de desajuste de esquema de base de datos** — Actualizaste el codigo SVN pero olvidaste la base de datos. Ejecuta:
```bash
mysql -p -f --database=asterisk < /usr/src/astguiclient/trunk/extras/upgrade_2.14.sql
```

---

## La Verdad Honesta Sobre VICIdial DIY vs. Administrado

Mira, escribimos toda esta guia porque creemos en la transparencia. VICIdial es un software increible. Es gratuito, es potente, y en las manos correctas, supera a marcadores que [cuestan 10 veces mas](/blog/vicidial-cost-2026/).

Pero "las manos correctas" esta haciendo mucho trabajo pesado en esa oracion.

Manejar VICIdial tu mismo significa que eres el administrador de sistemas, el DBA, el ingeniero de telefonia, el auditor de seguridad, y el gerente de relaciones con carriers. Cuando Asterisk se cae a las 9 AM un lunes y 50 agentes estan sentados sin hacer nada, tu eres el que esta en la terminal. Cuando tus [DIDs se marcan](/blog/vicidial-did-management/) como spam y tus tasas de conexion caen un 40%, tu eres el que esta llamando a los carriers.

**ViciStack existe porque hemos hecho esto mas de 200 veces.** Hemos construido y vendido mas de 200 call centers. Hemos contratado mas de 10,000 agentes. Hemos pasado mas de 15 anos aprendiendo cada peculiaridad de VICIdial, cada trampa de Asterisk, cada optimizacion de carrier que mueve la aguja.

Esto es lo que entregamos que esta guia no puede:

- **Precision de AMD del 92-96%** (vs. el 80-85% que la mayoria de las instalaciones autogestionadas logran). Esa diferencia significa 7-16% mas conversaciones en vivo por hora.
- **Migracion nocturna** — Todo tu entorno VICIdial, movido a nuestra infraestructura bare-metal optimizada mientras tus agentes duermen.
- **[Atestacion STIR/SHAKEN de nivel A](/blog/stir-shaken-vicidial-guide/)** configurada correctamente desde el primer dia.
- **Gestion de reputacion de DID** — Rotamos y monitoreamos tus numeros para que "Posible Spam" no se coma tus tasas de conexion.

> **Leiste Toda la Guia. Respeto.**
> Ahora imagina saltarte todo y hacer llamadas manana. Eso es ViciStack. [Obtener Tu Prueba de Concepto Gratuita →](/free-audit/)

---

## Recursos Esenciales (Marca Estos Como Favoritos)

- **Documentacion de ViciBox 12:** docs.vicibox.com — Especificaciones de hardware, fases de instalacion, redes, firewall
- **Foro de VICIdial:** forum.vicidial.org — Mas de 13,400 temas. Busca antes de publicar. Matt Florell (mflorell) y William Conley (williamconley) son las voces mas autorizadas
- **SVN de VICIdial:** svn://svn.eflo.net:3690/agc_2-X/trunk — El codigo fuente
- **Scripts de Instalacion de Carpenox:** github.com/carpenox/vicidial-install-scripts — El auto-instalador mejor mantenido para Alma/Rocky
- **Manual del Gerente de VICIdial:** Amazon, $45-65 — La referencia [completa de](/blog/es-stir-shaken-vicidial-guide/) Matt Florell
- **ViciStack:** vicistack.com — Cuando termines de hacerlo tu mismo

---

*Esta guia es mantenida por ViciStack y actualizada a medida que el ecosistema de VICIdial evoluciona. Ultima verificacion contra ViciBox 12.0.2 y VICIdial SVN trunk 2.14b0.5, marzo 2026. Encontraste algo desactualizado? [Dinoslo.](/free-audit/)*

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/es-vicidial-setup-guide).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
