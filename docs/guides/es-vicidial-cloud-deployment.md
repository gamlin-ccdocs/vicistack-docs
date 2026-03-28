# Despliegue de VICIdial en la Nube: AWS, GCP y DigitalOcean

**Ultima actualizacion: Marzo 2026 | Tiempo de lectura: ~18 minutos**

Cada guia de VICIdial asume que tienes un rack, una IP estatica, y un contrato de data center. Eso tenia sentido en 2014. En 2026, la mayoria de los nuevos despliegues de VICIdial que vemos en ViciStack empiezan con la misma pregunta: puedo correr esto en la nube?

La respuesta corta es si. La respuesta larga es si, pero VICIdial y la infraestructura cloud tienen suposiciones fundamentalmente diferentes sobre networking, scheduling de CPU, y I/O de almacenamiento que necesitas entender antes de levantar una instancia EC2 y preguntarte [por que](/blog/es-vicidial-vs-gohighlevel/) la mitad de tus llamadas tienen audio unidireccional.

---

## Cuando Cloud Tiene Sentido

### Cuando Cloud Tiene Sentido

**Escalamiento elastico para operaciones estacionales.** Si tu numero de agentes oscila de 20 en enero a 150 durante la temporada de inscripcion abierta, la infraestructura cloud te permite escalar servidores de telefonia arriba y abajo sin comprar hardware que se queda inactivo 8 meses al ano.

**Redundancia geografica.** Correr [clusters VICIdial](/blog/vicidial-cluster-guide/) a traves de multiples regiones te da recuperacion ante desastres que es casi imposible de lograr con hardware colocado.

**Sin gestion del ciclo de vida del hardware.** Sin reemplazos de discos a las 3 AM. Sin fallas de controlador RAID.

### Cuando Cloud No Tiene Sentido

**Operaciones de alta densidad sostenida.** Si estas corriendo 100+ agentes en una carga de trabajo estable y predecible, [bare metal](/blog/vicidial-setup-guide/) es mas barato. Periodo.

**Requisitos de latencia ultra-baja.** Bare metal te da nucleos de CPU dedicados y sin overhead de hypervisor.

**Startups con presupuesto limitado.** Si tienes 10 agentes y $500/mes para todo, un Dell PowerEdge reacondicionado en Hetzner u OVH por $50-80/mes va a rendir mejor que una instancia cloud de $200/mes.

---

## Desafios Especificos de VICIdial en Cloud

### NAT Traversal de SIP

Este es el punto de dolor mas grande y la razon por la que la mayoria de los despliegues cloud de VICIdial fallan la primera vez. SIP fue disenado en una era cuando cada dispositivo tenia una IP publica. Incrusta direcciones IP dentro del payload del protocolo, no solo en los headers de los paquetes.

La solucion involucra multiples capas:

**Configuracion SIP de Asterisk:**

```ini
; sip.conf - Configuracion NAT para despliegue cloud
externip=<TU_IP_ELASTICA>
localnet=10.0.0.0/8
localnet=172.16.0.0/12
localnet=192.168.0.0/16
nat=force_rport,comedia
directmedia=no
```

Para PJSIP (el enfoque moderno):

```ini
; pjsip.conf
[transport-udp]
type=transport
protocol=udp
bind=0.0.0.0
external_media_address=<TU_IP_ELASTICA>
external_signaling_address=<TU_IP_ELASTICA>
local_net=10.0.0.0/8
local_net=172.16.0.0/12
local_net=192.168.0.0/16
```

### Rendimiento de vCPU Compartida

Este es el secreto sucio de VICIdial cloud. Asterisk es extremadamente sensible al jitter de scheduling de CPU. Cuando Asterisk procesa paquetes de audio RTP, necesita acceso de CPU consistente sub-milisegundo.

**No corras VICIdial en instancias t3/t2 (AWS), instancias e2 (GCP), o Droplets basicos (DigitalOcean).** Usa instancias de la serie c (compute-optimized) o tenencia de host dedicada.

---

## Despliegue en AWS

### Dimensionamiento de Instancias

| Rol | Tipo de Instancia | Specs | Costo Mensual (us-east-1) |
|------|-------------|-------|------------------------|
| Servidor unico (<25 agentes) | c5.2xlarge | 8 vCPU, 16 GB RAM | ~$245/mes |
| Servidor DB (25-100 agentes) | m5.xlarge | 4 vCPU, 16 GB RAM | ~$140/mes |
| Servidor de telefonia/marcador | c5.2xlarge | 8 vCPU, 16 GB RAM | ~$245/mes |
| Servidor web | c5.xlarge | 4 vCPU, 8 GB RAM | ~$124/mes |

### Security Groups

**Servidor de Telefonia:**

