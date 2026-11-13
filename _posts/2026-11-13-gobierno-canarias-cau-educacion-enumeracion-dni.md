---
title: "El botón de ayuda que delata tu DNI en Canarias"
date: 2026-11-13 12:00:00 +0000
author: "Alejandro Müller Medina"
severity: high
cvss: "~8.2 (CVSS v4.0, estimación provisional)"
scope: "Endpoint REST público del CAU (Centro de Atención al Usuario) de la Consejería de Educación"
tags: [idor, information-disclosure, divulgacion-responsable, web, administracion-publica]
---

El portal de login del sistema educativo de Canarias (el que usan alumnado, familias y personal docente para entrar a sus servicios online) tiene un botón de "¿Necesitas ayuda?" que pide el DNI para identificarte antes de ayudarte. Ese formulario, pensado para algo tan modesto como decirte si tu documento está dado de alta, en realidad expone un endpoint público que cualquiera puede usar para comprobar, sin iniciar sesión ni nada parecido, si un DNI o NIE concreto existe en el sistema. Y no solo si existe: también revela de qué tipo de perfil es (alumnado, familia, personal, con o sin correo corporativo).

Este hallazgo se ha enviado a INCIBE-CERT para coordinar la divulgación con el organismo afectado. No se ha contactado todavía de forma directa con el Gobierno de Canarias. Las pruebas realizadas se limitaron a confirmar el comportamiento diferencial del endpoint con identificadores de control, sin consultar documentos reales de terceros.

## El camino hasta el fallo

El flujo empieza en la pantalla de login normal, la que pide usuario y contraseña o acceso con Cl@ve.

<figure>
  <img src="{{ '/assets/img/posts/gobcan/fig-login.png' | relative_url }}" alt="Pantalla de login del Sistema Centralizado de Autenticación del Gobierno de Canarias, con un enlace '¿Necesitas ayuda?' señalado con una flecha" width="1586" height="940" loading="lazy" decoding="async">
  <figcaption>El enlace "¿Necesitas ayuda?" del panel de login, el punto de entrada de todo esto.</figcaption>
</figure>

Al pulsar "Ayuda personalizada" dentro de ese menú, el formulario pide el DNI, NIE o CIAL del usuario para, supuestamente, orientarle sobre cómo recuperar el acceso.

<figure>
  <img src="{{ '/assets/img/posts/gobcan/fig-dni-input.png' | relative_url }}" alt="Formulario modal pidiendo DNI, Pasaporte o CIAL con un botón 'Comprobar'" width="1674" height="1040" loading="lazy" decoding="async">
  <figcaption>El formulario que pide el documento de identidad, sin ninguna sesión abierta ni comprobación previa.</figcaption>
</figure>

Al pulsar "Comprobar", el navegador lanza una petición `GET` directa a un endpoint REST público:

```
https://www3.gobiernodecanarias.org/educacion/cau_ce/webservices/
rest_checkusersdomain/api/checkuser/{identificador}
```

No hace falta estar autenticado, no hay token, no hay cookie de sesión que comprobar. El identificador va en la propia URL, como parámetro de ruta, y la respuesta depende únicamente de si ese documento está o no dado de alta.

<figure>
  <img src="{{ '/assets/img/posts/gobcan/fig-code-api.png' | relative_url }}" alt="Código JavaScript del panel (cas.js) visto en las herramientas de desarrollador, mostrando la llamada AJAX al endpoint checkuser y las ramas if/else según el valor de respuesta (METADIRECTORIO, LDAPALUMNADOCORREO, LDAPALUMNADO)" width="1848" height="742" loading="lazy" decoding="async">
  <figcaption>El propio JavaScript del panel, visto desde las herramientas de desarrollador. La URL completa se ha tapado en esta captura; el resto del código, que es el que importa aquí, queda visible.</figcaption>
</figure>

Ese fragmento de código es la prueba de que el fallo no es solo "existe o no existe". El frontend distingue varias respuestas posibles, y cada una activa una pantalla distinta:

- `METADIRECTORIO`
- `LDAPALUMNADOCORREO`
- `LDAPALUMNADO`
- `BBDD` / `BBDDCORREO` (observadas también, no visibles en este fragmento concreto)

Cada uno de esos valores corresponde a un directorio interno distinto según el tipo de usuario. En la práctica, eso significa que un DNI consultado no solo confirma que esa persona está en el sistema educativo canario: también revela, por el valor devuelto, si es alumno, familia o personal, y si tiene correo corporativo asignado. Si el identificador no está registrado en absoluto, la respuesta es simplemente nula.

## Lo que se confirmó

