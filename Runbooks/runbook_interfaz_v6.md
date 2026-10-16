# Runbook: Interfaz de Control Avanzada (`interface_v6`) - Mano Robótica Inspire RH56DFTP-2R

Guía técnica de referencia sobre la arquitectura de software, bloques de código, direccionamiento de registros Modbus TCP y parámetros de configuración de la interfaz interactiva por consola `interface_v6`, desarrollada en Python para la operación de la mano robótica Inspire RH56DFTP-2R.

---

## 1. Descripción General del Sistema

El programa `interface_v6` es una aplicación interactiva por consola diseñada para ejecutarse dentro de Jupyter Lab sobre la placa NVIDIA Jetson Orin Nano, utilizando un entorno virtual Conda (Miniforge) denominado `telecontrol` con Python 3.10. Su propósito es controlar de manera integral los 6 grados de libertad (DOF) de la mano robótica Inspire RH56DFTP-2R mediante el protocolo Modbus TCP.

Para superar las limitaciones de los métodos básicos del SDK (`inspire_rh56dftp`), esta interfaz implementa lectura y escritura directa sobre los registros internos del controlador. Esto permite manipular de forma independiente o simultánea los **ángulos de posición**, los **límites de fuerza** y las **velocidades de movimiento** de cada dedo, integrando además un sistema automatizado de bases de datos locales en formato `.csv` tanto para gestos individuales como para rutinas secuenciales.

---

## 2. Requisitos Previos de Red y Entorno

Antes de iniciar el script en Jupyter Lab, la interfaz Ethernet física (`eno1`) de la Jetson Orin Nano debe encontrarse en la misma subred que el controlador de la mano robótica (`192.168.123.0/24`) y con el puerto TCP `6000` habilitado:

```bash
# Activar el perfil de red estático (192.168.123.50/24)
sudo nmcli con up inspire-hand

# Verificar conectividad y apertura del puerto Modbus TCP (6000)
ping -c 4 192.168.123.211
nc -zv 192.168.123.211 6000
```

* **Dirección IP de la Jetson (`eno1`):** `192.168.123.50/24`
* **Dirección IP de la Mano Robótica (`IP_MANO`):** `192.168.123.211`
* **Puerto Modbus TCP:** `6000`
* **Kernel de Jupyter Lab:** `Python (Telecontrol)` (`Python 3.10.22`)

---

## 3. Configuraciones Globales a Tener Presentes

### 3.1 Mapeo de los 6 Grados de Libertad (DOF)
El arreglo global `NOMBRES` define el orden estricto de los 6 actuadores de la mano robótica, indexados de `0` a `5`:
* **Índice `0`:** `Meñique`
* **Índice `1`:** `Anular`
* **Índice `2`:** `Corazón`
* **Índice `3`:** `Índice`
* **Índice `4`:** `Flexión pulgar`
* **Índice `5`:** `Rotación pulgar`

### 3.2 Vectores y Perfiles Base de Seguridad
* **Postura Inicial (`ABIERTO`):** `[1000, 1000, 1000, 1000, 1000, 1000]`, correspondiente a la extensión total de los 6 DOF (`0` = cerrado total, `1000` = abierto total).
* **Fuerza Segura por Defecto (`FUERZA_SEGURA`):** `[200, 200, 200, 200, 200, 200]`, aplicada automáticamente en el Modo 0 para proteger el mecanismo frente a colisiones accidentales.
* **Fuerza Máxima (`MAX_FUERZA`):** `[1000, 1000, 1000, 1000, 1500, 1000]`. Los DOF `0, 1, 2, 3, 5` admiten un rango de `0` a `1000`, mientras que el índice `4` (`Flexión pulgar`) permite configurar una fuerza límite de hasta `1500`.
* **Velocidad Máxima (`MAX_VELOCIDAD`):** `[1000, 1000, 1000, 1000, 1000, 1000]`, con un rango operativo de `0` a `1000` por cada DOF.

