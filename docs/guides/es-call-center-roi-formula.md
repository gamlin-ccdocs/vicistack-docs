# Formula ROI de Call Center: Como Calcular y Mejorar

La mayoria de los operadores de call center pueden decirte su talk time, su [tasa de contacto](/glossary/contact-rate/), y su [tasa de conversion](/glossary/conversion-rate/). Preguntales sobre ROI y obtendras una mirada en blanco, un vago "somos rentables," o un numero que no cuenta la mitad de sus costos reales.

Esto no es un descuido menor. **El ROI es la unica metrica que te dice si tu call center es un motor de ganancias o un pozo de dinero.** Cada otra metrica es un insumo. El talk time te dice que tan ocupados estan los agentes. La tasa de conversion te dice que tan efectivos son. El [costo por lead](/glossary/cost-per-lead/) te dice que tan caro es cada resultado. Pero el ROI es la respuesta final — el numero que conecta cada dolar que gastas con cada dolar que ganas.

---

## La Formula de ROI de Call Center

La formula basica es sencilla:

**ROI = (Ingresos Generados - Costo Total) / Costo Total x 100**

Un ROI de 100% significa que ganaste el doble de lo que gastaste. Un ROI de 50% significa que ganaste $1.50 por cada dolar gastado. Un ROI negativo significa que estas perdiendo dinero.

Formula simple. La complejidad esta en obtener los insumos correctos.

### Ingresos Generados

La atribucion de ingresos es donde la mayoria de los calculos de ROI de call center fallan:

**Operacion de ventas directas (vendes [en la](/blog/es-vicidial-cloud-deployment/) llamada):**
Ingresos = Total de ventas cerradas x Valor promedio de venta

**Generacion de leads / programacion de citas:**
Ingresos = Leads calificados generados x Valor del lead

**Calificacion de leads / operacion de transferencia:**
Ingresos = Transferencias en vivo x Valor de transferencia

### Costo Total: La Parte Que Todos Subestiman

El denominador de la formula de ROI — costo total — es donde los operadores consistentemente se equivocan.

---

## Desglose de Costos del Call Center

### Mano de Obra: 60-70% del Costo Total

Para un agente en EE.UU. ganando $15/hora:
- Costo bruto por hora: $15.00
- Impuestos de nomina: $1.15
- Beneficios (25%): $3.75
- Amortizacion de entrenamiento: $0.50/hora
- **Tasa por hora totalmente cargada: $20.40**

Ese agente de $15/hora realmente cuesta $20.40/hora. A traves de un [piso de 50](/blog/es-vicidial-vs-gohighlevel/) agentes en turnos de 8 horas, eso es $8,160/dia, o aproximadamente **$176,800/mes** solo en mano de obra de agentes.

### Tecnologia: 15-20% del Costo Total

| Componente | Costo Mensual (50 agentes) |
|-----------|-------------------------|
| Infraestructura de servidores ([bare metal](/blog/vicidial-setup-guide/)) | $200-$600 |
| Troncales SIP | $500-$1,500 |
| Numeros DID | $100-$300 |
| Limpieza DNC | $50-$100 |
| Mano de obra admin (parcial o administrada) | $1,000-$3,000 |
| **Total VICIdial** | **$1,850-$5,500** |

La diferencia de costo de tecnologia es dramatica: **$15,000-$20,000/mes para una plataforma [hosted vs](/blog/hosted-vs-self-hosted-dialer-cost/). $2,000-$5,500/mes para VICIdial.**

---

## Economias Por Agente: El Bloque Constructor del ROI

### Costo Por Hora de Agente

| Componente de Costo | Por Agente/Hora |
|----------------|---------------|
| Mano de obra del agente (totalmente cargada) | $20.40 |
| Overhead de gestion (prorrateado) | $4.00 |
| Tecnologia (VICIdial/ViciStack) | $0.50-$1.25 |
| Tecnologia (Five9/Convoso) | $3.00-$5.00 |
| Telecom | $1.50-$3.00 |
| Overhead | $1.50-$2.50 |
| **Total (VICIdial)** | **$27.90-$31.15** |
| **Total (Plataforma hosted)** | **$30.40-$34.90** |

