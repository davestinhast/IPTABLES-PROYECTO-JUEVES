# M-FIREWALL v2.0
### Bloqueo de Facebook, YouTube y Hotmail con iptables en Kali Linux

**Autores:** Quezada / Espinola / Sanchez

---

## Explicacion simple del proyecto

Este script configura un firewall en Kali Linux usando iptables. Cuando se activa, bloquea el acceso a Facebook, YouTube y Hotmail. Ninguno de esos sitios carga mientras el firewall esta activo.

---

### Que es iptables

iptables es una herramienta del sistema operativo Linux que revisa cada paquete de red antes de que salga o entre a la maquina. Si una regla coincide con el paquete, lo acepta o lo rechaza. No hay ningun programa intermediario: el bloqueo ocurre dentro del kernel.

---

### Como se bloquean los sitios

Cuando el script se activa hace cuatro cosas al mismo tiempo:

**1. Bloqueo por IP**
Resuelve los dominios de Facebook, YouTube y Hotmail con dig y obtiene sus direcciones IP. Esas IPs se cargan en listas del kernel llamadas ipsets. Cada paquete que va hacia esas IPs es rechazado.

**2. Bloqueo por nombre de dominio en el paquete TCP (SNI)**
Cuando el navegador abre una conexion HTTPS, el primer mensaje que envia contiene el nombre del sitio en texto legible. iptables lee ese texto dentro del paquete y si encuentra "facebook.com", "youtube.com" o "hotmail.com", rechaza el paquete antes de que se complete la conexion.

**3. Bloqueo de consultas DNS**
Antes de conectarse a cualquier sitio, el navegador pregunta al servidor DNS que IP tiene ese dominio. El script bloquea esa pregunta. Sin la IP, el navegador no puede conectarse aunque eluda los otros bloqueos.

**4. Bloqueo en /etc/hosts**
El archivo /etc/hosts es consultado por el sistema operativo antes de hacer cualquier consulta DNS. El script agrega todos los dominios bloqueados apuntando a 0.0.0.0, que es una IP invalida. El sistema operativo devuelve error antes de intentar conectarse.

---

### Bloqueo cliente hacia servidor y permiso servidor hacia cliente

Cuando el navegador abre un sitio web pasan dos cosas:

1. Tu computadora **envia** una solicitud al servidor del sitio. Eso es **cliente hacia servidor**.
2. El servidor **responde** enviando el contenido de vuelta. Eso es **servidor hacia cliente**.

Un paquete es un pedazo pequeno de datos que viaja por la red. Tanto la solicitud como la respuesta estan hechas de paquetes.

El requisito del proyecto pide bloquear el paso 1 pero no el paso 2.

iptables sabe distinguir entre los dos porque el sistema operativo registra el estado de cada conexion:

- Cuando tu computadora intenta abrir una conexion nueva, el primer paquete tiene estado **NEW**.
- Cuando el servidor responde a una conexion que ya existe, esos paquetes tienen estado **ESTABLISHED**.

El script pone esta regla como la primera que iptables revisa:

```bash
iptables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT
```

Esa regla dice: si el paquete tiene estado ESTABLISHED, dejarlo pasar.

Despues de esa regla vienen los bloqueos de Facebook, YouTube y Hotmail.

Lo que ocurre con cada paquete:

- Tu computadora intenta abrir YouTube → el paquete tiene estado NEW → no coincide con la primera regla → llega a los bloqueos → rechazado.
- El servidor de YouTube responde a algo que ya estaba cargando → el paquete tiene estado ESTABLISHED → coincide con la primera regla → pasa sin ser bloqueado.

**Como demostrarlo al profesor:**

Ejecutar este comando con el firewall activo:

```bash
iptables -L FORWARD -n -v --line-numbers
```

La salida muestra todas las reglas de la cadena FORWARD en orden. La primera linea siempre es:

```
1    ACCEPT    all    state ESTABLISHED,RELATED
```

Eso prueba que la regla de permiso esta antes que los bloqueos. El numero de paquetes en esa linea aumenta cada vez que una respuesta de servidor pasa por ahi.

Para ver los paquetes rechazados:

```bash
journalctl -k | grep PM-DROP
```

Cada linea que aparece es un paquete que fue bloqueado. Son paquetes NEW, intentos de conexion nueva del cliente. Las respuestas del servidor no aparecen aqui porque fueron aceptadas antes de llegar al bloqueo.

---

### Bloqueo por MAC address

Cada tarjeta de red tiene una direccion MAC grabada en el hardware. iptables puede leer esa direccion en cada paquete que pasa por la cadena FORWARD usando el modulo xt_mac.

```bash
iptables -A PM_MACBLOCK -m mac --mac-source AA:BB:CC:DD:EE:FF -j PM_REJECT
```

Cuando un equipo con esa MAC envia cualquier paquete, iptables lo rechaza. No importa la IP que tenga ese equipo ni el puerto al que intente conectarse.

---

### Limite de conexiones simultaneas

El kernel de Linux lleva un registro de cuantas conexiones TCP activas tiene cada IP al mismo tiempo. El modulo connlimit lee ese contador. Si una IP supera el maximo configurado, el siguiente paquete de conexion nueva es rechazado.

```bash
iptables -A PM_CONNLIMIT -p tcp --dport 443 \
    -m connlimit --connlimit-above 5 --connlimit-mask 32 -j PM_REJECT
```

Si una IP ya tiene 5 conexiones abiertas al puerto 443 y trata de abrir una sexta, iptables la rechaza. Las 5 conexiones existentes siguen funcionando.

---

### Registro de paquetes rechazados

Cada vez que un paquete es rechazado, antes de rechazarlo el script lo registra en el log del kernel:

```bash
iptables -A PM_REJECT -j LOG --log-prefix "PM-DROP: " --log-level 4
```

Eso escribe una linea en el log del sistema con los datos del paquete: IP de origen, IP de destino, puerto y protocolo. Se puede ver en tiempo real con:

```bash
journalctl -k -f | grep PM-DROP
```

---

### Archivo de configuracion personalizado

Todo lo que el usuario configura en el menu (que sitios bloquear, que MACs bloquear, que limites aplicar) se guarda en:

```
/opt/mfirewall/config.conf
```

Este archivo reemplaza el uso de /etc/sysconfig/iptables que es donde normalmente se guardarian las reglas en otros sistemas. El script lee este archivo al arrancar y aplica la configuracion guardada.

Para verlo:

```bash
cat /opt/mfirewall/config.conf
```

O desde el menu del script con la opcion **[c]**.

---

## Demo: paso a paso para la presentacion

Esta secuencia cubre los 6 requisitos del proyecto en orden. Seguirla en la presentacion garantiza que el profesor vea cada punto funcionando.

### Paso 1: Iniciar el script

```bash
git clone https://github.com/davestinhast/IPTABLES-PROYECTO-JUEVES.git
cd IPTABLES-PROYECTO-JUEVES
chmod +x mfirewall.sh
sudo bash mfirewall.sh
```

### Paso 2: Ver el archivo de configuracion personalizado

Abrir una segunda terminal y mostrar que el archivo existe en la ruta personalizada del proyecto, no en `/etc/sysconfig/iptables`:

```bash
cat /opt/mfirewall/config.conf
```

