---
title: "TP-Link XGB430v Pro: cuando el cifrado \"propio\" sustituye a TLS (y falla)"
date: 2026-10-26 09:00:00 +0000
author: "Alejandro Müller Medina"
featured: true
severity: high
cvss: "8.8 (CVSS v3.1 y v4.0)"
scope: "Panel de administración web local del router"
tags: [hardware, router, tp-link, divulgacion-responsable, criptografia]
---

Router doméstico/ISP de fibra XGS-PON con Wi-Fi 7, distribuido en España por Digi bajo la marca DIGIFIBRA. Su panel de administración se sirve por HTTP sin TLS de forma predeterminada, y el fabricante compensa esa carencia con un esquema de cifrado propio implementado en JavaScript del lado del cliente. Ese esquema, no el estándar que sustituye, es el que falla, y falla de dos formas distintas que se refuerzan entre sí.

Este hallazgo ya ha sido reportado a **INCIBE-CERT** en dos informes separados, uno por cada fallo, para coordinación de la divulgación. Está en proceso de traslado al fabricante y al programa de divulgación de vulnerabilidades del ISP. Mientras ese proceso sigue abierto, esta entrada se queda en el nivel de detalle que ya es público en los informes formales: qué falla, por qué importa y cómo se verificó, sin publicar herramientas listas para usar contra dispositivos de terceros. Se actualizará con más detalle cuando exista parche o se publique un identificador CVE.

## El dispositivo

- **Fabricante / modelo:** TP-Link XGB430v Pro (BE7200 Dual-Band Wi-Fi 7, XGS-PON), hardware v1.0.
- **Firmware probado:** 0.2.0 3.2.1 v60c3.0 Build 250808 Rel.72518n.
- **Distribución:** ISP Digi, bajo marca comercial DIGIFIBRA.
- **HTTPS local:** desactivado por defecto; el usuario debe activarlo manualmente.

## Por qué un router "sin TLS" puede ser más grave de lo que parece

Que el panel de administración de un router se sirva en HTTP puro no es inusual por sí solo si el dispositivo solo es alcanzable desde la LAN. El problema aparece cuando, para compensarlo, el fabricante construye su propio reemplazo de TLS en JavaScript: RSA de 512 bits sin padding para transportar una clave, AES-128-CBC para los datos, un hash MD5 de usuario+contraseña como credencial, y un nonce para el login.

Un esquema así solo tiene que fallar en un punto para que la sustitución completa de TLS se venga abajo. Aquí falla en dos puntos independientes: uno en cómo se ejecuta y almacena ese cifrado en el navegador (hallazgo 1), y otro en cómo se genera la propia clave AES (hallazgo 2). El segundo es, de hecho, más grave que el primero, porque ni siquiera requiere interceptar y modificar tráfico: basta con escuchar.

## Hallazgo 1: ejecución y almacenamiento inseguros del cifrado en el cliente

**`eval()` sobre una respuesta de red no autenticada.** Antes incluso de iniciar sesión, el panel pide `/cgi/getGDPRParm` y `/cgi/getBusy`, y ejecuta la respuesta directamente con `eval(xmlHttpObj.responseText)`. Esa respuesta viaja por HTTP plano, sin ninguna protección de integridad: cualquiera en posición de interceptar el tráfico (misma red local, vía ARP spoofing, por ejemplo) puede modificarla en tránsito e inyectar JavaScript arbitrario que se ejecuta con el contexto completo de la página de login.

<figure>
  <img src="{{ '/assets/img/posts/tplink/fig-eval.png' | relative_url }}" alt="Captura de Burp Suite mostrando la respuesta de /cgi/getGDPRParm, de tipo application/javascript, que el navegador ejecuta con eval()" width="1307" height="492" loading="lazy" decoding="async">
  <figcaption>Respuesta de <code>/cgi/getGDPRParm</code>: JavaScript en texto plano que el navegador ejecuta sin verificar su origen.</figcaption>
</figure>

**Material criptográfico de sesión en `localStorage`.** Tras un login correcto, la clave AES de sesión, el hash MD5 de las credenciales, la clave pública RSA del dispositivo y el nonce quedan en `localStorage`, accesible por cualquier script que corra en el origen de la página, incluido el inyectado en el punto anterior.

