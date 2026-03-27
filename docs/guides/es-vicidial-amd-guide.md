# Configuracion de AMD en VICIdial: La Unica Guia Que No Te Hace Perder el Tiempo

**[La guia completa](/blog/es-open-source-call-center-software/) y sin rodeos sobre Answering [Machine Detection](/blog/vicidial-amd-vs-ai-amd/) en VICIdial — desde los internos del algoritmo hasta el codigo del dialplan y el evento de extincion de iOS 26 del que nadie habla.**

---

Si alguna vez viste a tus agentes sentados escuchando silencio, escuchaste un saludo de buzon de voz reproducirse en un auricular vacio, o te preguntaste [por que tu](/blog/es-vicidial-vs-gohighlevel/) AMD parece funcionar genial el martes y como basura el jueves — felicitaciones, experimentaste la funcion mas frustrante del marcado outbound.

El AMD de VICIdial ha sido debatido en foros desde 2008. El mismo Matt Florell — el tipo que *construyo* VICIdial — le dijo a una audiencia de Astricon en 2012 que su equipo "casi nunca sugiere usar" el AMD estandar. William Conley en PoundTeam rutinariamente les dice a los clientes que lo salteen por completo. Y sin embargo, cuando estas empujando 600,000 marcaciones al mes a traves de 100 centros de contacto, dejar que el 50% de las llamadas contestadas desperdicien el tiempo del agente en saludos de buzon de voz no es exactamente una estrategia de negocio.

Asi que aqui esta el trato. Despues de analizar cada hilo del foro, cada archivo fuente de Asterisk, cada interaccion con carriers, y cada despliegue real que pudimos encontrar — esta es [la guia](/blog/es-stir-shaken-vicidial-guide/). No la guia de "esto es lo que hacen las configuraciones." La guia que te dice *que realmente pasa* cuando AMD corre, por que se rompe, y que hacer al respecto en 2026.

---

## Como Funciona Realmente AMD (Es Mas Tonto de Lo Que Crees)

Matemos el misterio. La aplicacion AMD en Asterisk (`app_amd.c`) es una maquina de estados finitos de dos estados que procesa frames de audio de 20 milisegundos a una tasa de muestreo de 8kHz. Eso es todo. No hay [machine learning](/blog/vicidial-lead-scoring-ml/), no hay red neuronal, no hay AI. Es un detector de silencio con un contador de palabras atornillado encima.

Asi es la maquina de estados: empieza en un estado de **SILENCE**. Cuando detecta energia de audio por encima del `silenceThreshold` (por defecto 256), transiciona al estado **WORD**. Cuando el audio cae por debajo del umbral, vuelve a **SILENCE**. El algoritmo cuenta estas transiciones y mide sus duraciones. Luego toma una decision terminal basada en cual umbral se alcanza primero.

Cinco posibles estados terminales, en orden de prioridad:

**INITIALSILENCE** — No se detecto habla dentro de los `initialSilence` milisegundos. La linea se conecto pero nadie esta en casa. Esto dispara una clasificacion MACHINE porque los humanos reales tipicamente dicen "hola" dentro de 2 segundos.

**MAXWORDS** — El conteo de palabras excedio `maximumNumberOfWords`. Si el algoritmo cuenta 4+ segmentos de voz distintos, asume que es un saludo de buzon de voz listando opciones. MACHINE.

**LONGGREETING** — La duracion total acumulada de voz excedio los `greeting` milisegundos. Patrones de habla largos y continuos coinciden con saludos de buzon de voz. MACHINE.

**MAXWORDLENGTH** — Un solo segmento de voz continuo excedio `maximumWordLength` (no expuesto en los parametros por defecto de VICIdial pero existe en el codigo). Esto atrapa esos saludos de buzon de voz interminables. MACHINE.

**HUMAN** — Ninguno de los anteriores se disparo, y se detecto silencio despues del saludo. El algoritmo escucho una expresion corta ("Hola?") seguida de silencio (la persona esperando una respuesta). HUMAN.

Si *ninguna* de estas condiciones se cumple dentro de `totalAnalysisTime` (por defecto 5000ms), el resultado es **NOTSURE** con causa **TOOLONG**. Y aqui es donde empieza la diversion.

> **Tu AMD Esta Corriendo un Algoritmo de 2008 en 2026.**
> El AMD Crusher potenciado por AI de ViciStack alcanza 92-96% de precision. El mismo VICIdial, deteccion dramaticamente mejor. [Ver la Diferencia →](/free-audit/)