Cubre el requisito: *archivo personalizado en lugar de /etc/sysconfig/iptables*

### Paso 3: Escanear la red y bloquear una MAC

En el menu principal elegir opcion **6** (Escanear red).

El script detecta todos los equipos conectados con su IP y MAC. Seleccionar un equipo de la lista con su numero. Cuando el submenu pregunte que hacer, elegir opcion **1** (Bloquear MAC).

La MAC queda guardada en `/opt/mfirewall/config.conf`.

Cubre el requisito: *bloquear acceso de equipos clientes por su direccion MAC*

Si no hay otros equipos en la red, seleccionar el equipo `0)` (el propio Kali) para demostrar el bloqueo sobre si mismo.

### Paso 4: Configurar limite de conexiones

En el menu principal elegir opcion **4** (Limite de conexiones).

Elegir `a)` Agregar y completar:
- Protocolo: `tcp`
- Puerto: `443`
- Maximo: `5`
- IP objetivo: dejar vacio para todos, o escribir la IP del equipo escaneado en el paso anterior

La regla queda guardada en `/opt/mfirewall/config.conf`.

Cubre el requisito: *limitar el numero de conexiones simultaneas*

### Paso 5: Activar el firewall con los 3 sitios bloqueados

En el menu principal elegir opcion **2** (Activar Firewall).

Activar los 3 sitios:
- `1` para Facebook
- `2` para YouTube
- `3` para Hotmail
- `A` para activar

El script ejecuta paso a paso todos los comandos iptables mostrando cada uno en un segundo terminal. Al terminar, Firefox se abre automaticamente con los 3 sitios bloqueados.

Esto cubre 3 requisitos al mismo tiempo:

**Requisito: bloquear Facebook, YouTube y Hotmail**

El script resuelve las IPs, crea ipsets, aplica SNI matching, bloquea DNS, bloquea QUIC y aplica rangos CIDR de Google.

**Requisito: bloquear cliente→servidor, permitir servidor→cliente**

En `setup_base_chains()` la primera regla que se aplica es:

```bash
iptables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT
```

Esta regla permite que las respuestas de sesiones ya existentes pasen, pero bloquea cualquier conexion nueva hacia los sitios bloqueados. El kernel rastrea el estado de cada flujo con conntrack.

**Requisito: bloqueo por MAC y limite de conexiones**

Las reglas configuradas en los pasos 3 y 4 se aplican ahora:

```bash
iptables -A PM_MACBLOCK  -m mac --mac-source XX:XX:XX:XX:XX:XX -j PM_REJECT
iptables -A PM_CONNLIMIT -s <IP> -p tcp --dport 443 \
    -m connlimit --connlimit-above 5 --connlimit-mask 32 -j PM_REJECT
```

### Paso 6: Ver el registro de paquetes rechazados

En el menu principal elegir opcion **5** (Registro de paquetes).

Muestra las entradas `PM-DROP` del kernel en tiempo real. Cada paquete rechazado aparece con IP de origen, IP de destino, puerto y protocolo.

```bash
# Tambien se puede ver directamente en terminal:
journalctl -k | grep PM-DROP
```

Cubre el requisito: *guardar un registro de todos los paquetes rechazados*

### Paso 7: Verificar el archivo de configuracion final

```bash
cat /opt/mfirewall/config.conf
```

Debe mostrar todos los settings activos: sitios bloqueados, MACs, limites de conexion e interfaces. Este archivo es la fuente de verdad del firewall, equivalente a lo que normalmente iria en `/etc/sysconfig/iptables`.

### Paso 8 (opcional): Desactivar y mostrar que vuelve a funcionar

En el menu principal elegir opcion **8** (Desactivar Firewall).

El script elimina todas las reglas iptables, restaura DNS, limpia `/etc/hosts` y restaura Firefox. Los 3 sitios vuelven a cargar normalmente, lo que confirma que el bloqueo era real y no una configuracion del navegador.

---

## Si se usa iptables para bloquear: como y donde

Todo el bloqueo ocurre en el kernel de Linux a traves de iptables. Cuando un paquete intenta llegar a Facebook, YouTube o Hotmail, iptables lo intercepta antes de que salga de la maquina y lo descarta. No hay ningun proceso de usuario en el camino del bloqueo: el paquete muere en el kernel.

**El paquete es bloqueado en el momento en que iptables evalua la cadena PM_WEBBLOCK.**

Hay cuatro formas en que iptables ejecuta el bloqueo, dependiendo del sitio:

**Forma 1: bloqueo por IP de destino (ipset)**
```bash
# iptables compara la IP de destino del paquete contra un conjunto de IPs conocidas
iptables -A PM_WEBBLOCK -p tcp --dport 443 \
    -m set --match-set PM_FACEBOOK dst -j PM_REJECT

iptables -A PM_WEBBLOCK -p tcp --dport 443 \
    -m set --match-set PM_HOTMAIL dst -j PM_REJECT

iptables -A PM_WEBBLOCK -p tcp --dport 443 \
    -m set --match-set PM_YOUTUBE dst -j PM_REJECT
```
Si la IP de destino esta en el ipset, el paquete es rechazado.

**Forma 2: bloqueo por contenido del paquete TCP (SNI matching)**
```bash
# iptables lee el payload del paquete TCP y busca el nombre del dominio en texto plano
iptables -A PM_WEBBLOCK -p tcp --dport 443 \
    -m string --string "facebook.com" --algo bm -j PM_REJECT

iptables -A PM_WEBBLOCK -p tcp --dport 443 \
    -m string --string "hotmail.com" --algo bm -j PM_REJECT

iptables -A PM_WEBBLOCK -p tcp --dport 443 \
    -m string --string "youtube.com" --algo bm -j PM_REJECT
```
El TLS ClientHello contiene el nombre del servidor en texto plano (SNI). iptables lo lee y si encuentra el dominio, rechaza el paquete antes de que se complete la conexion.

**Forma 3: bloqueo por rango de IPs CIDR (YouTube)**
```bash
# iptables bloquea rangos completos de IPs de Google/YouTube usando la flag -d con CIDR
iptables -A PM_WEBBLOCK -d 34.107.0.0/16  -j PM_REJECT
iptables -A PM_WEBBLOCK -d 34.98.0.0/16   -j PM_REJECT

ip6tables -A PM_WEBBLOCK -d 2800:3f0::/32  -j REJECT
ip6tables -A PM_WEBBLOCK -d 2001:4860::/32 -j REJECT
ip6tables -A PM_WEBBLOCK -d 2607:f8b0::/32 -j REJECT
```
iptables acepta notacion CIDR nativa en la flag `-d`. No importa que IP especifica use YouTube en ese momento: si cae dentro del rango, es rechazada.

**Forma 4: bloqueo de DNS (port 53) y QUIC (UDP 443)**
```bash
# iptables bloquea queries DNS que contengan el nombre del dominio
iptables -A PM_WEBBLOCK -p udp --dport 53 \
    -m string --hex-string "|07|youtube|03|com" --algo bm -j PM_REJECT

# iptables bloquea QUIC/HTTP3 que usa UDP en vez de TCP
iptables -A PM_WEBBLOCK -p udp --dport 443 -j PM_REJECT
```
Sin resolucion DNS, el browser no puede obtener la IP. Sin UDP 443, YouTube no puede usar HTTP3.