<figure>
  <img src="{{ '/assets/img/posts/tplink/fig-localstorage.png' | relative_url }}" alt="Captura mostrando un localStorage.setItem interceptado para enviar las claves AES a un servidor del atacante" width="1307" height="560" loading="lazy" decoding="async">
  <figcaption>El mismo <code>eval()</code> permite sobrescribir <code>localStorage.setItem</code> y capturar la clave AES en cuanto se genera tras el login.</figcaption>
</figure>

**Robo de clave, control total.** Capturando la clave AES de `localStorage` junto con tráfico cifrado de la misma sesión, es posible descifrar sin romper ningún primitivo criptográfico: credenciales del administrador y la configuración Wi-Fi completa del dispositivo en texto plano, incluyendo contraseñas WPA2/WPA3 de la red principal, de invitados, de backhaul EasyMesh, de modo puente, y el PIN WPS.

<figure>
  <img src="{{ '/assets/img/posts/tplink/fig-decrypt-script.png' | relative_url }}" alt="Script en Python que descifra con AES-128-CBC el tráfico del panel usando la clave robada de localStorage" width="1438" height="1093" loading="lazy" decoding="async">
  <figcaption>Con la clave robada, descifrar offline el tráfico capturado es un script de pocas líneas.</figcaption>
</figure>

Con la misma clave robada también es posible cifrar peticiones propias, eligiendo libremente el objeto del modelo de datos del dispositivo (`oid`), y el servidor las acepta como si vinieran del administrador legítimo, sin conocer la contraseña real en ningún momento.

<figure>
  <img src="{{ '/assets/img/posts/tplink/fig-forge-request.png' | relative_url }}" alt="Petición cifrada a /cgi_gdpr construida por el atacante y aceptada por el servidor sin conocer la contraseña real" width="1328" height="588" loading="lazy" decoding="async">
  <figcaption>Petición propia, cifrada con la clave robada, aceptada por el servidor como si viniera del administrador.</figcaption>
</figure>

<figure>
  <img src="{{ '/assets/img/posts/tplink/fig-wifi-plain.png' | relative_url }}" alt="Respuesta descifrada con la configuración Wi-Fi completa del dispositivo, incluida la contraseña, en texto plano" width="1328" height="531" loading="lazy" decoding="async">
  <figcaption>Respuesta descifrada: configuración Wi-Fi completa, contraseña incluida, obtenida sin conocer la contraseña del panel.</figcaption>
</figure>

**Sin protección frente a repetición.** Una petición de acción capturada una única vez sigue siendo válida y aceptada por el servidor de forma indefinida, sin necesidad de descifrarla ni modificarla. Se probó durante más de 10 minutos de reenvíos idénticos. El cierre de sesión sí invalida correctamente la sesión y, con ella, esas peticiones repetidas.

**`HttpOnly` sin efecto real.** La cookie de sesión (`JSESSIONID`, marcada `HttpOnly`) y la cabecera `TokenID` viajan sin cifrar por ir sobre HTTP plano. Un atacante en posición de red las obtiene por inspección pasiva del tráfico, sin necesidad de ejecutar JavaScript para robarlas. La protección `HttpOnly` solo tiene sentido frente a un atacante que ya está dentro del navegador, no frente a uno que está en la red.

Este primer hallazgo necesita que el atacante module el tráfico en tránsito (un MITM activo), al menos para el paso inicial de inyección vía `eval()`.

## Hallazgo 2: la clave AES se genera con un generador de números no criptográfico

Este segundo informe apunta a la raíz del problema anterior: ¿de dónde sale esa clave AES que basta con robar? La función que la genera, `$.encrypt.AES.genKey()`, construye tanto la clave como el IV así:

```js
AES.prototype.genKey = function () {
    var key = (new Date().getTime() + "" + Math.random()*1000000000).substr(0, KEY_LEN);
    var iv  = (new Date().getTime() + "" + Math.random()*1000000000).substr(0, IV_LEN);
    ...
};
```