### Los Parametros por Defecto vs. Lo Que VICIdial Realmente Usa

| Parametro | Default Asterisk | Tipico VICIdial | Que Hace |
|---|---|---|---|
| initialSilence | 2500ms | 2000ms | Max silencio antes del habla |
| greeting | 1500ms | 2000ms | Max duracion acumulada de voz |
| afterGreetingSilence | 800ms | 1000ms | Silencio despues del saludo = HUMAN |
| totalAnalysisTime | 5000ms | 5000ms | Timeout fijo para analisis |
| minimumWordLength | 100ms | 120ms | Min segmento de voz para contar como "palabra" |
| betweenWordsSilence | 50ms | 50ms | Min silencio entre palabras |
| maximumNumberOfWords | 2 | 4 | Max palabras antes de MACHINE |
| silenceThreshold | 256 | 256 | Umbral de energia de audio |

VICIdial aumenta `greeting` a 2000ms (atrapando saludos de buzon de voz mas largos) y `maximumNumberOfWords` a 4 (reduciendo falsos positivos en humanos que dicen "Hola, habla con Jason" — eso son 4 palabras, lo cual activaria el default de 2).

Estos valores por defecto vinieron de una calibracion extensiva de la comunidad. Como noto un usuario del foro, ejecutaron mas de 10,000 llamadas para llegar a los valores por defecto actuales. Pero aqui esta la trampa que todos pasan por alto: estos parametros fueron calibrados en patrones de llamadas de la era Asterisk 1.4/1.8. Los saludos de buzon de voz en 2026 son diferentes. El comportamiento de los carriers es diferente. Todo el panorama de audio ha cambiado.

---

## El Dialplan: Que Realmente Pasa Cuando Una Llamada Se Conecta

Cuando VICIdial marca un numero con AMD habilitado, la llamada se enruta a la extension 8369 (o 8373 para campanas de survey/press-1). Asi se ve el dialplan en Asterisk 13.20+ — anotado con lo que realmente sucede en cada paso:

```
exten => 8369,1,AGI(agi://127.0.0.1:4577/call_log)
exten => 8369,n,Playback(sip-silence)
exten => 8369,n,AMD(2000,2000,1000,5000,120,50,4,256)
exten => 8369,n,AGI(VD_amd.agi,${EXTEN})
exten => 8369,n,AGI(agi-VDAD_ALL_outbound.agi,NORMAL-----LB-----${CONNECTEDLINE(name)})
exten => 8369,n,Hangup()
```

**Prioridad 1 — `call_log` AGI:** Llamada FastAGI en el puerto 4577. Registra el inicio de la llamada en `vicidial_log`, establece variables del canal como CAMPCUST y metadata de campana. En Asterisk 13+, esto *debe* ejecutarse antes de cualquier operacion de audio. Si lo tienes despues de `sip-silence` (que era el orden de Asterisk 11), el registro de llamadas se rompera silenciosamente. Este es el error numero uno que cometen los operadores al actualizar de Asterisk 11 a 13.

**Prioridad 2 — `Playback(sip-silence)`:** Esta es la linea mas importante del dialplan de AMD y la que nadie entiende. Reproduce un archivo de audio silencioso (tipicamente 250ms de silencio en formato GSM) para "preparar" el canal de audio. Sin esta linea, mas del 40% de tus llamadas retornaran NOAUDIODATA — significando que AMD recibio literalmente cero frames de audio para analizar. Algunos operadores con carriers problematicos ejecutan esta linea *dos veces*. Si ves altas tasas de NOAUDIODATA, duplicar este Playback es tu primer movimiento.

**Prioridad 3 — `AMD()`:** El analisis real de AMD. Esta linea *bloquea la ejecucion del dialplan* por hasta 5 segundos (el parametro `totalAnalysisTime`). Durante este tiempo, la persona del otro lado escucha silencio. AMD establece dos variables del canal: `AMDSTATUS` (HUMAN, MACHINE, NOTSURE, o HANGUP) y `AMDCAUSE` (la razon especifica).

**Prioridad 4 — `VD_amd.agi`:** El cerebro de enrutamiento. Este script Perl lee `AMDSTATUS` y `AMDCAUSE`, consulta la configuracion `cpd_amd_action` de la campana, verifica el Settings Container `amd_agent_route_options` si esta habilitado, y enruta en consecuencia. Las llamadas MACHINE se disposicionan (AA), se envian a voicemail drop (AM→AL), o se transfieren a un ingroup/[call menu](/blog/vicidial-ivr-setup/). Las llamadas HUMAN continuan a la siguiente linea.

