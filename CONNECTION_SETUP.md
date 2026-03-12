# Conexion Del Panel Remoto: Paso A Paso

Este documento resume todo lo que se tuvo que hacer para que la conexion entre:

- Replit backend
- UI web
- agente local en macOS

quedara funcional, y qué falta para que la UI vea al agente de forma estable en produccion.

## Estado actual

La parte local de macOS ya quedo funcionando:

- el agente conecta al backend de Replit
- `node-pty` ya puede abrir `/bin/zsh`
- el agente arranca automaticamente con `launchd`

El bloqueo restante es de despliegue en Replit:

- el agente queda conectado
- pero la UI sigue viendo `AGENT OFFLINE`
- `GET /api/agent/status` devuelve `connected:false`

Eso pasa porque el backend guarda el estado del agente en memoria y el deploy de Replit puede repartir requests entre instancias distintas.

## 1. Corregir el routing WebSocket en Replit

### Problema inicial

El agente intentaba conectar a:

```text
/ws/agent
```

pero en produccion Replit solo estaba ruteando correctamente bajo `/api`.

Resultado:

- `GET /api/healthz` respondia `200`
- `WS /ws/agent` respondia `502`

### Fix aplicado

Mover los WebSocket endpoints a:

```text
/api/ws/agent
/api/ws/browser
```

### Archivos que debian quedar consistentes

- backend relay: matcher de WebSocket
- frontend: URL del WebSocket del browser
- agente local: URL del WebSocket del agente

### Verificacion esperada

Estas pruebas deben pasar:

```text
wss://<app>.replit.app/api/ws/agent
wss://<app>.replit.app/api/ws/browser
```

Si despues del upgrade el socket se cierra con `4404 Unknown path`, el backend desplegado sigue con la logica vieja y hay que redeployar el bundle correcto.

## 2. Normalizar paths en el backend de Replit

### Problema detectado

Aunque `/api/ws/agent` ya entraba al backend, el deploy seguia cerrando con:

```text
4404 Unknown path
```

Eso indica que el backend seguia comparando paths exactos viejos.

### Fix requerido en el backend

El `ws-manager` debe aceptar ambas variantes:

```text
/ws/agent
/api/ws/agent
/ws/browser
/api/ws/browser
```

y tambien tolerar trailing slash.

### Validacion

En produccion:

- `/api/ws/agent` debe quedarse abierto
- `/api/ws/browser` debe autenticarse correctamente
- ya no debe aparecer `4404 Unknown path`

## 3. Redeployar el backend correcto en Replit

### Problema detectado

Varias veces el source ya tenia el fix, pero el binario desplegado seguia viejo.

### Lo que hubo que hacer

1. Rebuild del backend en Replit.
2. Verificar que el bundle final contenia las rutas nuevas.
3. Redeployar.

### Resultado esperado

Despues del redeploy:

- `/api/ws/agent` hace upgrade bien
- `/api/ws/browser` responde correctamente
- ya no hay `502` ni `4404`

## 4. Corregir el runtime local de Node en macOS

### Problema detectado

En la Mac el panel mostraba:

```text
[ERROR] Failed to spawn shell (/bin/zsh): posix_spawnp failed.
```

### Causa real

No era un problema de `/bin/zsh`.

Se verifico que:

- `/bin/zsh` existe
- `child_process.spawn('/bin/zsh')` funciona
- `node-pty` fallaba

La causa fue el runtime local:

```text
Node v25.6.1
```

`node-pty` no estaba funcionando correctamente con esa version en esta Mac.

## 5. Instalar Node 22 LTS

### Comando usado

```bash
brew install node@22
```

### Ruta usada

```text
/opt/homebrew/opt/node@22/bin/node
```

## 6. Recompilar `node-pty` con Node 22

### Problema

`node-pty` estaba compilado contra el ABI del Node viejo.

### Comando usado

```bash
cd /Users/alecksrodriguez/control/Nullzone-IT-Support-Agent
env PATH="/opt/homebrew/opt/node@22/bin:$PATH" npm rebuild node-pty --build-from-source
```

### Validacion

Se probo localmente y ya abrio PTY:

```text
SPAWN_OK
DATA ...
```

## 7. Fijar el agente para no usar Node no soportado

### Cambios hechos

Se actualizo el agente para fallar rapido si corre con una version no soportada de Node.

### Archivo

- `agent.js`

### Regla dejada

```text
Node soportado: >=18 y <25
Recomendado: Node 22 LTS
```

### Resultado

En vez de un error ambiguo de `posix_spawnp`, ahora el agente muestra un error claro si la version de Node no sirve.

