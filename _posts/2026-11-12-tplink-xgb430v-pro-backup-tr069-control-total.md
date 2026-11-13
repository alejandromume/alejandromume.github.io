---
title: "TP-Link XGB430v Pro: de una cuenta de \"usuario\" al control remoto permanente del router"
date: 2026-11-12 09:00:00 +0000
author: "Alejandro Müller Medina"
severity: critical
cvss: "9.3 (CVSS v4.0, con gestión remota activa)"
scope: "Backup de configuración, protocolo TR-069/CWMP y canal de gestión de Digi"
tags: [hardware, router, tp-link, divulgacion-responsable, tr-069, escalada-privilegios]
---

En el [artículo anterior sobre este router]({{ '/2026/10/tplink-xgb430v-pro-cifrado-inseguro/' | relative_url }}) ya quedó claro que el cifrado propio del panel web no aguanta. Lo que no contaba ahí es a dónde se llega desde el otro lado: una cuenta de "usuario" estándar, la que tiene cualquier miembro de la casa que no es el administrador, es suficiente para acabar con control remoto permanente del router. Y de paso, con las credenciales de la línea de fibra del cliente de Digi.

Dos informes nuevos, enviados a INCIBE-CERT el mismo día, describen dos fallos que por separado ya son serios y que encadenados forman algo bastante peor: un camino completo desde "tengo acceso de invitado a esta wifi" hasta "controlo este router para siempre, aunque lo reinicien".

## El dispositivo

- **Fabricante / modelo:** TP-Link XGB430v Pro (BE7200 Dual-Band Wi-Fi 7, XGS-PON), hardware v1.0.
- **Firmware probado:** 0.2.0 3.2.1 v60c3.0 Build 250808 Rel.72518n.
- **Distribución:** ISP Digi, bajo marca comercial DIGIFIBRA.

Mismo equipo que en la entrada anterior. Esta vez el punto de entrada no es la red, es el propio backup de configuración que el router deja descargar.

## Paso 1: el backup que cualquier "usuario" puede bajarse

El panel del XGB430v Pro distingue entre cuentas "admin" y cuentas "user". Es razonable esperar que una cuenta de nivel bajo no tenga acceso a nada que sea exclusivo del administrador. El endpoint de descarga del backup de configuración (`/cgi/..._backupcfg.bin`) no respeta esa distinción: lo sirve igual a una cuenta "user" que a una "admin". De hecho, la herramienta pública que automatiza este proceso exige explícitamente ejecutarse con una cuenta "user", porque el formato que devuelve el firmware para cuentas "admin" es distinto:

```python
if js_variable(merged, "userType") == "Admin":
    raise RouterError("Este firmware exporta otro formato para Admin. "
                       "Ejecuta con la cuenta user.")
```

Ese backup descargado está cifrado, pero con DES-ECB (un algoritmo retirado por el NIST en 2005) y con una clave de 56 bits que no depende del dispositivo concreto, ni del número de serie, ni de ninguna configuración del usuario. Depende solo del ProductID público del modelo:

```python
PRODUCT_ID = 1083710977   # 0x40A07BC1, fijo para el XGB430v Pro
BASE_KEY   = 0x748da50bf93e2dcf   # constante en el firmware

def derive_key(product_id):
    ascii_hex = f"{product_id:08x}".encode("ascii")
    return bytes(a ^ b for a, b in zip(BASE_KEY, ascii_hex))
```

Esa clave es idéntica en todas las unidades de este modelo que existen. No hay secreto de fábrica, no hay nada por dispositivo. Cualquiera con la constante `BASE_KEY` (pública, circula en repositorios abiertos) descifra el backup de cualquier XGB430v Pro del mundo en menos de un segundo.

Dentro del XML descifrado hay tres cosas que una cuenta "user" no debería poder ver jamás:

- Usuario y contraseña del **administrador** del panel (`X_TP_UserCfg` → `AdminName`, `AdminPwd`).
- Usuario y contraseña **PPPoE de la línea Digi**, los que el router usa para autenticarse contra la red del operador.
- Credenciales SIP/VoIP, si están configuradas.

