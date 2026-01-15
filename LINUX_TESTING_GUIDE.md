# Guía de Continuación - Máquina Linux

**Para retomar en la máquina Linux donde tienes ble-serial funcionando**

---

## ✅ Qué ya está hecho

El código BLE ya está implementado en CHIRP:
- ✅ `chirp/drivers/baofeng_uv17Pro.py` - Funciones BLE agregadas
- ✅ `chirp/drivers/baofeng_common.py` - Sin cambios
- ✅ Análisis completo del log en `PROJECT_STATUS.md`

---

## 🧪 Testing en Linux - Pasos Inmediatos

### 1. **Verificar que los cambios están aplicados**

```bash
cd ~/chirp  # O donde tengas CHIRP
git diff chirp/drivers/baofeng_uv17Pro.py
```

Deberías ver:
- ✅ Nuevas funciones: `_is_ble_port()`, `_do_ident_ble()`, `_upload_ble()`
- ✅ Método `sync_out()` en clase `UV5RMini`
- ✅ `BLE_BLOCK_SIZE = 0x80` en `UV5RMini`

### 2. **Iniciar ble-serial**

```bash
# En la máquina Linux
ble-serial -d <MAC_DE_TU_RADIO> /tmp/ttyBLE

# Ejemplo si tu radio es 5RMINI:
# ble-serial -d AA:BB:CC:DD:EE:FF /tmp/ttyBLE
```

Debería ver:
```
[INFO] Connected to device
[INFO] Serial port exposed at /tmp/ttyBLE
```

### 3. **Verificar el puerto**

```bash
ls -la /tmp/ttyBLE
# Debería estar accesible
```

### 4. **Abrir CHIRP**

```bash
# En otra terminal
python -m chirp.wxui.main &
```

### 5. **Seleccionar puerto BLE**

```
CHIRP → Radio → Download from Radio
         ↓
    Port: /tmp/ttyBLE
         ↓
    Model: Baofeng UV-5R Mini
         ↓
    Download
```

### 6. **Observar los logs**

En la terminal de CHIRP, deberías ver:

```
[INFO] Using BLE upload implementation
[INFO] Using BLE block size: 128 bytes
[INFO] BLE connection detected
[DEBUG] BLE-specific identification sequence
[DEBUG] Sending SEND! command for BLE
[DEBUG] ACK received for block 0x0000
[DEBUG] ACK received for block 0x0080
...
```

---

## ⚠️ Si algo falla

### Log muestra "Using USB upload implementation"
```
Significa: _is_ble_port() retorna False
Solución: El puerto NO tiene "ble" o "/tmp/ttyble" en el nombre
          Verifica: print(radio.pipe.port)
```

### Timeout error
```
Significa: El radio no responde en 3 segundos
Posibles causas:
1. Radio no conectado via BLE
2. ble-serial no funcionando
3. Latencia de BLE muy alta
Solución: Aumentar timeout en _do_ident_ble() a 5.0 segundos
```

### ACK error (Bad ack writing block)
```
Significa: Radio no retorna 0x06 esperado
Posibles causas:
1. Tamaño de bloque incorrecto (usa 128 bytes, no 64)
2. Datos corruptos en transmisión
3. Radio en estado incorrecto
Solución: Ver en el log qué datos recibió
```

---

## 📊 Debugging Avanzado

### Habilitar logs verbosos

```python
# En chirp/logger.py o donde configures logs
logging.getLogger('chirp.drivers.baofeng_uv17Pro').setLevel(logging.DEBUG)
logging.getLogger('chirp.drivers.baofeng_common').setLevel(logging.DEBUG)
```

### Ver comunicación serial

```bash
# En otra terminal, monitorear el puerto
cat /tmp/ttyBLE | xxd -g 1
```