**En todos los casos, el paquete llega a PM_REJECT:**
```bash
iptables -A PM_REJECT -j LOG --log-prefix "PM-DROP: " --log-level 4
iptables -A PM_REJECT -p tcp -j REJECT --reject-with tcp-reset
iptables -A PM_REJECT -j REJECT --reject-with icmp-port-unreachable
```
El kernel registra el evento en el log del sistema (visible con `journalctl -k | grep PM-DROP`) y envia un TCP RST o ICMP de error al cliente. La conexion nunca llega al servidor de destino.

Las capas adicionales (proxy DNS, /etc/hosts, politica Firefox) existen para evitar que el browser evada iptables por caminos alternativos como DNS over HTTPS o cache en memoria. Pero el bloqueo en si es siempre una regla iptables ejecutada en el kernel.

---

## Que hace cada comando en la ventana de ejecucion

Cuando activas el firewall se abre una segunda ventana que muestra cada comando en tiempo real. Esta guia explica que hace cada linea.

---

### FACEBOOK — comandos de bloqueo

```bash
ipset create PM_FACEBOOK hash:ip family inet hashsize 1024 maxelem 65536 -exist
```
Crea una lista de IPs en memoria del kernel. `hash:ip` = tabla hash de direcciones IP individuales. Esta lista se llama `PM_FACEBOOK` y puede contener hasta 65536 entradas.

```bash
ipset add PM_FACEBOOK 157.240.197.35 -exist
```
Agrega una IP de Facebook a la lista. El kernel resuelve los dominios de Facebook en el momento de activacion y agrega cada IP que encuentre. `-exist` evita error si la IP ya esta en la lista.

```bash
iptables -A PM_WEBBLOCK -p tcp --dport 443 -m set --match-set PM_FACEBOOK dst -j PM_REJECT
iptables -A PM_WEBBLOCK -p tcp --dport 80  -m set --match-set PM_FACEBOOK dst -j PM_REJECT
```
Agrega reglas a la cadena `PM_WEBBLOCK`. Si el paquete va al puerto 443 (HTTPS) o 80 (HTTP) **y** su IP de destino esta en `PM_FACEBOOK`, se envia a `PM_REJECT`. El `-m set --match-set` es el modulo de ipset.

```bash
iptables -A PM_WEBBLOCK -p tcp --dport 443 -m string --string "facebook.com" --algo bm -j PM_REJECT
iptables -A PM_WEBBLOCK -p tcp --dport 80  -m string --string "facebook.com" --algo bm -j PM_REJECT
```
Bloqueo por SNI. En TLS, el primer paquete (ClientHello) contiene el nombre del servidor en texto plano. iptables usa `-m string` para buscar `facebook.com` dentro del payload del paquete. Si lo encuentra, rechaza. `--algo bm` = algoritmo Boyer-Moore (rapido para busqueda de cadenas).

```bash
iptables -A PM_WEBBLOCK -p udp --dport 53 -m string --hex-string "|08|facebook|03|com" --algo bm -j PM_REJECT
iptables -A PM_WEBBLOCK -p tcp --dport 53 -m string --hex-string "|08|facebook|03|com" --algo bm -j PM_REJECT
```
Bloqueo DNS. Las consultas DNS tienen un formato binario (wire protocol): cada etiqueta va precedida de su longitud en hex. `|08|` = longitud 8, `facebook` = los 8 bytes del nombre, `|03|` = longitud 3, `com` = los 3 bytes del TLD. iptables busca ese patron binario exacto dentro del paquete UDP/TCP al puerto 53 (DNS).

---

### YOUTUBE — comandos adicionales que Facebook no necesita

```bash
iptables -A PM_WEBBLOCK -p udp --dport 443 -j PM_REJECT
```
Bloquea QUIC / HTTP3. YouTube usa el protocolo QUIC que corre sobre UDP puerto 443 (en vez de TCP). Sin esta regla, el browser descarga videos por QUIC aunque iptables bloquee TCP 443.

```bash
iptables -A PM_WEBBLOCK -d 34.107.0.0/16 -j PM_REJECT
iptables -A PM_WEBBLOCK -d 34.98.0.0/16  -j PM_REJECT
```
Bloqueo por rango CIDR. YouTube usa CDN de Google con rangos IP muy amplios. La flag `-d` con notacion CIDR bloquea un rango entero de IPs. Si la IP de destino cae dentro del rango `34.107.0.0/16` (65536 IPs), el paquete es rechazado sin importar el puerto.

```bash
ip6tables -A PM_WEBBLOCK -d 2800:3f0::/32 -j REJECT
ip6tables -A PM_WEBBLOCK -d 2001:4860::/32 -j REJECT
```
Lo mismo para IPv6. Firefox prefiere IPv6 cuando esta disponible y YouTube tiene CDN en rangos IPv6 de Google. `ip6tables` es el equivalente de iptables para trafico IPv6.

---

### HOTMAIL — mismo patron que Facebook

Los comandos son identicos en estructura a Facebook pero con los dominios y IPs de Microsoft:
- `PM_HOTMAIL` = ipset con IPs de outlook.live.com, hotmail.com, microsoftonline.com
- SNI matching para `hotmail.com`, `outlook.com`, `microsoftonline.com`, `live.com`
- DNS hex blocking para los mismos dominios

---

### OTROS COMANDOS (infraestructura del firewall)

```bash
iptables -N PM_REJECT
iptables -N PM_WEBBLOCK
iptables -N PM_MACBLOCK
iptables -N PM_CONNLIMIT
```
`-N` crea una cadena personalizada vacia. Las cadenas son listas de reglas que se evaluan en orden. Las cadenas propias permiten organizar las reglas por funcion.

```bash
iptables -A PM_REJECT -j LOG --log-prefix "PM-DROP: " --log-level 4
iptables -A PM_REJECT -p tcp -j REJECT --reject-with tcp-reset
iptables -A PM_REJECT -j REJECT --reject-with icmp-port-unreachable
```
`PM_REJECT` es la cadena final. Primero registra el paquete en el kernel con el prefijo `PM-DROP:` (visible en `journalctl -k | grep PM-DROP`). Luego envia TCP RST para conexiones TCP o ICMP port-unreachable para UDP. El cliente recibe un error inmediato en vez de esperar timeout.

```bash
iptables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT
```
Permite las respuestas de sesiones ya abiertas. `ESTABLISHED` = paquetes de una conexion en curso. `RELATED` = paquetes relacionados (como ICMP de error para una conexion TCP). Sin esta regla, el servidor no podria enviar respuestas a los clientes. Esta regla va PRIMERA para que las respuestas pasen antes de que WEBBLOCK las evalúe.

```bash
iptables -A FORWARD -j PM_MACBLOCK
iptables -A FORWARD -j PM_CONNLIMIT
iptables -A FORWARD -j PM_WEBBLOCK
iptables -A OUTPUT  -j PM_CONNLIMIT
iptables -A OUTPUT  -j PM_WEBBLOCK
```
Engancha las cadenas personalizadas en las cadenas del sistema. `FORWARD` = trafico que pasa a traves de Kali (de clientes). `OUTPUT` = trafico que genera el propio Kali. `-j PM_WEBBLOCK` significa "si el paquete llega aqui, evaluarlo contra PM_WEBBLOCK".

