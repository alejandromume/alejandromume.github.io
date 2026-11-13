---
title: "Routers en España: Wi-Fi 7, pero seguridad del 97"
date: 2026-11-13 09:00:00 +0000
author: "Alejandro Müller Medina"
featured: true
tags: [hardware, router, divulgacion-responsable, opinion, reflexion]
---

La caja dice Wi-Fi 7. Dice velocidades de varios gigabits, dice XGS-PON, dice "tecnología de nueva generación". Lo que no dice en ningún sitio es que para entrar al panel de administración de ese router, el que controla literalmente todo lo que pasa por tu red doméstica, basta con escuchar el tráfico un rato con un portátil cualquiera. Eso no es una metáfora. Es lo que he encontrado, de forma repetida, analizando routers de dos operadores distintos en las últimas semanas.

No he tenido que buscar mucho, ni atacar infraestructura de ningún operador: cogí el router que tengo en casa, el que me instaló mi propio proveedor de fibra, y miré cómo habla con el mundo. Luego hice lo mismo con el de otro operador. El patrón es el mismo en los dos, y es viejo: marketing de 2026, decisiones de seguridad que ya eran malas ideas cuando internet iba por módem. Es como comprarse un coche que aparca solo y frena con autonomía total, y descubrir que la puerta se abre soplando fuerte.

## El router no es un electrodoméstico más

Vale la pena pararse un segundo en por qué esto importa más que, digamos, que falle la seguridad de una freidora de aire conectada. El router es el único punto por el que pasa absolutamente todo tu tráfico doméstico: el banco, el trabajo, las cámaras de vigilancia del salón, el termostato, el móvil de tu hijo, el portátil con el que haces la declaración de la renta. Quien controla el router no necesita entrar en cada uno de esos sistemas por separado. Le basta con sentarse en el único sitio por el que todos pasan.

Y aun así, una y otra vez, el router es el dispositivo que menos atención de seguridad recibe de toda la casa. Se instala una vez, se mete en el mueble del recibidor detrás de las llaves y un paraguas que ya no se abre, nadie vuelve a tocarlo, y la mayoría de la gente ni siquiera sabe que tiene una contraseña de administrador distinta a la del Wi-Fi.

## Caso 1: la contraseña que viaja dos veces, y una de ellas sin tocar

Empiezo por el más sencillo de explicar, porque no hace falta saber nada de criptografía. El router Wi-Fi 7 que Movistar instala con sus ofertas de fibra más recientes, un Askey RTF8316VW, sirve su panel de administración por HTTP normal, sin el candado del `https://`. Para disimular esa carencia, alguien añadió una función en JavaScript que revuelve la contraseña antes de mandarla, con un XOR de clave fija: siempre el mismo valor, sumado letra a letra.

El problema no es ni siquiera que ese truco sea débil, y se revierte en una línea de código. El problema es que, al lado de la contraseña "revuelta", el formulario manda también la contraseña real, sin tocar, en el mismo paquete. Es como cifrar un mensaje con mucho cuidado, meterlo en una caja fuerte, y pegar el borrador original con celo en la puerta, por si acaso.

Cualquiera que esté escuchando esa red en el momento adecuado, sin modificar nada, sin hacer nada activo, se lleva la contraseña de administrador del router tal cual. (El análisis técnico completo, con capturas y la explicación paso a paso, está en la [entrada dedicada al router de Movistar]({{ '/2026/11/movistar-router-wifi7-askey-password-en-claro/' | relative_url }}).)

## Caso 2: cuando el fabricante decide inventarse su propio TLS

El segundo caso es el router XGB430v Pro de TP-Link que distribuye Digi bajo su marca DIGIFIBRA. Aquí el fabricante sí vio el problema de servir el panel por HTTP plano, y en vez de activar HTTPS, se construyó su propio sustituto desde cero: un esquema de cifrado en JavaScript con RSA y AES. Reinventar la rueda, pero cuadrada.

Sobre el papel lo es. En la práctica, la clave AES se genera combinando la hora exacta del sistema, que cualquiera que capture el tráfico ve directamente, con solo tres dígitos sacados de `Math.random()`. Ya escribí sobre [por qué `Math.random()` nunca debería tocar nada que importe]({{ '/2026/10/math-random-no-es-aleatorio/' | relative_url }}): aquí el espacio real de combinaciones que hay que probar se reduce a unas mil, y un portátil normal las recorre en menos de dos minutos. Sin modificar nada, solo escuchando.

Ese fallo venía acompañado de otro: el propio panel ejecutaba con `eval()` cualquier respuesta que recibiera de la red, sin comprobar de dónde venía, lo que abría la puerta a inyectar código ajeno con solo interceptar el tráfico activamente. El desglose completo de ambos fallos, con las capturas de pantalla, está en la [primera entrada sobre este router]({{ '/2026/10/tplink-xgb430v-pro-cifrado-inseguro/' | relative_url }}).