La diferencia de costo por hora-agente entre VICIdial y plataformas hosted es $2.50-$3.75. Eso suena pequeno hasta que multiplicas: 50 agentes x 8 horas x 22 dias laborales x $3.00 = **$26,400/mes** en ahorros de tecnologia solamente.

### Ejemplo Trabajado: Generacion de Leads de Seguros, 50 Agentes, VICIdial/ViciStack

Lado de ingresos:
- Contactos por hora: 7
- Tasa de calificacion de leads: 10%
- Pago por lead calificado: $65
- Ingresos por hora-agente: 7 x 10% x $65 = **$45.50**

Lado de costos:
- Costo por hora-agente (VICIdial): $29.00

**Margen por hora-agente: $45.50 - $29.00 = $16.50**

Calculo de ROI mensual:
- Ingresos mensuales totales: $45.50 x 8 horas x 22 dias x 50 agentes = **$400,400**
- Costo mensual total: $29.00 x 8 x 22 x 50 = **$255,200**
- **ROI: ($400,400 - $255,200) / $255,200 x 100 = 56.9%**

Ahora la misma operacion en Five9:
- Costo por hora-agente: $32.50
- Costo mensual total: $32.50 x 8 x 22 x 50 = **$286,000**
- **ROI: ($400,400 - $286,000) / $286,000 x 100 = 40.0%**

Mismos ingresos, diferente costo de tecnologia, **16.9 puntos porcentuales de diferencia de ROI**.

> **Conoce Tus Numeros Antes de Tu Proximo Ciclo de Presupuesto.**
> La auditoria gratuita de ViciStack incluye un analisis completo de ROI de tu operacion actual. [Obtener Tu Auditoria de ROI Gratuita →](/free-audit/)

---

## Las Optimizaciones de ROI Que Mas Mueven la Aguja

### 1. Calibracion de AMD: 15-20% de Aumento en Talk Time

[Answering Machine Detection](/glossary/amd/) filtra los saludos de buzon de voz antes de que lleguen a los agentes. AMD correctamente calibrado aumenta el talk time del agente en 15-20%.

En VICIdial 2.14-917a+, los parametros criticos de AMD viven en la configuracion de campana. Estos son los valores que usamos en operaciones de produccion de alta velocidad:

```
# Configuracion AMD en la campana (via admin GUI > Campaigns > Detail)
AMD Type:                 AMD
AMD Send Notification:    Y
Answering Machine Message: 8304|vm_greeting
AMD Agent Route Options:  CALLMENU
VM Message Group:          ---NONE---
```

El valor por defecto de `amd_type` en VICIdial causa que ~30% de las llamadas a contestadoras se conecten con agentes, desperdiciando talk time. La configuracion correcta de AMD redirige esas llamadas automaticamente y libera agentes para conexiones reales.

### 2. Gestion de DID: 20-40% de Mejora en Tasa de Respuesta

[Gestion de DID](/blog/vicidial-did-management/) — rotar caller IDs, usar numeros de presencia local, monitorear flags de spam, y reemplazar numeros marcados — tipicamente mejora la tasa de respuesta en 20-40%.

### 3. Gestion de Listas: 50-100% de Mejora en Tasa de Contacto

La segmentacion adecuada de leads, logica de reciclaje, filtrado de zona horaria, y [ordenamiento de leads](/settings/lead-order/) pueden duplicar tu [tasa de contacto](/glossary/contact-rate/).

### 4. Ajuste del Marcador Predictivo: 25-35% de Mejora en Utilizacion del Agente

Las configuraciones por defecto de VICIdial producen aproximadamente 28-32 minutos de talk time por hora-agente. Configuraciones optimizadas producen 45-55 minutos.

