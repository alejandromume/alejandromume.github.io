---
title: "HiperDino: un XSS en el buscador filtra el email de cualquier cuenta"
date: 2026-10-22 15:00:00 +0000
author: "Alejandro Müller Medina"
ref: "CVE-2026-0272"
severity: medium
cvss: "~6.0 (CVSS v4.0, estimación provisional)"
scope: "Buscador de hiperdino.es (Magento 2 / Adobe Commerce)"
tags: [xss, magento, divulgacion-responsable, web]
---

HiperDino es la cadena de supermercados de Canarias, y su tienda online corre sobre Magento 2 / Adobe Commerce. El buscador del sitio tiene un XSS reflejado que, con un solo enlace, ejecuta JavaScript en el navegador de cualquier visitante y, si la víctima tiene sesión iniciada, revela el email de su cuenta.

Este hallazgo se descubrió hace apenas unos días y se ha enviado a INCIBE-CERT pidiendo que coordine el contacto con el operador del sitio: a día de hoy no se ha localizado ni un `security.txt` ni ningún canal público de divulgación responsable en el dominio, y no ha habido contacto directo todavía. Al tratarse de una tienda real con clientes reales, y de un fallo sin parchear, esta entrada se queda deliberadamente corta en el apartado técnico: cuenta qué falla y por qué importa, pero no incluye el payload ni las URLs de prueba de concepto, que seguirían funcionando contra cualquier visitante del sitio hoy mismo. Se ampliará en cuanto haya confirmación de corrección.

## Dónde está el fallo

El buscador del sitio pasa el término de búsqueda (el parámetro `q` de la URL) a un bloque `<script>` inline que inicializa el `dataLayer` de Google Tag Manager en cada página de resultados, dentro de un objeto `globalParams`. El valor llega ahí sin el escapado necesario para ese contexto: no se neutraliza la secuencia de cierre de etiqueta `</script>`, así que es posible cerrar el bloque de script original desde el propio término de búsqueda y abrir uno nuevo con código arbitrario.

La causa raíz probable es la ausencia del flag `JSON_HEX_TAG` (junto a `JSON_HEX_AMP`, `JSON_HEX_APOS`, `JSON_HEX_QUOT`) al generar ese JSON en servidor con `json_encode()`, un fallo de escapado bastante típico cuando se incrusta un valor dinámico directamente dentro de una etiqueta `<script>` por interpolación de texto en vez de con un bloque de datos aparte.

## Qué se confirmó

- **Ejecución de JavaScript arbitrario:** un enlace al buscador con el payload adecuado ejecuta código en el contexto del sitio, confirmado con una prueba mínima (un diálogo de alerta). Las capturas de abajo lo demuestran con el payload oculto, para no dejar el enlace listo para copiar.

<figure>
  <img src="{{ '/assets/img/posts/hiperdino/fig-xss-alert.png' | relative_url }}" alt="Diálogo del navegador con el texto 'www.hiperdino.es dice 1', confirmando la ejecución de JavaScript inyectado en el buscador; el payload de la caja de búsqueda aparece oculto" width="1996" height="748" loading="lazy" decoding="async">
  <figcaption>Confirmación de la inyección: el diálogo de alerta se dispara desde el propio origen del sitio. El payload se ha ocultado.</figcaption>
</figure>

El payload real se envió codificado en base64 y ejecutado con `eval(atob(...))`, para esquivar cualquier filtro de caracteres especiales:

<figure>
  <img src="{{ '/assets/img/posts/hiperdino/fig-xss-eval-atob.png' | relative_url }}" alt="Diálogo de alerta confirmando la ejecución de un payload codificado en base64 con eval(atob(...)); la URL y la caja de búsqueda aparecen ocultas" width="1358" height="644" loading="lazy" decoding="async">
  <figcaption>El mismo fallo, con el payload en base64. URL y caja de búsqueda ocultas por el mismo motivo que arriba.</figcaption>
</figure>