## Caso 3: de invitado en tu propia wifi a dueño del router para siempre

El tercer caso, sobre el mismo router de TP-Link y Digi, es el que más me hizo pararme a pensar, porque no necesita nada de lo anterior: ni interceptar tráfico, ni estar en una posición de red especial, ni que nadie inicie sesión mientras tú escuchas.

El router deja descargar una copia de seguridad de su configuración a cualquiera que tenga una cuenta de "usuario" estándar, la que usa cualquier miembro de la casa que no es quien administra el router. Es la cuenta pensada para que tu sobrino cambie el nombre de la Wi-Fi por algo con emojis, no para que acabe con las llaves del reino. Esa copia va cifrada, pero con una clave que es exactamente la misma en todas las unidades de ese modelo que existen en el mundo, derivada de un número público que identifica el modelo, no el dispositivo concreto. Descifrarla es cuestión de segundos, y dentro aparece la contraseña de administrador del router en texto plano, junto con las credenciales de la línea de fibra del cliente frente al propio operador.

Con esa contraseña de administrador, el siguiente paso es casi absurdo de lo simple que resulta: el router usa TR-069, el protocolo con el que el operador lo gestiona en remoto, y nunca comprueba que el servidor de gestión al que se conecta sea realmente el del operador. Es un portero que deja pasar a cualquiera que diga "vengo de mantenimiento", sin pedir ni una identificación de pega. Basta con cambiar una dirección en el panel para que el router obedezca a un servidor cualquiera, montado en un portátil, con acceso a más de cuatro mil parámetros: contraseñas Wi-Fi, DNS de toda la casa, firewall, reinicios, actualizaciones de firmware. Y ese control sobrevive a que el dueño reinicie el router, porque el propio atacante puede desactivar el mecanismo que lo revertiría.

Lo cuento con más detalle, incluyendo lo que se puede hacer en la práctica con ese control y un hallazgo adicional sobre un canal de gestión permanente que mantiene el propio operador, en la [entrada sobre el backup y TR-069]({{ '/2026/11/tplink-xgb430v-pro-backup-tr069-control-total/' | relative_url }}).

## El hilo que conecta los tres casos

Da igual el fabricante, da igual el operador: en los tres casos la raíz es exactamente la misma. Alguien decidió que el estándar que llevamos usando desde hace treinta años para proteger comunicaciones, TLS, el que hace que un navegador muestre un candado, era prescindible, y que un invento propio bastaría. Ya escribí sobre [por qué los ataques man-in-the-middle siguen funcionando en 2026]({{ '/2026/11/ataques-mitm-por-que-siguen-funcionando/' | relative_url }}) precisamente por este motivo: no es que falte "algo de cifrado" en estos dispositivos. Es que falta el único mecanismo capaz de impedir que alguien se cuele en medio de la conversación haciéndose pasar por el otro extremo, y nada de lo que se construye a mano para sustituirlo llega a estar a la altura.

Lo que más me llama la atención no es que fallen. Es que fallan todos de la misma forma, con la misma lógica de fondo: "TLS da pereza configurar bien, así que hacemos algo nuestro que parezca seguro". Da la sensación de que alguien soltó esa frase en un congreso del sector hacia 2015 y media industria se la apuntó en una servilleta. Ese "algo nuestro" nunca ha pasado la revisión pública que TLS lleva décadas pasando, y se nota en cuanto alguien se para a mirarlo con calma.

## Por qué pasa esto, y no es casualidad

Un sector que mueve miles de millones de euros no comete estos fallos por casualidad. Hay varias razones, y se alimentan entre sí.

La primera es que la seguridad no vende en la caja del router. Lo que vende es Wi-Fi 7, velocidad, número de antenas. Nadie ha visto nunca un anuncio que diga "ahora con autenticación mutua TLS", así que ahí no se invierte tiempo de ingeniería si se puede evitar.

La segunda es la fragmentación de responsabilidad. El router lo fabrica una empresa, como TP-Link o Askey, pero lo distribuye y lo mantiene otra, el operador. Cuando algo falla, cada parte señala a la otra, y mientras tanto el dispositivo sigue en casa de millones de clientes exactamente igual de expuesto.

La tercera es la prisa. Sacar un router Wi-Fi 7 antes que la competencia importa más, comercialmente, que dedicar meses a una auditoría de seguridad seria. El resultado es un dispositivo rápido por fuera y descuidado por dentro, justo la combinación más peligrosa: cuanto más nuevo y conectado es un aparato, más cosas hay detrás que proteger.

## Lo más grave: la solución ya existe, y no la quieren pagar

Esto no es un problema sin resolver. Es un problema resuelto desde hace años que casi ninguna operadora española se molesta en aplicar.