## 8. Actualizar el instalador de `launchd`

### Problema

El instalador podia tomar el `node` global equivocado.

### Cambios hechos

Se actualizo `launchd-install.sh` para:

- preferir `node@22`
- permitir `NODE_BIN` explicito
- rechazar Node fuera del rango soportado

### Archivo

- `launchd-install.sh`

## 9. Instalar el agente como LaunchAgent

### Comando usado

```bash
cd /Users/alecksrodriguez/control/Nullzone-IT-Support-Agent
NODE_BIN=/opt/homebrew/opt/node@22/bin/node ./launchd-install.sh
```

### Resultado

Se instalo:

```text
~/Library/LaunchAgents/com.remoteshell.agent.plist
```

y el proceso quedo corriendo como:

```text
/opt/homebrew/opt/node@22/bin/node /Users/alecksrodriguez/control/Nullzone-IT-Support-Agent/agent.js
```

### Log

```text
/tmp/remoteshell-agent.log
```

## 10. Eliminar procesos duplicados del agente

### Problema detectado

Habia dos agentes corriendo a la vez y uno expulsaba al otro:

```text
Disconnected  code=4000  reason=Replaced by new agent connection
```

### Fix

Se dejo solo el proceso levantado por `launchd`.

## 11. Validar la UI real

### Lo que se verifico

1. El login del panel funciono.
2. El browser llego a `/terminal`.
3. El backend acepto sockets en `/api/ws/...`.

### Problema restante observado

Aunque el agente ya esta conectado, la UI sigue viendo:

```text
AGENT OFFLINE
```

y el endpoint:

```text
GET /api/agent/status
```

responde:

```json
{"connected":false,"connectedAt":null,"agentId":null,"ptyActive":false}
```

## 12. Bloqueo final: Replit debe usar Reserved VM

### Causa

El backend guarda el agente en memoria, algo equivalente a:

```ts
let agent = ...
```

Con `Autoscale`, el agente puede caer en una instancia y la UI/API en otra.

Entonces:

- el agente realmente esta conectado
- pero la UI consulta otra instancia
- esa otra instancia responde `connected:false`

### Fix requerido en Replit

Cambiar el deployment target de:

```text
Autoscale
```

a:

```text
Reserved VM
```

y luego redeployar.

## 13. Paso final que falta hacer en Replit

En Replit:

1. Abrir `Deploy`.
2. Cambiar el deployment target a `Reserved VM`.
3. Hacer `Redeploy`.

## 14. Verificacion final despues del cambio a Reserved VM

Despues del redeploy en `Reserved VM`, esto debe pasar:

1. El agente queda conectado.
2. `GET /api/agent/status` devuelve `connected:true`.
3. La UI cambia de `AGENT OFFLINE` a `Agent connected`.
4. La terminal ya no muestra `Waiting for agent connection`.
5. Al abrir sesion en el panel, la shell PTY arranca y responde.

## 15. Comandos utiles

### Ver version de Node 22

```bash
/opt/homebrew/opt/node@22/bin/node -v
```

### Rebuild de `node-pty`

```bash
cd /Users/alecksrodriguez/control/Nullzone-IT-Support-Agent
env PATH="/opt/homebrew/opt/node@22/bin:$PATH" npm rebuild node-pty --build-from-source
```

### Reinstalar LaunchAgent

```bash
cd /Users/alecksrodriguez/control/Nullzone-IT-Support-Agent
NODE_BIN=/opt/homebrew/opt/node@22/bin/node ./launchd-install.sh
```

### Ver logs del agente

```bash
tail -f /tmp/remoteshell-agent.log
```

### Ver si el LaunchAgent esta cargado

```bash
launchctl list | rg 'com\\.remoteshell\\.agent'
```

### Descargar el agente

```bash
launchctl unload ~/Library/LaunchAgents/com.remoteshell.agent.plist
rm ~/Library/LaunchAgents/com.remoteshell.agent.plist
```

## 16. Resumen corto

Para que esto funcione de punta a punta:

1. Replit debe exponer WebSockets bajo `/api/ws/...`.
2. El backend debe aceptar y normalizar esos paths.
3. El backend correcto debe estar redeployado.
4. La Mac debe correr con Node 22, no Node 25.
5. `node-pty` debe estar recompilado con Node 22.
6. El agente debe correr con `launchd`.
7. Replit debe usar `Reserved VM`, no `Autoscale`.

Sin el paso 7, la UI puede seguir viendo al agente como desconectado aunque la conexion del agente exista.