Verás bytes en hex:
```
00000000: 50 52 4f 47 52 41 4d 43  4f 4c 4f 52 50 52 4f 55  PROGRAMCOLORPROU
00000010: 06 46 01 36 01 74 04 00  05 20 02 20 02 60 01 03  .F.6.t... . .`..
...
```

### Capturar nuevo log

```bash
# Si quieres capturar otro btsnoop_hci.log
# En Android/Linux, según donde hagas el sniffer:
adb shell "btsnoop_log"
# O desde el adaptador BLE
```

---

## 🔍 Validación de Parámetros

### Verificar tamaño de bloques

Cuando veas en logs:
```
[DEBUG] Sending address 0x0000 (BLE block size: 128)
[DEBUG] Sending address 0x0080 (BLE block size: 128)
[DEBUG] Sending address 0x0100 (BLE block size: 128)
```

✅ Correcto = direcciones incrementan en 0x80 (128 bytes)
❌ Incorrecto = incrementan en 0x40 (64 bytes)

### Verificar comando SEND!

En logs debería ver:
```
[DEBUG] Sending SEND! command for BLE
[DEBUG] SEND! command acknowledged
```

Si no lo ves, el comando no se envía correctamente.

---

## 📈 Pruebas Progresivas

### Fase 1: Solo lectura (Download)
```
1. CHIRP → Radio → Download from Radio
2. Verifica que descarga correctamente
3. Compara con descarga via USB
```

### Fase 2: Escritura (Upload)
```
1. Modifica un canal en CHIRP
2. CHIRP → Radio → Upload to Radio
3. Verifica en el radio físico que el cambio se grabó
```

### Fase 3: Datos completos
```
1. Descarga configuración actual
2. Carga archivo .img de otra radio
3. Sube a la 5R Mini via BLE
4. Verifica que funciona
```

---

## 📝 Checklist para Retomar

- [ ] Código BLE está en `baofeng_uv17Pro.py`
- [ ] ble-serial instalado y funcionando
- [ ] Puerto `/tmp/ttyBLE` accesible
- [ ] Radio emparejada via BLE en Linux
- [ ] CHIRP actualizado con los cambios
- [ ] Logs configurados en DEBUG
- [ ] Test de descarga completado
- [ ] Test de carga completado
- [ ] Verificar tamaño de bloques en logs
- [ ] Documentar cualquier diferencia vs USB

---

## 🚀 Próximos Pasos Después de Testing

### Si todo funciona en Linux:
1. ✅ Documentar logs exitosos
2. ✅ Comparar con logs via USB
3. ✅ Ir a Windows con NRF52840

### Si hay problemas:
1. Capturar nuevo btsnoop_hci.log
2. Analizar con tshark
3. Comparar con log anterior
4. Ajustar parámetros según sea necesario

---

## 💾 Archivos de Referencia

En el proyecto ya tienes:
- `PROJECT_STATUS.md` - Estado completo del proyecto
- `BT_COMMUNICATION_ANALYSIS.md` - Análisis técnico del log
- `BLE_MINIMAL_CHANGES.md` - Explicación de cambios
- `btsnoop_hci.log` - Log original capturado
- `bt_analysis.txt` - Extracción de tshark

---

## 🔗 Comandos Rápidos

```bash
# Iniciar ble-serial
ble-serial -d AA:BB:CC:DD:EE:FF /tmp/ttyBLE &

# Abrir CHIRP con debug
PYTHONPATH=. python -m chirp.wxui.main 2>&1 | tee chirp.log

# Ver logs de CHIRP en tiempo real
tail -f chirp.log | grep -i ble

# Monitorear puerto BLE
cat /tmp/ttyBLE | xxd -g 1

# Cancelar ble-serial
pkill ble-serial
```

---

## 📞 Notas para Próxima Sesión

- **Objetivo**: Validar que funciona en Linux exactamente como en el log capturado
- **Entregables**: Logs de prueba exitosa
- **Siguiente**: Firmware NRF52840 para Windows

---

**Estado**: Listo para Testing en Linux
**Máquina**: La que tienes con ble-serial
**Tiempo estimado**: 15-30 minutos para validar

¡Cuando estés en Linux, ejecuta el checklist y avísame qué tal funciona!