```bash
iptables -A PM_MACBLOCK -m mac --mac-source AA:BB:CC:DD:EE:FF -j PM_REJECT
```
Bloqueo por MAC. `-m mac` carga el modulo `xt_mac` que lee la direccion MAC del frame Ethernet. Si la MAC de origen coincide, el paquete va a PM_REJECT. Solo funciona en FORWARD porque en OUTPUT el trafico ya salio de la tarjeta de red y no tiene MAC visible.

```bash
iptables -A PM_CONNLIMIT -s 192.168.1.10 -p tcp --dport 443 \
    -m connlimit --connlimit-above 5 --connlimit-mask 32 -j PM_REJECT
```
Limite de conexiones. `-m connlimit` usa conntrack para contar las conexiones activas de esa IP. `--connlimit-above 5` = si tiene mas de 5 conexiones simultaneas al puerto 443, rechazar la siguiente. `--connlimit-mask 32` = contar por IP individual (no por subred).

```bash
iptables -t nat -A OUTPUT    -p udp --dport 53 -j REDIRECT --to-ports 53
iptables -t nat -A PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 53
```
NAT REDIRECT para el proxy DNS. Redirige TODAS las consultas DNS (puerto 53) al proxy local que corre en 127.0.0.1:53. Asi aunque un equipo configure otro servidor DNS, las consultas igual pasan por el proxy del firewall. `-t nat` = tabla NAT, no la tabla filter.

```bash
iptables -t nat -A OUTPUT -p udp --dport 53 -m mark --mark 0x1CA3 -j RETURN
```
Anti-loop del proxy DNS. El proxy cuando hace consultas al servidor DNS real marca los paquetes con `0x1CA3`. Esta regla los deja pasar sin redirigir, evitando que el proxy se llame a si mismo infinitamente.

---

## Opciones del menu: para que sirve cada una y como usarlas

El menu principal esta ordenado como una secuencia de pasos. Ejecutarlos en orden cubre todos los requisitos del proyecto.

---

### PASO 1 — Configurar interfaces WAN / LAN  `[tecla 1]`

**Para que sirve:**
Le indica al script cual tarjeta de red de Kali es la que conecta a internet (WAN) y cual conecta a los equipos de la red local (LAN). Con una sola tarjeta de red (como eth0), ambas son la misma.

Esta informacion se guarda en `/opt/mfirewall/config.conf` y se muestra en el dashboard. Las reglas iptables funcionan independientemente de este valor, pero el profesor lo ve en pantalla.

**Como usarlo:**
Al entrar, el script detecta automaticamente la interfaz activa con `ip route get 8.8.8.8`. Si la deteccion es correcta, presionar `0` para volver. Si no, usar `m` para escribirla manualmente o `c` para borrar y volver a detectar.

Opciones disponibles dentro del menu:
- `a` Re-detectar automaticamente
- `m` Cambiar manualmente
- `c` Limpiar / borrar lo guardado
- `0` Volver al menu principal

---

### PASO 2 — Activar Firewall  `[tecla 2]`

**Para que sirve:**
Crea todas las cadenas iptables, resuelve las IPs de los sitios, aplica todas las capas de bloqueo (ipset, SNI, DNS, QUIC, CIDR, IPv6, /etc/hosts, proxy DNS, politica Firefox) y abre el navegador con los 3 sitios bloqueados para demostrar que funciona.

Este paso cubre 4 requisitos del proyecto al mismo tiempo:
- Bloqueo de Facebook, YouTube y Hotmail
- Bloqueo cliente→servidor con ESTABLISHED,RELATED para servidor→cliente
- Aplicacion de las reglas MAC y connlimit configuradas en pasos anteriores
- Log de paquetes rechazados (PM-DROP activo desde el momento de activacion)

**Como usarlo:**
Dentro del wizard, activar o desactivar cada sitio con su numero:
- `1` Facebook
- `2` YouTube
- `3` Hotmail

Cuando los 3 muestren `[ACTIVO]`, presionar `A` para aplicar todas las reglas. El script ejecuta los comandos iptables uno por uno y al terminar abre Firefox con los 3 sitios automaticamente.

---

### PASO 3 — Bloqueo por MAC address  `[tecla 3]`

**Para que sirve:**
Bloquea equipos especificos por su direccion MAC de hardware. iptables usa el modulo `xt_mac` para leer la MAC del frame Ethernet de cada paquete que llega a la cadena FORWARD. Si la MAC coincide con una de la lista, el paquete es rechazado y se registra en el log.

```bash
iptables -A PM_MACBLOCK -m mac --mac-source AA:BB:CC:DD:EE:FF -j PM_REJECT
```

El equipo bloqueado no puede enviar ningun paquete a traves de este servidor.

**Como usarlo:**
- `a` Agregar: muestra los equipos detectados en la red con su IP y MAC. Elegir un numero de la lista para bloquear esa MAC. No hay que escribir nada manualmente.
- `d` Eliminar: muestra las MACs guardadas con numeros. Elegir cual borrar.
- `0` Volver

Si no hay otros equipos en la red, el propio Kali aparece en la lista y puede bloquearse a si mismo para demostrar el requisito.

---

### PASO 4 — Limite de conexiones simultaneas  `[tecla 4]`

**Para que sirve:**
Limita cuantas conexiones TCP/UDP puede abrir cada IP al mismo tiempo hacia un puerto especifico. El kernel cuenta las conexiones activas usando conntrack. Si una IP supera el limite, cada paquete nuevo es rechazado.

```bash
iptables -A PM_CONNLIMIT -p tcp --dport 443 \
    -m connlimit --connlimit-above 5 --connlimit-mask 32 -j PM_REJECT
```

Se puede aplicar a todos los equipos de la red o solo a una IP especifica.

**Como usarlo:**
- `a` Agregar: pide protocolo (`tcp`/`udp`), puerto (`443`), maximo de conexiones (`5`) e IP objetivo (dejar vacio para todos, o escribir la IP de un equipo concreto).
- `d` Eliminar: muestra las reglas con numeros. Elegir cual borrar.
- `0` Volver

---

### PASO 5 — Ver registro de paquetes bloqueados  `[tecla 5]`

**Para que sirve:**
Muestra en pantalla cada paquete que iptables ha rechazado. Cada rechazo queda registrado en el log del kernel con el prefijo `PM-DROP:`, incluyendo IP de origen, IP de destino, puerto y protocolo.

Esta es la evidencia de que el firewall esta funcionando. El profesor puede ver en tiempo real que los paquetes hacia Facebook, YouTube o Hotmail son interceptados y rechazados por el kernel.

```bash
# Ver directamente en terminal sin el script:
journalctl -k | grep "PM-DROP"
```

**Como usarlo:**
Presionar `5` en el menu principal. Muestra los ultimos logs del archivo interno del script y las ultimas entradas del kernel. Presionar Enter para volver al menu.

---

### PASO 6 — Escanear red  `[tecla 6]`

**Para que sirve:**
Detecta todos los equipos conectados a la red local con su IP, MAC y fabricante. Usa `arp-scan` y `ip neigh` para descubrirlos. Desde el listado se puede bloquear o limitar cualquier equipo sin escribir su MAC o IP manualmente.

