---
title: "Cómo Claude empezó a firmar lo que escribe (y por qué no es infalible)"
date: 2026-09-24 10:00:00 +0000
tags: [ia, deteccion-ia, anthropic, analisis]
---

Los detectores de IA eran ridículamente fáciles de engañar. Hasta ahora, al menos en parte.

Los sistemas tradicionales (Turnitin, GPTZero y similares) analizan métricas estadísticas del texto, sobre todo **perplejidad** (cuánto le "sorprende" a un modelo de lenguaje la secuencia de palabras elegida) y **burstiness** (cuánto varía la complejidad entre frases, algo que los humanos hacemos de forma irregular y los modelos, por defecto, menos). El problema de fondo es que esas métricas no demuestran nada: son una correlación estadística, no una prueba. Basta con reescribir algunas frases, cambiar la estructura de una oración o aplicar unas pocas modificaciones manuales para mover esos números lo suficiente como para que el detector falle, en cualquiera de las dos direcciones.

## El cambio de enfoque de Anthropic

Anthropic ha incorporado a los modelos Claude, actuales y anteriores, un watermark estadístico invisible basado en **SynthID-Text**, la técnica de marca de agua desarrollada originalmente por Google DeepMind. El despliegue llegó en el verano de 2026, con el sistema activo desde principios de agosto y una explicación técnica más detallada publicada por Anthropic semanas después. A diferencia de los detectores externos, este watermark no analiza el texto desde fuera para adivinar si "parece" generado por IA: la marca la introduce el propio modelo mientras escribe.

La idea, simplificada, es la siguiente: en cada paso de generación, un modelo de lenguaje no elige una única palabra siguiente, sino que calcula una distribución de probabilidad sobre miles de palabras candidatas y muestrea una de ellas. SynthID-Text interviene justo ahí: usando una clave pseudoaleatoria derivada del contexto ya generado, sesga ligeramente esa elección hacia un subconjunto concreto de palabras candidatas, sin cambiar de forma perceptible la calidad ni la naturalidad del texto. Para quien lee, el resultado es indistinguible de un texto sin marcar. Pero quien conoce la clave puede analizar, después, si la secuencia de palabras elegidas encaja con ese sesgo estadístico mejor de lo que encajaría por azar. Cuanto más largo el texto, más fiable es la señal.

Es un cambio de paradigma real: pasamos de intentar adivinar si un texto "parece" de IA, un juicio subjetivo y fácil de manipular, a buscar una señal que el propio modelo introdujo de forma deliberada y que el autor del texto no puede ver ni quitar sin reescribir el contenido.

## Por qué, aun así, no es infalible

El sistema funciona bien como escudo frente al caso más simple y más común: copiar y pegar el texto generado tal cual, o con ediciones ligeras. Ahí la marca sobrevive.

Pero se rompe, y se rompe por completo, en varios escenarios:

- **Parafraseo profundo**, reescribiendo el contenido con tus propias palabras manteniendo solo las ideas.
- **Traducción** a otro idioma y vuelta, o directamente publicar en un idioma distinto al generado.
- **Pasarlo por otro modelo competidor** para que lo reescriba con otro estilo, algo cada vez más trivial de automatizar.

En cualquiera de esos casos, la secuencia exacta de palabras cambia lo suficiente como para que el sesgo estadístico original desaparezca. El watermark no es una firma criptográfica inquebrantable: es una señal estadística que depende de que el texto, a nivel de elección de palabras, se parezca razonablemente al que generó el modelo.

Hay además dos matices importantes, más allá de los técnicos:

- **No es prueba definitiva.** Incluso cuando la señal se detecta, el resultado es una probabilidad de que el texto proceda de Claude, no una certeza binaria. Se puede usar como indicio, no como veredicto.
- **La detección no es universal.** La herramienta que permitiría a cualquiera, fuera de Anthropic, comprobar si un texto lleva esta marca se ha anunciado pero no está disponible de forma abierta todavía; de momento, verificar la marca depende en buena medida de la propia Anthropic.

La medida tampoco ha sido recibida sin fricción: ha generado críticas de quienes consideran que una marca de agua de este tipo es poco efectiva en la práctica (precisamente por lo fácil que es romperla con un parafraseo) y de quienes la ven como un problema de privacidad para las personas que usan el modelo, más que como una solución de detección robusta. El contexto regulatorio tampoco es casual: buena parte de la presión para incorporar este tipo de marcado viene de la AI Act europea.

## El balance

¿Estamos entrando en una nueva era de detección de contenido generado por IA? En parte sí, y en parte no tanto como suena. Sí, porque la lógica ha cambiado de raíz: ya no se trata de adivinar a partir de un estilo, sino de verificar una señal introducida deliberadamente, lo que en principio da muchísima menos libertad al falso positivo contra un texto humano que simplemente "escribe raro". No, porque la señal sigue siendo frágil frente a cualquier reescritura mínimamente seria, y porque la capacidad real de verificarla sigue, por ahora, concentrada en manos de quien puso la marca.

Como paralelismo con lo que ya hemos visto en criptografía: un watermark estadístico débil se parece más a una ofuscación que a una protección real, de la misma manera que un generador de números "aleatorios" basado en una fórmula predecible [no es lo mismo que uno criptográficamente seguro]({{ '/2026/10/math-random-no-es-aleatorio/' | relative_url }}). La diferencia entre "difícil de romper en el caso común" y "imposible de romper" importa, y en seguridad casi siempre se paga cara la confusión entre ambas.