- El endpoint responde de forma diferenciada según el estado del identificador, sin exigir ninguna sesión ni token previo.
- Se repitió la consulta 15 veces de forma consecutiva, con identificadores distintos, desde la misma sesión de red. No hubo ningún HTTP 429, ningún retardo creciente, ningún bloqueo ni CAPTCHA. El servicio respondió igual de rápido en la petición número 15 que en la primera.
- Nada en el flujo limita qué identificadores se pueden probar. Al ser un DNI o NIE español, el espacio de valores posibles es finito y generable por fuerza bruta estructurada (el propio formato del documento reduce mucho el rango a comprobar).

La combinación de ambas cosas, sin límite de peticiones y con una respuesta que distingue perfiles, convierte un formulario de ayuda en una herramienta de enumeración masiva: cualquiera en Internet puede, con un script de pocas líneas, confirmar qué documentos de identidad están vinculados al sistema educativo canario y a qué tipo de perfil corresponde cada uno.

## Hallazgos adicionales, mismo barrio de servicios

Revisando los servicios REST hermanos bajo la misma ruta `/educacion/cau_ce/webservices/` aparecieron dos cosas más, que no se han explotado a fondo pero que conviene anotar:

- El endpoint `/webservices/rest_clave_authentication/api/clave`, con el parámetro `service`, revela el nombre interno de la aplicación IIS y tiene un manejo de errores inconsistente, el típico síntoma de una API que no filtra lo que devuelve cuando algo sale mal.
- El propio `rest_checkusersdomain/api` expone en producción el scaffold de desarrollo por defecto de ASP.NET Web API, incluido el controlador de ejemplo `ValuesController`. Es la plantilla que Visual Studio genera al crear un proyecto nuevo, la que nadie debería ver fuera de un entorno de desarrollo.

Ninguno de los dos es tan directo como el fallo principal, pero ambos apuntan en la misma dirección: esta familia de servicios se desplegó sin una revisión de qué queda expuesto hacia fuera.

## Requisitos para explotarlo

Ninguno. El endpoint es público, no exige sesión, no exige token, y es alcanzable por cualquiera con conexión a Internet desde la pantalla de login, sin necesidad de tener ninguna cuenta previa en el sistema.

## Impacto

Exposición masiva de información personal indirecta: confirmación de la existencia de un DNI/NIE en el sistema educativo canario, junto con el tipo de perfil asociado (alumnado, familia o personal, con o sin correo corporativo). Sobre una base de usuarios que cubre a buena parte del alumnado, familias y personal docente de Canarias, esto habilita campañas de phishing dirigido con mucha más precisión de la habitual: saber de antemano que un DNI concreto pertenece a un padre o madre con hijos en el sistema, por ejemplo, permite construir un cebo mucho más convincente que un envío genérico.

- **CVSS v4.0 (estimación provisional):** `CVSS:4.0/AV:N/AC:L/AT:N/PR:N/UI:N/VC:H/VI:N/VA:N/SC:N/SI:N/SA:N` ≈ 8.2 (Alto). Pendiente de verificación con la calculadora oficial de FIRST.
- **CWE:**
  - CWE-200 (exposición de información sensible a un actor no autorizado), como causa principal.
  - CWE-307 (restricción insuficiente de intentos de autenticación/consulta), por la ausencia total de rate limiting.
  - CWE-862 (ausencia de autorización) en el propio endpoint.

## Qué debería cambiar

1. Exigir autenticación, o como mínimo un CAPTCHA, antes de permitir cualquier consulta a este endpoint.
2. Implementar límite de peticiones por IP o por sesión (unos pocos intentos por hora son más que suficientes para un caso de uso legítimo de "ayuda").
3. Que la respuesta pública deje de distinguir el tipo de directorio interno. Un resultado binario genérico (existe / no existe) ya es más de lo que debería darse sin autenticar, pero al menos no filtra el perfil del titular.
4. Revisar si hay registro de consultas masivas a este endpoint, para detectar abuso retroactivo mientras no esté corregido.
5. Retirar de producción el scaffold por defecto de ASP.NET Web API y revisar el resto de servicios bajo la misma ruta en busca de exposiciones similares.

## Línea temporal

| Fecha | Hecho |
|---|---|
| 2026-11-13 | Descubrimiento y verificación del comportamiento diferencial del endpoint |
| 2026-11-13 | Envío a INCIBE-CERT, solicitando coordinación con el organismo afectado |
| (pendiente) | Contacto directo con el Gobierno de Canarias y actualización de esta entrada |

---

*Divulgación responsable: las pruebas se limitaron a confirmar el comportamiento del endpoint con identificadores de control, sin consultar documentos reales de terceros ni extraer datos de personas identificables. La URL completa del endpoint se ha tapado en la captura del código. Esta entrada no incluye ningún script de enumeración ni lista de identificadores probados.*