**Como usarlo:**
Al entrar aparece la tabla de equipos detectados. Escribir el numero del equipo para ver el submenu de acciones:
- `1` Bloquear MAC: agrega una regla iptables que rechaza todo su trafico
- `2` Limitar conexiones: pide puerto y maximo, aplica connlimit solo para esa IP
- `3` Ambas: bloquea MAC y limita conexiones al mismo tiempo
- `0` Cancelar

Presionar `r` para reescanear la red si algun equipo nuevo se conecto.

---

### PASO 7 — Dashboard en vivo  `[tecla 7]`

**Para que sirve:**
Vista en tiempo real del estado completo del firewall. Muestra que sitios estan bloqueados, cuantas reglas iptables hay activas, que MACs estan bloqueadas, los limites de conexion configurados y los ultimos paquetes PM-DROP del kernel. Se actualiza automaticamente cada segundo.

**Como usarlo:**
Presionar `7` en el menu principal. El dashboard ocupa toda la pantalla y se refresca solo. Presionar `q` para salir y volver al menu.

---

### Opciones de mantenimiento

**`[8]` Desactivar Firewall:**
Elimina todas las reglas iptables creadas por el script, detiene el proxy DNS, restaura `/etc/resolv.conf`, limpia `/etc/hosts` y restaura la politica de Firefox. Los 3 sitios vuelven a cargar normalmente. Util para demostrar que el bloqueo era real.

**`[9]` Reset total de red:**
Hace lo mismo que desactivar pero ademas elimina todos los ipsets, cadenas personalizadas y cualquier regla residual. Equivale a un estado limpio como si el script nunca hubiera corrido. Usar si algo quedo mal despues de un error.

**`[0]` Salir:**
Cierra el script. Las reglas iptables que se aplicaron siguen activas en el kernel hasta que se desactiven con `[8]` o se reinicie el sistema.

---

## Requisitos del proyecto

1. Bloquear Facebook, YouTube y Hotmail
2. Bloquear trafico cliente→servidor, permitir respuestas servidor→cliente (ESTABLISHED,RELATED)
3. Bloqueo por MAC address
4. Limite de conexiones simultaneas por IP
5. Log de paquetes rechazados en el kernel (`PM-DROP`)
6. Archivo de configuracion personalizado `/opt/mfirewall/config.conf`
7. Minimo 10 comandos iptables

---

## Como ejecutar

```bash
git clone https://github.com/davestinhast/IPTABLES-PROYECTO-JUEVES.git
cd IPTABLES-PROYECTO-JUEVES
chmod +x mfirewall.sh
sudo bash mfirewall.sh
```

Dependencias:

```bash
apt install iptables ipset dnsutils iproute2 python3 conntrack
```

---

## Que es iptables

iptables es el firewall integrado en el kernel de Linux. Cada paquete de red que entra, sale o pasa por la maquina es inspeccionado por iptables antes de llegar a su destino. Si una regla coincide con el paquete, el kernel ejecuta la accion asignada (`ACCEPT`, `REJECT`, `DROP`, `LOG`) en ese instante, sin que el paquete toque ningun proceso de usuario.

Hay tres cadenas principales en la tabla `filter`:

- `INPUT`: paquetes que llegan a esta maquina
- `OUTPUT`: paquetes que esta maquina genera
- `FORWARD`: paquetes que pasan a traves de esta maquina (modo router/gateway)

Cuando Kali actua como gateway entre una red de clientes e internet, el trafico de los clientes pasa por `FORWARD`. El trafico del propio Kali pasa por `OUTPUT`. Este proyecto bloquea en ambas cadenas.

---

## Como se construyo el firewall: paso a paso

### Paso 1: Cadenas personalizadas

En lugar de agregar reglas directamente en FORWARD y OUTPUT, se crearon cadenas propias organizadas por funcion:

```bash
iptables -N PM_REJECT       # registra y rechaza
iptables -N PM_WEBBLOCK     # bloqueo por sitio web
iptables -N PM_MACBLOCK     # bloqueo por MAC address
iptables -N PM_CONNLIMIT    # limite de conexiones por IP
```

Cada cadena tiene una unica responsabilidad. Esto permite modificar un tipo de bloqueo sin tocar los demas.

### Paso 2: Cadena PM_REJECT

`PM_REJECT` es el objetivo final de cualquier paquete bloqueado. Hace dos cosas: primero lo registra en el kernel y luego lo descarta.

```bash
iptables -A PM_REJECT -j LOG --log-prefix "PM-DROP: " --log-level 4
iptables -A PM_REJECT -p tcp -j REJECT --reject-with tcp-reset
iptables -A PM_REJECT -j REJECT --reject-with icmp-port-unreachable
```

- `-j LOG --log-prefix "PM-DROP: "`: escribe una linea en el kernel por cada paquete rechazado. Se lee con `journalctl -k | grep PM-DROP` o `dmesg | grep PM-DROP`.
- `--reject-with tcp-reset`: para TCP envia un RST al cliente, la conexion muere de inmediato.
- `--reject-with icmp-port-unreachable`: para UDP y otros protocolos envia un ICMP de error.

### Paso 3: Bloquear cliente→servidor y permitir servidor→cliente

Este es el requisito central del proyecto. Se implementa con una sola regla iptables mas las cadenas de bloqueo, en un orden especifico.

**Que significa este requisito:**

- Cliente→servidor: el cliente intenta abrir una conexion nueva hacia Facebook, YouTube o Hotmail. Esto se bloquea.
- Servidor→cliente: el servidor responde a una sesion que ya existia antes de activar el firewall. Esto se permite.

**Por que importa el orden:**

iptables evalua las reglas en el orden en que aparecen en la cadena. La primera regla que coincide gana. Por eso la regla de ESTABLISHED,RELATED debe ir antes de las cadenas de bloqueo:

```bash
# PASO 3.1 — Primero: dejar pasar las respuestas de sesiones existentes
iptables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT
```

Esta regla dice: si el paquete pertenece a una conexion que ya existe (estado ESTABLISHED) o esta relacionado con una (RELATED), dejarlo pasar sin revisar nada mas. El kernel de Linux rastrea el estado de cada flujo TCP en una tabla interna llamada conntrack.

```bash
# PASO 3.2 — Despues: enganchar las cadenas de bloqueo para trafico nuevo
iptables -A FORWARD -j PM_MACBLOCK
iptables -A FORWARD -j PM_CONNLIMIT
iptables -A FORWARD -j PM_WEBBLOCK
iptables -A OUTPUT  -j PM_WEBBLOCK
```

El trafico nuevo (estado NEW) no coincide con la primera regla, baja por la cadena y llega a PM_WEBBLOCK donde se aplican los bloqueos.

**Estados que maneja el kernel (conntrack):**

- `NEW`: primer paquete de una conexion. El SYN del cliente cuando intenta abrir youtube.com. Esto cae en PM_WEBBLOCK y es rechazado.
- `ESTABLISHED`: la conexion ya fue aceptada y hay intercambio de datos. Pasa directo por la regla de ACCEPT.
- `RELATED`: paquete relacionado a una conexion existente, como mensajes ICMP de error o conexiones FTP de datos. Tambien pasa por ACCEPT.

**Flujo de un paquete bloqueado vs uno permitido:**

