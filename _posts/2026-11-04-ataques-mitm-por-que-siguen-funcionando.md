---
title: "Qué es un ataque man-in-the-middle y por qué en 2026 sigue funcionando"
date: 2026-11-04 10:00:00 +0000
tags: [mitm, redes, criptografia, analisis]
---

MITM significa "man in the middle", hombre en el medio. La idea de fondo es vieja y no tiene nada de sofisticado: alguien se coloca físicamente entre dos partes que creen estar hablando directamente, y desde ahí escucha, o incluso modifica, lo que pasa por su posición. Lo raro, en 2026, no es que este ataque exista. Es que siga funcionando contra dispositivos nuevos, de fabricantes grandes, que deberían tenerlo resuelto desde hace una década.

## Cómo se llega a ese "en el medio"

Para interceptar tráfico primero hay que colocarse en su camino. En una red local, la forma más común es el ARP spoofing. Cuando tu portátil quiere hablar con el router, pregunta a toda la red "¿quién tiene esta dirección IP?", y el dispositivo correspondiente responde con su dirección física (MAC). Ese protocolo, ARP, no verifica nada: no hay firma, no hay credencial, no hay forma de comprobar que la respuesta viene de quien dice venir. Un atacante en la misma red puede simplemente responder él primero, afirmando "esa IP soy yo", y tu equipo se lo cree sin comprobar nada más. A partir de ahí, todo tu tráfico pasa primero por el atacante antes de llegar a su destino real, y él decide si lo deja pasar tal cual, lo copia, o lo modifica.

Otra vía habitual es un punto de acceso Wi-Fi falso: una red abierta o con el mismo nombre que una legítima, que la víctima conecta sin pensarlo. En ese caso el atacante no necesita engañar a ningún protocolo, porque la víctima se conecta a él voluntariamente, convencida de que es la red de siempre.

Hecho esto, el atacante no necesita "romper" ningún cifrado complicado. Solo necesita que lo que pasa por su posición no esté cifrado, o que lo esté con algo tan débil que da igual.

## Por qué TLS existe (y por qué sigue siendo la única solución real)

TLS, el protocolo detrás del candado y el `https://` del navegador, se diseñó exactamente para esto. No solo cifra el contenido para que quien lo intercepte no pueda leerlo: también autentica el extremo, es decir, demuestra criptográficamente que el servidor con el que hablas es quien dice ser, mediante un certificado firmado por una autoridad en la que el navegador ya confía. Sin esa autenticación, el cifrado por sí solo no basta, porque un atacante en el medio podría simplemente presentarse como si fuera el servidor legítimo y negociar su propio canal cifrado con la víctima, sin que nadie note la diferencia.

Esto es justo lo que falla cuando un dispositivo no fuerza HTTPS: no es que falte "un cifrado más". Es que falta el único mecanismo que impide que alguien en el medio se haga pasar por el otro extremo.

## Dos routers domésticos, el mismo problema de fondo

En las últimas semanas he analizado dos routers distintos, de dos fabricantes distintos, distribuidos por dos operadores españoles distintos. Los dos comparten exactamente esta causa raíz, aunque la resuelven (mal) de formas distintas.

En el [router Wi-Fi 7 de Movistar]({{ '/2026/11/movistar-router-wifi7-askey-password-en-claro/' | relative_url }}) el panel se sirve por HTTP sin TLS, y el fabricante intenta compensarlo con una función JavaScript que aplica un XOR de un solo byte a la contraseña antes de enviarla. El problema ahí ni siquiera requiere estar activamente en el medio modificando tráfico: basta con escuchar de forma pasiva, porque el campo de contraseña original viaja igual, sin tocar, junto al valor "ofuscado". Solo el segundo hallazgo de ese router, el de la suplantación del formulario de login mediante un frame reescrito, necesita ya un MITM activo como el que describo arriba.

En el [router XGB430v Pro de TP-Link/Digi]({{ '/2026/10/tplink-xgb430v-pro-cifrado-inseguro/' | relative_url }}) el fabricante fue un paso más allá: en vez de una ofuscación simple, construyó su propio sustituto de TLS en JavaScript, con RSA y AES. Sobre el papel suena más serio. En la práctica, la clave AES se generaba combinando la hora exacta del sistema con solo tres dígitos de `Math.random()`, lo que reducía el espacio de búsqueda real a unas 1.000 combinaciones por milisegundo de margen. Ahí tampoco hacía falta modificar nada: con capturar pasivamente el login y su hora de llegada, probar esas combinaciones en un portátil normal tardaba un par de minutos.

Dos implementaciones distintas, dos fabricantes distintos, y el mismo patrón exacto: alguien decidió que no hacía falta TLS porque ya tenían "algo propio" que lo sustituía. En ambos casos esa sustitución es más fácil de romper que configurar correctamente el estándar que evitaban usar.

## Por qué sigue siendo peligroso en 2026, y no una curiosidad de manual

Hay tres razones por las que este ataque, que no usa ninguna técnica nueva, sigue abriendo puertas reales.

La primera es que la posición de ataque es barata y común. No hace falta acceso a infraestructura de operador ni capacidad de vigilancia estatal. Basta con estar en la misma red Wi-Fi que la víctima: una oficina, un domicilio compartido, una red doméstica donde cualquier dispositivo comprometido (una cámara IP, un termostato, un asistente barato) puede usarse como punto de partida para el ARP spoofing.

La segunda es que el "casi TLS" sigue siendo habitual. Un dispositivo que ofrece HTTPS como opción desactivada por defecto da la sensación de estar resuelto, porque la opción existe. Pero un mecanismo de seguridad que depende de que el usuario lo active manualmente protege, en la práctica, a casi nadie: la inmensa mayoría de los dispositivos se quedan con la configuración de fábrica para siempre.

La tercera, la que explican los dos routers anteriores, es la tentación de "hacerlo a mano". Cuando un equipo de desarrollo decide implementar su propio cifrado en vez de usar TLS, que lleva treinta años siendo revisado públicamente por miles de criptógrafos, el resultado casi nunca es mejor. Es, como mucho, igual de bueno pero sin el escrutinio. Y como hemos visto, ni siquiera suele ser igual de bueno.

## La defensa no cambia con el tiempo

Lo que protege de un MITM en 2026 es lo mismo que lo protegía hace quince años: TLS correctamente implementado, forzado por defecto, sin opción de desactivarlo ni de acceder en paralelo por HTTP. Ningún truco del lado cliente, por ingenioso que parezca sobre el papel, sustituye a eso. La diferencia entre un dispositivo seguro y uno que no lo es no está en si tiene "algo de cifrado". Está en si ese algo impide que alguien se cuele entre tú y el otro extremo haciéndose pasar por él, que es, al final, todo lo que un MITM necesita conseguir.
