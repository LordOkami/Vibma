# Fork de Vibma — por qué existe y qué cambia

Fork local de [ufira-ai/vibma](https://github.com/ufira-ai/vibma) (MIT). Upstream
está **sin desarrollo activo** desde que Figma sacó su MCP nativo, así que no hay
a quién mandarle esto como PR ni versión futura que lo incorpore. Esa es
justamente la razón de forkear en lugar de esperar.

## El cambio: `frames.export` puede escribir a disco

Un parámetro nuevo, `path`. Sin él, el comportamiento es idéntico al de upstream.

```
frames(method: "export", id: "4995:204", path: "fastlane/screenshots/en-US/EXPORT_04.png")
→ Wrote 1483920 bytes to /Users/.../EXPORT_04.png
```

### Por qué hacía falta

`frames.export` devolvía la imagen **solo** como contenido MCP, es decir, al
contexto del agente. Eso está bien para mirar un frame y es inservible para
cualquier otra cosa:

- un PNG a tamaño real cuesta miles de tokens recibirlo,
- y una vez recibido **no se puede guardar**: el contenido MCP no es un buffer
  que el agente pueda escribir.

Con lo cual exportar un conjunto —las capturas de App Store de un locale, una
página de artboards— no era lento: era **imposible**. El caso que lo destapó
fueron 160 frames de Kotomaji (16 locales × 10 capturas), que había que sacar a
mano desde la UI de Figma mientras todo lo demás del pipeline estaba
automatizado.

`path` convierte `export` de herramienta de inspección en paso de build.

### Los ficheros tocados

| Fichero | Cambio |
|---|---|
| `schema/tools/frames.yaml` | Declara el parámetro `path` en `export` |
| `packages/core/src/tools/types.ts` | `methodFormatters` pasa a recibir `(result, params)` |
| `packages/core/src/tools/registry.ts` | El call site pasa `params` al formateador |
| `packages/core/src/tools/mcp-registry.ts` | El formateador de `export` escribe a disco si hay `path` |

El formateador solo recibía `result`, y por eso no podía ver `path`. Pasar
`params` es el cambio de fondo; lo demás cuelga de ahí.

Detalles de la implementación que no son obvios:

- **`SVG_STRING` se escribe como UTF-8, no como base64.** Es el único formato que
  llega como texto, y tratarlo como binario produce un fichero corrupto sin
  ningún error.
- **Los directorios se crean solos** (`mkdirSync recursive`), porque el caso de
  uso es escribir en `<out>/<locale>/` y exigir que existan las 16 carpetas
  convierte un comando en un script.
- **Un fallo de escritura devuelve `isError`**, no una excepción: el agente tiene
  que poder distinguir "no se pudo escribir" de "el export falló".

## Por qué no se parcheó el paquete de npm

La configuración MCP resolvía `npx -y @ufira/vibma@latest`, que vive en
`~/.npm/_npx/<hash>/`. Cualquier parche ahí lo borra la siguiente resolución, en
silencio. Un fork con fuente es lo único que sobrevive.

## Cómo se usa

La configuración MCP apunta a este directorio en lugar de al paquete de npm.
Tras cualquier cambio en `packages/`:

```bash
npm run build     # core + adapter-figma + tunnel
```

y reiniciar la sesión para que recargue el servidor MCP.

El relay (`packages/tunnel`) no está tocado: `npx @ufira/vibma-tunnel` del
paquete publicado sigue valiendo, igual que `npm run socket` desde aquí.

## Lo que sigue sin poder hacerse

El relay admite **un cliente `mcp` y uno `plugin` por canal**. Dos procesos que
quieran hablar con el mismo fichero de Figma a la vez necesitan dos canales, y el
plugin solo está en uno. No es algo que este fork arregle; es el diseño del
relay.
