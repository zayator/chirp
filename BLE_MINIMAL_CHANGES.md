# Modificaciones para Soporte BLE - Cambios Mínimos No-Invasivos

## Resumen
Se añadieron **nuevas funciones específicas para BLE** sin modificar el código existente que funciona con USB.

## Archivos Modificados

### 1. **chirp/drivers/baofeng_uv17Pro.py**

#### Nuevas Funciones Añadidas:

```python
def _is_ble_port(radio):
    """Detecta automáticamente puertos BLE"""
    # Retorna True si puerto contiene 'ble' o '/tmp/ttyble'
```

```python
def _do_ident_ble(radio):
    """Identificación BLE con parámetros ajustados"""
    - Timeout: 3.0 segundos (vs 1.5 para USB)
    - Envía comando "SEND!" adicional
    - Logging mejorado
```

```python
def _upload_ble(radio):
    """Upload con bloques de 128 bytes para BLE"""
    - Usa BLE_BLOCK_SIZE (0x80 = 128 bytes)
    - Mejor validación de ACKs
    - Logging detallado
```

#### Código Original Sin Cambios:
```python
def _do_ident(radio):
    # ✅ INTACTO - Funciona como antes
    
def _upload(radio):
    # ✅ INTACTO - Funciona como antes
```

#### Cambio en Clase UV5RMini:

**Agregado**: BLE_BLOCK_SIZE = 0x80

**Agregado**: Método sync_out() que detecta BLE automáticamente:
```python
def sync_out(self):
    """Upload a radio - Auto-select BLE o USB"""
    if _is_ble_port(self):
        _upload_ble(self)  # Usa funciones BLE
    else:
        _upload(self)      # Usa funciones USB normales
```

### 2. **chirp/drivers/baofeng_common.py**

✅ **SIN CAMBIOS** - Todo el código original se mantiene igual

## Flujo de Ejecución

### Conexión USB (comportamiento original):
```
Usuario → Selecciona COM4
          ↓
    sync_out()
          ↓
    _is_ble_port() → False
          ↓
    _upload() [FUNCIÓN ORIGINAL]
          ↓
    Funciona exactamente como antes ✅
```

### Conexión BLE (nuevo):
```
Usuario → Selecciona /tmp/ttyBLE
          ↓
    sync_out()
          ↓
    _is_ble_port() → True
          ↓
    _upload_ble() [FUNCIÓN NUEVA]
          ↓
    - Timeout: 3.0s
    - Bloque: 128 bytes
    - Comando SEND!
          ↓
    ✅ Funciona con BLE
```

## Ventajas de este Enfoque

✅ **Seguro**: No toca código que funciona
✅ **Minimal**: Solo añade lo necesario para BLE
✅ **Reversible**: Se puede quitar fácilmente
✅ **Claro**: Muy obio dónde está la lógica BLE
✅ **Testeable**: Funciones separadas, fácil debuguear
✅ **Mantenible**: USB y BLE son independientes

## Testing

Para probar:

1. **USB (debe funcionar como siempre)**:
   ```
   - Conecta radio por USB
   - Intenta subir canales
   - Debe funcionar igual que antes
   ```

2. **BLE (nueva funcionalidad)**:
   ```
   - Inicia ble-serial: /tmp/ttyBLE
   - Abre CHIRP
   - Selecciona puerto /tmp/ttyBLE
   - Intenta subir canales
   - Debe funcionar con bloques de 128 bytes
   ```

## Logs Esperados

**USB**:
```
Using USB upload implementation
```

**BLE**:
```
Using BLE upload implementation
Using BLE block size: 128 bytes
BLE-specific identification sequence
Sending SEND! command for BLE
```

## Rollback

Si algo falla, solo elimina:
- Función `_is_ble_port()`
- Función `_do_ident_ble()`
- Función `_upload_ble()`
- BLE_BLOCK_SIZE en UV5RMini
- Método sync_out() en UV5RMini

El código USB seguirá funcionando sin cambios.