```
BLOQUEADO: cliente intenta abrir youtube.com
  SYN (estado NEW)
    v FORWARD
    v regla ESTABLISHED,RELATED → no coincide, sigue
    v PM_MACBLOCK → MAC no bloqueada, sigue
    v PM_CONNLIMIT → limite no alcanzado, sigue
    v PM_WEBBLOCK → IP de YouTube en ipset → PM_REJECT
    v kernel envia TCP RST al cliente
    v cliente recibe "conexion rechazada"

PERMITIDO: respuesta de un servidor a una sesion ya abierta
  ACK/DATA (estado ESTABLISHED)
    v FORWARD
    v regla ESTABLISHED,RELATED → coincide → ACCEPT
    v paquete llega al cliente
    (nunca llega a PM_WEBBLOCK)
```

**Verificar que la regla esta activa:**

```bash
iptables -L FORWARD -n -v --line-numbers
```

La primera linea debe ser:
```
1   ACCEPT  all  --  *  *  0.0.0.0/0  0.0.0.0/0  state ESTABLISHED,RELATED
```

Si esa regla no es la primera, el orden esta mal y el bloqueo puede romper trafico legitimo.

### Paso 4: IPv6 en paralelo

Firefox puede usar registros AAAA (IPv6) y conectarse a YouTube directamente sin pasar por ninguna regla IPv4. Se creo la misma estructura en ip6tables:

```bash
ip6tables -N PM_WEBBLOCK
ip6tables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT
ip6tables -A FORWARD -j PM_WEBBLOCK
ip6tables -A OUTPUT  -j PM_WEBBLOCK
```

Las reglas de bloqueo se duplican en ip6tables cuando se aplican.

---

## Como se bloquea cada sitio

### Facebook y Hotmail: bloqueo simple

Facebook y Hotmail tienen dominios con IPs relativamente estables. Dos tecnicas bastan:

**Tecnica 1: ipset con IPs resueltas al activar**

Al arrancar el firewall, el script resuelve cada dominio con `dig` y carga las IPs en un ipset:

```bash
ipset create PM_FACEBOOK hash:ip family inet hashsize 1024 maxelem 65536
ipset add PM_FACEBOOK <IP-resuelta>

iptables -A PM_WEBBLOCK -p tcp --dport 80  -m set --match-set PM_FACEBOOK dst -j PM_REJECT
iptables -A PM_WEBBLOCK -p tcp --dport 443 -m set --match-set PM_FACEBOOK dst -j PM_REJECT
```

Cualquier paquete TCP que vaya al puerto 80 o 443 de una IP de Facebook es rechazado antes de salir.

**Tecnica 2: SNI matching en TLS ClientHello**

El primer mensaje de una conexion HTTPS (TLS ClientHello) contiene el nombre del dominio en texto plano en el campo SNI (Server Name Indication). iptables puede buscar esa cadena dentro del payload del paquete TCP con el modulo `-m string`:

```bash
iptables -A PM_WEBBLOCK -p tcp --dport 443 \
    -m string --string "facebook.com" --algo bm -j PM_REJECT

iptables -A PM_WEBBLOCK -p tcp --dport 443 \
    -m string --string "fbcdn.net" --algo bm -j PM_REJECT

iptables -A PM_WEBBLOCK -p tcp --dport 443 \
    -m string --string "hotmail.com" --algo bm -j PM_REJECT

iptables -A PM_WEBBLOCK -p tcp --dport 443 \
    -m string --string "microsoftonline.com" --algo bm -j PM_REJECT
```

Si el nombre coincide, el paquete es rechazado antes de que se complete el handshake TLS. Facebook y Hotmail no usan ECH (Encrypted Client Hello), por lo que el SNI sigue en texto plano y el matching funciona de forma directa.

---

### YouTube: por que no basta con comandos simples

YouTube es infraestructura de Google y tiene tres caracteristicas tecnicas que hacen que los metodos que funcionan para Facebook fallen completamente:

---

#### Problema 1: Anycast

Google opera cientos de rangos IP distribuidos en todo el mundo. La misma peticion a youtube.com puede resolverse a IPs distintas segun la ubicacion geografica y la carga de los servidores en ese momento. Resolver los dominios con `dig` al activar el firewall captura las IPs actuales, pero YouTube puede conectar desde otras IPs que no estaban en el ipset.

**Efecto:** el ipset bloqueaba las IPs conocidas, pero Firefox establecia conexion con IPs nuevas de rangos como `34.107.0.0/16` que no habian sido resueltas al arrancar.

**Solucion parcial:** se agregaron bloques CIDR especificos para rangos que no captura el ipset:

```bash
iptables -A PM_WEBBLOCK -d 34.107.0.0/16  -j PM_REJECT
iptables -A PM_WEBBLOCK -d 34.98.0.0/16   -j PM_REJECT
```

---

#### Problema 2: ECH (Encrypted Client Hello)

Firefox 118 y superiores cifran el campo SNI dentro del TLS ClientHello usando una clave publica del servidor obtenida via DNS (registro HTTPS/SVCB). Esto hace que el SNI ya no sea texto plano en el paquete, por lo que `-m string --string "youtube.com"` no encuentra nada y el paquete pasa sin ser bloqueado.

**Efecto:** las reglas SNI que bloqueaban Facebook y Hotmail no bloqueaban YouTube porque el campo estaba cifrado.

**Solucion:** se deshabilitaron ECH y las prefetch de DNS en Firefox mediante politica enterprise:

```json
{
  "policies": {
    "DNSOverHTTPS": { "Enabled": false, "Locked": true },
    "Preferences": {
      "network.dns.echconfig.enabled":        { "Value": false, "Status": "locked" },
      "network.dns.use_https_rr_as_altsvc":   { "Value": false, "Status": "locked" },
      "security.tls.ech.grease_http3":        { "Value": false, "Status": "locked" },
      "network.dns.disableIPv6":              { "Value": true,  "Status": "locked" }
    }
  }
}
```

Esto fuerza a Firefox a no obtener la clave ECH, dejando el SNI en texto plano y permitiendo que el matching funcione.

---

#### Problema 3: QUIC / HTTP3

YouTube usa el protocolo QUIC que corre sobre UDP en el puerto 443 (HTTP3). Un bloqueo solo en TCP no alcanza porque QUIC no usa TCP en absoluto.

**Efecto:** aunque todas las reglas TCP estaban activas, Firefox conectaba con YouTube via UDP 443 y el trafico pasaba sin restricciones.

**Solucion:**

```bash
iptables  -A PM_WEBBLOCK -p udp --dport 443 -j PM_REJECT
ip6tables -A PM_WEBBLOCK -p udp --dport 443 -j REJECT
```

Esto bloquea completamente HTTP3/QUIC. El navegador cae automaticamente a HTTP2 sobre TCP, donde aplican el resto de reglas.

---

#### Problema 4: IPv6 con ECH activo

Firefox, cuando puede obtener registros AAAA via DNS, prefiere conectar a YouTube usando IPv6. Los rangos IPv6 de Google (AS15169) son rangos distintos de los IPv4, y cuando ECH esta activo el SNI esta cifrado sobre IPv6 igual que sobre IPv4, haciendo que el matching no funcione.

**Efecto:** todos los contadores de PM_WEBBLOCK IPv4 mostraban 0 paquetes porque Firefox estaba usando conexiones ESTABLISHED sobre `2800:3f0:4003:c03::5e` (IPv6 de Google) que ni siquiera llegaban al bloqueo de texto.

