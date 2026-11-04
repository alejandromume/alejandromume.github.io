---
title: "Movistar: su nuevísimo router Wi-Fi 7 deja ver su contraseña a cualquiera en la red"
date: 2026-11-04 09:00:00 +0000
author: "Alejandro Müller Medina"
severity: high
cvss: "~8.8 (CVSS v4.0, estimación provisional)"
scope: "Panel de administración web local del router"
tags: [hardware, router, movistar, askey, divulgacion-responsable, criptografia]
---

El Router Smart WiFi 7 que Movistar instala con sus ofertas de fibra más recientes es un Askey RTF8316VW. Su panel de administración (la página donde se configura el router, normalmente en `192.168.1.1`) se sirve por HTTP, sin TLS. TLS es el cifrado que hace que una web empiece por `https://` y que el candado aparezca en el navegador: sin él, cualquier dato que viaje entre tu equipo y el router puede leerse por el camino. El formulario de login intenta compensar esa falta con una función JavaScript que "ofusca" la contraseña antes de enviarla. La función existe, se ejecuta y el resultado se envía. Pero la contraseña original también.

Este hallazgo está reportado a **INCIBE-CERT** para coordinar la divulgación con el fabricante y con el operador. Todavía no he contactado directamente con Askey ni con Movistar. Por eso esta entrada cuenta qué falla y cómo se comprobó, y no incluye ninguna herramienta lista para usar contra equipos ajenos. Se actualizará cuando haya parche o identificador CVE.

## El dispositivo

```text
Fabricante     Askey Computer Corp.
Modelo         Router Smart WiFi 7 RTF8316VW (según etiqueta)
Acceso         GPON / XGS-PON  (fibra óptica hasta el domicilio)
Distribución   Movistar / Telefónica (España)
HTTPS          opcional, certificado autofirmado, no forzado
```

## Hallazgo 1: dos contraseñas por el precio de una

El login está en `/te_acceso_router.asp` y envía los datos por POST a `/cgi-bin/te_acceso_router.cgi`. El archivo `te_util.js` incluye esta función, que se llama justo antes del envío:

```js
function mess_userpass(str)
{
    return str.split('').map(function(c){
        return String.fromCharCode(c.charCodeAt(0) ^ 0x1f);
    }).join('');
}
```

Es un XOR con una clave fija de un byte. El XOR es una operación matemática simple que combina cada carácter con una clave, y su particularidad es que aplicarla dos veces con la misma clave devuelve el texto original. Aquí la clave es siempre la misma (0x1f), así que quien vea el valor "ofuscado" lo revierte en un instante, sin necesidad de probar nada. Como protección vale poco, pero el problema real es otro. El input original del formulario (`name="Password"`) no se vacía ni se deshabilita, y viaja en el mismo cuerpo de la petición:

```http
POST /cgi-bin/te_acceso_router.cgi HTTP/1.1
Host: 192.168.1.1
Content-Type: application/x-www-form-urlencoded

Password=Hunter2!&curWebPage=...&loginUsername=~{rvq&loginPassword=Wjqkzm->
         └───────┬──┘                                       └─────┬─────┘
         texto claro                                         "ofuscada" con XOR
```

Los valores de arriba son de ejemplo. En el equipo de pruebas aparecía exactamente esa estructura con mis credenciales reales. La parte "ofuscada" sobra: la contraseña ya está en el campo `Password`, sin transformar.

Para comprobar que la función hace lo que dice, basta con un intérprete de Python:

```python
>>> s = "Hunter2!"
>>> o = "".join(chr(ord(c) ^ 0x1f) for c in s)
>>> o
'Wjqkzm->'
>>> "".join(chr(ord(c) ^ 0x1f) for c in o)
'Hunter2!'
```

Un atacante que capture el tráfico de forma pasiva (misma red local, o cualquier salto intermedio mientras no haya TLS) lee la contraseña de administrador tal cual. No tiene que modificar nada ni invertir ninguna función.

## Hallazgo 2: el login se puede sustituir sin cambiar la barra de direcciones

La página raíz del panel monta un frameset con JavaScript inline, también por HTTP:

```js
document.writeln("<frameset rows='*,60px' border='0' frameborder='0'>");
document.writeln("   <frame id='mainFrame' src='/te_wifi.asp' name='mainFrame' ...>");
document.writeln("   <frame id='footerfrm' src='te_footer.asp' name='footerfrm' ...>");
document.writeln("</frameset>");
```

El login legítimo se carga en `mainFrame`. Si alguien en posición de red puede reescribir esa respuesta, cambia el `src` de ese frame por una página suya. A esto se le llama ataque MITM (man in the middle, "hombre en el medio"): el atacante se coloca entre la víctima y el router, por ejemplo falseando la tabla ARP de la red local o montando un punto de acceso Wi-Fi que imita al legítimo, y desde ahí puede leer o incluso modificar el tráfico que pasa por sus manos. La navegación de nivel superior no cambia, así que el navegador sigue mostrando `192.168.1.1`:

```text
  Navegador de la víctima                   Atacante (MITM)               Router
  ───────────────────────                   ───────────────               ──────
  GET http://192.168.1.1/  ───────────────────────►  ─────────────────────►
                           ◄───────────────────────  ◄─────────────────────  frameset original
                                   │
                          reescribe mainFrame.src
                                   │
  barra de direcciones: 192.168.1.1     ┌──────────────────────────────┐
  ┌────────────────────────────────────┤ frame interno: login clonado │
  │ usuario  [______]                   │ servido desde el atacante    │
  │ clave    [______]                   └──────────────────────────────┘
  └────────────────────────────────────
```

