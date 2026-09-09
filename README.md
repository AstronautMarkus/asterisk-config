# Reyes X BeerCat asterisk-config
 
<div align="center">
  <img height="350" alt="image" src="https://github.com/user-attachments/assets/14353d99-2d63-4687-9456-d1e94e2cd7af"/>
</div>

Configuración de la centralita Asterisk para la familia Reyes: dialplan, IVR y audios asociados. La estructura de carpetas del repositorio espeja las rutas reales del servidor, para que el despliegue sea una simple copia de archivos.

## Estructura del repositorio

```
etc/asterisk/
  extensions.conf      dialplan: extensiones internas, IVR 5000 y subrutina de resultado de llamada
  musiconhold.conf     clase de MOH "cisco" usada durante el Dial()

var/lib/asterisk/
  sounds/custom/       audios del IVR (formato 8000 Hz, mono, PCM s16le)
  moh/cisco/           audio de espera para la clase MOH "cisco"
```

Cada ruta dentro del repo corresponde 1:1 con su ubicación real en el servidor (por ejemplo `etc/asterisk/extensions.conf` va a `/etc/asterisk/extensions.conf`).

## Despliegue

```bash
sudo cp -r var/lib/asterisk/sounds/custom/* /var/lib/asterisk/sounds/custom/
sudo cp -r var/lib/asterisk/moh/cisco/*     /var/lib/asterisk/moh/cisco/
sudo cp etc/asterisk/extensions.conf         /etc/asterisk/
sudo cp etc/asterisk/musiconhold.conf        /etc/asterisk/
sudo asterisk -rx "core reload"
```
## Extensiones internas

Contexto `[familia-reyes]`, marcado directo entre extensiones:

| Extension | Persona |
|-----------|---------|
| 3004      | Carla   |
| 1404      | Tiago   |
| 2406      | Isaura  |
| 0301      | Carlos  |
| 1603      | Marcos  |

## IVR (extensión 5000)

Al llamar a la extensión 5000 se responde la llamada, se reproduce `bienvenida-familia` y se entra al menú principal (`[ivr-familia]`).

### Menú principal

- Reproduce `menu-familia` y espera 5 segundos una opción.
- Si no hay respuesta, vuelve a reproducir el mismo menú y espera otros 5 segundos.
- Si tras el segundo intento sigue sin haber respuesta, reproduce `despedida-familia` y cuelga.
- Una opción inválida reproduce `opcion-invalida` y reintenta (máximo dos intentos combinados con los timeouts) antes de despedirse.

Opciones del menú:

| Tecla | Acción                          |
|-------|----------------------------------|
| 1     | Llamar a Carla                   |
| 2     | Llamar a Tiago                   |
| 3     | Llamar a Isaura                  |
| 4     | Llamar a Carlos                  |
| 5     | Llamar a Marcos                  |
| 9     | Repetir el menú                  |
| 0     | Salir (`despedida-familia`)      |

### Resultado de la llamada

Al elegir una persona, el Dial() tiene un timeout de 120 segundos (2 minutos). Al finalizar el intento de llamada, la subrutina `[resultado-llamada]` evalúa `DIALSTATUS`:

| DIALSTATUS                | Audio reproducido |
|----------------------------|--------------------|
| ANSWER                     | (ninguno, la llamada se completó) |
| BUSY                       | `ocupado`          |
| NOANSWER / CANCEL          | `no-contesta`      |
| CHANUNAVAIL / otro         | `no-en-linea`      |

Después de reproducir el audio correspondiente, se reproduce `volver-a-llamar` y se espera una respuesta:

| Tecla | Acción |
|-------|--------|
| 1     | Vuelve al menú principal |
| 2     | Reproduce `llamada-finalizada` y cuelga |

Una opción inválida o un timeout reintenta una vez (reutilizando `opcion-invalida` o `no-seleccion` según el caso); al segundo fallo, finaliza la llamada.

## Audios

Todos los archivos en `var/lib/asterisk/sounds/custom/` y `var/lib/asterisk/moh/cisco/` están normalizados a 8000 Hz, mono, PCM s16le (formato nativo de Asterisk). Para convertir un audio nuevo al mismo formato:

```bash
ffmpeg -y -i origen.wav -ar 8000 -ac 1 -c:a pcm_s16le destino.wav
```