### 3.3 Estructura de Directorios y Bases de Datos Dinámicas (`.csv`)
Al ejecutarse, el código verifica y crea automáticamente dos carpetas locales en el sistema mediante `os.makedirs(..., exist_ok=True)`:
* **Directorio de Gestos (`DIRECTORIO_GUARDADO`):** `~/Documents/Jupyter/ManoRH56DFTP-2R/Gestos`
* **Directorio de Rutinas (`DIRECTORIO_RUTINAS`):** `~/Documents/Jupyter/ManoRH56DFTP-2R/Rutinas`

La interfaz cuenta con **4 modos de operación** (`modo_actual_idx` de `0` a `3`). Para evitar que se mezclen filas con diferente número de columnas, el programa selecciona automáticamente un par de archivos `.csv` exclusivos para cada modo:

| Índice | Modo de Operación (`MODOS_NOMBRES`) | Archivo de Gestos (`ARCHIVOS_CSV`) | Archivo de Rutinas (`ARCHIVOS_RUTINAS_CSV`) | Columnas Guardadas en Gestos |
| :--- | :--- | :--- | :--- | :--- |
| **`0`** | `Solo Ángulos (Fuerza Segura)` | `gestos_0_solo_angulos.csv` | `rutinas_0_solo_angulos.csv` | `7` (`Nombre` + 6 Ángulos) |
| **`1`** | `Ángulos + Fuerza configurable` | `gestos_1_angulos_fuerza.csv` | `rutinas_1_angulos_fuerza.csv` | `13` (`Nombre` + 6 Ángulos + 6 Fuerzas) |
| **`2`** | `Ángulos + Velocidad configurable` | `gestos_2_angulos_velocidad.csv` | `rutinas_2_angulos_velocidad.csv` | `13` (`Nombre` + 6 Ángulos + 6 Velocidades) |
| **`3`** | `Ambas (Ángulo + Fuerza + Velocidad)` | `gestos_3_completos.csv` | `rutinas_3_completos.csv` | `19` (`Nombre` + 6 Ángulos + 6 Fuerzas + 6 Velocidades) |

### 3.4 Mapa de Registros Modbus TCP
La comunicación directa con el hardware utiliza las siguientes direcciones decimales de registros Modbus:

| Constante | Registro Decimal | Método del SDK Utilizado | Descripción Técnica |
| :--- | :--- | :--- | :--- |
| `REG_DEFAULT_FORCE_SET` | `1044` | `mano._write_and_ack(1044, ...)` | Establece el límite de fuerza segura por defecto. |
| `REG_ANGLE_SET` | `1486` | `mano.send_raw(angulos)` | Consigna de posición angular objetivo (`0-1000`). |
| `REG_FORCE_SET` | `1498` | `mano._write_and_ack(1498, ...)` | Consigna de fuerza límite por DOF (`0-1000` / `1500`). |
| `REG_SPEED_SET` | `1522` | `mano._write_and_ack(1522, ...)` | Consigna de velocidad de movimiento por DOF (`0-1000`). |
| `REG_ANGLE_ACT` | `1546` | `mano.read_holding(1546, 6)` | Lectura en tiempo real de los sensores de ángulo actual. |
| `REG_FORCE_ACT` | `1582` | `mano.read_holding(1582, 6)` | Lectura en tiempo real de los sensores de fuerza actual. |

---

## 4. Explicación Detallada de los Bloques de Código

### Bloque 1: Detección de Entorno Jupyter y Generación de Cabeceras
* **Compatibilidad con Jupyter Lab (`ENTORNO_JUPYTER` y `limpiar_pantalla`):** El bloque `try-except` importa `clear_output` de `IPython.display`. Al refrescar el menú, `clear_output(wait=False)` limpia la salida de la celda sin bloquear la aparición de los cuadros de entrada (`input()`), e incorpora llamadas previas a `sys.stdout.flush()` y `time.sleep(0.2)` para sincronizar el renderizado en el navegador web.
* **Función `obtener_cabeceras(modo_idx)`:** Construye dinámicamente la primera fila del archivo `.csv` de gestos concatenando los prefijos `Ang_`, `Fue_` y `Vel_` con los nombres de los 6 dedos según el modo activo.