Con una sesión de invitado en la wifi y cinco segundos de ejecución de un script, cualquiera obtiene la contraseña de administrador del router y las credenciales de la fibra del cliente. Sin ARP spoofing, sin capturar nada, sin estar en posición de red activa en ningún momento del proceso.

## Lo que ya se puede hacer solo con esto

Antes de seguir con la segunda mitad de la cadena, merece la pena parar aquí un momento, porque esta parte sola ya tiene consecuencias fuera del router:

- Las credenciales PPPoE son las de la línea del cliente frente a la red de Digi. Con ellas, alguien puede autenticarse directamente contra el operador usando la identidad de ese cliente, con las implicaciones que eso tenga según la política de la red del ISP.
- Digi no da al usuario ninguna forma sencilla de cambiar esas credenciales sin pasar por el operador. Si se filtran, no es un problema que el cliente pueda resolver cambiando una contraseña desde el panel.
- Y con la contraseña de administrador ya en la mano, el router entero queda abierto: wifi, configuración de red, y el punto que sigue.

## Paso 2: el router confía en cualquiera que diga ser su gestor remoto

El segundo informe ataca algo distinto: el protocolo TR-069/CWMP, que es el que usan los operadores para gestionar remotamente el router (actualizar firmware, cambiar parámetros, diagnosticar averías) sin que el usuario tenga que tocar nada.

TR-069 define que el CPE (el router) se autentica frente al servidor de gestión, el ACS. Lo que no implementa este firmware es la otra mitad: en ningún momento el router comprueba que el ACS al que se conecta sea quien dice ser. No hay certificado, no hay verificación de identidad, no hay nada. Si el parámetro `Device.ManagementServer.URL` apunta a un servidor, el router se conecta ahí y obedece.

Y ese parámetro es de escritura, modificable desde el panel web por cualquiera que tenga sesión de administrador. La sesión de administrador que, un paso antes, se consiguió descifrando el backup.

La cadena completa, verificada de principio a fin sobre un dispositivo real:

1. **Cuenta "user"** → backup → DES-ECB con clave pública → contraseña de administrador en claro (paso 1).
2. Con esa contraseña, entrar al panel y cambiar `Device.ManagementServer.URL` a un servidor propio, montado como ACS falso.
3. El router envía automáticamente un mensaje `INFORM` con `EventCode "0 BOOTSTRAP"` al nuevo ACS, iniciando sesión con él como si fuera legítimo.
4. Ese ACS puede entonces leer el árbol de datos completo del dispositivo: **8.298 parámetros**, de los cuales más de **4.000 son de escritura**.

Todo esto se probó con un servidor HTTP casero en Python escuchando en el puerto 7547, implementando únicamente los cuatro mensajes TR-069 necesarios (`InformResponse`, `GetParameterNames`, `GetParameterValues`, `SetParameterValues`). Cambiar la contraseña de una red wifi desde ese servidor de prueba y verla reflejada al instante en el panel del router no requirió nada más sofisticado que eso.

## Ejemplos reales de lo que permite esta cadena

Con un ACS controlado por el atacante y 4.000 parámetros de escritura disponibles, esto deja de ser un ejercicio teórico. Algunas cosas concretas que se verificaron o que se derivan directamente de los parámetros expuestos:

- **Cambiar la contraseña de cualquier red wifi del router** (`Device.WiFi.AccessPoint.{1-16}.Security.KeyPassphrase`): la principal, la de invitados, la de backhaul EasyMesh, la de modo puente, y el PIN WPS. Se demostró en la prueba de concepto, cambiando una contraseña wifi y viéndola reflejada en el panel sin que el router pidiera ninguna confirmación.
- **Redirigir el DNS de toda la LAN** (`Device.DNS.Relay.Forwarding.1.DNSServer`), que es RW: cualquier dispositivo de la casa que resuelva nombres a través del router pasa a depender de un servidor DNS elegido por el atacante. Eso habilita phishing selectivo, intercepción de tráfico o simplemente vigilancia de qué dominios visita cada dispositivo.
- **Tocar firewall, redirección de puertos y DMZ**: exponer a Internet un NAS, una cámara IP o cualquier equipo interno que el propietario nunca quiso publicar.
- **Reiniciar el dispositivo o forzar una descarga de firmware** (`Reboot`, `Download`) en cualquier momento, de forma silenciosa, sin que el administrador legítimo vea ninguna alerta.
- **Mantener el control tras un reinicio**, desactivando `Device.ManagementServer.PeriodicInformEnable` mientras se reconfirma la URL del ACS atacante. Ambos parámetros son RW, y el router acepta el cambio sin más.

