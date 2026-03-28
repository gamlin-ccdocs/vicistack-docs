# STIR/SHAKEN para VICIdial: La Guia Completa de Implementacion 2026

*Publicado por ViciStack — la plataforma administrada de VICIdial construida por operadores, para operadores.*

---

Si estas corriendo un call center VICIdial en 2026 y piensas que STIR/SHAKEN es solo otra casilla de cumplimiento que puedes ignorar con seguridad — felicitaciones, estas a punto de aprender como se siente una caida del 50% en tasas de respuesta.

La realidad incomoda que nadie [en la](/blog/es-vicidial-cloud-deployment/) comunidad VICIdial esta explicando con suficiente claridad: **el cumplimiento de STIR/SHAKEN es necesario pero ni de lejos suficiente.** Conseguir que tus llamadas se firmen con atestacion de nivel A es la Capa 1 de un stack de cumplimiento y reputacion de 13 capas. La mayoria de los operadores de VICIdial se detienen en la Capa 1 y luego se preguntan por que sus numeros se marcan como "Posible Spam" a los seis dias de una campana.

---

## Que Hace Realmente STIR/SHAKEN (Y Que No)

STIR/SHAKEN es un framework de autenticacion criptografica de llamadas. Eso es todo. STIR (Secure Telephone Identity Revisited) define los estandares IETF (RFC 8224, 8225, 8226) para firmar digitalmente llamadas telefonicas. SHAKEN es el framework de despliegue norteamericano construido sobre esos estandares.

Cuando tu servidor VICIdial dispara un SIP INVITE a traves de Asterisk, esa llamada llega a tu proveedor de troncal SIP. Su Authentication Service (STI-AS) mira tres cosas: Te conocen (KYC)? Te asignaron este numero de telefono? La llamada se origino en su red?

**La concepcion erronea critica que les cuesta dinero a los operadores de VICIdial cada dia**: STIR/SHAKEN NO bloquea llamadas. NO etiqueta llamadas como spam. Autentica identidad. Punto.

Las decisiones reales de bloqueo y etiquetado? Esas las toman los motores de analytics de los carriers — Scam Shield de T-Mobile (impulsado por First Orion), ActiveArmor de AT&T (impulsado por Hiya), y Call Filter de Verizon (impulsado por TNS). Estos tres sistemas controlan la reputacion de llamadas para mas de 200 millones de suscriptores inalambricos de EE.UU., y actualizan sus modelos **cada seis minutos**.

---

## Los Tres Niveles de Atestacion y Por Que Solo Uno Importa

**Full Attestation (A):** Tu carrier verifico tu identidad, tu eres dueno del numero, y la llamada empezo en su red. Este es el unico nivel que mueve la aguja en entregabilidad.

**Partial Attestation (B):** Tu carrier te conoce, pero no puede confirmar que eres dueno del numero especifico. Los datos de la industria muestran que las llamadas con atestacion B y C son aproximadamente **tres veces mas propensas** a ser marcadas como robocalls.

**Gateway Attestation (C):** Tu carrier no sabe de donde vino la llamada. Las llamadas con nivel C estan funcionalmente muertas al llegar.

**La regla de oro:** Compra tus DIDs directamente de tu proveedor de troncal SIP. Si el carrier asigno el numero y estas enviando la llamada desde su red, eso es automaticamente nivel A.

---

## Tu Eleccion de Carrier Es Todo el Juego

VICIdial no implementa STIR/SHAKEN. Tu carrier lo hace.

Tu carrier debe tener:
- **Su propio token SPC y certificado.** Desde el 18 de septiembre de 2025, la FCC prohibio los certificados STIR/SHAKEN de terceros.
- **Estar en la Robocall Mitigation Database (RMD).** En agosto de 2025, la FCC removio 1,388 proveedores del RMD en un solo mes. Los call centers usando esos carriers vieron sus operaciones cesar en 48 horas.
- **Ser un CLEC con sus propios recursos de numeracion.** Para nivel A automatico.
- **Infraestructura construida para marcadores predictivos.** VICIdial necesita 2-3x mas canales concurrentes que agentes activos.

---

## Tus Configuraciones de VICIdial Estan Alimentando los Algoritmos de Spam

La conexion que nadie hace con suficiente explicitud: **la [configuracion de](/blog/es-vicidial-amd-guide/) tu campana de VICIdial genera patrones de llamadas especificos que los motores de analytics de carriers interpretan como firmas de spam.**

**AMD es el asesino silencioso de reputacion.** Cuando el modulo AMD de Asterisk detecta un saludo de buzon de voz y desconecta despues de 2-3 segundos de analisis de audio, genera volumenes masivos de llamadas de muy corta duracion. Las llamadas de [menos de](/blog/es-vicidial-setup-guide/) 30 segundos son la senal de spam mas fuerte en los algoritmos de carriers.