El origen técnico: una autoridad de certificación no puede emitir un certificado HTTPS válido para una IP privada como `192.168.1.1`, porque esa dirección no identifica a nadie en concreto, la tiene tu router y el de medio vecindario a la vez. Si el fabricante pone ahí un certificado propio sin que nadie lo avale, el navegador suelta la pantalla roja que asusta a cualquier usuario normal. Entre esa pantalla roja y quedarse en HTTP sin más, la mayoría elige quedarse en HTTP, que al menos no da miedo.

La solución tiene nombre: asignar a cada router un dominio propio, del estilo `miroutero.fabricante.com`, que resuelve a esa misma IP local, con un certificado real para ese dominio que el navegador acepta sin rechistar. ASUS lo hace. Fritz!Box lo hace. Esa infraestructura ya existe y funciona, hoy, en routers que se venden en España. Simplemente no son los que reparten la mayoría de los operadores.

¿Por qué no lo hace todo el mundo? Porque mantenerlo vivo cuesta dinero de forma continua: servidores renovando certificados durante toda la vida útil del dispositivo, gestión de qué pasa cuando el usuario cambia su DNS, un equipo dedicado a que no se rompa, año tras año, sin retorno visible para marketing. Para una marca que compite por precio, o un operador que reparte el router casi gratis con la fibra, eso es un gasto sin beneficio aparente. Entre pagarlo o asumir el riesgo, la decisión ya la conocemos.

La industria sí ha encontrado una salida, pero no es arreglar el panel: es abandonarlo. Cada vez más fabricantes sustituyen el acceso por navegador por una app móvil que habla con el router a través de su propia nube, sin pasar nunca por `192.168.1.1`. Resuelve el problema haciendo que deje de existir el sitio donde ocurría, y de paso mete tu router dentro de la infraestructura permanente del fabricante para siempre. Es cambiar un problema de seguridad local por uno de dependencia a perpetuidad, y llamarlo mejora de experiencia de usuario.

Al final sigues entrando por HTTP plano, o cediéndolo a una app en la nube, por la misma razón de siempre: a quien fabrica y distribuye le sale más barato asumir ese riesgo que pagar la infraestructura que ya demostró que esto se puede hacer bien.

## Un ejemplo que no necesita ser hipotético

No hace falta imaginar un escenario de película. Piensa en un bloque de pisos con fibra de un operador que reparte el mismo modelo de router a todos sus clientes. Alguien con acceso breve a la Wi-Fi de invitados de un vecino, o sentado en la cafetería de abajo si la gestión remota está activada de fábrica (y en al menos un caso lo estaba), termina con la contraseña de administrador de cualquier vecino en segundos, sin levantar ninguna alerta, y mantiene ese control de forma indefinida. El router, mientras tanto, sigue con su lucecita azul tan tranquila.

De ahí a redirigir el DNS de esa casa para colar una página de banca falsa, a abrir a Internet la cámara IP que alguien instaló pensando que solo la vería desde el móvil, o a autenticarse frente al propio operador con las credenciales de fibra de otra persona, hay un paso más pequeño del que debería ser razonable.

## Qué puedes hacer tú, ahora mismo

Mientras el sector resuelve esto a su ritmo, hay cosas que dependen solo de ti:

- Cambia la contraseña de administrador del router por una que no sea la de fábrica, si el panel lo permite.
- Si hay opción de acceder por HTTPS, actívala, aunque el navegador avise de un certificado autofirmado.
- Desactiva la gestión remota del router si no la necesitas y el operador te deja hacerlo.
- No reutilices la contraseña del router en ningún otro sitio, ni siquiera por comodidad el primer día.
- Desconfía de redes Wi-Fi abiertas o compartidas para entrar al panel de tu propio router: ese es exactamente el escenario que hace falta para explotar varios de estos fallos.

Ninguna de estas medidas arregla el fallo de fondo. Pero reduce de verdad la ventana de exposición mientras el fabricante y el operador no se muevan.

## Qué debería cambiar, y no depende de ti

Lo que de verdad resuelve esto no está en manos del usuario final, está en manos de quien diseña y distribuye estos dispositivos:

- HTTPS activado por defecto en el panel de administración, sin opción de desactivarlo ni de dejar HTTP disponible en paralelo.
- Ningún control de acceso que dependa del rol del usuario puede fallar en la práctica, y mucho menos en algo tan sensible como un backup de configuración.
- Material criptográfico generado con fuentes de aleatoriedad reales, no con la hora del sistema disfrazada de clave.
- Autenticación mutua en cualquier protocolo de gestión remota, para que un router no obedezca a cualquiera que diga ser su operador.
- Auditorías de seguridad independientes antes de que un modelo salga al mercado, no después de que alguien externo las haga gratis y las reporte.

Lo único que no se ha actualizado en treinta años es, precisamente, lo que más debería.

---

*Esta entrada reúne y pone en contexto hallazgos ya reportados individualmente a INCIBE-CERT, con el detalle técnico completo en cada artículo enlazado. Ninguno de esos informes incluye herramientas listas para usar contra dispositivos de terceros mientras la coordinación con fabricantes y operadores siga abierta.*