- **Divulgación del email de cuenta:** la cookie `email_user`, que guarda el email de la cuenta en texto plano, no lleva el flag `HttpOnly`, así que es legible directamente desde JavaScript. El mismo XSS que ejecuta código puede leer esa cookie y exfiltrarla sin pasos adicionales.
- **Control positivo:** la cookie de sesión (`PHPSESSID`) sí lleva `HttpOnly` y no aparece en `document.cookie`, así que no se puede leer directamente por este camino.

Eso último no cierra el riesgo del todo. El código inyectado corre en el mismo origen que la sesión activa, así que cualquier petición (`fetch`/`XHR`) lanzada desde ese mismo payload incluiría automáticamente la cookie de sesión al llegar al servidor, aunque el JavaScript nunca llegue a leer su valor. Eso abriría la puerta a leer o modificar datos de la cuenta autenticada, por ejemplo desde la página de "mi cuenta", sin necesidad de robar nada de `document.cookie`. Esa vía no se ha explotado a fondo en la prueba de concepto, precisamente para no tocar datos reales de la cuenta usada en las pruebas, así que el impacto real podría ser bastante mayor del que refleja la puntuación CVSS de abajo.

## Requisitos

Ninguno para ejecutar el payload en sí: no hace falta estar autenticado ni saber nada de la víctima de antemano, basta con que abra el enlace. Para que se filtre específicamente el email de cuenta, la víctima necesita tener una sesión iniciada en el sitio en el momento de abrir el enlace, porque la cookie `email_user` solo existe en ese caso.

## Impacto

Con un único enlace, cualquier visitante del buscador (con sesión o sin ella) puede acabar ejecutando JavaScript arbitrario en su propio navegador. Como mínimo queda confirmada la divulgación del email de la cuenta de cualquier usuario autenticado que abra el enlace, y como se explica arriba, el techo real del impacto probablemente sea mayor.

- **CVSS v4.0 (estimación provisional):** `CVSS:4.0/AV:N/AC:L/AT:N/PR:N/UI:P/VC:L/VI:L/VA:N/SC:N/SI:N/SA:N` ≈ 6.0 (Media). Cálculo manual pendiente de recalcular con la calculadora oficial antes de cerrarse; dado lo señalado sobre la cookie de sesión, es más una cota inferior que una cifra definitiva.
- **CWE:** CWE-79 (Cross-Site Scripting). Causa raíz probable: CWE-116 (neutralización incorrecta de la salida).

Como nota aparte, de menor severidad: el mismo script de inicialización expone en producción el valor literal `"environment":"qa"`, lo que podría apuntar a un entorno de pruebas mal etiquetado o expuesto por error (CWE-215), pendiente también de confirmación.

## Qué debería cambiar

1. Aplicar el escapado correcto al generar `globalParams` en servidor, con `JSON_HEX_TAG` (y compañía) en `json_encode()`, para neutralizar `</script>` dentro de datos incrustados en bloques de script.
2. En general, no incrustar datos controlados por el usuario directamente dentro de una etiqueta `<script>` por interpolación de texto; mejor un bloque `<script type="application/json">` que se parsea después, o un atributo `data-*` leído de forma segura.
3. Aplicar ese mismo escapado en cualquier otro punto donde se refleje el término de búsqueda.
4. Una Content-Security-Policy estricta (con nonce o hash para scripts) limitaría el impacto aunque se reintrodujera un fallo parecido.
5. Revisar si la cookie `email_user` necesita ser legible por JavaScript; si solo hace falta en el servidor, marcarla como `HttpOnly`.
6. Confirmar y, si procede, corregir la exposición de `"environment":"qa"` en producción.

## Línea temporal

| Fecha | Hecho |
|---|---|
| 2026-10-22 | Descubrimiento |
| 2026-10-27 | Envío a INCIBE-CERT, solicitando coordinación del contacto con el operador (no se ha localizado canal de divulgación responsable propio) |
| (pendiente) | Contacto y respuesta del operador |

---

*Divulgación responsable: el fallo sigue sin corregir y afecta a una tienda con clientes reales, así que esta entrada no incluye el payload ni las URLs de prueba de concepto. Se ampliará con el detalle técnico completo cuando haya confirmación de corrección.*