El timestamp actual (`Date.now()`) tiene 13 dígitos, y el resultado se trunca a 16 caracteres. Eso significa que los primeros 13 caracteres de cada valor son el instante exacto en milisegundos, un dato público que cualquiera que capture el tráfico ve directamente en la cabecera de su propia herramienta de captura. Solo los 3 caracteres restantes proceden de `Math.random()`, que no es un generador criptográficamente seguro y que en este uso concreto aporta del orden de 10 bits de entropía real, en vez de los 128 bits nominales de una clave AES-128.

Ya escribí sobre por qué `Math.random()` nunca debería usarse para nada que importe: [es una fórmula determinista, no una fuente de azar]({{ '/2026/10/math-random-no-es-aleatorio/' | relative_url }}), y con tan solo un puñado de salidas se puede reconstruir su estado interno completo. Aquí el problema es aún más directo, porque ni siquiera hace falta observar varias salidas del generador: el propio timestamp, que viaja en claro, ya hace la mayor parte del trabajo de reducir el espacio de búsqueda.

Hay un segundo matiz que reduce aún más el problema: las dos llamadas a `Date.now()` (una para la clave, otra para el IV) se ejecutan de forma síncrona en el mismo hilo, así que casi siempre devuelven el mismo milisegundo. Y en AES-CBC, el IV solo afecta a la decodificación correcta del primer bloque de 16 bytes de texto plano; el resto de bloques se descifran bien aunque el IV usado sea incorrecto, mientras coincida en los primeros 13 caracteres. Como el campo de la contraseña no suele caer en ese primer bloque, en la práctica el espacio de búsqueda útil se reduce a las 1.000 combinaciones posibles de la clave.

<figure>
  <img src="{{ '/assets/img/posts/tplink/fig-prng-wireshark.png' | relative_url }}" alt="Captura de Wireshark con la petición de login capturada y su Epoch Arrival Time en milisegundos" width="1307" height="823" loading="lazy" decoding="async">
  <figcaption>Captura pasiva de una petición de login con Wireshark: el timestamp de llegada se lee directamente en la cabecera del paquete.</figcaption>
</figure>

<figure>
  <img src="{{ '/assets/img/posts/tplink/fig-prng-code.png' | relative_url }}" alt="Fragmento de código genKey() que construye la clave y el IV a partir de Date.now() y Math.random()" width="1307" height="504" loading="lazy" decoding="async">
  <figcaption>El código que genera clave e IV: timestamp más tres dígitos de <code>Math.random()</code>.</figcaption>
</figure>

Con eso, un atacante puramente pasivo (que solo escucha, sin modificar ni inyectar nada) puede:

1. Capturar una petición de login cifrada y anotar su instante de llegada con precisión de milisegundos.
2. Generar localmente candidatos de clave e IV combinando ese timestamp, con un margen de tolerancia de unas pocas decenas de milisegundos, con las 1.000 combinaciones posibles de los tres dígitos aleatorios.
3. Probar cada candidato contra el tráfico cifrado capturado, verificando el padding PKCS7 y que el resultado sea un JSON válido.

El espacio de búsqueda resultante, del orden de 2·10⁸ combinaciones para una ventana de ±100 ms, se recorre en minutos con un portátil de consumo. En la prueba real realizada contra el propio dispositivo, la clave correcta apareció tras unos 3,2·10⁷ intentos, en menos de 2 minutos, revelando usuario y contraseña del panel en claro.

<figure>
  <img src="{{ '/assets/img/posts/tplink/fig-prng-bruteforce.png' | relative_url }}" alt="Captura de terminal mostrando el script de fuerza bruta encontrando la clave AES y descifrando usuario y contraseña" width="1396" height="732" loading="lazy" decoding="async">
  <figcaption>Fuerza bruta offline de la clave: menos de dos minutos para recuperar la clave AES real y descifrar el login.</figcaption>
</figure>

La diferencia con el primer hallazgo es justo esa: aquí no hace falta tocar el tráfico, ni colocarse de forma activa entre la víctima y el router. Con capturar pasivamente lo que el propio dispositivo emite por defecto ya es suficiente para comprometer la sesión.

