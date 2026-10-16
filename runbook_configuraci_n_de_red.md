# Runbook: Configuración de Red para la Mano Robótica Inspire RH56DFTP-2R en NVIDIA Jetson Orin Nano

Este documento detalla el procedimiento técnico para establecer una conexión de red estable y segura entre la placa NVIDIA Jetson Orin Nano (ejecutando Ubuntu 22.04 LTS) y el controlador de la mano robótica Inspire RH56DFTP-2R. El controlador de la mano opera mediante el protocolo **Modbus TCP** y viene preconfigurado con una dirección IP estática de fábrica.

---

## 1. Contexto de Red y Requisitos

- **Protocolo de comunicación:** Modbus TCP.
- **Dirección IP de fábrica (Mano Robótica):** `192.168.123.211`
- **Puerto TCP de control:** `6000`
- **Interfaz física en la Jetson:** Puerto Ethernet (usualmente `eno1`).

Para que la Jetson Orin Nano pueda enviar comandos al controlador de la mano de forma directa a través de un cable de red (sin pasar por un router que asigne IPs automáticamente), es estrictamente necesario que ambos dispositivos pertenezcan a la misma "familia" o segmento de subred (`192.168.123.X`).

---

## 2. Procedimiento de Configuración Paso a Paso

Las siguientes instrucciones deben ejecutarse directamente en la terminal de Ubuntu de la Jetson Orin Nano.

### 2.1 Verificación del Estado Inicial de la Interfaz

Antes de modificar la red, inspeccione el estado actual de la tarjeta Ethernet física (identificada en el sistema como `eno1`):

```bash
ip addr show eno1
```
> **Nota:** Este comando permite comprobar si la interfaz está activa (estado `UP`) y confirmar que no posea una dirección IP conflictiva asignada previamente.

### 2.2 Asignación Temporal de Dirección IP

Para establecer la comunicación inmediata, asigne de manera temporal una dirección IP estática que sitúe a la Jetson dentro de la misma subred que la mano robótica:

```bash
sudo ip addr add 192.168.123.50/24 dev eno1
```
> **Explicación del comando:**
> - `sudo`: Ejecuta el comando con privilegios de administrador.
> - `ip addr add`: Instrucción para añadir una nueva dirección a la interfaz.
> - `192.168.123.50/24`: Define la IP `.50` para la Jetson, utilizando una máscara de subred de 24 bits (`255.255.255.0`). Esto le indica al sistema que cualquier dispositivo con una IP que comience en `192.168.123.` pertenece a su conexión local.
> - `dev eno1`: Especifica que esta configuración se aplica a la tarjeta Ethernet física.

Para confirmar que la IP se asignó correctamente en el sistema, vuelva a consultar la interfaz:

```bash
ip addr show eno1
```

### 2.3 Validación de la Conectividad Física (Ping)

Compruebe que el cableado es correcto y que la Jetson puede comunicarse físicamente con la mano robótica sin pérdida de datos:

```bash
ping -c 4 192.168.123.211
```
> **Explicación:** Envía exactamente 4 paquetes de prueba (`-c 4`, count) al controlador de la mano. Si la respuesta es exitosa (0% packet loss), significa que el enlace a nivel de red está operativo y el cable está en buen estado.

### 2.4 Creación de un Perfil de Red Permanente

Las direcciones IP asignadas con el comando anterior (`ip addr add`) son temporales y se borran si la Jetson se reinicia. Para garantizar que la configuración se mantenga de forma permanente, es necesario crear un perfil en **NetworkManager**:

```bash
sudo nmcli con add type ethernet ifname eno1 con-name inspire-hand ipv4.method manual ipv4.addresses 192.168.123.50/24
```
> **Explicación del comando (`nmcli`):**
> - `con add type ethernet`: Crea un nuevo perfil de conexión para una red cableada.
> - `ifname eno1`: Vincula este perfil exclusivamente al puerto físico `eno1`.
> - `con-name inspire-hand`: Asigna el nombre descriptivo "inspire-hand" al perfil para reconocerlo fácilmente en el futuro.
> - `ipv4.method manual`: Indica que la configuración no buscará un router (DHCP), sino que será estática/manual.
> - `ipv4.addresses 192.168.123.50/24`: Fija la dirección IP de forma permanente para la placa.

Para aplicar y activar formalmente este nuevo perfil sobre la tarjeta de red:

```bash
sudo nmcli con up inspire-hand
```

### 2.5 Validación del Puerto de Control Modbus TCP

El hecho de que el equipo responda al `ping` no garantiza que esté listo para recibir instrucciones de movimiento (podría estar encendido pero con el servicio de control bloqueado). La mano utiliza el puerto `6000` para escuchar las tramas del SDK en Python. 

Verifique que este puerto esté abierto usando la herramienta Netcat (`nc`):

```bash
nc -zv 192.168.123.211 6000
```
> **Explicación:**
> - `nc`: Llama a la herramienta Netcat.
> - `-z`: Modo de escaneo pasivo (Zero-I/O), verifica la apertura del puerto sin enviar datos basura que puedan confundir a la mano.
> - `-v`: Modo "verbose" para mostrar un resultado explícito y legible en la terminal.
>
> **Resultado esperado:** Debería recibir un mensaje indicando `succeeded` o `open` (Ej. *Connection to 192.168.123.211 6000 port [tcp/*] succeeded!*). Esto confirma que el controlador está a la escucha y listo para ser operado.

---

## 3. Conclusión y Siguientes Pasos

Una vez que el comando `nc` reporta el puerto `6000` como abierto, la infraestructura de red queda formalmente validada. La NVIDIA Jetson Orin Nano está ahora preparada para ejecutar los scripts de telecontrol y enviar comandos Modbus de ángulo, fuerza y velocidad a los servomotores a través de la librería Python.