**Prioridad 5 — `agi-VDAD_ALL_outbound.agi`:** Enruta la llamada clasificada como humana a un agente disponible via asignacion de servidor balanceada por carga. Aqui es donde la persona que ha estado sentada en silencio por 2-5 segundos finalmente escucha una voz humana. Si no ha colgado ya.

Para comparacion, la extension 8368 (sin AMD) salta las lineas AMD() y VD_amd.agi por completo — las llamadas van directamente de sip-silence al enrutamiento de agentes.

---

## El Problema NOTSURE (Y Por Que AMD Agent Route Options Lo Cambio Todo)

Aqui esta el secreto sucio del AMD de VICIdial: el comportamiento por defecto enruta tanto las llamadas HUMAN *como* NOTSURE a los agentes. Eso significa que cuando AMD no puede tomar una decision en 5 segundos (TOOLONG), la llamada va a un agente de todas formas — despues de que el cliente ya soporto mas de 5 segundos de silencio. Cuando AMD no recibe datos de audio del carrier (NOAUDIODATA), tambien enruta a los agentes — quienes entonces reciben un canal silencioso sin nadie al otro lado.

[En la](/blog/es-vicidial-cloud-deployment/) revision SVN 3200+, VICIdial agrego **AMD Agent Route Options** — un Settings Container que te permite definir exactamente que combinaciones de `AMDSTATUS,AMDCAUSE` se enrutan a los agentes. Esta es genuinamente la funcion de AMD mas importante agregada en anos, y la mayoria de los operadores no saben que existe.

El Settings Container vive a nivel de campana. Cada linea es un par separado por comas:

**Configuracion conservadora (solo humanos definitivos):**
```
HUMAN,HUMAN
```

**Configuracion moderada (humanos + llamadas ambiguas, bloquear canales muertos):**
```
HUMAN,HUMAN
NOTSURE,TOOLONG
```

**Consenso minimo de la comunidad — bloquear NOAUDIODATA como minimo:**
```
HUMAN,HUMAN
NOTSURE,TOOLONG
NOTSURE,INITIALSILENCE
NOTSURE,MAXWORDS
```

El tutorial de Striker24x7 recomienda especificamente bloquear NOAUDIODATA, y ademas sugiere bloquear TOOLONG tambien. La logica es solida: las llamadas TOOLONG ya han consumido mas de 5 segundos de silencio. Enrutarlas a agentes crea una experiencia terrible para el cliente. Si, perderas algunos humanos legitimos que resultan tener patrones de saludo inusuales. Pero esos humanos ya estaban irritados por 5 segundos de silencio de todas formas.

> **Cuantos Humanos Vivos Estas Colgandoles?**
> Si no has configurado AMD Agent Route Options, la respuesta es "demasiados." [Obtener Tu Auditoria de AMD Gratuita →](/free-audit/)

---

## La Guerra de Versiones de Asterisk: Elige Tu Veneno

Esta es la parte que nadie quiere escuchar. La version de Asterisk que estas corriendo determina fundamentalmente que tan bien funciona AMD, y no hay version que haga todo bien.

**Asterisk 11 (la era dorada, ahora EOL):** AMD funciona al 70-80% de precision con los defaults calibrados por la comunidad. Multiples operadores confirmaron: "100 llamadas, 1 o 2 contestadoras llegan a los agentes." El problema? Asterisk 11 esta en fin de vida, tiene vulnerabilidades [de seguridad](/blog/es-vicidial-security-hardening/) conocidas, no soporta WebRTC, y — segun Kumba — "se va a crashear/colgar/convertir en zombie aleatoriamente 3-4 veces al dia en un sistema de alta carga."

**Asterisk 13.0-13.17 (los anos rotos):** AMD esta *fundamentalmente no funcional*. La documentacion ASTERISK_13.txt de VICIdial explicitamente establece: "NOTA: Asterisk 13.20 y superior tiene una version funcional de la app AMD, a diferencia de versiones anteriores de Asterisk 13." GOautodial 4 se distribuyo con Asterisk 13.17.2 y AMD estaba roto de fabrica. Si estas corriendo cualquier cosa por debajo de 13.20, tu AMD esta retornando TOOLONG en la mayoria de las llamadas. Punto.