## Requisitos para explotarlo

- **Hallazgo 1 (eval + localStorage):** posición de intercepción y modificación del tráfico HTTP (MITM activo), y una sesión de administrador activa durante la ventana de ataque.
- **Hallazgo 2 (clave predecible):** posición de intercepción puramente pasiva del tráfico, y haber capturado (o tener ya capturada) la petición de login.
- Común a ambos: "HTTPS local" desactivada, que es el estado de fábrica.

No se ha evaluado si el panel es accesible desde la WAN; de serlo, la explotabilidad pasa de "red adyacente" a "red remota" en términos de CVSS, y el impacto sube considerablemente.

## Impacto

Compromiso total de la sesión de administración y divulgación de toda la configuración Wi-Fi del dispositivo, incluidas las contraseñas en texto plano, sin necesidad de conocer ni crackear la contraseña del panel. Sobre routers domésticos, eso abre la puerta a la red completa, no solo al dispositivo. El segundo hallazgo agrava el primero porque convierte un ataque que exigía intercepción activa en uno que se consigue solo con escuchar.

- **CVSS v3.1 / v4.0:** 8.8 (Alto) en ambos informes.
  - Hallazgo 1: `CVSS:3.1/AV:A/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H` · `CVSS:4.0/AV:A/AC:L/AT:N/PR:N/UI:N/VC:H/VI:H/VA:H/SC:N/SI:N/SA:N`
  - Hallazgo 2: mismo vector en ambas versiones.
- **CWE:**
  - Hallazgo 1: CWE-319 (transmisión en claro de información sensible), con CWE-95 (eval injection), CWE-922 (almacenamiento inseguro de material sensible), CWE-294 (bypass de autenticación por captura-repetición) y CWE-284 (control de acceso incorrecto) como relacionados.
  - Hallazgo 2: CWE-330 (uso de valores insuficientemente aleatorios), con CWE-338 (uso de un PRNG criptográficamente débil, `Math.random()`) como causa específica.

## Qué debería cambiar

1. **HTTPS activado por defecto**, con certificado autofirmado o gestionado por dispositivo, en lugar de una opción manual desactivada de fábrica.
2. **Eliminar `eval()` sobre contenido de red.** Los endpoints que hoy devuelven JavaScript ejecutable deberían devolver JSON, parseado con `JSON.parse()` y nunca ejecutado como código.
3. **No guardar material criptográfico de sesión en `localStorage`** ni en ningún sitio accesible por script; si hace falta persistencia, usar un mecanismo equivalente a una cookie `HttpOnly` real, o regenerarlo en cada carga.
4. **Protección anti-repetición real** (nonce de un solo uso o ventana temporal corta) en todas las peticiones de acción, no solo en el login.
5. **Sustituir MD5** por un hash de contraseñas diseñado para ello (bcrypt, scrypt o Argon2) con sal por usuario.
6. **Sustituir `Math.random()` por `window.crypto.getRandomValues()`** para generar cualquier material criptográfico (claves, IVs, nonces), sin derivar ningún componente de un valor público o predecible como una marca de tiempo. Generar clave e IV de forma independiente, con longitud completa de bytes aleatorios.

Los puntos 1 y 6 van de la mano: comparten el mismo vector de acceso (posición de red) y se refuerzan mutuamente en el impacto, así que deberían corregirse juntos.

## Línea temporal

| Fecha | Hecho |
|---|---|
| 2026-10-26 | Descubrimiento y verificación del hallazgo 1 (eval + localStorage) |
| 2026-10-27 | Envío a INCIBE-CERT del informe del hallazgo 1 |
| 2026-10-27 | Descubrimiento, verificación y envío a INCIBE-CERT del hallazgo 2 (clave AES predecible) |
| (pendiente) | Traslado al fabricante y actualización de esta entrada |

---

*Divulgación responsable: los valores sensibles de las capturas (SSID, BSSID, claves, credenciales) están tapados en los informes originales. No se publican aquí herramientas ni scripts listos para explotar el fallo en dispositivos de terceros mientras la coordinación con el fabricante siga abierta.*
