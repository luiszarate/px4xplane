# Manual de Usuario Extensivo — Plugin PX4↔X-Plane (`px4xplane`)

> **Versión del documento:** 1.0  
> **Idioma:** Español  
> **Objetivo:** Guía completa para instalar, operar, diagnosticar y extender el plugin que conecta X-Plane con PX4 SITL.

---

## Tabla de Contenido

1. [Introducción](#1-introducción)
2. [¿Qué es este plugin? (SITL + mensajes HIL)](#2-qué-es-este-plugin-sitl--mensajes-hil)
3. [Arquitectura general](#3-arquitectura-general)
4. [Flujo de datos MAVLink](#4-flujo-de-datos-mavlink)
5. [Requisitos previos](#5-requisitos-previos)
6. [Instalación](#6-instalación)
7. [Estructura de carpetas recomendada](#7-estructura-de-carpetas-recomendada)
8. [Primer arranque paso a paso](#8-primer-arranque-paso-a-paso)
9. [Uso diario del plugin](#9-uso-diario-del-plugin)
10. [UI interna del plugin y lectura de estado](#10-ui-interna-del-plugin-y-lectura-de-estado)
11. [Archivo `config.ini` en detalle](#11-archivo-configini-en-detalle)
12. [Mapeo de actuadores (canales PX4 → datarefs X-Plane)](#12-mapeo-de-actuadores-canales-px4--datarefs-x-plane)
13. [Ajustes de tasas MAVLink y rendimiento](#13-ajustes-de-tasas-mavlink-y-rendimiento)
14. [Sensores, ruido y EKF2](#14-sensores-ruido-y-ekf2)
15. [Cómo agregar una nueva aeronave (custom airframe)](#15-cómo-agregar-una-nueva-aeronave-custom-airframe)
16. [Cómo adaptar parámetros PX4 para una aeronave nueva](#16-cómo-adaptar-parámetros-px4-para-una-aeronave-nueva)
17. [Checklist de validación de un airframe nuevo](#17-checklist-de-validación-de-un-airframe-nuevo)
18. [Troubleshooting avanzado](#18-troubleshooting-avanzado)
19. [Buenas prácticas operativas](#19-buenas-prácticas-operativas)
20. [FAQ](#20-faq)
21. [Glosario](#21-glosario)
22. [Anexos útiles](#22-anexos-útiles)

---

## 1) Introducción

Este manual describe de forma integral cómo usar `px4xplane`, un plugin para X-Plane que permite simular aeronaves controladas por PX4 en modo SITL.

El foco está en:

- Uso real en operación diaria.
- Entender arquitectura para depuración efectiva.
- Configuración de aeronaves existentes.
- Creación de aeronaves personalizadas.
- Resolución de problemas de control, estabilidad y conexión.

---

## 2) ¿Qué es este plugin? (SITL + mensajes HIL)

El ecosistema funciona así:

- **PX4 corre como SITL** (controlador en software).
- **X-Plane** representa la dinámica/física visual.
- **`px4xplane`** es el puente que intercambia mensajes MAVLink.

Aunque normalmente se dice “SITL”, gran parte del intercambio usa mensajes de tipo **HIL_
*** (`HIL_SENSOR`, `HIL_GPS`, `HIL_STATE_QUATERNION`, `HIL_RC_INPUTS_RAW`, y recepción de `HIL_ACTUATOR_CONTROLS`).

> Resumen práctico: **SITL de PX4 + interfaz MAVLink HIL para sensor/actuador**.

---

## 3) Arquitectura general

### 3.1 Componentes clave del plugin

- **`ConnectionManager`**
  - Abre socket TCP (puerto 4560 por defecto).
  - Espera y acepta conexión PX4.
  - Envía/recibe paquetes MAVLink.

- **`MAVLinkManager`**
  - Construye y envía mensajes de sensores simulados.
  - Decodifica comandos de actuadores que llegan desde PX4.

- **`DataRefManager`**
  - Lee datarefs de X-Plane (sensores simulados).
  - Escribe datarefs (motores/superficies) para aplicar comandos de PX4.
  - Activa `override_*` para que X-Plane acepte control externo.

- **`ConfigManager`**
  - Carga `config.ini`.
  - Interpreta mapeo de canales y rangos por aeronave.
  - Configura opciones de debug y tasas MAVLink.

- **`UIHandler` / HUD**
  - Presenta estado de conexión, valores de HIL, canales activos, etc.

### 3.2 Ciclo de ejecución

Por frame de X-Plane:

1. Se verifica conexión.
2. Se envían mensajes de sensores según tasas configuradas.
3. Se reciben mensajes de actuadores desde PX4.
4. Se aplican valores de control sobre datarefs de la aeronave.

---

## 4) Flujo de datos MAVLink

### 4.1 Del simulador a PX4

- `HIL_SENSOR`: IMU/Baro/Mag/Temp
- `HIL_GPS`: GPS simulado
- `HIL_STATE_QUATERNION`: estado “ground truth”
- `HIL_RC_INPUTS_RAW`: entradas RC

### 4.2 De PX4 al simulador

- `HIL_ACTUATOR_CONTROLS`: salidas de control (motores/superficies)

### 4.3 Implicación operativa

Si el avión “arma pero no se estabiliza”, normalmente el problema está en:

- Mapeo de canales incorrecto.
- Airframe/params no alineados.
- `HIL_ACTUATOR_CONTROLS` llegando en cero.
- Índices/rangos de datarefs no compatibles con el modelo X-Plane.

---

## 5) Requisitos previos

- X-Plane 11/12 funcional.
- Plugin `px4xplane` instalado correctamente.
- Entorno PX4 SITL compatible (idealmente el recomendado por el proyecto).
- QGroundControl (recomendado para inspección de actuadores, parámetros y estado EKF).

---

## 6) Instalación

### 6.1 Desde binarios precompilados

1. Descargar release del plugin.
2. Copiar la carpeta completa `px4xplane/` a:
   - `X-Plane/Resources/plugins/`
3. No copiar sólo `.xpl` aislado; copiar estructura completa.

### 6.2 Compilando desde fuente

Opción CMake:

```bash
git clone --recursive https://github.com/alireza787b/px4xplane.git
cd px4xplane
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
cmake --build .
```

Luego copiar la carpeta de salida generada en `build/.../px4xplane` al directorio de plugins de X-Plane.

---

## 7) Estructura de carpetas recomendada

Estructura esperada (v3+):

```text
px4xplane/
├── 64/
│   ├── win.xpl | lin.xpl | mac.xpl
│   └── config.ini
├── px4_airframes/   (o px4_params según distribución)
└── README.md
```

> **Importante:** `config.ini` debe estar junto al binario (`64/`).

---

## 8) Primer arranque paso a paso

1. Iniciar X-Plane.
2. Confirmar que el plugin aparece en el menú de plugins.
3. Seleccionar “Connect to SITL” (o equivalente en menú/UI del plugin).
4. Iniciar PX4 SITL con el airframe correcto.
5. Confirmar estado “Connected”.
6. En pestaña “Controls”, verificar que los canales HIL varían.
7. Armar y validar respuesta en motores/superficies.

---

## 9) Uso diario del plugin

### 9.1 Flujo típico

- Elegir aeronave en X-Plane.
- Elegir configuración `config_name` correspondiente.
- Conectar plugin ↔ PX4.
- Confirmar salud de sensores/EKF.
- Armar.
- Volar.

### 9.2 Desconexión segura

El plugin implementa lógica para evitar “ghost commands” al desconectar/reconectar, reseteando actuadores y estado MAVLink.

---

## 10) UI interna del plugin y lectura de estado

### 10.1 Señales clave a observar

- **SITL Status**: conectado/no conectado.
- **Valores HIL de canales**: si no cambian, PX4 no está entregando salida útil.
- **Channel Configurations**: lista de mapeos y datarefs activos.
- **Current value vs Range**: útil para validar escalado.

### 10.2 ¿Qué indica una falla de estabilización?

- Motores giran pero canales de control no tienen diferencial útil.
- Superficies no se mueven aunque canales sí cambian.
- Rangos invertidos o demasiado pequeños.

---

## 11) Archivo `config.ini` en detalle

`config.ini` contiene:

- `config_name` activo.
- Flags de debug.
- Tasas MAVLink.
- Secciones por aeronave (`[Alia250]`, `[ehang184]`, etc.).
- Líneas `channelN = ...` con mapeo a datarefs.

Ejemplo conceptual:

```ini
config_name = Alia250

[Alia250]
channel0 = sim/flightmodel/engine/ENGN_thro_use, floatArray, [0], [-1 1]
channel1 = sim/flightmodel/engine/ENGN_thro_use, floatArray, [1], [-1 1]
channel2 = sim/flightmodel/engine/ENGN_thro_use, floatArray, [2], [-1 1]
channel3 = sim/flightmodel/engine/ENGN_thro_use, floatArray, [3], [-1 1]
```

Formato general por entrada:

```text
<dataref>, <tipo>, <indices>, [<min> <max>]
```

Donde:

- `tipo`: `float` o `floatArray`
- `indices`: `0` (si no aplica) o `[0]`, `[1]`, etc.
- `range`: escala de salida en X-Plane

---

## 12) Mapeo de actuadores (canales PX4 → datarefs X-Plane)

### 12.1 Principio

`HIL_ACTUATOR_CONTROLS.controls[i]` (rango típico -1..+1) se escala al rango definido por cada `channeli`.

### 12.2 Riesgos comunes

- Índice de motor mal puesto (`[2]` en lugar de `[1]`).
- Dataref incorrecto para el modelo de avión elegido.
- Rango invertido (ej. `[+1 -1]`) causando control al revés.
- Usar configuración de otra aeronave sin adaptar geometría.

### 12.3 Multi-dataref por canal

Se permite usar `|` para aplicar la misma señal a múltiples superficies/datarefs.

---

## 13) Ajustes de tasas MAVLink y rendimiento

Parámetros principales:

- `mavlink_sensor_rate_hz`
- `mavlink_gps_rate_hz`
- `mavlink_state_rate_hz`
- `mavlink_rc_rate_hz`

### Recomendaciones

- Mantener FPS de X-Plane alto y estable (ideal 60+).
- Evitar sobrecargar con tasas muy altas si tu hardware no da FPS suficiente.
- Si hay oscilaciones/fallas EKF, primero revisar FPS + rates antes de tocar gains.

---

## 14) Sensores, ruido y EKF2

El plugin incorpora modelado de ruido y calibración para comportamiento más realista.

### 14.1 Qué te importa como usuario

- EKF2 es sensible a calidad temporal (timing) y coherencia de sensores.
- Saltos de tiempo, FPS bajo o mapeos incoherentes pueden afectar estabilidad.
- Si hay warnings de bias o innovaciones altas, no siempre es “PID malo”; a menudo es cadena sensor-tiempo-config.

### 14.2 Parámetros PX4 relevantes para diagnóstico

- `EKF2_*` (ruidos, gates, delays)
- `IMU_*`
- `COM_ARM_*`
- `CA_*` y `PWM_MAIN_FUNC*`

---

## 15) Cómo agregar una nueva aeronave (custom airframe)

### 15.1 Estrategia recomendada

1. Elegir una base similar (`Alia250`, `ehang184`, etc.).
2. Duplicar sección en `config.ini`.
3. Renombrar sección (ej. `[MiAeronaveVTOL]`).
4. Ajustar mapeo de canales según tu aeronave en X-Plane.
5. Cambiar `config_name` a la nueva sección.

### 15.2 Procedimiento detallado

1. Identifica motores/superficies reales en Plane Maker.
2. Mapea cada salida de PX4 a su dataref correspondiente.
3. Valida dirección de giro y signo de control.
4. Ajusta rangos por canal (deflexión realista).
5. Reinicia X-Plane (la configuración se carga al inicio).

### 15.3 Ejemplo plantilla

```ini
[MiAeronaveCustom]
autoPropBrakes = 0,1,2,3

; Motores
channel0 = sim/flightmodel/engine/ENGN_thro_use, floatArray, [0], [-1 1]
channel1 = sim/flightmodel/engine/ENGN_thro_use, floatArray, [1], [-1 1]
channel2 = sim/flightmodel/engine/ENGN_thro_use, floatArray, [2], [-1 1]
channel3 = sim/flightmodel/engine/ENGN_thro_use, floatArray, [3], [-1 1]

; Superficies
channel4 = sim/flightmodel/controls/wing2l_ail1def, float, 0, [-20 20]
channel5 = sim/flightmodel/controls/wing2r_ail1def, float, 0, [-20 20]
channel6 = sim/flightmodel/controls/hstab1_elv1def, float, 0, [-20 20]
channel7 = sim/flightmodel/controls/vstab1_rud1def, float, 0, [-20 20]
```

---

## 16) Cómo adaptar parámetros PX4 para una aeronave nueva

Además de `config.ini`, necesitas que PX4 “piense” correctamente tu geometría.

### 16.1 Mínimos a revisar

- `CA_ROTOR_COUNT`
- `CA_ROTORx_PX/PY/PZ`, `CA_ROTORx_CT`, `CA_ROTORx_KM`
- `PWM_MAIN_FUNC*`
- Gains de rate/attitude (MC/FW/VTOL según aplique)

### 16.2 Regla de oro

**No mezclar**: geometría de un airframe con mapeo físico de otro, salvo que estés copiando un modelo realmente equivalente.

---

## 17) Checklist de validación de un airframe nuevo

### 17.1 Pre-arm

- [ ] Conexión plugin↔PX4 estable.
- [ ] No warnings críticos de EKF.
- [ ] `config_name` correcto.
- [ ] Canales HIL vivos en UI.

### 17.2 Actuadores

- [ ] Cada canal mueve el actuador esperado.
- [ ] Signo correcto (roll/pitch/yaw).
- [ ] Sin saturación inmediata.

### 17.3 Vuelo inicial

- [ ] Hover corto o taxi controlado.
- [ ] Respuesta estable a comandos pequeños.
- [ ] Log revisado tras prueba.

---

## 18) Troubleshooting avanzado

## 18.1 “Conecta, arma, encienden motores, pero no estabiliza”

Causas típicas:

1. Mapeo de canales no corresponde a la aeronave real.
2. `config.ini` equivocado o en ruta incorrecta.
3. Airframe PX4 distinto al perfil de `config_name`.
4. Control allocation (`CA_*`) no representa geometría real.
5. Canales llegan en cero o casi cero.

Acciones:

- Verificar valores HIL en UI Controls.
- Verificar mapeo por canal en `config.ini`.
- Verificar orden físico de motores.
- Revisar QGC Actuators para salida real.

## 18.2 “No conecta PX4 con plugin”

- Confirmar puerto TCP (4560 por defecto).
- Confirmar que plugin está en menú y activo.
- Revisar si otro proceso ocupa el puerto.
- Reiniciar ambos lados en orden limpio.

## 18.3 “Hay conexión, pero actuadores no se mueven”

- Revisar `override_*` habilitado (plugin lo hace al conectar).
- Verificar datarefs válidos para ese avión.
- Comprobar que `actuatorConfigs` no está vacío.

## 18.4 “EKF inestable / bias alto / arming checks fallan”

- Reducir complejidad inicial (rates moderados).
- Revisar FPS real de X-Plane.
- Revisar configuración de ruido y parámetros EKF2.
- Verificar que aeronave esté quieta al inicio de calibración.

## 18.5 “Control invertido”

- Invertir rango en `config.ini` (ej. `[-20 20]` ↔ `[20 -20]`) o corregir signo en PX4 allocation.
- No hacer doble inversión (en ambos lados a la vez).

---

## 19) Buenas prácticas operativas

- Cambiar una sola variable por prueba.
- Registrar configuración probada (commit o backup).
- Mantener una “config base estable” y crear variantes.
- Validar primero control básico, luego tuning fino.
- Evitar debug extremo en vuelos largos (logs gigantes).

---

## 20) FAQ

### ¿Es HIL o SITL?

Es SITL de PX4 con intercambio MAVLink que usa mensajes `HIL_*` para sensores/actuadores.

### ¿Puedo usar cualquier avión de X-Plane?

Sí, pero necesitas mapear correctamente datarefs y adaptar parámetros PX4.

### ¿Puedo usar la config de Alia250 en otro avión?

Sólo como punto de partida. Si la geometría no es equivalente, tendrás falta de control/estabilidad.

### ¿Debo reiniciar X-Plane tras cambiar `config.ini`?

Sí, recomendado. La carga de configuración ocurre al iniciar plugin/sim.

---

## 21) Glosario

- **SITL**: Software In The Loop.
- **HIL**: Hardware In The Loop (aquí en mensajes MAVLink).
- **DataRef**: Variable interna de X-Plane accesible por plugins.
- **EKF2**: Filtro de estimación de estado de PX4.
- **CA (Control Allocation)**: Asignación de esfuerzos de control a actuadores.

---

## 22) Anexos útiles

## A) Matriz de mapeo recomendada para pruebas

Crear una tabla por aeronave:

| Canal PX4 | Dataref X-Plane | Actuador físico | Dirección esperada |
|---|---|---|---|
| ch0 | ... | motor FL | + aumenta RPM |
| ch1 | ... | motor FR | + aumenta RPM |
| ... | ... | ... | ... |

## B) Procedimiento de “smoke test” de 5 minutos

1. Conectar.
2. Confirmar HIL activo.
3. Mover pitch/roll/yaw mínimo.
4. Verificar motor/superficie esperada.
5. Abort si algo no coincide.

## C) Política de cambios de configuración

- Guardar copias versionadas de `config.ini`.
- Nombrar perfiles por fecha + aeronave + objetivo (`2026-03-19_myvtol_hovertest.ini`).

---

## Cierre

Si estás migrando desde una aeronave existente (por ejemplo Alia250) hacia una custom, la clave es validar **alineación completa de la cadena**:

1. Geometría PX4 (`CA_*`, `PWM_MAIN_FUNC*`)  
2. Mapeo plugin (`channelN` → dataref + rango)  
3. Modelo X-Plane (índices de motor/superficie correctos)

Con esa trilogía alineada, la mayoría de problemas de “arma pero no estabiliza” desaparecen.
