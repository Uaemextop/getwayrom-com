# 📱 Diagnóstico del Dispositivo — Análisis de Logs

## Información del Dispositivo

| Campo | Valor |
|-------|-------|
| **SoC** | MediaTek MT6769V/CB (Helio G85 / MT6768) |
| **Kernel** | `6.6.82-android15-8-gb715d421b585-dirty-android15-lamu-ksu` |
| **Android** | 15 |
| **Root** | KernelSU Next |
| **Marca / Launcher** | Motorola (com.motorola.launcher3) |
| **Operador** | Telcel (México) |
| **Pantalla** | 720p (HD+) |

### Módulos KernelSU instalados

| Módulo | Función |
|--------|---------|
| `zygisksu` | Implementación de Zygisk para KernelSU |
| `playintegrityfix` | Pasar la verificación de Play Integrity |
| `tricky_store` | Manipulación del keystore para integridad |
| `zygisk_vector` | Módulo Zygisk adicional |
| `TA_utl` | Utilidad Trusted Application |
| `adguardcert` | Certificado de AdGuard (bloqueo de anuncios) |

---

## 🔴 Problemas Encontrados

### 1. Firmware Bug — Kernel Desalineado en Boot

```
[Firmware Bug]: Kernel image misaligned at boot, please fix your bootloader!
```

**Severidad:** 🔴 CRÍTICA

**Explicación:** El kernel personalizado (compilación `-dirty` con KernelSU) no está correctamente alineado en memoria al inicio. Esto es causado por el bootloader o por cómo se flasheó la imagen del kernel. Un kernel desalineado puede provocar:
- Inestabilidad general del sistema
- Accesos a memoria no óptimos
- Reinicios aleatorios
- Congelamientos de la interfaz (UI freezes)

---

### 2. Fallo de Inicialización USB (MUSB)

```
phy phy-11cc0000.usb-phy.0: phy init failed --> -2
[MUSB]mt_usb_init 4475: mt_usb_init error
[MUSB]musb_init_controller 2613: musb_init_controller failed with status -2
musb-hdrc: probe of musb-hdrc failed with error -2
```

**Severidad:** 🟠 ALTA

**Explicación:** El controlador USB no pudo inicializarse correctamente. El error `-2` (ENOENT) indica que un recurso necesario no se encontró. Esto puede causar:
- Problemas de carga
- Desconexiones USB aleatorias
- Inestabilidad del subsistema de energía
- Reinicios cuando se conecta/desconecta el cargador

---

### 3. Advertencias del Kernel (Kernel Warnings con Call Trace)

Se detectaron **11 WARNING** en el kernel:

```
WARNING: CPU: 7 PID: 337 at mm/page_alloc.c:4774 __alloc_pages+0x164/0x270
WARNING: CPU: 2 PID: 11 at kernel/tracepoint.c:340 tracepoint_add_func
WARNING: CPU: 7 PID: 1 at lib/kobject.c:736 kobject_put
WARNING: CPU: 7 PID: 1 at lib/refcount.c:28 refcount_warn_saturate
WARNING: CPU: 7 PID: 1 at kernel/irq/manage.c:1902 free_irq
WARNING: CPU: 6 PID: 337 at fs/proc/generic.c:376 proc_register
WARNING: CPU: 1 PID: 0 at kernel/time/tick-sched.c:1364 tick_nohz_idle_exit
```

**Severidad:** 🟠 ALTA

**Explicación:**
- **`__alloc_pages`**: Fallo de asignación de memoria en el kernel. El sistema tiene presión de memoria.
- **`kobject_put` / `refcount_warn_saturate`**: Errores de conteo de referencias, indicando posibles fugas de memoria del kernel.
- **`free_irq`**: Problemas liberando interrupciones, típico de drivers mal portados al kernel custom.
- **`tick_nohz_idle_exit`**: Problema con el scheduler/timer cuando el CPU sale del modo idle — puede causar congelamientos breves.

El kernel está marcado como **Tainted: G W O** (módulos propietarios + warnings + out-of-tree), lo que confirma inestabilidad.

---

### 4. Presión de Memoria (Low Memory)

```
am_low_memory: 2
```

```
lowmemorykiller: Using psi monitors for memory pressure detection
```

**Severidad:** 🟡 MEDIA