La víctima ve la URL correcta y un formulario idéntico al original. Ni siquiera llega a hablar con el router real: las credenciales se las queda el atacante en cuanto las escribe.

La cabecera `Content-Security-Policy` (CSP) de esa respuesta no ayuda aquí. Es un mecanismo con el que una web le dice al navegador qué scripts o estilos puede ejecutar, pensado para frenar ataques donde alguien cuela código ajeno dentro de una página legítima. Pero viaja por el mismo canal HTTP, así que quien reescribe el cuerpo reescribe también la cabecera, o la elimina. Una CSP protege frente a contenido inyectado dentro de un canal que ya es de confianza. No frente a alguien que controla el canal entero.

En la prueba de este segundo punto no desplegué un MITM completo. Verifiqué que la respuesta se compone como se describe, sin ninguna protección de integridad, y que el razonamiento sobre el tráfico reescrito se sostiene. Lo dejo dicho tal cual porque la diferencia con el hallazgo 1, que sí reproduje de principio a fin, importa.

## Un nonce que no cambia

Dentro de esa CSP hay un detalle adicional. Las respuestas del panel llevan un nonce para `script-src` y `style-src`, un valor que debería generarse al azar en cada respuesta para que un script solo se ejecute si lleva ese código exacto, como una contraseña de un solo uso para cada carga de página. Este es el valor, decodificado:

```console
$ echo VGVsZWZvbmljYS1jc3Atbm9uY2U= | base64 -d
Telefonica-csp-nonce
```

Es el mismo en todas las respuestas que observé. Un nonce sirve si es impredecible y distinto en cada respuesta. Con uno fijo, cualquier script que un atacante consiga inyectar puede llevar ese valor y la política lo acepta.

No lo cuento como vulnerabilidad aparte, porque no he encontrado un punto de inyección XSS que lo haga explotable. Queda como una señal más de cómo está montada la seguridad del panel.

## Requisitos para explotarlo

| | Hallazgo 1 | Hallazgo 2 |
|---|---|---|
| Posición en la red | Intercepción pasiva | Intercepción y modificación (MITM activo) |
| Cuenta o privilegio previo | Ninguno | Ninguno |
| Acción de la víctima | Iniciar sesión mientras se captura | Abrir `http://192.168.1.1/` |
| Condición común | Panel por HTTP (estado de fábrica) | Panel por HTTP (estado de fábrica) |

Aunque se active el acceso por HTTPS, el HTTP puede seguir disponible en paralelo sin redirección. No lo he verificado en este modelo. No he evaluado tampoco si el panel es alcanzable desde la WAN. Si lo fuera, el vector pasaría de red adyacente a remoto y la puntuación subiría.

## Impacto

Con la contraseña de administrador se controla el router: configuración Wi-Fi, DNS, reenvío de puertos y, en la práctica, el tráfico de toda la red doméstica. Los dos hallazgos llegan a la misma credencial por caminos distintos, uno solo escuchando y otro suplantando el formulario.

Para medir la gravedad de este tipo de fallos se usan dos escalas estándar en el sector. CVSS da una puntuación de 0 a 10 según qué tan fácil es explotar el fallo y qué daño causa (8.8 se considera "alto"). CWE clasifica el tipo de error de fondo, como un catálogo de categorías de fallos de seguridad conocidos.

```text
CVSS v4.0   CVSS:4.0/AV:A/AC:L/AT:N/PR:N/UI:N/VC:H/VI:H/VA:H/SC:N/SI:N/SA:N
CVSS v3.1   CVSS:3.1/AV:A/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H   →  8.8 (Alto)
CWE         CWE-319  transmisión en claro de información sensible (causa raíz)
            CWE-327  algoritmo criptográfico roto o arriesgado
            CWE-300  canal accesible por intermediarios (MITM)
```

La puntuación v4.0 es una estimación mía, pendiente del cálculo oficial de FIRST.

## Qué debería cambiar

1. **HTTPS por defecto**, con redirección obligatoria desde HTTP y el puerto 80 realmente cerrado. Es lo único que cierra también el hallazgo 2.
2. **Dejar de enviar el campo `Password` en claro.** Se vacía o se deshabilita antes del envío.
3. **Sustituir el XOR por un mecanismo criptográfico de verdad**, sobre un canal cifrado.
4. **Generar el nonce de la CSP de forma aleatoria en cada respuesta**, en el servidor.

Los puntos 2 a 4 mejoran cosas, pero ninguno sustituye al primero. Sin TLS, cualquier formulario, frame, script o cabecera del panel puede reescribirse en tránsito.

## Mientras tanto, para quien tenga uno

- Entrar al panel lo menos posible, y solo desde un equipo de confianza conectado por cable o a una Wi-Fi propia.
- Si hay opción de acceder por HTTPS, usarla (el navegador avisará del certificado autofirmado, es esperable).
- No reutilizar la contraseña del router en ningún otro sitio.
- No abrir el panel desde una Wi-Fi pública ni compartida.

## Línea temporal

| Fecha | Hecho |
|---|---|
| 2026-10-29 | Descubrimiento y verificación del hallazgo 1 en mi propio equipo |
| 2026-11-04 | Informe a INCIBE-CERT |
| (pendiente) | Traslado al fabricante y al operador, y actualización de esta entrada |

---

*Divulgación responsable: las pruebas se hicieron solo contra mi propio dispositivo. Los valores de las credenciales mostrados arriba son de ejemplo, y no se publican herramientas ni páginas de phishing listas para usar mientras la coordinación siga abierta.*
