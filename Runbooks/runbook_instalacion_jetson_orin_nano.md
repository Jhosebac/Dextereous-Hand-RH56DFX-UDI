# Runbook: Instalación de NVIDIA Jetson Orin Nano Developer Kit

Guía de referencia para la configuración, puesta en marcha y flasheo inicial del kit de desarrollo Jetson Orin Nano mediante NVIDIA SDK Manager y almacenamiento NVMe.

---

## 1. Documentación Oficial de Referencia

Para la configuración inicial de la Jetson Orin Nano, se tomó como base la guía oficial de NVIDIA:
- [NVIDIA Jetson Orin Nano Developer Kit Quick Start Guide](https://docs.nvidia.com/jetson/orin-nano-devkit/user-guide/latest/quick_start.html)

---

## 2. Contenido del Kit de Desarrollo

El kit oficial de desarrollo de la Jetson Orin Nano incluye:
- Módulo y placa base **Jetson Orin Nano Developer Kit**.
- Fuente de alimentación oficial con salida de **19 V @ 2.37 A**.
  > **Nota de compatibilidad eléctrica:** Incluye cables de conexión a corriente para tomas de tipo B y tipo I, seleccionables según la región.

---

## 3. Requisitos Previos: Almacenamiento

Antes de iniciar la instalación de JetPack y los paquetes del sistema operativo, es necesario contar con una unidad de almacenamiento instalada en la placa:

- **Opciones de almacenamiento:**
  - **Tarjeta microSD:** El manual recomienda clase UHS-1 con capacidad igual o superior a 64 GB.
  - **SSD NVMe M.2:** La placa integra dos ranuras M.2 compatibles con factores de forma **M.2 2280** y **M.2 2230**.

- **Configuración utilizada:**
  - Se optó por una unidad **Crucial PCIe 3.0 M.2 2280 de 512 GB**.
  - **Justificación:** Ofrece tasas de lectura/escritura muy superiores frente a una microSD UHS-1, proporcionando mejor fluidez en el desarrollo de proyectos y previniendo cuellos de botella prematuros en operaciones de E/S.

---

## 4. Configuración de Pantalla (*Monitor-Attached*)

> **Importante:** La salida de video debe estar conectada a la pantalla **antes** de suministrar energía eléctrica a la placa; de lo contrario, el sistema podría no inicializar la señal de video durante el proceso de arranque y flasheo.

- La salida de video nativa de la Jetson Orin Nano es un puerto **DisplayPort**.
- Para este procedimiento se utilizó un adaptador pasivo/activo de **DisplayPort a HDMI** según disponibilidad de hardware.

---

## 5. Preparación del Host y SDK Manager

Para la preparación e instalación del software se utilizó la herramienta oficial **NVIDIA SDK Manager**:

1. **Requisitos de cuenta:**
   - Requiere iniciar sesión con una cuenta de [NVIDIA Developer](https://developer.nvidia.com/). El registro es gratuito y solo requiere una dirección de correo electrónico válida.
2. **Descarga de SDK Manager:**
   - Descargar el instalador correspondiente desde el portal oficial de NVIDIA.
3. **Verificación del Powershell 7**
   - Comprobar que el equipo cuente con el PowerShell 7 instalado, se puede comprobar con el comando `$PSVersionTable` dentro del PowerShell de Windows. La consola en la primera linea mostrará la versión actual.
   - En caso de tener una versión anterior ejecute el siguiente comando en el PowerShell, `winget search --id Microsoft.PowerShell --exact`, y paso seguido `winget install --id Microsoft.PowerShell --source winget`. Si al final la consola le arroja el mensaje "*Successfully installed*" en la última linea, el PowerShell 7 quedó instalado correctamente.
   - Para comprobar de manera adicional, se puede usar el mismo comando anterior `$PSVersionTable` o el `pwsh` y la consola le arrojara "*PowerShell 7.x.x*"
5. **Controladores USB / Modo APX (Host Windows / Entorno de Flasheo):**
   - Para garantizar que el host reconozca correctamente el dispositivo en modo de recuperación a través del bus USB, se recomienda tener los controladores al día.
   - En este entorno se instaló la utilidad **Zadig (v2.9)** para asociar el controlador USB adecuado al dispositivo en modo APX, permitiendo el reconocimiento sin fallas por parte de las herramientas de flasheo.

---

## 6. Procedimiento: Modo de Recuperación (*Force Recovery Mode*)

Para que el SDK Manager reconozca el módulo y pueda grabar el firmware y el sistema operativo, la Jetson debe encenderse en modo de recuperación (*Recovery Mode*). Siga estos pasos en orden estricto:

1. **Desconectar la alimentación:** Asegúrese de que el conector de alimentación de la Jetson esté desconectado.
2. **Puente de recuperación:** Realice un puente eléctrico (*jumper*) entre el pin **`FC REC`** (pin 10) y un pin **`GND`** (pin 9, o cualquier otro terminal de tierra disponible en la cabecera de pines).
3. **Conexión de datos:** Conecte un cable USB desde un puerto USB-A de la computadora *host* hacia el puerto **USB-C** de la Jetson (puerto de flasheo y comunicación de datos).
4. **Conexión de video y periféricos:**
   - Conecte el cable de salida de imagen (**DisplayPort**).
   - Conecte teclado y ratón (pueden conectarse después, pero se recomienda tenerlos listos).
5. **Encendido:** Conecte el adaptador de corriente al puerto *DC barrel jack* (19 V) de la Jetson.
6. **Retirar el puente:** Tras 2 a 3 segundos de haber energizado la placa, retire el puente entre `FC REC` y `GND`. La placa permanecerá en modo de recuperación.

---

## 7. Flasheo y Descarga con NVIDIA SDK Manager

Con la placa reconocida en modo de recuperación, continúe en la interfaz gráfica de SDK Manager:

1. **Paso 01 (Hardware Selection):** Seleccione la categoría **Jetson**.
2. **Paso 02 (Target Hardware):** Seleccione el hardware destino correspondiente al Developer Kit:
   - Identificador de módulo: **`P3767-0005 module`** (Jetson Orin Nano DevKit).
3. **Paso 03 (Target Operating System):** Seleccione la versión de software a instalar:
   - **JetPack 6.2.3** (basado en **Ubuntu 22.04 LTS - Jammy Jellyfish**, con **Jetson Linux 36.5.2** y kernel Linux 5.15).
4. **Paso 04 (SDK Components):** Componentes adicionales (Holoscan, DeepStream, etc.). En este caso se mantuvieron los componentes base sin paquetes adicionales.
5. **Configuración de credenciales temporales:**
   - Durante el proceso de flasheo preliminar, el asistente solicitará definir un usuario y contraseña para la Jetson.
   - *Recomendación:* Utilizar nombres de usuario en minúsculas y registrar una contraseña segura que recuerde con facilidad, ya que será la clave con privilegios `sudo`.

---

## 8. Finalización y Buenas Prácticas

1. **Primer arranque (OOBE - Out of Box Experience):**
   - Al terminar de grabar la imagen en la unidad NVMe, la Jetson reiniciará y mostrará señal en el monitor.
   - Aparecerá el asistente inicial de configuración de Ubuntu para finalizar la creación del perfil de usuario y zona horaria.
2. **Sincronización Host-Target:**
   - Una vez que la Jetson completa la configuración inicial e inicia el escritorio de Ubuntu, el SDK Manager en el host detectará la dirección IP asignada a la Jetson para proceder a la instalación final de los componentes SDK base.
3. **Recomendaciones de estabilidad:**
   - **No interrumpir el proceso:** Espere a que el SDK Manager en el equipo host confirme que la instalación finalizó al 100%.
   - **No ejecutar instalaciones concurrentes:** Aunque se puede usar la interfaz de Ubuntu mientras el host transfiere paquetes, **no** ejecute `apt update`, `apt upgrade` ni instale paquetes desde la terminal de la Jetson mientras el SDK Manager siga trabajando, para evitar bloqueos del gestor de paquetes (`dpkg lock`).
   - **Evitar reinicios:** No reinicie ni apague el dispositivo desde el entorno gráfico de Ubuntu hasta que el asistente del host concluya satisfactoriamente.