**Explicación:** El sistema reportó eventos de baja memoria desde el arranque. En un dispositivo MT6768 con resolución 720p, esto sugiere que:
- La RAM disponible es limitada (probablemente 4GB o menos)
- Los módulos de KernelSU + Zygisk consumen memoria adicional
- El módulo `moto_swap` (hybrid swap) está activado, pero puede no ser suficiente

---

### 5. Errores de SEPolicy en ZygiskSU

```
ksud::sepolicy: apply rule NormalPerm(...) failed: Invalid argument (os error 22)
```

**Severidad:** 🟡 MEDIA

**Explicación:** Las reglas de SELinux del módulo `zygisksu` no pudieron aplicarse correctamente. Esto puede causar:
- Fallos al inyectar en procesos de aplicaciones
- Bloqueos de apps que dependen de Zygisk
- Inestabilidad del sistema cuando SELinux niega operaciones

---

### 6. Bandera `has_battery_removed=1`

```
Kernel command line: ... has_battery_removed=1 ...
```

**Severidad:** 🟡 MEDIA

**Explicación:** El sistema detecta que la batería fue removida o desconectada previamente. Esto puede indicar:
- La batería está degradada o tiene mal contacto
- El conector de la batería está flojo
- El kernel reporta inconsistencias de energía que pueden causar apagones repentinos

---

### 7. Errores I2C / Drivers de Hardware

```
i2c-mt65xx: addr: 21, transfer ACK error
i2c-mt65xx: addr: 60, transfer ACK error
AW35616: Could not communicate with device over i2c!
aw35616 init fail
sc2150-driver: read chip ID fail(-6)
rt1711h: read chip ID fail(-6)
```

**Severidad:** 🟡 MEDIA

**Explicación:** Múltiples dispositivos I2C (USB Type-C controller, cargador) fallan al inicializarse. Esto afecta:
- El controlador USB Type-C (detección de cable y carga)
- La gestión de energía del cargador
- Puede contribuir a reinicios durante la carga

---

## ✅ Soluciones con Root (KernelSU)

### Solución 1: Reflashear Kernel Correctamente Alineado

El problema principal es el **Firmware Bug del kernel desalineado**. Necesitas:

```bash
# 1. Descargar una versión estable del kernel KSU para tu dispositivo
# Busca una build que NO sea "-dirty" y que sea compatible con MT6768

# 2. Flashear con KernelSU Manager o via fastboot:
adb reboot bootloader
fastboot flash boot boot-ksu-stable.img
fastboot reboot
```

**Alternativa:** Si compilas tu propio kernel, asegúrate de que la imagen esté alineada:
```bash
# En la configuración del kernel, verificar:
CONFIG_ARM64_ALIGN_KERNEL=y
```

---

### Solución 2: Optimizar Memoria con Root

Ejecuta estos comandos como root para mejorar la gestión de memoria:

```bash
# Conectar via ADB con root
adb shell
su

# Ajustar el LMK (Low Memory Killer) para ser menos agresivo
echo "0,100,200,300,900,906" > /sys/module/lowmemorykiller/parameters/adj
echo "18432,23040,27648,32256,55296,80640" > /sys/module/lowmemorykiller/parameters/minfree

# Ajustar swappiness para usar mejor la swap/zram
echo 60 > /proc/sys/vm/swappiness

# Limpiar caché de inodes y dentries
echo 3 > /proc/sys/vm/drop_caches

# Ajustar la presión de memoria para evitar OOM
echo 10 > /proc/sys/vm/vfs_cache_pressure
```

Para hacer estos cambios **permanentes**, crea un script de servicio KernelSU:

```bash
# Crear script de optimización
mkdir -p /data/adb/service.d
cat > /data/adb/service.d/memory_optimize.sh << 'EOF'
#!/system/bin/sh
# Esperar a que el sistema arranque completamente
sleep 30

# Optimizar parámetros de memoria
echo 60 > /proc/sys/vm/swappiness
echo 10 > /proc/sys/vm/vfs_cache_pressure
echo 50 > /proc/sys/vm/dirty_ratio
echo 20 > /proc/sys/vm/dirty_background_ratio

# Optimizar scheduler de I/O
for queue in /sys/block/*/queue; do
    echo "cfq" > "$queue/scheduler" 2>/dev/null
    echo 128 > "$queue/read_ahead_kb" 2>/dev/null
done
EOF
chmod 755 /data/adb/service.d/memory_optimize.sh
```