**Asterisk 13.20+ (el compromiso):** AMD funciona de nuevo pero notablemente peor que Asterisk 11. Usuario del foro bbakirtas: "100 llamadas 1 o 2 AA con Asterisk 11, mismas configuraciones en Asterisk 13 = 10 AA." Otro operador: "Cuando vuelvo a Asterisk 11 se reduce casi a la mitad." La refactorizacion del stack SIP en Asterisk 13 introdujo cambios de timing que afectan el manejo del stream de audio de AMD.

**Asterisk 18 (la actual):** La version para la que VICIdial esta desarrollando activamente. Viene con soporte AudioSocket para integracion de AMD externo, PJSIP reemplazando chan_sip, y ConfBridge reemplazando MeetMe. El algoritmo AMD en si no ha cambiado — mismo codigo `app_amd.c` — pero el manejo SIP mejorado y el stack de audio moderno pueden proporcionar mejoras marginales. Mas importante, AudioSocket permite transmitir audio de llamadas a servicios de AMD con AI externos, que es donde esta el verdadero futuro.

**El enfoque hibrido (lo que hacen los operadores inteligentes):** Corre AMD en servidores de telefonia Asterisk 11 que solo manejan marcacion y analisis AMD. Corre agentes en servidores Asterisk 13/18 WebRTC para la interfaz moderna del agente. La [arquitectura de cluster](/blog/vicidial-cluster-guide/) de VICIdial soporta esto nativamente.

> **Todavia Corriendo AMD en Asterisk 13.17? Uy.**
> ViciStack despliega en Asterisk 18 con cada optimizacion de AMD de esta guia incorporada. [Hablemos →](/free-audit/)

---

## La Funcion AMDMINLEN: Salvando Tus DIDs (SVN 3873)

En septiembre de 2024, Matt Florell agrego una funcion pequena pero critica a `VD_amd.agi` que aborda uno de los efectos secundarios mas costosos del AMD: penalizaciones de Short Duration Percent (SDP) del carrier.

Cuando AMD identifica correctamente un buzon de voz y cuelga, la duracion de la llamada es frecuentemente de 2-4 segundos. Los carriers rastrean SDP como metrica de calidad. Demasiadas llamadas de corta duracion y tu carrier comienza a penalizarte — enrutamiento degradado, tarifas por minuto mas altas, o terminacion directa. Tus [DIDs se marcan](/blog/vicidial-did-management/), su reputacion se desploma, y tus tasas de conexion se derrumban.

La solucion es una sola variable de dialplan establecida *antes* de la linea Dial en tu entrada de carrier (no en `extensions.conf`):

```
exten => _8567.,1,AGI(agi://127.0.0.1:4577/call_log)
exten => _8567.,n,Set(__AMDMINLEN=7)
exten => _8567.,n,Dial(SIP/carrier/${EXTEN:4},,tToR)
exten => _8567.,n,Hangup
```

`AMDMINLEN=7` asegura que `VD_amd.agi` mantenga la llamada viva por un minimo de 7 segundos antes de colgar, incluso si AMD ya determino que es una maquina. La llamada no va a un agente — simplemente se mantiene conectada silenciosamente hasta que se alcanza el minimo. Esto elimina la penalizacion de corta duracion sin afectar la productividad del agente.

Si estas corriendo AMD y *no* usando AMDMINLEN, estas activamente quemando tus DIDs. Esto no es opcional en 2026.

> **Tus DIDs Estan Muriendo y AMD Esta Jalando el Gatillo.**
> AMDMINLEN es no negociable en 2026. ViciStack lo configura automaticamente en cada carrier. [Protege Tus Numeros →](/free-audit/)

---

## iOS 26: El Evento de Extincion del AMD

Aqui esta la parte que deberia aterrorizar a cada operador outbound que lee esto.

En septiembre de 2025, Apple lanzo iOS 26 con filtrado de llamadas a nivel de sistema. Cuando esta habilitado, las llamadas de numeros desconocidos son contestadas automaticamente por iOS mismo, que reproduce un mensaje: "Por favor diga su nombre y el motivo de su llamada." La respuesta del llamante se transcribe en vivo en la pantalla de bloqueo. El dueno del iPhone — que nunca lo escucho sonar — lee la transcripcion y decide si contestar.

Piensa en lo que esto significa para AMD.

La llamada se conecta. El carrier envia un 200 OK. Asterisk ve una llamada contestada. AMD empieza a analizar. Pero en lugar de escuchar a un humano decir "Hola?" o un saludo de buzon de voz, AMD escucha el prompt de filtrado de Apple — que es una voz femenina pregrabada diciendo una oracion completa. Para la maquina de estados finitos de dos estados de AMD, esto se ve exactamente como un saludo de buzon de voz: un segmento de voz largo, potencialmente multiples palabras, seguido de silencio mientras el sistema de filtrado espera que el llamante hable.

