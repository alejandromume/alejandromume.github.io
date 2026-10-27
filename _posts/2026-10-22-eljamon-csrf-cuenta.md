---
title: "El Jamón: un CSRF sin token deja cambiar el email de cualquier cuenta"
date: 2026-10-22 09:00:00 +0000
author: "Alejandro Müller Medina"
ref: "CVE-2026-0271"
severity: high
cvss: "~7.1 (CVSS v4.0, estimación provisional; el fabricante o INCIBE podrían reevaluarla al alza)"
scope: "Portal de supermercadoseljamon.com (plataforma Comerzzia / Liferay)"
tags: [csrf, account-takeover, divulgacion-responsable, web]
---

Supermercados El Jamón vende alimentación online con entrega a domicilio, sobre una plataforma de comercio electrónico (Comerzzia, construida sobre portlets Liferay). Varias de las acciones que modifican datos de la cuenta, incluida la dirección de entrega y, más grave aún, el email de contacto, no llevan ningún token anti-CSRF. Eso significa que un enlace o un botón en una página ajena, sin que la víctima haga nada más que hacer clic con su sesión abierta, puede cambiar esos datos en su nombre.

El hallazgo más grave de los dos (el que afecta al email) es, en la práctica, un precursor directo de toma de cuenta completa: si el atacante puede redirigir el email de contacto, puede después iniciar el flujo de "olvidé mi contraseña" y recibir él mismo el correo de recuperación. Se descubrió hace pocos días, se ha enviado a INCIBE-CERT pidiendo coordinación con el operador (no hay canal de divulgación responsable público en el dominio) y no ha habido contacto directo todavía. Dado que el fallo sigue activo, afecta a una tienda real con clientes reales y el camino hacia el impacto más grave es trivial de automatizar, esta entrada no incluye el HTML de prueba de concepto ni los nombres de campo exactos: cuenta el mecanismo y el impacto, no el exploit.

## El patrón de fondo

El portal usa, para las acciones normales de Liferay, un token (`p_auth`) que protege las URLs de acción contra este tipo de ataque. Los endpoints afectados aquí no lo llevan, a pesar de modificar datos de la cuenta:

- Uno de ellos cambia la dirección principal de entrega del usuario autenticado, por `GET`, sin exigir ningún token ni validar que el identificador de fidelizado de la petición corresponda a la sesión activa. Es decir, se puede explotar a ciegas: no hace falta saber nada de la víctima de antemano, el servidor aplica el cambio igualmente sobre la cuenta de quien tenga la sesión abierta.
- El segundo, más grave, reescribe de una sola vez todos los datos personales del fidelizado (nombre, apellidos, fecha de nacimiento, documento de identidad, teléfonos, email y dirección) mediante un formulario `POST`. El campo del email viaja por el mismo mecanismo que el resto, sin ninguna comprobación adicional: ni confirmación por correo al nuevo valor, ni reautenticación con la contraseña actual.

Un tercer endpoint, de menor severidad (añadir artículos a una lista de deseos), comparte exactamente la misma causa raíz: se repitió la prueba varias veces y cada clic creó una lista nueva en la cuenta de la víctima, sin que esta hiciera nada más.

<figure>
  <img src="{{ '/assets/img/posts/eljamon/fig-listas.png' | relative_url }}" alt="Pantalla de 'mis listas' de la cuenta, mostrando varias listas de prueba (CSRFTEST2, CSRFTEST3, CSRFTEST4) creadas por repetición del mismo ataque" width="2478" height="984" loading="lazy" decoding="async">
  <figcaption>Tres listas de prueba, una por cada repetición del mismo clic: el servidor no detecta ni limita la repetición.</figcaption>
</figure>

Los tres apuntan a lo mismo: falta una capa de protección anti-CSRF en el conjunto de acciones bajo `/delegate/*AjaxServletFood` y en las action URLs del portal, no es un fallo puntual de un único formulario.

## Qué se confirmó

- **Cambio de dirección sin interacción visible:** con la víctima autenticada, un solo clic en un enlace controlado por el atacante (ni siquiera hace falta un formulario, basta con navegar a una URL) deja la cuenta con una nueva dirección principal, confirmado consultando "mis direcciones" después del clic. El sitio no fija el atributo `SameSite` en la cookie de sesión, así que la mitigación por defecto del navegador para navegación de página completa no se aplica.

<figure>
  <img src="{{ '/assets/img/posts/eljamon/fig-direccion.png' | relative_url }}" alt="Pantalla de 'mis direcciones' de la cuenta, mostrando la dirección principal cambiada a valores de prueba (CALLE CSRFTEST 777, CSRFTESTCITY, CSRFTESTPROV) tras el clic en el enlace malicioso" width="1938" height="844" loading="lazy" decoding="async">
  <figcaption>La dirección principal de la cuenta, cambiada a los valores de prueba tras un solo clic. Ningún otro paso.</figcaption>
</figure>