**Tu metodo de marcacion importa mas de lo que piensas.** RATIO a 3:1 con 10 agentes dispara 30 llamadas simultaneas. **ADAPT_AVERAGE** produce el patron de trafico mas suave.

**Configuraciones optimas de VICIdial para gestion de reputacion:**

```
Dial Method:           ADAPT_AVERAGE
Auto Dial Level:       1.0  (dejar que adapt lo suba)
Adaptive Drop %:       2.0
Drop Action:           MESSAGE
Drop Exten:            8304  (grabacion safe harbor)
Dial Timeout:          28
Available Only Tally:  Y
Calls per DID per day: 75   (bajo configuraciones de DID Rotation)
```

---

## El Stack Completo de Cumplimiento: 13 Capas, No Solo 1

**La Cadena de Entregabilidad:**
1. Atestacion STIR/SHAKEN de nivel A (lado del carrier)
2. Registro CNAM ($0.15-$2/numero)
3. Inscripcion en Free Caller Registry
4. Registro individual en motores de analytics (Hiya, First Orion, TNS)
5. Monitoreo continuo de reputacion de numeros
6. Branded calling (opcional pero cada vez mas importante)

**La Cadena de Cumplimiento Legal:**
7. Documentacion de consentimiento (TrustedForm o Jornaya)
8. Limpieza DNC Federal
9. Limpieza DNC Estatal (11-13 estados mantienen listas separadas)
10. Identificacion de celulares y cumplimiento TCPA
11. Consultas a la Reassigned Numbers Database
12. Limpieza de litigantes
13. Gestion de DNC interna y grabacion de llamadas (VICIdial maneja esto nativamente)

**Para un centro [de 50 puestos](/blog/es-vicidial-vs-gohighlevel/)**, el cumplimiento minimo viable corre **$3,000-$5,000/mes**. El stack completo recomendado llega a **$10,000-$18,000/mes**.

Eso suena caro hasta que calculas la alternativa. El no cumplimiento le cuesta a un centro de 50 puestos un estimado de **$143,000-$768,000 por mes** en conexiones perdidas, salarios de agentes desperdiciados, desgaste acelerado de DIDs, y costos de remediacion.

---

## El Camino de Implementacion en VICIdial: Tu Manual de 90 Dias

**Semana 1-2: Auditoria de Carrier**
- Verifica el estado RMD de tu carrier
- Confirma que tienen su propio token SPC y certificado
- Solicita confirmacion de nivel de atestacion para tus DIDs especificos

**Semana 3-4: Higiene de Numeros**
- Registra cada DID outbound en FreeCallerRegistry.com
- Configura CNAM para todos los numeros
- Ejecuta una verificacion de reputacion de referencia

**Semana 5-8: Configuracion de VICIdial**
- Cambia a ADAPT_AVERAGE si estas corriendo RATIO arriba de 2.0
- Establece porcentaje de abandono al 2% maximo
- Cambia Drop Action de HANGUP a MESSAGE o IN_GROUP
- Establece dial timeout a 28 segundos minimo
- Limita llamadas por DID por dia a 75 maximo

**Semana 9-12: Monitoreo y Optimizacion**
- Implementa rastreo diario de tasas de respuesta por carrier
- Rastrea ratio de llamadas de corta duracion (llamadas contestadas de 6 segundos o menos como % del total). Manten por debajo del 15%
- Monitorea codigos de respuesta SIP 603 — un pico repentino significa bloqueo activo

> **90 Dias Es Mucho Tiempo. Nosotros Podemos Hacerlo en 48 Horas.**
> ViciStack migra toda tu operacion a infraestructura optimizada y en cumplimiento de la noche a la manana. [Inicia Tu Migracion →](/free-audit/)

---

## La Linea Final

STIR/SHAKEN es el sistema operativo de la entregabilidad de llamadas moderna. No es toda la historia — es la Capa 1 de 13 — pero sin el, nada mas importa.

Los operadores que construyan el stack completo de cumplimiento y gestion de reputacion ahora veran tasas de respuesta, tasas de conversion, e ingresos por puesto que hacen que la inversion sea evidente por si misma. Los operadores que sigan tratando STIR/SHAKEN como problema de alguien mas se encontraran gastando mas dinero para llegar a menos personas hasta que la economia colapse por completo.

**Deja de adivinar. Empieza a construir.** [Contacta a ViciStack →](/free-audit/)

---

*ViciStack es la plataforma administrada de VICIdial que maneja cumplimiento STIR/SHAKEN, optimizacion de carriers, gestion de reputacion de numeros, y configuracion del marcador — para que tus llamadas realmente conecten.*

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/es-stir-shaken-vicidial-guide).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