Nada de esto exige haber crackeado nada, ni fuerza bruta, ni condiciones de red especiales. Exige haber pasado por el paso 1 una sola vez.

## Si la gestión remota está activa, ni hace falta estar en la LAN

En al menos una unidad analizada, la opción "Gestión remota" del panel venía activada de fábrica. Con ella activa, el panel de administración es accesible desde Internet, y todo el paso 2 se puede ejecutar completamente desde fuera, sin haber estado nunca en la red local del objetivo. El único requisito pasa a ser disponer de la contraseña de administrador, que se puede haber obtenido en cualquier momento anterior con solo un acceso breve a la LAN (la wifi de invitados de una tarde es suficiente).

Es la diferencia entre "vulnerabilidad del router de mi vecino" y "vulnerabilidad de cualquier router de este modelo con gestión remota activa, desde cualquier parte del mundo".

## El canal de gestión que Digi mantiene y que nadie puede apagar

Durante las pruebas apareció algo que no estaba en el guion. El router mantiene una interfaz de red dedicada, llamada `ipoe_gpon_2_0_s` (VLAN ID 2, servicio `VOIP,TR069`), que aparece en el panel como "Connected" de forma permanente y sin ningún control para desconectarla. No es la conexión a Internet del usuario, que es otra interfaz distinta (`pppoe_gpon_20_1`, VLAN 20) y sí se puede desconectar.

Esa VLAN 2 inyecta por DHCP (Option 43) la URL del ACS legítimo de Digi en cada arranque, porque el firmware trae la directiva `INCLUDE_TR069_ACSURL_FROM_DHCP=1` activa. El efecto práctico: aunque un atacante haya redirigido `Device.ManagementServer.URL` a su propio servidor, en el siguiente reinicio el ACS de Digi vuelve a imponer su propia URL, sin que el usuario pueda evitarlo sin modificar el firmware.

La URL real del ACS de Digi, confirmada durante las pruebas, es `https://tr069-gpon.digimobil.es:9191`. Lo relevante no es tanto esa dirección en sí, sino lo que implica: Digi mantiene, sobre todos los routers de sus clientes, un canal de gestión permanente, no revocable por el usuario y no advertido en ninguna parte, con capacidad de tocar cualquiera de esos más de 4.000 parámetros RW. Eso convierte al propio operador en un actor con el mismo nivel de acceso que el ACS malicioso del ejemplo anterior, solo que de forma legítima y constante.

## Otras puertas que se quedaron abiertas sin que nadie las haya tocado

Revisando el JavaScript que el propio router sirve públicamente (`oid_str.js`) aparecen varias directivas de compilación activas que no se han auditado a fondo en estos informes, pero que conviene dejar anotadas porque amplían la superficie de ataque real del dispositivo:

- `INCLUDE_TR369=1`: el protocolo sucesor de TR-069, TR-369/USP, también está activo. Es razonable esperar una superficie de ataque comparable, o mayor, a la descrita aquí.
- `INCLUDE_PACKET_CAPTURE=1` junto a `INCLUDE_PKTCAP_UPLOAD=1`: el firmware puede capturar tráfico de red y subirlo a un servidor remoto configurable. Un vector de exfiltración adicional sobre todo lo que pase por la LAN.
- `INCLUDE_REMOTE_HTTP_SUPER_CTRL=1`: una función de "control super" cuyo alcance real no está documentado públicamente.
- `INCLUDE_REMOTE_TELNET=1` e `INCLUDE_REMOTE_SSH=1`: acceso remoto por terminal, también presente en el firmware.

Ninguna de estas cuatro se ha explotado ni verificado en profundidad para este reporte. Se mencionan porque existir ya dice algo: este firmware trae mucho más encendido de lo que un router doméstico necesitaría mostrar hacia fuera.

## Requisitos para explotar la cadena