AMD lo clasifica como MACHINE. Cuelga. El dueno del iPhone nunca supo que llamaste.

Con iPhone manteniendo 59% de participacion en el mercado de EE.UU. y la adopcion de iOS alcanzando historicamente mas del 70% en 6 meses, esto no es un problema futuro. Es un problema de *ahora*. El AMD tradicional — el tonto detector de silencio que hemos estado cuidando desde 2008 — no puede distinguir entre un prompt de filtrado de iOS, un saludo de buzon de voz, y un humano diciendo hola.

> **iOS 26 Lo Cambio Todo. Tu AMD Aun No Lo Sabe.**
> Nuestro AMD con AI distingue humanos, buzones de voz, Y prompts de filtrado de iOS. El AMD estandar no puede. [Ver Nuestra Precision de AMD →](/pricing/)

---

## Cuando Usar AMD (Y Cuando Desactivarlo)

Despues de todo lo anterior, aqui esta la evaluacion honesta.

**AMD tiene sentido cuando:**
- Tu tasa de buzon de voz excede el 40% (comun en B2C residencial)
- Tienes 20+ agentes y las matematicas de productividad funcionan
- Puedes tolerar 20-30% de tasa de falsos positivos (humanos legitimos colgados)
- Estas corriendo AMDMINLEN para proteger DIDs
- Tienes AMD Agent Route Options configurados para bloquear NOAUDIODATA y TOOLONG
- Tu tasa de abandono de la campana tiene espacio por debajo del 3% de la FCC

**AMD no tiene sentido cuando:**
- Estas llamando a listas con muchos celulares donde el filtrado de iOS 26 es prevalente
- Tus agentes pueden identificar buzones de voz [en menos de](/blog/es-vicidial-setup-guide/) 1 segundo (entrenalos a disposicionar rapidamente)
- Estas corriendo menos de 10 agentes (las colgadas por silencio duelen mas que el tiempo de buzon de voz)
- Tu carrier esta generando alto FAS (carrier contesta antes que el humano)
- Estas en un vertical regulado donde la regla de abandono del 3% se esta aplicando activamente

**La alternativa del dial timeout:** Establece el timeout de marcacion de tu campana a 22-24 segundos. La mayoria de los sistemas de buzon de voz contestan a los 24-25 segundos (4-5 timbres). Al colgar antes de que el buzon conteste, evitas todo el problema de AMD. Perderas humanos que tardan mucho en contestar, pero eliminas falsos positivos, silencio, y riesgo de abandono de la FCC.

---

## Las Matematicas Economicas: Cuando AMD Se Paga Solo

Hagamos las cuentas que nadie mas publica.

**Supuestos:** 50 agentes, $14/hora totalmente cargado, turno de 8 horas. Sin AMD, los agentes manejan ~15 llamadas/hora, 50% son buzones de voz que requieren ~20 segundos cada uno para identificar y disposicionar. Eso son 7.5 buzones x 20 segundos = 150 segundos por hora por agente desperdiciados en buzones. A traves de 50 agentes en 8 horas: **16,667 minutos de tiempo de agente desperdiciado por dia.** A $14/hora, eso es aproximadamente **$3,889/dia en nomina desperdiciada.**

**Con AMD al 80% de precision:** AMD atrapa el 80% de esos buzones (6 de 7.5 por hora). El tiempo de buzon del agente baja a 30 segundos/hora. Ahorro: ~**$3,111/dia.** Pero AMD agrega 2-3 segundos de silencio a *cada* llamada contestada por humano. Con una tasa de colgado del ~3% del cliente debido al silencio (conservador), pierdes aproximadamente 18 conversaciones en vivo por dia. Si tu tasa de conversion es 5% y el ingreso promedio por conversion es $500, eso son **$450/dia en ingresos perdidos.**

**Beneficio neto diario: ~$2,661.** Eso es aproximadamente $66,500/mes — significativo a escala. Pero esa tasa de falsos positivos del 20-30% significa que AMD tambien esta colgando a 90-135 *humanos reales* por dia que contestaron el telefono y fueron desconectados. Esos humanos no te van a llamar de vuelta. Estan presentando quejas.

> **Las Matematicas Dicen Que Estas Perdiendo $2,600/Dia.**
> Los clientes de ViciStack ven 7-16% mas conversaciones en vivo por hora. Mismos agentes, mismos leads, mas ingresos. [Demostrarlo Gratis →](/free-audit/)