---

### Solución 3: Reducir Módulos KernelSU

Tienes **6 módulos** activos que consumen recursos. Recomendación:

```bash
# Desde KernelSU Manager o terminal:
su

# Deshabilitar módulos no esenciales temporalmente:
# Si no necesitas bloqueo de anuncios a nivel sistema:
touch /data/adb/modules/adguardcert/disable

# Si no necesitas el módulo zygisk_vector:
touch /data/adb/modules/zygisk_vector/disable

# Reiniciar para aplicar
reboot
```

**Módulos que puedes mantener:**
- `zygisksu` — Necesario para Zygisk
- `playintegrityfix` — Para apps bancarias/Google Pay
- `tricky_store` — Para integridad del keystore

**Módulos que puedes deshabilitar para probar estabilidad:**
- `adguardcert` — Usa AdGuard sin certificado del sistema
- `zygisk_vector` — Evalúa si es necesario
- `TA_utl` — Evalúa si es necesario

---

### Solución 4: Corregir Errores USB/Carga

```bash
su

# Reiniciar el subsistema USB
echo 0 > /sys/class/typec/port0/vconn_source 2>/dev/null
echo "none" > /config/usb_gadget/g1/UDC 2>/dev/null
sleep 1
# Re-habilitar
cat /sys/class/udc/*/uevent 2>/dev/null

# Si tienes problemas de carga, verificar el estado de la batería:
cat /sys/class/power_supply/battery/status
cat /sys/class/power_supply/battery/capacity
cat /sys/class/power_supply/battery/health
cat /sys/class/power_supply/battery/voltage_now
cat /sys/class/power_supply/battery/current_now
cat /sys/class/power_supply/battery/temp
```

---

### Solución 5: Deshabilitar Kernel Warnings Innecesarios (Temporal)

```bash
su

# Reducir la verbosidad del kernel para evitar overhead de logging
echo 4 > /proc/sys/kernel/printk

# Deshabilitar panic en warnings del kernel
echo 0 > /proc/sys/kernel/panic_on_warn
```

---

### Solución 6: Monitorear Temperaturas

Los congelamientos pueden estar relacionados con thermal throttling:

```bash
su

# Ver temperaturas actuales de los sensores
for tz in /sys/class/thermal/thermal_zone*/; do
    type=$(cat "$tz/type" 2>/dev/null)
    temp=$(cat "$tz/temp" 2>/dev/null)
    temp_c=$((temp / 1000))
    echo "$type: ${temp_c}°C (raw: ${temp})"
done

# Si la temperatura es alta (>45°C), el thermal throttling
# puede estar causando los congelamientos
```

---

## 🎯 Resumen y Plan de Acción

| Prioridad | Acción | Impacto Esperado |
|-----------|--------|-----------------|
| 🔴 1 | Flashear kernel KSU estable (no `-dirty`) correctamente alineado | Elimina el firmware bug y la mayoría de las warnings |
| 🔴 2 | Reducir módulos KernelSU (deshabilitar `adguardcert`, `zygisk_vector`, `TA_utl`) | Reduce consumo de RAM y posibles conflictos |
| 🟠 3 | Aplicar optimizaciones de memoria (`service.d` script) | Reduce eventos de low memory y OOM |
| 🟠 4 | Verificar salud de la batería (`has_battery_removed=1`) | Previene apagones por voltaje bajo |
| 🟡 5 | Monitorear temperaturas del SoC | Detectar si el throttling causa freezes |

### ⚠️ Nota Importante

El kernel que tienes es una **compilación custom marcada como `-dirty`** (modificada) con el tag `android15-lamu-ksu`. La fuente del kernel es de **@qdykernel** (Telegram: `https://t.me/qdykernel`). Contacta al desarrollador del kernel para reportar:
1. El firmware bug de alineación
2. Las warnings en `__alloc_pages`, `kobject_put`, `free_irq`
3. Los fallos de inicialización USB (MUSB)

Si el desarrollador no puede corregirlo, considera usar un kernel KernelSU diferente y estable para el MT6768.