Puedes verificar la utilizacion actual de tus agentes con una consulta directa a la tabla de logs:

```sql
SELECT user, SUM(talk_sec) / 3600 AS talk_hours,
       SUM(pause_sec) / 3600 AS pause_hours,
       SUM(wait_sec) / 3600 AS wait_hours,
       ROUND(SUM(talk_sec) / (SUM(talk_sec) + SUM(pause_sec) + SUM(wait_sec)) * 100, 1) AS pct_talk
FROM vicidial_agent_log
WHERE event_time >= DATE_SUB(NOW(), INTERVAL 7 DAY)
GROUP BY user
ORDER BY pct_talk DESC;
```

Y para rastrear tu costo real por contacto — el insumo mas critico del calculo de ROI — esta consulta cruza los intentos de llamada contra los contactos exitosos:

```sql
SELECT campaign_id,
       COUNT(*) AS total_calls,
       SUM(IF(status IN ('SALE','XFER','CALLBK'), 1, 0)) AS contacts,
       ROUND(SUM(IF(status IN ('SALE','XFER','CALLBK'), 1, 0)) / COUNT(*) * 100, 2) AS contact_rate_pct
FROM vicidial_log
WHERE call_date >= DATE_SUB(NOW(), INTERVAL 30 DAY)
GROUP BY campaign_id;
```

Estos numeros alimentan directamente la formula de ROI. Si tu `contact_rate_pct` esta por debajo de 8%, tienes un problema de listas o de [configuracion de AMD](/blog/es-vicidial-amd-guide/) antes de tocar cualquier otra variable.

---

## El Efecto Compuesto de Multiples Optimizaciones

**Escenario base: Operacion outbound de 50 agentes, VICIdial, configuraciones por defecto**
- ROI mensual: 7.6%

**Despues de todas las optimizaciones:**
- Calibracion de AMD → contactos/hora mejora
- Gestion de DID → tasa de respuesta mejora 30%
- Gestion de listas → tasa de contacto mejora 50%
- Ajuste del marcador → utilizacion del agente al 80%

Ingreso optimizado por hora-agente: **$81.25**

Ingresos mensuales: **$715,000**
Costo mensual: **$255,200**
**ROI mensual: 180.2%**

De 7.6% de ROI a 180.2%. Los mismos 50 agentes. Los mismos leads. Los mismos telefonos. La diferencia es configuracion, no gasto de capital.

---

## Preguntas Frecuentes

### Cual es un buen ROI para un call center outbound?

Depende mucho de tu vertical. Operaciones de ventas directas B2C tipicamente apuntan a 80-150% de ROI. Operaciones de generacion de leads apuntan a 40-80%. Operaciones BPO con contratos por transferencia apuntan a margenes de 20-40%.

### Cual es la diferencia de ROI entre VICIdial y plataformas hosted como Five9?

Para una operacion de 50 agentes, la diferencia de costo de tecnologia sola es $120,000-$180,000/ano. Pero la verdadera ventaja de ROI de VICIdial no es solo costo — es configurabilidad. Los clientes de ViciStack consistentemente logran 10-20% mayor utilizacion de agentes que operaciones comparables en plataformas hosted.

### Con que frecuencia debo recalcular el ROI?

Semanalmente. No mensualmente, no trimestralmente — semanalmente. Los calculos mensuales de ROI pierden problemas a corto plazo. El rastreo semanal de ROI con atribucion de ingresos rezagada te da el ciclo de retroalimentacion mas rapido. Tenemos una guia paso a paso de como configurar reportes automatizados de ROI en [ViciStack](https://vicistack.com/blog/call-center-roi-formula/) usando los datos nativos de la base de datos de VICIdial.

---

*Esta guia es mantenida por el equipo de ViciStack y actualizada a medida que los benchmarks de la industria y los costos de tecnologia evolucionan. Ultima actualizacion: Marzo 2026.*

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/es-call-center-roi-formula).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