**Escenario mínimo** (gestión remota desactivada):
- Haber tenido acceso a la red local del router en algún momento, por breve que sea (la wifi de invitados cuenta).
- Credenciales de cuenta "user", que en los dispositivos que distribuye Digi pueden ser las de fábrica o conocidas por otros medios.
- No se necesita cuenta de administrador, ni interacción del administrador legítimo, ni posición de red activa tipo MITM en ningún punto.

**Escenario máximo** (gestión remota activa, que se ha observado habilitada de fábrica en al menos una unidad):
- Ningún requisito de acceso físico ni local.
- Solo credenciales de administrador, obtenibles como en el escenario mínimo, y conectividad a la IP pública del router.

## Impacto y CVSS

| Hallazgo | CVSS v4.0 | Vector |
|---|---|---|
| Backup con DES-ECB y clave pública | 8.1 (Alto) | `AV:A/AC:L/AT:N/PR:L/UI:N/VC:H/VI:N/VA:N/SC:N/SI:N/SA:N` |
| TR-069 sin autenticación del ACS (gestión remota activa) | 9.3 (Crítico) | `AV:N/AC:L/AT:N/PR:L/UI:N/VC:H/VI:H/VA:H/SC:N/SI:N/SA:N` |
| TR-069 sin autenticación del ACS (solo LAN) | 8.7 (Alto) | `AV:A/AC:L/AT:N/PR:L/UI:N/VC:H/VI:H/VA:H/SC:N/SI:N/SA:N` |

CWE principales:

- **CWE-321** (clave criptográfica hardcodeada) y **CWE-326** (cifrado inadecuado, DES) en el backup.
- **CWE-269** (gestión incorrecta de privilegios): una cuenta "user" accede a datos exclusivos de "admin".
- **CWE-312** (almacenamiento en claro de información sensible): cifrado con clave pública equivale a no cifrar.
- **CWE-306** (ausencia de autenticación para una función crítica) como causa raíz del fallo en TR-069.
- **CWE-923** (restricción incorrecta del canal de comunicación al destino previsto), relacionado.

## Qué debería cambiar

1. **Restringir la descarga del backup a cuentas "admin"**, denegando a cuentas "user" con un 403 directo.
2. **Sustituir DES-ECB** por AES-256-GCM o ChaCha20-Poly1305, con clave derivada de un secreto único por dispositivo, generado en fábrica, no de constantes compartidas en el firmware.
3. **No incluir las credenciales PPPoE del ISP en un backup exportable por el usuario**, o cifrarlas en un segmento aparte que solo el operador pueda descifrar.
4. **Autenticación mutua TLS (mTLS) en TR-069**, tal como define el Amendment 5 del propio estándar: el ACS debe presentar un certificado válido y el CPE debe verificarlo antes de aceptar ninguna instrucción.
5. Mientras no haya mTLS, al menos **validar la URL del ACS contra una lista de dominios autorizados por el operador** y **avisar al administrador si `Device.ManagementServer.URL` cambia**.
6. Revisar qué hace cada una de las directivas de compilación activas (TR-369, captura de paquetes, "super control", Telnet/SSH remoto) y desactivar las que no tengan una justificación de producto clara.

Todo esto es complementario a lo que ya se pedía en el informe anterior sobre cifrado del panel: activar HTTPS por defecto y generar claves AES con una fuente de aleatoriedad real. Son puertas distintas, pero todas dan a la misma casa.

## Línea temporal

| Fecha | Hecho |
|---|---|
| 2026-10-26 | Descubrimiento y envío a INCIBE-CERT del fallo de `eval()` y almacenamiento inseguro en el panel |
| 2026-10-27 | Descubrimiento y envío a INCIBE-CERT de la clave AES predecible en el login |
| 2026-11-12 | Descubrimiento y envío a INCIBE-CERT del backup con DES-ECB y clave pública |
| 2026-11-12 | Descubrimiento y envío a INCIBE-CERT de la ausencia de autenticación del ACS en TR-069 |
| (pendiente) | Traslado al fabricante y al programa de divulgación del ISP |

---

*Divulgación responsable: los valores sensibles de las pruebas (credenciales, direcciones, capturas) están sustituidos por marcadores en los informes originales. No se publica aquí ninguna herramienta ni script listo para ejecutar contra dispositivos de terceros mientras la coordinación con el fabricante y el operador siga abierta.*
