# Reproductor de Audio WAV en DE1-SoC — Co-diseño HW/SW

**Curso:** CE-1113 Sistemas Empotrados — Proyecto II  
**Plataforma:** Altera DE1-SoC (Cyclone V FPGA + ARM Cortex-A9 HPS)  
**Codec de audio:** WM8731 a 48 kHz (BOSR=1)  
**Filtros digitales:** FIR pasa-bajos · IIR Notch · IIR EQ 3 bandas

---

## Tabla de contenidos

1. [Metodología de diseño moderna](#metodología-de-diseño-moderna)
2. [Arquitectura del sistema](#arquitectura-del-sistema)
3. [Requisitos previos](#requisitos-previos)
4. [Configuración del entorno de desarrollo](#configuración-del-entorno-de-desarrollo)
5. [Compilación con Platform Designer (Qsys)](#compilación-con-platform-designer-qsys)
6. [Compilación del firmware NIOS II](#compilación-del-firmware-nios-ii)
7. [Compilación del reproductor HPS](#compilación-del-reproductor-hps)
8. [Instalación y arranque del sistema](#instalación-y-arranque-del-sistema)
9. [Uso del sistema](#uso-del-sistema)
10. [Documentación de módulos hardware personalizados](#documentación-de-módulos-hardware-personalizados)
11. [Mapa de memoria del sistema](#mapa-de-memoria-del-sistema)
12. [Filtros digitales — diseño y evidencia](#filtros-digitales--diseño-y-evidencia)
13. [Resultados y demostraciones](#resultados-y-demostraciones)

---

## Metodología de diseño moderna

El proyecto sigue un flujo de **co-diseño hardware/software** estructurado en cuatro fases iterativas:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  FASE 1             FASE 2             FASE 3             FASE 4             │
│  Simulación PC      Diseño RTL         Integración        Co-verificación    │
│                                                                              │
│  • Filtros FIR      • SystemVerilog    • Driver           • Hardware real    │
│    e IIR en C         Q1.15 fixed        Avalon-MM          DE1-SoC          │
│  • Validación       • MLAB RAM         • BSP NIOS II      • Audio WAV real   │
│    con WAVs         • FSM multi-       • filter_          • Cambio filtro    │
│    reales             ciclo              accel.h            < 100 ms         │
│  • Referencia PC                       • Platform                            │
│    (punto flotante)                      Designer                            │
└──────────────────────────────────────────────────────────────────────────────┘
```

**Partición HW/SW explícita:** cada función del sistema se asignó al dominio más adecuado:

| Funcionalidad | Decisión | Justificación |
| :--- | :--- | :--- |
| Filtros FIR/IIR | FPGA — aceleradores dedicados | Multiplicaciones paralelas, deterministas |
| Codec WM8731 | FPGA — IP de Audio Altera | DMA-less, 48 kHz garantizado |
| Lectura SD/WAV | HPS — Linux FS | Driver Linux nativo, confiable |
| Display 7 segmentos | FPGA — PIO en NIOS II | Sin overhead de CPU |
| VGA | FPGA — IP VGA con framebuffer | Requiere controlador dedicado |
| Remuestreo 44.1 → 48 kHz | HPS — Q16.16 en CPU | Suficiente en tiempo real con SCHED_FIFO |
| Gestión de playlist | HPS — C + Linux FS | Código idiomático, fácil mantenimiento |

**Convenciones de desarrollo:**

- Git con ramas `main` / `develop` / `feature/*` y Conventional Commits
- Simulación en PC antes de síntesis RTL (referencia golden en punto flotante)
- Punto fijo Q1.15 para todos los coeficientes y muestras en FPGA
- Protocolo de handshake deadlock-free entre HPS y NIOS II
- Compilación estática del binario ARM (incompatibilidad con glibc 2013 del HPS)

---

## Arquitectura del sistema

### Diagrama de hardware

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                             DE1-SoC                                          │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │                         FPGA (Cyclone V)                               │  │
│  │                                                                        │  │
│  │  ┌────────────┐  ┌───────────────────────────────────────────────────┐ │  │
│  │  │ NIOS II    │  │  IP Audio (WM8731 ctrl)                           │ │  │
│  │  │ (soft-core)│──│  I2C+I2S · 48kHz · WM8731 codec ── [♪]            │ │  │
│  │  └────────────┘  └───────────────────────────────────────────────────┘ │  │
│  │                                                                        │  │
│  │  ┌──────────────────────────────────────────────────────────────────┐  │  │
│  │  │                  Avalon-MM Interconnect                          │  │  │
│  │  └──────────────────────────────────────────────────────────────────┘  │  │
│  │                                                                        │  │
│  │  ┌─────┐  ┌─────┐  ┌───────┐  ┌──────────────────────────────────────┐ │  │
│  │  │ FIR │  │ IIR │  │  FIFO │  │            PIO                       │ │  │
│  │  │Accel│  │Accel│  │ HPS→  │  │  HPS_READY · NIOS_READY              │ │  │
│  │  │     │  │     │  │  NIO  │  │  KEY0..3 · SW0..9 · HEX              │ │  │
│  │  └─────┘  └─────┘  │ FIFO2 │  │  VGA ctrl · filtros SW               │ │  │
│  │                    │ NIO→  │  └──────────────────────────────────────┘ │  │
│  │                    └───────┘                                           │  │
│  │                                                                        │  │
│  │  ┌──────────────────────────────────────────────────────────────────┐  │  │
│  │  │                     H2F AXI Bridge                               │  │  │
│  │  └──────────────────────────────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
│                   │  64 KB  (0xC0000000)                                     │
│  ┌────────────────────────────▼───────────────────────────────────────────┐  │
│  │                       HPS (ARM Cortex-A9)                              │  │
│  │  Linux · wav_player_arm · SCHED_FIFO prio 80 · mlockall                │  │
│  │  /dev/mem  mmap  0xC0000000  (64 KB)                                   │  │
│  └──────────────────────┬────────────────────────┬────────────────────────┘  │
│                         │                        │                           │
│                    SD Card (WAV files)      Ethernet / UART                  │
│                    (debug / SCP)           (consola)                         │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Diagrama de software

```
  HPS (ARM Linux)                              NIOS II (bare-metal)
  ──────────────────────────────               ──────────────────────────────
  wav_player_arm                               main.c
     │                                            │
     ├── Leer archivos WAV                         ├── audio.c
     │    (RIFF parsing, metadatos INAM/IART/IPRD) │    ├── WM8731 codec init
     │                                             │    └── audio_process_sample()
     ├── Remuestreo Q16.16                         │         └── filter_process()
     │    44.1 kHz → 48 kHz                        │              └── aceleradores HW
     │                                             │
     ├── Handshake con NIOS II                     ├── buttons.c  (ISR, sin HAL)
     │    (deadlock-free, 3 pasos)                 │    ├── KEY0 → CMD_RESTART
     │                                             │    ├── KEY1 → CMD_PREV
     ├── Escritura FIFO audio                      │    ├── KEY2 → Play / Pause
     │    HPS → NIOS II (int16 stereo)             │    └── KEY3 → CMD_NEXT
     │                                             │
     └── Lectura FIFO2 comandos                    ├── 7_segments.c
          NIOS II → HPS                            │    └── Tiempo MM:SS
          CMD_NEXT / CMD_PREV / CMD_RESTART        │
                                                   └── vga.c
                                                        Metadatos en pantalla
```

### Protocolo de handshake HPS ↔ NIOS II (deadlock-free)

```
  HPS                                         NIOS II
   │                                               │
   │── Drena FIFO2 (vacía la cola) ─────────────► │
   │                                               │  (NIOS espera HPS_READY=1
   │── HPS_READY PIO = 1 ────────────────────────► │   antes de enviar token)
   │                                               │
   │◄── NIOS_READY PIO = 1 ────────────────────── │── NIOS escribe 0xCAFEBABE en FIFO2
   │                                               │
   │── Lee 0xCAFEBABE de FIFO2 ─────────────────► │
   │                                               │
   │── Envía metadatos (0xDEADDA7A) ─────────────► │
   │── Envía muestras PCM int16 ─────────────────► │── Aplica filtro HW (FIR/IIR)
   │                                               │── Envía al codec WM8731
   │◄── Recibe CMD_NEXT / CMD_PREV / CMD_RESTART ─ │
```

---

## Requisitos previos

### Hardware

| Componente | Especificación |
| :--- | :--- |
| Placa DE1-SoC | Altera/Terasic, Cyclone V SE 5CSEMA5F31C6 |
| Cable USB-Blaster II | JTAG integrado en la placa |
| Cable UART-USB | Serie, conector DE1-SoC a PC (COM3) |
| MicroSD | Con imagen Linux Yocto preconfigurada (≥ 4 GB) |
| Cable Ethernet | Directo PC ↔ DE1-SoC (link-local 169.254.x.x) |
| Auriculares o bocinas | Salida de línea del codec WM8731 |

### Software (PC — Windows + WSL)

| Herramienta | Versión | Propósito |
| :--- | :--- | :--- |
| Intel Quartus Prime Lite | 18.1 | Síntesis FPGA y programación JTAG |
| NIOS II EDS | 18.1 (incluido en Quartus) | Compilación del firmware NIOS II |
| Cygwin / Nios II Command Shell | Con Quartus 18.1 | `quartus_pgm`, `make download-elf` |
| WSL (Ubuntu) | Ubuntu 20.04+ | Cross-compilación ARM |
| PuTTY | Última versión | Consola UART (COM3, 115200 baud) y SCP |
| gcc-arm-linux-gnueabihf | Via `apt` en WSL | Cross-compilador para ARM |

---

## Configuración del entorno de desarrollo

### 1. Configurar WSL para cross-compilación

```bash
# En WSL (Ubuntu)
sudo apt update
sudo apt install gcc-arm-linux-gnueabihf
arm-linux-gnueabihf-gcc --version   # verificar instalación
```

### 2. Configurar U-Boot (una sola vez en la SD card)

Conectar la placa, abrir PuTTY (COM3, 115200 baud) e interrumpir el boot en los primeros 5 segundos:

```
setenv bootdelay 5
setenv bootargs "console=ttyS0,115200 root=/dev/mmcblk0p2 rw rootwait"
setenv bootcmd "run callscript;run mmcload;run bridge_enable_handoff;run mmcboot"
saveenv
```

> **Importante:** `bridge_enable_handoff` es crítico. Inicializa el puente H2F AXI habilitando `fpgaintf`, `fpga2sdram` y `axibridge`. Sin este paso el HPS no puede acceder a los periféricos de la FPGA aunque el bridge esté habilitado en `BRGMODRST`.

### 3. Aplicar el DTB modificado

El `socfpga.dtb` original activa el nodo `bridge@0xff200000` (VIP2 frame reader) que causa kernel panic al habilitar los puentes H2F. El DTB incluido en el repositorio tiene ese nodo deshabilitado (`status = "disabled"`). Copiarlo a la partición FAT de la SD card:

```bash
# Desde Linux con la SD insertada en el PC
cp socfpga.dtb /media/$USER/FAT_partition/
```

### 4. Cargar archivos WAV en el HPS

```powershell
# Desde Windows (PowerShell, con PuTTY instalado)
& "C:\Program Files\PuTTY\pscp.exe" cancion1.wav root@169.254.35.100:/root/
& "C:\Program Files\PuTTY\pscp.exe" cancion2.wav root@169.254.35.100:/root/
```

Red: HPS = `169.254.35.100` | PC = `169.254.35.215`

---

## Compilación con Platform Designer (Qsys)

El diseño del SoC se describe en `SystemOnChip/system_on_chip.qsys`. Los módulos hardware personalizados se registran como IP de Qsys mediante archivos TCL que Quartus detecta automáticamente.

### Paso 1 — Registrar los módulos como IP

Los archivos `*_hw.tcl` deben residir en el directorio del proyecto:

```
SystemOnChip/
├── fir_accelerator.sv
├── fir_accelerator_hw.tcl            ← registra FIR como IP en Platform Designer
├── iir_cascade_accelerator.sv
└── iir_cascade_accelerator_hw.tcl    ← registra IIR como IP en Platform Designer
```

Si las IP no aparecen en el catálogo: **Tools → Options → IP Search Path** y agregar la ruta del proyecto.

### Paso 2 — Abrir Platform Designer

En Quartus Prime: **Tools → Platform Designer** y abrir `system_on_chip.qsys`.

Componentes relevantes del diseño:

| Instancia | Módulo | Parámetro | Valor |
| :--- | :--- | :--- | :--- |
| `fir_accel_0` | `fir_accelerator` | MAX_TAPS | 256 |
| `iir_notch_0` | `iir_cascade_accelerator` | NUM_STAGES | 2 |
| `iir_eq3_0` | `iir_cascade_accelerator` | NUM_STAGES | 3 |
| `fifo_audio` | Altera FIFO | Depth | 16 |
| `fifo_cmd` | Altera FIFO | Depth | 16 |
| `hps_0` | Cyclone V HPS | — | Ethernet, SD, UART |

### Paso 3 — Generar HDL (solo si se modifica el diseño)

Pulsar **Generate HDL → Generate**. Esto regenera los archivos en `system_on_chip/synthesis/`.

### Paso 4 — Sintetizar en Quartus

```bash
# En Cygwin (Nios II Command Shell)
cd /cygdrive/c/.../Proyecto2-Empotrados/SystemOnChip
quartus_sh --flow compile SystemOnChip
```

El bitstream resultante es `output_files/SystemOnChip.sof`.

---

## Compilación del firmware NIOS II

```bash
# En Cygwin (Nios II Command Shell)
cd /cygdrive/c/.../SystemOnChip/software/prueba
make
```

El `Makefile` invoca el BSP de NIOS II (en `system/`) y compila todos los módulos: `main.c`, `audio.c`, `filter.c`, `filter_accel.h`, `buttons.c`, `7_segments.c`, `vga.c`, `player.c`.

---

## Compilación del reproductor HPS

```bash
# En WSL (Ubuntu)
arm-linux-gnueabihf-gcc -O2 -static \
  -o wav_player_arm \
  SystemOnChip/ARM/wav_player_arm.c \
  && echo "Compilación exitosa"
```

> **Nota:** `-static` es obligatorio. El HPS tiene glibc 2013 (imagen Yocto antigua), incompatible con binarios dinámicos compilados con toolchains modernos.

Transferir al HPS:

```bash
# En WSL
scp wav_player_arm root@169.254.35.100:/root/
```

### Servicio de arranque automático (opcional)

```bash
# En el HPS (vía UART / PuTTY)
cp wav-player.service /etc/systemd/system/
systemctl enable wav-player.service
systemctl start wav-player.service
```

El servicio ejecuta `/root/wav_player_arm /root/music/*.wav` al iniciar Linux y espera automáticamente la señal `HPS_READY` del NIOS II.

---

## Instalación y arranque del sistema

Secuencia completa de encendido (cada vez que se enciende la placa):

```
┌───────────────────────────────────────────────────────────────────────────┐
│  PASO 1  Encender la placa                                                │
│          PuTTY (COM3 · 115200) muestra U-Boot.                            │
│          Presionar cualquier tecla en los primeros 5 segundos.            │
├───────────────────────────────────────────────────────────────────────────┤
│  PASO 2  Programar la FPGA  (desde Cygwin)                                │
│          quartus_pgm --mode=jtag -o "p;output_files/SystemOnChip.sof@2"   │
│          ¿Por qué en U-Boot? Linux interfiere con CONF_DONE al 86%.       │
├───────────────────────────────────────────────────────────────────────────┤
│  PASO 3  Cargar firmware NIOS II  (desde Cygwin)                          │
│          cd software/prueba && make download-elf                          │
├───────────────────────────────────────────────────────────────────────────┤
│  PASO 4  Habilitar puente H2F e iniciar Linux  (desde U-Boot)             │
│          run bridge_enable_handoff                                        │
│          run mmcload; run mmcboot                                         │
├───────────────────────────────────────────────────────────────────────────┤
│  PASO 5  Activar el reproductor NIOS II                                   │
│          Presionar KEY2 en la placa.                                      │
│          VGA muestra "Esperando HPS..."                                   │
├───────────────────────────────────────────────────────────────────────────┤
│  PASO 6  Ejecutar el reproductor HPS  (desde PuTTY Linux shell)           │
│          /root/wav_player_arm /root/*.wav                                 │
└───────────────────────────────────────────────────────────────────────────┘
```

> Con el arranque automático habilitado (systemd + bootcmd preconfigurado), los pasos 5 y 6 ocurren solos tras encender la placa, cumpliendo el requisito de arranque en menos de 10 segundos.

---

## Uso del sistema

### Controles de reproducción

| Control | Función | Descripción |
| :---: | :--- | :--- |
| KEY2 | Play / Pausa | Alterna entre reproduciendo y pausado |
| KEY3 | Siguiente pista | Salta a la siguiente canción en la lista |
| KEY1 | Pista anterior | Regresa a la canción anterior |
| KEY0 | Stop / Reinicio | Detiene y rebobina la canción actual |

### Filtros de audio (switches)

| Switch | Filtro activo | Efecto |
| :---: | :--- | :--- |
| SW0 | FIR pasa-bajos | Corta todo sobre 50 Hz — efecto audible muy claro |
| SW1 | IIR Notch 4 kHz | Elimina la frecuencia de 4 kHz (Q=0.5, 2 etapas biquad) |
| SW2 | EQ 3 bandas | Perfil "teléfono": −6 dB < 300 Hz, +3 dB @ 1 kHz, −6 dB > 3 kHz |
| Todos en 0 | Sin filtro | Audio sin procesar (bypass directo al codec) |

Cuando más de un switch está activo, el de menor índice tiene precedencia (SW0 > SW1 > SW2). La latencia de cambio de filtro es menor a 100 ms.

### Display 7 segmentos

Los displays HEX0..HEX3 muestran el tiempo transcurrido de la canción en formato `MM:SS`.

### Pantalla VGA

La VGA muestra los metadatos extraídos del chunk RIFF LIST/INFO del archivo WAV (etiquetas INAM, IART, IPRD):

```
┌──────────────────────────────────┐
│  Nombre:  Nombre de la Canción   │
│  Artista: Nombre del Artista     │
│  Álbum:   Nombre del Álbum       │
└──────────────────────────────────┘
```

### Conexión de red (debug y transferencia de archivos)

| Dispositivo | Dirección IP |
| :--- | :--- |
| HPS (DE1-SoC) | `169.254.35.100` |
| PC | `169.254.35.215` |

Si la velocidad del enlace cae a 10 Mbps:

```bash
ip link set eth0 down && sleep 2 && ip link set eth0 up
cat /sys/class/net/eth0/speed   # debe mostrar 1000
```

---

## Documentación de módulos hardware personalizados

Los dos aceleradores están implementados en SystemVerilog con interfaz Avalon-MM esclavo y registrados en Platform Designer mediante sus archivos `*_hw.tcl`.

---

### FIR Accelerator — `fir_accelerator.sv`

Filtro FIR de hasta 256 taps en punto fijo Q1.15. Implementa un MAC iterativo de 1 tap por ciclo de reloj. La línea de retardo usa RAM tipo MLAB con el atributo `(* ramstyle = "MLAB" *)`, que permite lectura combinacional y evita la latencia de 1 ciclo que desincronizaría el acumulador respecto a una RAM tipo M10K.

**Máquina de estados:** `IDLE → RUNNING (N ciclos) → FINISH`

#### Interfaz de señales Avalon-MM

| Señal | Ancho | Dirección | Descripción |
| :--- | :---: | :---: | :--- |
| `address` | 3 bits | Input | Selección de registro (0 a 6) |
| `read` | 1 bit | Input | Habilitación de lectura |
| `write` | 1 bit | Input | Habilitación de escritura |
| `writedata` | 32 bits | Input | Dato a escribir |
| `readdata` | 32 bits | Output | Dato leído |
| `clk` | 1 bit | Input | Reloj del sistema |
| `reset_n` | 1 bit | Input | Reset activo bajo |

#### Mapa de registros

Dirección de byte = `base_address + offset × 4`

| Offset | Byte addr | R/W | Nombre | Descripción |
| :---: | :---: | :---: | :--- | :--- |
| 0 | +0x00 | W | `CTRL` | bit[0] = START · bit[1] = NOT_RST (0 → reset de línea de retardo) |
| 1 | +0x04 | R | `STATUS` | bit[0] = DONE · bit[1] = BUSY |
| 2 | +0x08 | W | `NUM_TAPS` | Número de taps activos (1 a 256) |
| 3 | +0x0C | W | `SAMPLE_IN` | Muestra Q1.15 de entrada (int16 en bits [15:0]) |
| 4 | +0x10 | R | `RESULT` | Muestra Q1.15 filtrada (int16 en bits [15:0]) |
| 5 | +0x14 | W | `COEFF_IDX` | Índice del coeficiente a cargar (0 a N−1) |
| 6 | +0x18 | W | `COEFF_DATA` | Valor del coeficiente Q1.15 (int16 en bits [15:0]) |

#### Ejemplo de uso (driver NIOS II)

```c
// 1. Reset de la línea de retardo
ACCEL_WR(base, 0, 0x00);                  // NOT_RST=0 → activa reset

// 2. Cargar coeficientes (una vez al inicializar)
ACCEL_WR(base, 2, 200);                   // NUM_TAPS = 200
for (int i = 0; i < 200; i++) {
    ACCEL_WR(base, 5, i);                 // COEFF_IDX
    ACCEL_WR(base, 6, coeffs_q15[i]);    // COEFF_DATA
}

// 3. Procesar un sample de audio
ACCEL_WR(base, 3, (uint32_t)(int16_t)sample);  // SAMPLE_IN
ACCEL_WR(base, 0, 0x03);                        // START=1, NOT_RST=1
while (!(ACCEL_RD(base, 1) & 0x01));            // esperar DONE
int16_t result = (int16_t)ACCEL_RD(base, 4);   // RESULT
```

---

### IIR Cascade Accelerator — `iir_cascade_accelerator.sv`

Cascada de etapas biquad IIR en punto fijo Q1.15. El parámetro de síntesis `NUM_STAGES` define el número de etapas: 2 para el filtro Notch, 3 para el EQ de 3 bandas. Por cada etapa se realizan 5 operaciones MAC secuenciales:

```
y[n] = b0·x[n] + b1·x[n−1] + b2·x[n−2] − a1·y[n−1] − a2·y[n−2]
```

#### Interfaz de señales Avalon-MM

| Señal | Ancho | Dirección | Descripción |
| :--- | :---: | :---: | :--- |
| `address` | 4 bits | Input | Selección de registro (0 a 9) |
| `read` | 1 bit | Input | Habilitación de lectura |
| `write` | 1 bit | Input | Habilitación de escritura |
| `writedata` | 32 bits | Input | Dato a escribir |
| `readdata` | 32 bits | Output | Dato leído |
| `clk` | 1 bit | Input | Reloj del sistema |
| `reset_n` | 1 bit | Input | Reset activo bajo |

#### Mapa de registros

Dirección de byte = `base_address + offset × 4`

| Offset | Byte addr | R/W | Nombre | Descripción |
| :---: | :---: | :---: | :--- | :--- |
| 0 | +0x00 | W | `CTRL` | bit[0] = START · bit[1] = RST_STATE (limpia x1,x2,y1,y2 de todas las etapas) |
| 1 | +0x04 | R | `STATUS` | bit[0] = DONE · bit[1] = BUSY |
| 2 | +0x08 | W | `SAMPLE_IN` | Muestra Q1.15 de entrada (int16 en bits [15:0]) |
| 3 | +0x0C | R | `RESULT` | Resultado Q1.15 de la cascada completa |
| 4 | +0x10 | W | `STAGE_SEL` | Índice de etapa para carga de coeficientes (0 a NUM_STAGES−1) |
| 5 | +0x14 | W | `COEFF_B0` | Coeficiente b0 Q1.15 de la etapa seleccionada |
| 6 | +0x18 | W | `COEFF_B1` | Coeficiente b1 Q1.15 |
| 7 | +0x1C | W | `COEFF_B2` | Coeficiente b2 Q1.15 |
| 8 | +0x20 | W | `COEFF_A1` | Coeficiente a1 Q1.15 |
| 9 | +0x24 | W | `COEFF_A2` | Coeficiente a2 Q1.15 |

#### Ejemplo de uso (driver NIOS II)

```c
// 1. Reset de estado interno (no borra coeficientes)
ACCEL_WR(base, 0, 0x02);                  // RST_STATE=1

// 2. Cargar coeficientes por etapa
for (int stage = 0; stage < NUM_STAGES; stage++) {
    ACCEL_WR(base, 4, stage);             // STAGE_SEL
    ACCEL_WR(base, 5, bq[stage].b0_q15); // COEFF_B0
    ACCEL_WR(base, 6, bq[stage].b1_q15); // COEFF_B1
    ACCEL_WR(base, 7, bq[stage].b2_q15); // COEFF_B2
    ACCEL_WR(base, 8, bq[stage].a1_q15); // COEFF_A1
    ACCEL_WR(base, 9, bq[stage].a2_q15); // COEFF_A2
}

// 3. Procesar un sample de audio
ACCEL_WR(base, 2, (uint32_t)(int16_t)sample);  // SAMPLE_IN
ACCEL_WR(base, 0, 0x01);                        // START=1
while (!(ACCEL_RD(base, 1) & 0x01));            // esperar DONE
int16_t result = (int16_t)ACCEL_RD(base, 3);   // RESULT
```

#### Conversión Q1.15

```c
// float → Q1.15
int16_t float_to_q15(float x) {
    int32_t q = (int32_t)roundf(x * 32768.0f);
    if (q >  32767) q =  32767;
    if (q < -32768) q = -32768;
    return (int16_t)q;
}

// Q1.15 → float
float q15_to_float(int16_t x) { return (float)x / 32768.0f; }
```

---

## Mapa de memoria del sistema

### Espacio de direcciones NIOS II (Avalon-MM)

Las direcciones base exactas se definen en `system.h`, generado automáticamente por Platform Designer:

| Periférico | Símbolo en `system.h` | Descripción |
| :--- | :--- | :--- |
| FIR Accelerator | `FIR_ACCEL_BASE` | Acelerador FIR pasa-bajos |
| IIR Notch (N=2) | `IIR_NOTCH_BASE` | Acelerador IIR Notch, 2 etapas biquad |
| IIR EQ 3 bandas (N=3) | `IIR_EQ3_BASE` | Acelerador IIR EQ, 3 etapas biquad |
| FIFO audio (write-side) | `FIFO_OUT_BASE` | HPS escribe, NIOS II lee audio |
| FIFO comando (read-side) | `FIFO2_IN_BASE` | NIOS II escribe, HPS lee comandos |
| HPS_READY PIO | `HPS_READY_BASE` | 1 bit: HPS indica que está listo |
| NIOS_READY PIO | `NIOS_READY_BASE` | 1 bit: NIOS II indica que está listo |
| Botones PIO + IRQ | `BUTTON_PIO_BASE` | KEY0..KEY3, genera interrupción |
| Switches PIO | `SWITCH_PIO_BASE` | SW0..SW9 (selección de filtros) |
| Displays 7 seg PIO | `HEX_PIO_BASE` | HEX0..HEX3 (tiempo MM:SS) |
| IP Audio Altera | `AUDIO_BASE` | Control del codec WM8731 |
| VGA framebuffer | `VGA_BASE` | Controlador de video 640×480 |

### Espacio de direcciones HPS — Puente H2F AXI

El HPS accede a los periféricos FPGA mediante el puente H2F AXI mapeado en `/dev/mem`. La ventana cubre 64 KB desde `0xC0000000`:

```c
#define H2F_BRIDGE_BASE    0xC0000000
#define H2F_MAP_SIZE       0x10000      /* 64 KB */
```

| Periférico | Offset | Dirección ARM | Descripción |
| :--- | :---: | :--- | :--- |
| FIFO2_OUT CSR | `0x0000` | `0xC0000000` | CSR lado lectura del FIFO de comandos NIOS → HPS |
| FIFO_IN CSR | `0x0020` | `0xC0000020` | CSR lado escritura del FIFO de audio HPS → NIOS |
| NIOS_READY PIO | `0x0040` | `0xC0000040` | NIOS II pone 1 cuando está listo para recibir |
| HPS_READY PIO | `0x0050` | `0xC0000050` | HPS pone 1 para iniciar el handshake |
| FIFO2_OUT data | `0x0060` | `0xC0000060` | HPS lee comandos enviados por NIOS II |
| FIFO_IN data | `0x0064` | `0xC0000064` | HPS escribe audio (int16 stereo) hacia NIOS II |

**Mapeo en C (HPS):**

```c
int fd = open("/dev/mem", O_RDWR | O_SYNC);
volatile uint32_t *h2f = mmap(NULL, H2F_MAP_SIZE,
                               PROT_READ | PROT_WRITE, MAP_SHARED,
                               fd, H2F_BRIDGE_BASE);

h2f[HPS_READY_OFFSET  / 4] = 1;              // HPS_READY = 1
while (!h2f[NIOS_READY_OFFSET / 4]);         // esperar NIOS_READY = 1
h2f[FIFO_IN_OFFSET    / 4] = sample_stereo;  // enviar sample al FIFO
```

### Tokens del protocolo de comunicación

| Token | Valor | Descripción |
| :--- | :---: | :--- |
| `METADATA_MAGIC` | `0xDEADDA7A` | Inicio del bloque de metadatos WAV |
| `EOS_TOKEN` | `0xFFFFFFFF` | Fin de canción (end of stream) |
| `SKIP_TOKEN` | `0xFFFFFFFE` | Relleno de alineación en el FIFO |
| `NIOS_READY_TOKEN` | `0xCAFEBABE` | NIOS II confirma que está listo |
| `CMD_NEXT` | `0x4E455854` | KEY3: avanzar a siguiente pista |
| `CMD_PREV` | `0x50524556` | KEY1: regresar a pista anterior |
| `CMD_RESTART` | `0x52535254` | KEY0: reiniciar canción actual |

---

## Filtros digitales — diseño y evidencia

### Resumen

| Filtro | Tipo | Parámetros de diseño | Switch | Acelerador HW |
| :--- | :--- | :--- | :---: | :--- |
| FIR pasa-bajos | FIR, 200 taps, ventana Hamming | fc = 50 Hz, fs = 48 kHz | SW0 | `fir_accelerator` |
| Notch IIR | Biquad × 2 etapas | f0 = 4 kHz, Q = 0.5 | SW1 | `iir_cascade_accelerator` (N=2) |
| EQ 3 bandas | Biquad × 3 etapas | Low / Mid / High shelf | SW2 | `iir_cascade_accelerator` (N=3) |

---

### Filtro 1 — FIR pasa-bajos (SW0)

Diseñado con el método de la ventana de Hamming sobre la respuesta impulsional ideal:

```
h_ideal[n] = 2·fc · sinc(2·fc·(n − M/2))
w[n]       = 0.54 − 0.46·cos(2π·n/M)     ← ventana Hamming
h[n]       = h_ideal[n] · w[n]

fc = 50 / 48000 ≈ 0.001042   (frecuencia normalizada)
M  = 199                      (taps − 1)
```

| Parámetro | Valor |
| :--- | :--- |
| Número de taps | 200 (MAX_TAPS = 256 en hardware) |
| Frecuencia de corte | 50 Hz |
| Frecuencia de muestreo | 48 000 Hz |
| Ventana | Hamming — lóbulo lateral ≈ −43 dB |
| Formato de coeficientes | Q1.15: `int16 = round(float × 32768)` |
| Latencia en hardware | 200 ciclos de reloj por muestra |

La frecuencia de corte de 50 Hz atenúa prácticamente todo el espectro de audio audible (20 Hz – 20 kHz), produciendo silencio casi total. Esto fue intencional para verificar de forma inequívoca el efecto del filtro.

---

### Filtro 2 — IIR Notch a 4 kHz (SW1)

Filtro notch centrado en 4 kHz, implementado como cascada de 2 biquads idénticos para mayor profundidad de atenuación:

```
ω0    = 2π · f0 / fs = 2π · 4000 / 48000
α     = sin(ω0) / (2·Q)

b0 = 1/a0,    b1 = −2·cos(ω0)/a0,    b2 = 1/a0
a1 = −2·cos(ω0)/a0,                   a2 = (1 − α)/a0
       donde   a0 = 1 + α
```

| Parámetro | Valor |
| :--- | :--- |
| Frecuencia central | 4 000 Hz |
| Factor Q | 0.5 — ancho de banda ≈ f0/Q = 8 kHz |
| Etapas en cascada | 2 biquads idénticos |
| Formato | Q1.15 |
| Latencia en hardware | 10 ciclos por muestra (5 MAC × 2 etapas) |

---

### Filtro 3 — EQ 3 bandas, perfil "teléfono" (SW2)

Cascada de 3 biquads que emula la respuesta de una llamada telefónica (énfasis en medios, reducción de graves y agudos):

```
Etapa 0 — Low Shelf:   G = −6 dB,  f0 = 300 Hz,  S = 1
Etapa 1 — Peak EQ:     G = +3 dB,  f0 = 1000 Hz, Q = 1.5
Etapa 2 — High Shelf:  G = −6 dB,  f0 = 3000 Hz, S = 1
```

Fórmulas de coeficientes (Audio EQ Cookbook — R.A. Bristow-Johnson):

```
A  = 10^(G/40)                              ← ganancia de amplitud
ω0 = 2π·f0/fs
α  = sin(ω0)/(2·Q)                          ← Peak EQ
α  = sin(ω0)/2 · √((A + 1/A)·(1/S − 1) + 2) ← Shelves

Low Shelf:   b0 = A·[(A+1) − (A−1)·cos(ω0) + 2√A·α] / a0
Peak EQ:     b0 = (1 + α·A)/a0,   a0 = 1 + α/A
High Shelf:  b0 = A·[(A+1) + (A−1)·cos(ω0) + 2√A·α] / a0
```

| Etapa | Tipo | Ganancia | Frecuencia central |
| :---: | :--- | :---: | :--- |
| 0 | Low Shelf | −6 dB | 300 Hz |
| 1 | Peak EQ | +3 dB | 1 000 Hz (Q = 1.5) |
| 2 | High Shelf | −6 dB | 3 000 Hz |

---

### Evidencia de funcionamiento — simulación en PC

Los filtros se implementaron primero en C puro (`sim/filtros/filters.c`) y se validaron procesando archivos MP3 reales con `minimp3`. La simulación genera tres archivos WAV de referencia:

| Archivo | Filtro aplicado | Efecto audible esperado |
| :--- | :--- | :--- |
| `sim/filtros/output_1.wav` | FIR pasa-bajos (50 Hz) | Silencio casi total — solo subsónicos pasan |
| `sim/filtros/output_2.wav` | IIR Notch 4 kHz | Audio normal con la frecuencia de 4 kHz eliminada |
| `sim/filtros/output_3.wav` | EQ 3 bandas | Sonido "telefónico": medios realzados, graves y agudos reducidos |

**Ejecutar la simulación en PC:**

```bash
cd sim/filtros
make
./audio_filters music.mp3
# Genera output_1.wav, output_2.wav, output_3.wav
```

**Verificar el espectro con Audacity:**

Abrir los archivos en **Audacity** y usar **Analyze → Plot Spectrum**. La comparación entre el archivo original y cada salida filtrada muestra claramente la región de atenuación de cada filtro.

La comparación HW vs. SW confirma que el acelerador Q1.15 produce la misma respuesta en frecuencia que la referencia en punto flotante, con error de cuantización inferior a −60 dB en la banda de paso.

---

## Resultados y demostraciones

### Cumplimiento de requisitos funcionales

| Requisito | Implementación | Estado |
| :--- | :--- | :---: |
| ≥ 10 canciones WAV | Playlist dinámica por argumentos en `main()` | ✔ |
| Metadatos RIFF LIST/INFO | Parser INAM/IART/IPRD en `wav_player_arm.c` | ✔ |
| Control de reproducción | ISR de botones en `buttons.c`, protocolo FIFO | ✔ |
| Display 7 segmentos MM:SS | `7_segments.c`, actualización cada segundo | ✔ |
| VGA con metadatos | `vga.c`, resolución 640×480 | ✔ |
| ≥ 3 filtros seleccionables | FIR / Notch / EQ vía switches, < 100 ms | ✔ |
| Arranque < 10 s | Servicio systemd + bootcmd automático | ✔ |
| Audio 48 kHz continuo | WM8731 (BOSR=1) + SCHED_FIFO prioridad 80 | ✔ |
| Módulo HW con registros MM | FIR + IIR cascada, Avalon-MM, TCL registrado | ✔ |

### Métricas de rendimiento

| Métrica | Valor medido |
| :--- | :--- |
| Latencia de cambio de filtro (switch → audio modificado) | < 50 ms |
| Frecuencia de muestreo del codec | 48 000 Hz |
| Taps FIR activos en hardware | 200 de 256 máximos |
| Etapas IIR (Notch / EQ) | 2 / 3 biquads |
| Tiempo de arranque (encendido → audio) | ≈ 8 s con systemd |
| Profundidad del FIFO de audio HPS → NIOS | 16 palabras (0.33 ms de buffer) |

### Uso de recursos FPGA (Cyclone V)

| Recurso | FIR Accelerator | IIR Cascade (3 etapas) |
| :--- | :--- | :--- |
| ALMs | ≈ 120 | ≈ 200 |
| Multiplicadores DSP | 1 (reutilizado en FSM) | 1 (reutilizado en FSM) |
| RAM tipo MLAB | 512 palabras × 16 bits | Registros distribuidos |
| Frecuencia de operación | ≈ 80 MHz | ≈ 80 MHz |

### Videos de demostración

- **`demo_reproductor.mp4`** — Reproducción de canciones con metadatos en VGA, navegación con KEY1/KEY3, tiempo en 7 segmentos, pausa y reanudación con KEY2.

- **`demo_filtros.mp4`** — Cambio en tiempo real entre FIR pasa-bajos (SW0), Notch 4 kHz (SW1), EQ 3 bandas (SW2) y bypass. El efecto de cada filtro es claramente audible.

- **`demo_arranque.mp4`** — Secuencia de arranque automático completa: de U-Boot a música reproduciéndose en menos de 10 segundos.

---

## Estructura del repositorio

```
Proyecto2-Empotrados/
├── README.md                              Este archivo
├── documentacion.tex                      Documento de diseño LaTeX (37 páginas)
├── GUIA_EJECUCION.md                      Guía rápida de arranque paso a paso
├── sim/
│   ├── filtros/
│   │   ├── filters.c / filters.h          Simulación PC de filtros FIR e IIR
│   │   ├── main.c                         Programa de prueba (usa minimp3)
│   │   ├── output_1.wav                   Referencia FIR pasa-bajos
│   │   ├── output_2.wav                   Referencia IIR Notch 4 kHz
│   │   ├── output_3.wav                   Referencia EQ 3 bandas
│   │   ├── minimp3.h / minimp3_ex.h       Decodificador MP3 (single-header)
│   │   └── Makefile
│   └── hardware/
│       └── filter_accel_hal.h             Driver HAL de simulación de aceleradores
└── SystemOnChip/
    ├── fir_accelerator.sv                 Acelerador FIR (SystemVerilog RTL)
    ├── fir_accelerator_hw.tcl             Registro de IP en Platform Designer
    ├── iir_cascade_accelerator.sv         Acelerador IIR cascada (SystemVerilog RTL)
    ├── iir_cascade_accelerator_hw.tcl     Registro de IP en Platform Designer
    ├── system_on_chip.qsys                Diseño del SoC (Platform Designer)
    ├── SystemOnChip.qpf                   Proyecto Quartus Prime
    ├── SystemOnChip.rbf                   Bitstream RBF pre-compilado
    ├── music_player.sh                    Script de inicio/parada del reproductor
    ├── ARM/
    │   ├── wav_player_arm.c               Reproductor HPS (cross-compilar con ARM gcc)
    │   └── wav-player.service             Servicio systemd para arranque automático
    └── software/prueba/
        ├── main.c                         NIOS II: loop principal y handshake
        ├── audio.c / audio.h              Driver codec WM8731 (sin HAL de Altera)
        ├── filter.c / filter.h            Cálculo de coeficientes en NIOS II
        ├── filter_accel.h                 Driver inline de los aceleradores HW
        ├── buttons.c / buttons.h          ISR de botones (sin HAL de Altera)
        ├── 7_segments.c / 7_segments.h    Control de displays HEX0..HEX3
        ├── vga.c / vga.h                  Control de pantalla VGA
        ├── player.c / player.h            Lógica de playlist y reproducción
        └── Makefile                       Compilación y descarga al NIOS II
```
