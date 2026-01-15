# Análisis de Comunicación Bluetooth - Baofeng UV-5R Mini

## Hallazgos Clave del Log BTSnoop

### 1. **Protocolo Utilizado: BLE (Bluetooth Low Energy) con GATT**
- **NO es RFCOMM** como el serial tradicional
- Usa **ATT (Attribute Protocol)** sobre L2CAP
- Opcodes principales:
  - `0x52`: Write Request (envío de datos al radio)
  - `0x1b`: Handle Value Notification (respuesta del radio)

### 2. **Secuencia de Inicialización**

```
Frame 241: TX -> 50524f4752414d434f4c4f5250524f55
             ASCII: "PROGRAMCOLORPROU"
             
Frame 243: RX <- 06 (ACK)

Frame 244: TX -> 46 ("F")

Frame 246: RX <- 01360174040005200220026001035003

Frame 247: TX -> 4d ("M")

Frame 249: RX <- 35524d494e4920202b4c3030303030
             ASCII: "5RMINI  +L00000"
             ^^^ Identificación del radio UV-5R Mini
             
Frame 250: TX -> 53454e4421050d010101041108050d0d01110f091209100400
             ASCII: "SEND!" + data
```

### 3. **Comandos de Escritura (Upload)**

Formato del comando Write:
```
W [ADDR_HIGH][ADDR_LOW][SIZE] [DATA...]

Ejemplos:
Frame 253: 57 00 00 80 [128 bytes de datos]
           W  addr=0x0000, size=0x80 (128 bytes)
           
Frame 256: 57 00 80 80 [128 bytes de datos]
           W  addr=0x0080, size=0x80
           
Frame 259: 57 01 00 80 [128 bytes de datos]
           W  addr=0x0100, size=0x80
```

### 4. **Patrón de Comunicación**

```
TX: Write Command (128 bytes de datos)
RX: ACK (0x06)
TX: Write Command (128 bytes de datos)
RX: ACK (0x06)
... [repetir]
```

### 5. **Tamaño de Bloques**
- **Bloque de escritura: 128 bytes (0x80)**
- Cada bloque recibe un ACK antes de enviar el siguiente
- Direcciones incrementales: 0x0000, 0x0080, 0x0100, 0x0180, etc.

### 6. **Magic Strings Identificados**

1. **Inicio de sesión**: `PROGRAMCOLORPROU` (16 bytes)
2. **Identificación**: Respuesta contiene `5RMINI  +L00000`
3. **Comando adicional**: `SEND!` seguido de datos de configuración

## Diferencias con la Implementación Actual de CHIRP

### Implementación Actual (baofeng_uv17Pro.py):
```python
BAUD_RATE = 115200
BLOCK_SIZE = 0x40  # 64 bytes
Magic: MSTRING_UV17PROGPS = b"PROGRAMCOLORPROU"
```

### Log Bluetooth:
```
No hay baudrate (es BLE)
BLOCK_SIZE = 0x80  # 128 bytes en Bluetooth
Magic: "PROGRAMCOLORPROU" ✓ (coincide)
Protocolo: BLE GATT (no serial tradicional)
```

## Recomendaciones para Adaptación

### 1. **Detectar Conexión BLE**
- Identificar si el puerto es un dispositivo BLE
- Adaptar el tamaño de bloque: 64 bytes (USB) vs 128 bytes (BLE)

### 2. **Ajustar Timeouts**
- BLE puede tener latencias mayores
- Aumentar timeout de 1.5s a 3-5s para Bluetooth

### 3. **Implementar Protocolo GATT**
- La comunicación actual asume serial directo
- BLE requiere escribir a características GATT específicas

### 4. **Comando "SEND!"**
- El log muestra un comando adicional después del magic string
- Datos: `53454e4421050d010101041108050d0d01110f091209100400`

### 5. **ACK Pattern**
- Esperar `0x06` después de cada bloque
- No continuar hasta recibir ACK

## Archivos a Modificar

1. **chirp/drivers/baofeng_uv17Pro.py**
   - Agregar soporte para BLE
   - Ajustar BLOCK_SIZE dinámicamente
   - Implementar comando "SEND!"

2. **chirp/drivers/baofeng_common.py**
   - Detectar tipo de conexión (USB vs BLE)
   - Ajustar timeouts para BLE
   - Implementar espera de ACK estricta

3. **chirp/wxui/clone.py**
   - Detectar puertos BLE
   - No filtrar dispositivos Bluetooth emparejados

## Siguiente Paso

Crear una capa de adaptación BLE que traduzca las operaciones seriales a operaciones GATT, permitiendo que el driver actual funcione sin cambios mayores.