**Solucion:** bloqueo CIDR por rangos IPv6 de Google:

```bash
ip6tables -A PM_WEBBLOCK -d 2800:3f0::/32  -j REJECT
ip6tables -A PM_WEBBLOCK -d 2001:4860::/32 -j REJECT
ip6tables -A PM_WEBBLOCK -d 2607:f8b0::/32 -j REJECT
ip6tables -A PM_WEBBLOCK -d 2404:6800::/32 -j REJECT
ip6tables -A PM_WEBBLOCK -d 2a00:1450::/32 -j REJECT
```

---

#### Problema 5: DNS bypass y cache del navegador

Firefox puede guardar en cache las respuestas DNS y los contenidos de YouTube via service workers. Si alguna vez cargo YouTube con el firewall desactivado, el navegador puede servir la pagina desde cache local sin hacer ninguna peticion de red, haciendo que parezca que el bloqueo no funciona cuando en realidad si esta activo.

**Prueba:** abrir YouTube en ventana privada (Ctrl+Shift+P). La ventana privada no tiene cache. Si no carga, el bloqueo funciona correctamente a nivel de red.

**Solucion:** al activar el firewall, el script mata Firefox, borra toda la cache y los service workers de los dominios bloqueados, y luego abre Firefox de nuevo con las URLs bloqueadas para que el resultado sea visible de inmediato.

---

## Las 8 capas de bloqueo activas para YouTube

```
Peticion a youtube.com
    |
    v
[CAPA 1] /etc/hosts (inmutable con chattr +i)
         0.0.0.0 youtube.com
         :: youtube.com
         Resolucion falla directamente en el sistema operativo.
    |
    v
[CAPA 2] DNS Proxy Python3 en 127.0.0.1:53
         Devuelve NXDOMAIN para dominios bloqueados.
         /etc/resolv.conf fijado a 127.0.0.1, NetworkManager deshabilitado.
    |
    v
[CAPA 3] NAT REDIRECT: interceptar DNS a nivel kernel
         Cualquier paquete UDP/TCP al puerto 53 es redirigido al proxy,
         sin importar cual DNS este configurado en el sistema o en el browser.
         iptables -t nat -A OUTPUT -p udp --dport 53 -j REDIRECT --to-ports 53
    |
    v
[CAPA 4] DNS hex-string blocking en port 53
         Bloquea queries DNS con el nombre del dominio en wire protocol.
         iptables -A PM_WEBBLOCK -p udp --dport 53
             -m string --hex-string "|07|youtube|03|com" --algo bm -j PM_REJECT
    |
    v
[CAPA 5] SNI matching en TLS ClientHello (con ECH desactivado)
         iptables -A PM_WEBBLOCK -p tcp --dport 443
             -m string --string "youtube.com" --algo bm -j PM_REJECT
    |
    v
[CAPA 6] QUIC/HTTP3 bloqueado en UDP 443
         iptables -A PM_WEBBLOCK -p udp --dport 443 -j PM_REJECT
    |
    v
[CAPA 7] ipset con IPs resueltas al activar + CIDR especificos
         iptables -A PM_WEBBLOCK -m set --match-set PM_YOUTUBE dst -j PM_REJECT
         iptables -A PM_WEBBLOCK -d 34.107.0.0/16 -j PM_REJECT
         iptables -A PM_WEBBLOCK -d 34.98.0.0/16  -j PM_REJECT
    |
    v
[CAPA 8] CIDR IPv6: rangos completos de Google en IPv6
         ip6tables -A PM_WEBBLOCK -d 2800:3f0::/32  -j REJECT
         ip6tables -A PM_WEBBLOCK -d 2001:4860::/32 -j REJECT
         ip6tables -A PM_WEBBLOCK -d 2607:f8b0::/32 -j REJECT
    |
    v
PM_REJECT: LOG en kernel (PM-DROP) + RST al cliente
```

Cada capa existe porque la anterior puede ser evadida. La combinacion de CIDR IPv4, CIDR IPv6, QUIC bloqueado, DNS bloqueado y ECH desactivado cubre todos los vectores de bypass confirmados durante las pruebas.

---

## Todos los comandos iptables del proyecto

```bash
# Cadenas personalizadas
iptables -N PM_REJECT
iptables -N PM_WEBBLOCK
iptables -N PM_MACBLOCK
iptables -N PM_CONNLIMIT

# PM_REJECT: log en kernel + rechazo
iptables -A PM_REJECT -j LOG --log-prefix "PM-DROP: " --log-level 4
iptables -A PM_REJECT -p tcp -j REJECT --reject-with tcp-reset
iptables -A PM_REJECT -j REJECT --reject-with icmp-port-unreachable

# Enganchar cadenas en FORWARD y OUTPUT
iptables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -j PM_MACBLOCK
iptables -A FORWARD -j PM_CONNLIMIT
iptables -A FORWARD -j PM_WEBBLOCK
iptables -A OUTPUT  -j PM_WEBBLOCK

# Bloqueo por MAC address
iptables -A PM_MACBLOCK -m mac --mac-source AA:BB:CC:DD:EE:FF -j PM_REJECT

# Limite de conexiones simultaneas por IP
iptables -A PM_CONNLIMIT -p tcp --dport 443 \
    -m connlimit --connlimit-above 50 --connlimit-mask 32 -j PM_REJECT

# ipset: bloqueo por IPs resueltas al activar
iptables -A PM_WEBBLOCK -p tcp --dport 80 \
    -m set --match-set PM_YOUTUBE dst -j PM_REJECT
iptables -A PM_WEBBLOCK -p tcp --dport 443 \
    -m set --match-set PM_YOUTUBE dst -j PM_REJECT
iptables -A PM_WEBBLOCK -p tcp --dport 80 \
    -m set --match-set PM_FACEBOOK dst -j PM_REJECT
iptables -A PM_WEBBLOCK -p tcp --dport 443 \
    -m set --match-set PM_FACEBOOK dst -j PM_REJECT
iptables -A PM_WEBBLOCK -p tcp --dport 80 \
    -m set --match-set PM_HOTMAIL dst -j PM_REJECT
iptables -A PM_WEBBLOCK -p tcp --dport 443 \
    -m set --match-set PM_HOTMAIL dst -j PM_REJECT

# SNI matching en TLS ClientHello
iptables -A PM_WEBBLOCK -p tcp --dport 443 \
    -m string --string "youtube.com" --algo bm -j PM_REJECT
iptables -A PM_WEBBLOCK -p tcp --dport 443 \
    -m string --string "googlevideo.com" --algo bm -j PM_REJECT
iptables -A PM_WEBBLOCK -p tcp --dport 80 \
    -m string --string "youtube.com" --algo bm -j PM_REJECT
iptables -A PM_WEBBLOCK -p tcp --dport 443 \
    -m string --string "facebook.com" --algo bm -j PM_REJECT
iptables -A PM_WEBBLOCK -p tcp --dport 443 \
    -m string --string "fbcdn.net" --algo bm -j PM_REJECT
iptables -A PM_WEBBLOCK -p tcp --dport 443 \
    -m string --string "hotmail.com" --algo bm -j PM_REJECT
iptables -A PM_WEBBLOCK -p tcp --dport 443 \
    -m string --string "microsoftonline.com" --algo bm -j PM_REJECT

# QUIC/HTTP3: bloquear YouTube sobre UDP 443
iptables  -A PM_WEBBLOCK -p udp --dport 443 -j PM_REJECT
ip6tables -A PM_WEBBLOCK -p udp --dport 443 -j REJECT

# DNS hex-string blocking (wire protocol DNS, port 53)
iptables -A PM_WEBBLOCK -p udp --dport 53 \
    -m string --hex-string "|07|youtube|03|com" --algo bm -j PM_REJECT
iptables -A PM_WEBBLOCK -p tcp --dport 53 \
    -m string --hex-string "|07|youtube|03|com" --algo bm -j PM_REJECT
iptables -A PM_WEBBLOCK -p udp --dport 53 \
    -m string --hex-string "|0b|googlevideo|03|com" --algo bm -j PM_REJECT
iptables -A PM_WEBBLOCK -p udp --dport 53 \
    -m string --hex-string "|08|facebook|03|com" --algo bm -j PM_REJECT
iptables -A PM_WEBBLOCK -p udp --dport 53 \
    -m string --hex-string "|07|hotmail|03|com" --algo bm -j PM_REJECT
iptables -A PM_WEBBLOCK -p udp --dport 53 \
    -m string --hex-string "|07|outlook|03|com" --algo bm -j PM_REJECT

# NAT REDIRECT: interceptar todo trafico DNS al proxy local
iptables -t nat -A OUTPUT    -p udp --dport 53 -m mark --mark 0x1CA3 -j RETURN
iptables -t nat -A OUTPUT    -p tcp --dport 53 -m mark --mark 0x1CA3 -j RETURN
iptables -t nat -A OUTPUT    -p udp --dport 53 -j REDIRECT --to-ports 53
iptables -t nat -A OUTPUT    -p tcp --dport 53 -j REDIRECT --to-ports 53
iptables -t nat -A PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 53
iptables -t nat -A PREROUTING -p tcp --dport 53 -j REDIRECT --to-ports 53

# CIDR especificos para rangos Anycast de YouTube no capturados por ipset
iptables -A PM_WEBBLOCK -d 34.107.0.0/16  -j PM_REJECT
iptables -A PM_WEBBLOCK -d 34.98.0.0/16   -j PM_REJECT

# CIDR IPv6: rangos de Google/YouTube en IPv6 (AS15169)
ip6tables -A PM_WEBBLOCK -d 2800:3f0::/32  -j REJECT
ip6tables -A PM_WEBBLOCK -d 2001:4860::/32 -j REJECT
ip6tables -A PM_WEBBLOCK -d 2607:f8b0::/32 -j REJECT
ip6tables -A PM_WEBBLOCK -d 2404:6800::/32 -j REJECT
ip6tables -A PM_WEBBLOCK -d 2a00:1450::/32 -j REJECT
```