| Puerto | Protocolo | Origen | Proposito |
|------|----------|--------|---------|
| 5060 | UDP | IPs del Carrier | Senalizacion SIP |
| 5061 | TCP | IPs del Carrier | SIP TLS |
| 10000-20000 | UDP | 0.0.0.0/0 | Media RTP |
| 4569 | UDP | Servidores del cluster | IAX2 inter-servidor |
| 3306 | TCP | VPC CIDR | MySQL |
| 443 | TCP | IPs de agentes | WebRTC/ViciPhone |

**Critico:** El rango de puertos RTP (10000-20000) debe estar abierto a 0.0.0.0/0 en UDP. Estos puertos llevan el audio de voz real.

### Ejemplo Terraform para VICIdial en AWS

```hcl
provider "aws" {
  region = "us-east-1"
}

resource "aws_vpc" "vicidial" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true
  tags = { Name = "vicidial-vpc" }
}

resource "aws_instance" "vicidial" {
  ami                    = "ami-XXXXXXXX" # AlmaLinux 9 o openSUSE Leap 15.6
  instance_type          = "c5.2xlarge"
  key_name               = "tu-key-pair"
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.vicidial.id]

  root_block_device {
    volume_type = "gp3"
    volume_size = 100
    iops        = 3000
  }
}

resource "aws_eip" "vicidial" {
  instance = aws_instance.vicidial.id
  domain   = "vpc"
}
```

---

## Despliegue en GCP

### Dimensionamiento de Instancias

| Rol | Tipo de Maquina GCP | Equivalente AWS | Costo Mensual |
|------|-----------------|---------------|-------------|
| Servidor unico (<25 agentes) | n2-standard-8 | c5.2xlarge | ~$260/mes |
| Servidor DB (25-100 agentes) | n2-highmem-4 | m5.xlarge | ~$155/mes |
| Servidor de telefonia | c2-standard-8 | c5.2xlarge | ~$275/mes |

**Ventaja de GCP: c2-standard-8.** La serie c2 corre en procesadores Intel Cascade Lake dedicados de 3.1 GHz sin sobrecomprometimiento de CPU. Esto es lo mas cercano a bare metal que puedes obtener en una VM cloud.

**Ventaja de GCP: descuentos por uso sostenido.** A diferencia de [AWS, GCP](/blog/vicidial-cloud-deployment/) automaticamente aplica descuentos por uso sostenido cuando una instancia corre mas del 25% del mes. Sin compromiso requerido.

---

## Despliegue en DigitalOcean

DigitalOcean es la opcion cloud mas simple para VICIdial. Menos flexibilidad que AWS o GCP, pero dramaticamente menos complejidad.

### Dimensionamiento de Droplets

| Rol | Tipo de Droplet | Specs | Costo Mensual |
|------|-------------|-------|-------------|
| Servidor unico (<25 agentes) | CPU-Optimized 8 vCPU | 8 vCPU, 16 GB RAM | $168/mes |
| Servidor DB | General Purpose 8 GB | 2 vCPU, 8 GB RAM | $68/mes |
| Servidor de telefonia | CPU-Optimized 8 vCPU | 8 vCPU, 16 GB RAM | $168/mes |

**Critico: Usa CPU-Optimized Droplets.** Los Droplets Basic y General Purpose usan vCPUs compartidas con rendimiento burstable. Los CPU-Optimized proporcionan nucleos vCPU dedicados.

```bash
# Crear un CPU-Optimized Droplet via doctl CLI
doctl compute droplet create vicidial-server \
  --region nyc1 \
  --size c-8 \
  --image almalinux-9-x64 \
  --ssh-keys TU_FINGERPRINT_SSH_KEY \
  --wait
```

---

## Patron de Arquitectura: Servidor Unico (Menos de 25 Agentes)

```
┌──────────────────────────┐
│   Instancia Cloud         │
│   (c5.2xlarge / c2-8)    │
│                           │
│   ┌─────────────────┐    │
│   │ Asterisk 18     │    │
│   │ Apache/PHP      │    │
│   │ MariaDB         │    │
│   │ VICIdial Screens│    │
│   └─────────────────┘    │
│                           │
│   IP Elastica/Estatica    │
└──────────────────────────┘
         │
         ▼
    Carrier SIP
```

> **Despliegue Cloud Sin Los Dolores de Cabeza.**
> ViciStack maneja la infraestructura para que puedas enfocarte en correr campanas. Rendimiento bare metal, simplicidad tipo cloud. [Obtener Una Cotizacion Personalizada.](/pricing/)

---

*Esta guia es mantenida por ViciStack y actualizada a medida que los proveedores cloud y los requisitos de VICIdial evolucionan. Ultima actualizacion: Marzo 2026.*

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/es-vicidial-cloud-deployment).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