---

## El Checklist de Configuracion: Si Vas a Usar AMD, Hazlo Bien

Aqui esta la implementacion practica, sintetizada de todo lo anterior:

**Version de Asterisk:** 13.38.3-vici minimo. Idealmente 18.26.0 para futura integracion de AMD con AI. Nunca corras 13.0-13.17.

**Orden del dialplan (Asterisk 13+):** `call_log` → `sip-silence` → `AMD()` → `VD_amd.agi` → `agi-VDAD_ALL_outbound.agi`. Si actualizaste desde Asterisk 11 y no cambiaste el orden de call_log/sip-silence, arreglalo ahora.

**AMDMINLEN:** Agrega `Set(__AMDMINLEN=7)` antes de la linea Dial en la entrada del dialplan de tu carrier. No negociable.

**AMD Agent Route Options:** Habilita y crea un Settings Container bloqueando como minimo NOTSURE-NOAUDIODATA. Muy recomendado tambien bloquear NOTSURE-TOOLONG.

**WaitForSilence para voicemail drops:** El default `2000,2,30` funciona para la mayoria de los carriers. Si los voicemail drops se cortan temprano o no se reproducen despues del beep, aumenta la deteccion de silencio a 3000ms o agrega una tercera iteracion.

**Codec:** Usa ulaw (G.711) en trunks de carrier siempre que sea posible. G.729 requiere transcodificacion intensiva de CPU a slin para el analisis AMD, y el proceso de transcodificacion puede introducir artefactos que degradan la precision de AMD. Como William Conley lo dijo sin rodeos: "el codec por defecto es ulaw. el codec mas preferido es ulaw. el mejor codec es ulaw."

**Monitoreo:** Habilita `$AMD_LOG` en `VD_amd.agi`. Revisa el AMD Log Report en Admin Utilities semanalmente. Rastrea tu distribucion de AMDSTATUS: apunta a menos del 5% de NOTSURE-TOOLONG y menos del 2% de NOTSURE-NOAUDIODATA. Si NOAUDIODATA excede el 5%, tienes un problema de carrier — no un problema de AMD.

**Dimensionamiento de CPU:** AMD realiza procesamiento DSP en cada canal activo simultaneamente. En un servidor de telefonia dedicado, planifica aproximadamente 100-150 canales AMD concurrentes por CPU moderno de cuatro nucleos. Monitorea degradacion no lineal — la precision de AMD se desploma cuando la utilizacion de CPU excede 70-80% porque el procesamiento de frames empieza a perder sus ventanas de timing de 20ms.

---

## La Linea Final

El AMD estandar de VICIdial es un algoritmo de la era 2008 corriendo en un mundo de 2026. Nunca fue genial — Matt Florell lo dijo el mismo — y el panorama ha cambiado dramaticamente en su contra. El filtrado de llamadas de iOS, el [filtrado de carriers por STIR/SHAKEN](/blog/stir-shaken-vicidial-guide/), y los sistemas de buzon de voz modernos han cambiado los patrones de audio que AMD estaba calibrado para detectar.

Pero para operaciones outbound de alto volumen donde 40-60% de las llamadas contestadas son buzones de voz, incluso un AMD con 70-80% de precision proporciona valor economico significativo. La clave es implementarlo correctamente (AMDMINLEN, Agent Route Options, orden adecuado del dialplan, higiene de codecs) y saber cuando desactivarlo.

El futuro es AMD potenciado por AI — clasificacion sub-segundo usando streaming de audio en tiempo real, capaz de distinguir entre humanos, saludos de buzon de voz, prompts de filtrado de iOS, y IVRs de carriers. El soporte AudioSocket de Asterisk 18 hace esto arquitectonicamente factible hoy. Los operadores que se muevan al AMD con AI primero tendran una ventaja competitiva medible en tasas de conexion y productividad de agentes.

Hasta entonces, tienes un detector de silencio con un contador de palabras y un monton de configuracion para hacer bien.

*En ViciStack, hemos ajustado AMD a traves de mas de 100 centros de contacto procesando mas de 600,000 marcaciones mensuales. Nuestra plataforma optimiza cada parametro cubierto en esta guia automaticamente — desde configuracion de AMDMINLEN hasta ajuste de AMD especifico por carrier hasta monitoreo de AMDSTATUS en tiempo real. Si tu precision de AMD te esta costando dinero, [deberiamos hablar](/free-audit/).*

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/es-vicidial-amd-guide).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