- **Modificación de datos personales por formulario cross-site:** un formulario `POST` alojado en cualquier otro origen, con los mismos campos que el formulario legítimo, cambia los datos personales de la cuenta activa. Se confirmó el cambio sobre un campo de contacto (el móvil), y el campo del email se procesa exactamente por el mismo camino, sin comprobación diferenciada. La página de prueba, vista desde fuera, no se distingue en nada de un enlace de oferta cualquiera:

<figure>
  <img src="{{ '/assets/img/posts/eljamon/fig-poc-page.png' | relative_url }}" alt="Página de prueba de concepto con el título 'Prueba CSRF - cambio de móvil' y un único botón de texto 'Mira esta oferta especial'" width="1302" height="290" loading="lazy" decoding="async">
  <figcaption>Así de poco llamativa es la página que dispara el cambio: un botón sin relación aparente con el sitio objetivo.</figcaption>
</figure>

- **No es un control anti-CSRF real:** durante las pruebas, alguna petición cross-site devolvió un aviso de "sesión caducada" justo después de iniciar sesión, pero repitiendo la misma petición tras un nuevo login funcionaba sin cambios. Eso encaja con un balanceador de carga con retardo de replicación entre nodos, no con una protección real, que habría bloqueado la petición de forma consistente en todos los intentos.

## Requisitos

En ambos casos, lo único que necesita la víctima es tener sesión activa en el sitio en el navegador donde hace clic en el enlace o carga la página con el formulario malicioso. El atacante no necesita ningún dato previo de la víctima: para el cambio de dirección, ni siquiera un identificador válido; para los datos personales, solo conocer el formato del formulario, que es público porque está en el HTML de la página legítima.

## Impacto

El cambio de dirección habilita un vector de fraude físico directo: al tratarse de un comercio con entrega a domicilio, un pedido futuro legítimo de la víctima podría acabar entregándose en una dirección bajo control del atacante, sin que la víctima lo note hasta que el pedido no llegue.

El segundo hallazgo es más grave: dado que el email de contacto se puede sobrescribir por el mismo mecanismo que el resto de campos del formulario, sin reautenticación ni confirmación, un atacante puede redirigirlo a una dirección propia y, acto seguido, iniciar la recuperación de contraseña para hacerse con el control completo de la cuenta, incluyendo historial de pedidos, datos de fidelización y cualquier medio de pago guardado.

- **CVSS v4.0 (estimación provisional):** `CVSS:4.0/AV:N/AC:L/AT:N/PR:N/UI:P/VC:N/VI:H/VA:N/SC:N/SI:N/SA:N` ≈ 7.1 (Alta) para ambos hallazgos. El propio informe señala que el segundo, al habilitar toma de cuenta completa, probablemente merece una puntuación mayor una vez recalculada con la calculadora oficial y con métricas de contexto de negocio.
- **CWE:** CWE-352 (Cross-Site Request Forgery) en los tres endpoints. El segundo se relaciona también con CWE-640 (mecanismo de recuperación de contraseña débil), porque permite desviar el email de la cuenta sin verificación.

## Qué debería cambiar

1. Exigir un token anti-CSRF real (equivalente al `p_auth` que ya protege otras acciones del mismo portal) en todos los endpoints de `/delegate/*AjaxServletFood` y en las action URLs que modifiquen estado, validado en servidor antes de procesar nada.
2. Migrar las acciones de `GET` a `POST` donde corresponda, y validar también el origen de la petición (cabeceras `Origin`/`Referer` o `Sec-Fetch-Site`).
3. Fijar explícitamente `SameSite=Strict` (o, como mínimo, `Lax` de forma explícita) en la cookie de sesión, en vez de depender del valor por defecto del navegador.
4. Para el cambio de email en particular, dado lo crítico del campo: exigir la contraseña actual o una confirmación por correo al nuevo valor antes de aplicar el cambio, en vez de aceptarlo junto con el resto del formulario sin verificación.
5. Revisar el resto de acciones del portal en busca del mismo patrón, ya que se ha confirmado en al menos tres endpoints independientes con la misma causa raíz.

## Línea temporal

| Fecha | Hecho |
|---|---|
| 2026-10-22 | Descubrimiento de ambos hallazgos |
| 2026-10-27 | Envío a INCIBE-CERT, solicitando coordinación del contacto con el operador (no se ha localizado canal de divulgación responsable propio) |
| (pendiente) | Contacto y respuesta del operador |

---

*Divulgación responsable: ambos fallos siguen sin corregir y afectan a una tienda con clientes reales y datos personales reales, con un camino directo a toma de cuenta completa. Las capturas usan únicamente datos de prueba creados por el propio investigador en su cuenta (valores "CSRFTEST…"), nunca datos de terceros. Esta entrada no incluye el HTML de prueba de concepto ni los nombres de campo exactos de los formularios. Se ampliará con el detalle técnico completo cuando haya confirmación de corrección.*
