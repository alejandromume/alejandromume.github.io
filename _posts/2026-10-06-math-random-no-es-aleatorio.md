---
title: "Tu generador de tokens \"seguros\" es totalmente predecible"
date: 2026-10-06 10:00:00 +0000
tags: [criptografia, random, javascript, analisis]
---

Muchos desarrolladores usan `Math.random()` para generar tokens de sesión, códigos de descuento o contraseñas temporales. Funciona rápido, no da errores y, visualmente, el resultado parece entrópico. Es justo ese aspecto el que engaña: no estás generando ruido, estás evaluando una fórmula. Y toda fórmula, por complicada que parezca, se puede despejar.

## Por qué no es aleatorio

Los navegadores y motores como Node.js (sobre V8) usan, desde hace años, un algoritmo llamado **xorshift128+**. Su funcionamiento interno es sencillo: parte de un estado de 128 bits (dos números de 64 bits) y en cada llamada lo transforma con una serie de operaciones XOR y desplazamientos de bits, para luego devolver un fragmento de ese estado como el número "aleatorio".

Lo importante no es lo que hace, sino lo que *es*: una recurrencia lineal sobre GF(2), el cuerpo binario donde XOR hace de suma. Eso tiene una consecuencia muy concreta. Un sistema de ecuaciones lineales, por muchas variables que tenga, se resuelve. No hace falta fuerza bruta ni GPUs: con eliminación de Gauss-Jordan, la de toda la vida, basta.

Si un atacante consigue 3 o 4 salidas consecutivas de `Math.random()` de tu aplicación, no está "robando tokens": está recolectando ecuaciones. Con esas pocas muestras, cualquier solver como **Z3** (el SMT solver de Microsoft Research) reconstruye el estado interno completo del generador en milisegundos. A partir de ahí, no solo sabe los tokens que ya emitiste: predice con 100% de precisión los próximos miles que vas a emitir, sin necesidad de volver a observar nada.

Esto no es una debilidad teórica de laboratorio. Es exactamente el mismo patrón que [documenté al auditar un router TP-Link]({{ '/2026/10/tplink-xgb430v-pro-cifrado-inseguro/' | relative_url }}): la clave AES de sesión del panel de administración se construía con `Date.now()` más tres dígitos de `Math.random()`. Ahí ni siquiera hacía falta resolver un sistema de ecuaciones completo (aunque se podría): bastaba con que la marca de tiempo, que viaja en claro por la red, redujera el espacio de búsqueda a unas pocas decenas de millones de combinaciones, recorribles en minutos con un portátil normal. Cuando el "azar" de tu clave depende de una fuente predecible y de un generador lineal, el ataque deja de ser una curiosidad académica y se convierte en una forma de robar contraseñas de verdad.

## ¿La solución? No hace falta instalar nada externo

Los entornos de desarrollo modernos ya separan, de forma nativa, los dos mundos: el generador rápido y determinista para cosas que no importan (animaciones, barajar una lista en una demo), y el generador criptográficamente seguro para todo lo que sí importa.

- En JavaScript de navegador: `window.crypto.getRandomValues()`.
- En Python: el módulo `secrets`, no `random`.
- En Node.js: `crypto.randomBytes()` / `crypto.randomUUID()`.
- A nivel de sistema operativo en Linux: la llamada `getrandom()`.

Esa última es la que conviene entender un poco mejor, porque es la base de casi todo lo demás. `getrandom()` recoge entropía de fuentes físicas: ruido térmico del hardware, variaciones de temporización del procesador, interrupciones del sistema. Esa entropía física alimenta un generador criptográfico (en Linux, basado en ChaCha20, el mismo cifrado de flujo que usa buena parte del tráfico TLS moderno) que expande esos pocos bits de ruido real en un flujo de salida indistinguible del azar perfecto, a efectos prácticos.

Al mezclar datos físicos del mundo real con el algoritmo, la ecuación lineal que un atacante podría intentar resolver deja de existir como tal: ya no hay una fórmula determinista que relacione una salida con la siguiente. Predecir el próximo bit pasa de ser "álgebra" a ser, literalmente, imposible con la información disponible.

## Una curiosidad final

Si nos ponemos estrictos, en software no existe la aleatoriedad pura: incluso un CSPRNG es, en el fondo, un algoritmo determinista alimentado por una semilla de entropía física. Un **UUIDv4** puede ser, a efectos prácticos, imposible de adivinar (hay más combinaciones posibles que granos de arena en el planeta), pero sigue dependiendo de esa fuente criptográficamente segura para generarse.

La aleatoriedad fundamental, la que no depende de ningún algoritmo, solo aparece en el mundo físico: en el ruido térmico, en la desintegración radiactiva, en los procesos cuánticos. Todo lo demás, por bien diseñado que esté, es una aproximación extraordinariamente buena. La diferencia entre un generador seguro y uno inseguro no es si "parece" aleatorio. Es si alguien, con las herramientas adecuadas, puede escribir la ecuación que lo describe.