### Bloque 2: Lectura de Sensores con Conversión de Complemento a Dos (`leer_estado_completo`)
Lee los 6 registros de posición real (`1546`) y los 6 registros de fuerza real (`1582`):
* **Corrección de desbordamiento (`65535`):** Dado que el bus Modbus transmite enteros de 16 bits sin signo (`uint16`), pequeñas variaciones negativas en los sensores de fuerza o ángulo se reportan en crudo como valores cercanos a `65535`. La función aplica la conversión a entero con signo de 16 bits (`v if v < 32768 else v - 65536`) y acota las lecturas de fuerza negativas a `0` mediante `max(0, ...)`.
* **Seguimiento de velocidad por software (`velocidades_cfg`):** Como el hardware no posee un sensor de lectura de velocidad física, la interfaz mantiene en la variable `velocidades_cfg` el último vector de velocidades enviado al registro `1522`.

### Bloque 3: Inyección de Parámetros y Monitoreo de Llegada (`enviar_parametros`)
Centraliza el envío de órdenes hacia el controlador de la mano robótica:
1. **Acotamiento de seguridad (*Clamping*):** Restringe numéricamente los vectores de fuerza entre `0` y `1500` y los vectores de velocidad entre `0` y `1000` antes de transmitirlos.
2. **Escritura secuencial con pausa:** Escribe en los registros `1044` (`default_fuerzas`), `1498` (`fuerzas`) y `1522` (`velocidades`) usando `mano._write_and_ack(...)` con pausas de `0.05s` entre tramas, y finalmente ejecuta `mano.send_raw(angulos)`.
3. **Verificación de llegada con `timeout=5.0` y `tolerancia=40`:** Entra en un ciclo `while` que consulta cada `0.1s` el registro `1546` (`REG_ANGLE_ACT`). Cuando los 6 dedos se encuentran dentro de un margen de `±40` unidades respecto al ángulo objetivo, imprime el tiempo exacto de ejecución (`✓ Gesto completado en X.XXs`). Si transcurren 5 segundos sin alcanzar el ángulo (por ejemplo, al sujetar un objeto sólido que detiene los dedos por límite de fuerza), finaliza la espera de forma segura e informa por pantalla.

### Bloque 4: Captura Interactiva de Vectores (`recoger_matriz`)
Utilizada en la Opción 3 (Configuración Completa) para pedir los valores de los 6 DOF uno por uno:
* Muestra entre corchetes el valor actual del sensor (`[Actual: X]`).
* Si el usuario ingresa `-1` (o presiona `ENTER`), la función conserva el valor actual de ese dedo sin alterarlo.
* Aplica automáticamente el tope de `1500` exclusivamente cuando se configura `"Fuerza límite"` en el índice `4` (`Flexión pulgar`), restringiendo el resto de entradas al rango `0-1000`.

### Bloque 5: Gestión de Bases de Datos `.csv` para Gestos y Rutinas
* **`cargar_gestos_csv(ruta_csv)` y `ejecutar_gesto_desde_fila(...)`:** Leen el archivo `.csv` del modo actual, omiten la cabecera y desempaquetan las columnas según corresponda: columnas `1:7` para ángulos, `7:13` para fuerzas (Modos 1 y 3), y `7:13` (Modo 2) o `13:19` (Modo 3) para velocidades.
* **`ejecutar_secuencia_gestos(...)`:** Recorre una lista de índices numéricos, ejecuta cada gesto en orden e introduce una pausa de transición de `0.3s` (`time.sleep(0.3)`) entre posturas consecutivas.
* **`exportar_rutina_csv(...)` y `cargar_rutinas_csv(...)`:** Almacenan y leen secuencias completas en el archivo de rutinas del modo activo bajo la estructura de 3 columnas: `["Nombre_Rutina", "Secuencia_Indices", "Secuencia_Nombres"]` (ejemplo: `Abecedario_LSC | 0,1,2,3 | A -> B -> C -> D`).

---

## 5. Opciones del Menú Principal (`main`)

Al iniciar `main()`, el programa abre la mano en posición segura (`ABIERTO` con `FUERZA_SEGURA`) y despliega 12 opciones interactivas:

* **Opción `1` (Cambiar Modo Actual):** Alterna de manera circular entre los 4 modos de operación (`0 -> 1 -> 2 -> 3 -> 0`) y cambia automáticamente los archivos `.csv` activos de gestos y rutinas.
* **Opción `2` (Modo Manual Incremental):** Lee la postura actual de la mano y permite mover **un único dedo (`0-5`)** manteniendo los demás en su posición física actual.
* **Opción `3` (Modo Configuración Completa):** Solicita mediante `recoger_matriz` los valores de los 6 DOF (ángulos y, según el modo, fuerzas y/o velocidades).
* **Opción `4` (Reestablecer a Posición Inicial):** Lleva todos los dedos a la postura `ABIERTO` (`1000`) y restablece `velocidades_cfg` a `MAX_VELOCIDAD`.
* **Opción `5` (Guardar Posición en Memoria RAM):** Toma una lectura de los sensores actuales (`angulos`, `fuerzas`) junto con `velocidades_cfg` y los guarda temporalmente en el diccionario `posicion_memoria`.
* **Opción `6` (Ejecutar Posición desde Memoria RAM):** Envía a la mano la postura almacenada en `posicion_memoria` aplicando los parámetros correspondientes al modo activo.
* **Opción `7` (Añadir Posición Actual a Base de Datos de Gestos `.csv`):** Lee el estado físico actual de la mano, solicita un nombre identificador y agrega el registro al final del archivo `.csv` de gestos del modo actual.
* **Opción `8` (Cargar y Ejecutar UN Gesto Guardado):** Despliega la lista numerada de gestos disponibles en el `.csv` del modo actual y ejecuta el gesto seleccionado por su índice.
* **Opción `9` (Crear, Ejecutar y/o Exportar Rutina Secuencial):** Permite escribir una secuencia de índices separados por coma (ej. `0,2,1,0`), muestra la vista previa de los nombres de los gestos concatenados con `->`, permite guardarla en el `.csv` de rutinas del modo activo y ejecutarla en el acto.
* **Opción `10` (Importar y Ejecutar Rutina Guardada `.csv`):** Muestra todas las rutinas almacenadas en el modo actual indicando sus pasos e índices, y ejecuta de principio a fin la rutina seleccionada.
* **Opción `11` (Estado Actual de Cada Dedo):** Consulta y muestra en pantalla los valores instantáneos de ángulo (`1546`) y fuerza (`1582`) de los 6 DOF sin comandar ningún movimiento.
* **Opción `0` (Salir del Programa):** Pide confirmación, abre la mano por seguridad (`ABIERTO`) y cierra formalmente la conexión Modbus TCP (`mano.close()`).

---

## 6. Consideraciones Operativas y Prevención de Colisiones (LSC)

1. **Coordinación de velocidades entre Pulgar e Índice:** En la validación de posturas del abecedario y números (0-10) en Lengua de Señas Colombiana (LSC), las trayectorias mecánicas de la **Flexión/Rotación del pulgar** (índices `4` y `5`) y del **dedo Índice** (índice `3`) pueden cruzarse. Para evitar colisiones o solapamientos entre estos dedos, se debe operar en el **Modo 2 (`Ángulos + Velocidad`)** o **Modo 3 (`Ambas`)**, asignando una velocidad menor al dedo que debe superponerse para desfasar su llegada y lograr transiciones limpias.
2. **Integridad de índices entre archivos `.csv`:** Dado que las rutinas en `rutinas_X_*.csv` guardan la secuencia haciendo referencia al número de fila (índice `0, 1, 2...`) del archivo `gestos_X_*.csv` del mismo modo, no se deben eliminar filas intermedias manualmente en el archivo de gestos si ya existen rutinas creadas sobre esos índices.
3. **Liberación del socket TCP:** Siempre se debe cerrar el programa con la **Opción `0`** o mediante interrupción controlada (`KeyboardInterrupt`), lo cual garantiza la ejecución del bloque `finally: mano.close()` y evita que el puerto `6000` quede retenido.

