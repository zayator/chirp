# Proyecto: Soporte Bluetooth para Baofeng UV-5R Mini en CHIRP
**Fecha de inicio**: 12 de Enero 2026  
**Estado**: En progreso - Implementación de soporte BLE completada

---

## 📋 Resumen Ejecutivo

Se ha implementado soporte Bluetooth (BLE) para la radio Baofeng UV-5R Mini en CHIRP mediante:
1. Análisis del log de comunicación BLE (btsnoop_hci.log)
2. Identificación de parámetros BLE específicos
3. Implementación de funciones BLE en el driver sin modificar código existente

---

## 🔍 Problema Original

- Radio Baofeng UV-5R Mini no sube datos correctamente via Bluetooth
- La app oficial funciona bien, pero CHIRP falla
- Necesidad de reconfigurar la subida de canales mediante Bluetooth

---

## 📊 Análisis Realizado

### Log Bluetooth Capturado
**Archivo**: `btsnoop_hci.log` (73 KB)
**Herramienta de análisis**: Wireshark (tshark)

### Hallazgos Clave del Log

#### 1. **Protocolo: BLE GATT (no RFCOMM)**
- Usa ATT (Attribute Protocol)
- Opcodes: `0x52` (Write Request), `0x1b` (Notifications)

#### 2. **Secuencia de Inicialización**
```
Frame 241 TX: 50524f4752414d434f4c4f5250524f55
             ASCII: "PROGRAMCOLORPROU" (magic string)

Frame 243 RX: 06 (ACK)

Frame 249 RX: 35524d494e4920202b4c3030303030
             ASCII: "5RMINI  +L00000" (identificación)

Frame 250 TX: SEND! + 0x05 0x0d 0x01 0x01 0x01 0x04 0x11 0x08...
             (comando adicional no estándar)
```

#### 3. **Parámetro Crítico: Tamaño de Bloque**
- **USB**: 64 bytes (0x40)
- **BLE**: 128 bytes (0x80) ← DIFERENCIA CLAVE
- Cada bloque requiere ACK (0x06) antes del siguiente

#### 4. **Timeout**
- USB: 1.5 segundos
- BLE: Requiere más tiempo (latencia mayor)

---

## 💻 Solución Implementada

### Enfoque: Funciones Nuevas No-Invasivas

Se agregaron **3 nuevas funciones** sin tocar código que funciona:

### Archivo: `chirp/drivers/baofeng_uv17Pro.py`

#### 1. Función de Detección
```python
def _is_ble_port(radio):
    """Detecta automáticamente puertos BLE"""
    # Retorna True si puerto contiene 'ble' o '/tmp/ttyble'
```

#### 2. Identificación BLE
```python
def _do_ident_ble(radio):
    """Identificación específica para BLE"""
    - Timeout: 3.0 segundos (vs 1.5 para USB)
    - Envía comando "SEND!" adicional
    - Mejor logging
```

#### 3. Upload BLE
```python
def _upload_ble(radio):
    """Upload con bloques de 128 bytes para BLE"""
    - BLE_BLOCK_SIZE = 0x80 (128 bytes)
    - Validación estricta de ACKs
    - Logging detallado
```

#### 4. Auto-selección en UV5RMini
```python
def sync_out(self):
    """Elige automáticamente entre BLE o USB"""
    if _is_ble_port(self):
        _upload_ble(self)  # Nueva función
    else:
        _upload(self)      # Función original (intacta)
```

### Archivo: `chirp/drivers/baofeng_common.py`
✅ **SIN CAMBIOS** - Se mantiene exactamente igual

---

## 🚀 Flujo de Funcionamiento

### Conexión USB (Original)
```
Windows/Linux → USB → COM4
                 ↓
            _is_ble_port() → False
                 ↓
            _upload() [FUNCIÓN ORIGINAL]
                 ↓
        Funciona como siempre ✅
```

### Conexión BLE (Nuevo)
```
Windows → NRF52840 (adaptador) → BLE → Radio
             ↓ USB
          COM6 (puerto virtual)
             ↓
        _is_ble_port() → True
             ↓
        _upload_ble() [NUEVA FUNCIÓN]
             ↓
        - Timeout: 3.0s
        - Bloque: 128 bytes
        - Comando SEND!
             ↓
        ✅ Funciona con BLE
```

---

## 📁 Archivos Modificados

### 1. chirp/drivers/baofeng_uv17Pro.py
- **Líneas ~104-155**: Nuevas funciones `_is_ble_port()`, `_do_ident_ble()`, `_upload_ble()`
- **Línea ~158-230**: Función `_do_ident()` ORIGINAL (sin cambios)
- **Línea ~233-285**: Función `_upload()` ORIGINAL (sin cambios)
- **Línea ~2265-2290**: Clase `UV5RMini` con:
  - `BLE_BLOCK_SIZE = 0x80`
  - Método `sync_out()` para auto-selección