Total de comandos iptables/ip6tables activos: mas de 40.

---

## Por que se fueron agregando capas

| Problema encontrado durante pruebas | Causa tecnica | Capa que lo resolvio |
|---|---|---|
| YouTube cargaba con /etc/hosts bloqueado | Firefox cachea DNS en memoria RAM | Proxy DNS Python3 en :53 |
| Proxy DNS no bloqueaba porque NetworkManager sobreescribia resolv.conf | NM vuelve a escribir `nameserver 8.8.8.8` | `dns=none` en NetworkManager + `chattr +i` en resolv.conf |
| SNI matching fallaba para YouTube | Firefox 118+ cifra el ClientHello con ECH | Politica enterprise que deshabilita ECH en Firefox |
| YouTube conectaba por UDP sin pasar por reglas TCP | QUIC/HTTP3 usa UDP 443, no TCP | `iptables -A PM_WEBBLOCK -p udp --dport 443 -j PM_REJECT` |
| Contadores PM_WEBBLOCK IPv4 en cero, YouTube cargaba igual | Firefox usaba IPv6 (2800:3f0::/32) que no pasaba por ip6tables con reglas suficientes | CIDR IPv6 completos del AS15169 de Google |
| Algunas IPs de YouTube no estaban en el ipset | Anycast: las IPs rotan por ubicacion y carga | CIDR IPv4 adicionales: 34.107.0.0/16, 34.98.0.0/16 |
| YouTube parecia cargar en Firefox normal pero no en ventana privada | Service worker cache: Firefox sirve la pagina desde almacenamiento local | Al activar el firewall se borran los service workers y cache antes de reabrir Firefox |

---

## Verificar el bloqueo

```bash
# Ver reglas activas con contadores de paquetes
iptables -L PM_WEBBLOCK -n --line-numbers -v

# Ver ipsets cargados
ipset list PM_YOUTUBE

# Ver paquetes rechazados en el kernel en tiempo real
journalctl -k -f | grep PM-DROP

# Probar DNS bloqueado
dig +short youtube.com @127.0.0.1

# Ver log del firewall
tail -f /var/log/mfirewall.log
```

---

## Menu del script

```
1  Interfaces WAN / LAN      configurar tarjetas de red (auto-detectado)
2  Activar Firewall          elegir sitios y aplicar todas las capas
3  Bloqueo por MAC           agregar o eliminar MACs bloqueadas
4  Limite de conexiones      configurar connlimit por protocolo:puerto
5  Registro de paquetes      logs PM-DROP del kernel
6  Escanear red              ver equipos conectados con IP y MAC
7  Dashboard en vivo         monitoreo de estado en tiempo real
8  Desactivar Firewall       restaurar internet al estado original
9  Reset total               eliminar todas las reglas y restaurar internet
0  Salir
```

Al activar el firewall, el script:
1. Muestra cada paso con barra de progreso animada
2. Abre un segundo terminal con los comandos iptables ejecutados en tiempo real
3. Al terminar: mata Firefox, borra cache y service workers, abre Firefox con los 3 sitios bloqueados para que el bloqueo sea visible de inmediato

### Opcion 9: Escanear red

Detecta todos los equipos conectados a la red local y muestra su IP, MAC e identificador de fabricante. Permite seleccionar cualquier equipo de la lista para agregarlo al bloqueo por MAC sin tener que escribir la direccion manualmente.

Si no hay otros equipos conectados, el propio Kali aparece en la lista (posicion 0) con su MAC, lo que permite configurar y probar el bloqueo por MAC sobre el mismo equipo.

Fuentes de deteccion que usa el script en orden de prioridad:
- `arp-scan --localnet`: escaneo activo completo con identificador de fabricante
- `ip neigh show`: tabla ARP del kernel con dispositivos vistos recientemente
- `ip link show`: MAC e IP del propio equipo (siempre disponible)

Despues de seleccionar un equipo, su MAC queda guardada en `/opt/mfirewall/config.conf`. El bloqueo se aplica la proxima vez que se activa el firewall con la opcion 1.

---

## Estructura del proyecto

```
mfirewall.sh                  script principal
/opt/mfirewall/
    config.conf               configuracion persistente (sitios, MACs, limites, interfaces)
/var/log/mfirewall.log        log de activaciones y desactivaciones
/tmp/mfirewall_dnsproxy.py    proxy DNS Python3 (se genera al activar el firewall)
```