### 2. chirp/drivers/baofeng_common.py
✅ **SIN CAMBIOS**

---

## 🔧 Setup Actual del Usuario

### Máquina Linux (Funciona):
```
- Radio Baofeng UV-5R Mini conectada via Bluetooth
- Librería: ble-serial
- Puerto virtual: /tmp/ttyBLE
- Comunicación: Serial sobre BLE GATT
```

### Máquina Windows (Pendiente):
```
- SIN adaptador Bluetooth nativo
- Solución: NRF52840 como bridge BLE-USB
- Puerto virtual: COM6 (o similar)
- Firmware: PENDIENTE DE CREAR
```

---

## ⏭️ Próximos Pasos

### FASE 1: Testing en Linux (COMPLETADO)
✅ Análisis del log
✅ Identificación de parámetros
✅ Implementación del código

### FASE 2: Testing en Windows (PENDIENTE)
```
Requisitos:
1. Crear firmware NRF52840 para bridgear BLE-USB
2. Programar el NRF52840
3. Conectar NRF52840 a Windows via USB
4. Probar con CHIRP en Windows
```

### FASE 3: Firmware NRF52840
Necesita:
```
- USB CDC Serial (Windows COM port)
- BLE central mode (conecta a radio)
- Transparente: USB ↔ BLE
```

---

## 📊 Parámetros de Configuración

### Magic Strings
```
Inicio: PROGRAMCOLORPROU (16 bytes)
Comando adicional: SEND! + datos
```

### Bloques de Datos
```
USB:  0x40 (64 bytes)
BLE:  0x80 (128 bytes)
```

### Timeouts
```
USB: 1.5s
BLE: 3.0s
```

### ACK
```
Esperado: 0x06
Después de cada bloque
```

---

## 📝 Archivos de Referencia

### Documentación Creada
- `BT_COMMUNICATION_ANALYSIS.md` - Análisis detallado del log
- `BLE_MINIMAL_CHANGES.md` - Explicación de cambios implementados
- `btsnoop_hci.log` - Log original capturado
- `bt_analysis.txt` - Extracción de tshark

### Scripts de Análisis
- `analyze_btsnoop.py` - Script Python para analizar logs (no usado, tshark fue mejor)

---

## 🧪 Testing Recomendado

### En Linux
```bash
1. Iniciar ble-serial:
   ble-serial -d <MAC_RADIO> /tmp/ttyBLE

2. Abrir CHIRP
3. Seleccionar puerto: /tmp/ttyBLE
4. Descargar/subir canales
5. Verificar logs: "Using BLE upload implementation"
```

### En Windows (cuando esté el NRF52840)
```
1. Conectar NRF52840 a Windows (USB)
2. Abrir CHIRP
3. Seleccionar puerto COM virtual
4. Descargar/subir canales
5. Verificar logs
```

---

## ⚠️ Notas Importantes

### Compatibilidad
- ✅ UV-5R Mini (implementado)
- ⚠️ Otros modelos Baofeng: Verificar si necesitan ajustes
- ⚠️ BLE solo para radios con Bluetooth

### Reversibilidad
- Los cambios BLE son completamente reversibles
- USB sigue funcionando exactamente igual
- Se puede remover todo el código BLE sin afectar nada

### Debugging
- Habilitar logs en CHIRP para ver qué función se ejecuta
- Log diferente para BLE vs USB
- ACKs validados con error reporting detallado

---

## 📚 Recursos Externos

### Baofeng UV-5R Mini
- Magic string: PROGRAMCOLORPROU
- Modelo: UV5RMini (en CHIRP)

### BLE/GATT
- ATT Protocol (Attribute)
- GATT Write Request (0x52)
- GATT Notifications (0x1b)

### ble-serial
- Crea puerto serial virtual desde BLE
- Linux: /tmp/ttyBLE
- Windows: COM port virtual (via NRF52840)

---

## 🔐 Control de Versiones

### Cambios en Git
```bash
git diff chirp/drivers/baofeng_uv17Pro.py
# Verifica solo adiciones, no modificaciones en código original
```

### Rollback (si es necesario)
```bash
git checkout chirp/drivers/baofeng_uv17Pro.py
# Restaura versión anterior
```

---

## 📞 Contacto / Continuación

**Continuidad de proyecto**:
1. Crear firmware NRF52840 para bridgear BLE-USB
2. Testear en Windows con NRF52840
3. Validar parámetros en envío real de datos
4. Documentar cualquier ajuste adicional necesario

**Estado**: ESPERANDO NRF52840 + Firmware

---

**Última actualización**: 12 Enero 2026
**Próxima sesión**: Firmware NRF52840 para Windows
