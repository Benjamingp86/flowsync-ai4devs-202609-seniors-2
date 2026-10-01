# Prompts

Aquí van **todos los prompts que lanzaste** para hacer el ejercicio, en el orden en que los
lanzaste, con el modelo y la herramienta de cada uno.

Esto no es papeleo. Lo que se revisa es **cómo pediste las cosas**, no solo lo que salió: un
resultado flojo con un prompt bueno y un resultado flojo con un prompt vago necesitan feedback
distinto, y sin este archivo no se distinguen.

## Cómo rellenarlo

- Un apartado `## Prompt N` por cada prompt.
- **Pega el prompt tal cual lo lanzaste**, dentro del bloque de código, aunque ocupe diez líneas
  y aunque tenga faltas. No lo reescribas para que quede bien: el que arreglaste mentalmente
  después no es el que lanzaste.
- Incluye también los que **no funcionaron**. Suelen ser los más útiles de leer.
- `Modelo` y `Herramienta` en todos. Si cambiaste de una a otra a mitad, se nota aquí.

Borra el ejemplo de abajo cuando escribas el primero.

---

## Prompt 1

**Modelo:** Opus 5.5
**Herramienta:** Claude Code (app de escritorio, código leído en WSL)

```
ok  hay va el prompt inicial : "Lee el código de FlowSync que hace el registro, el login, la sesión y el perfil, y escríbeme un documento que describa cómo se comporta hoy, con este formato…"
```

**Qué salió:** el prompt no decía el formato ni dónde guardar el archivo (acaba en "con este formato…"); el agente tomó el formato, las reglas y la ruta `docs/spec-viva/bgp.md` de lo hablado antes en la misma sesión. Escribió 20 requisitos de las dos capas y probó algunas respuestas de la API con curl contra el backend local.

## Prompt 2

**Modelo:** Opus 5.5
**Herramienta:** Claude Code (app de escritorio, código leído en WSL)

```
enséñame el código del requisito credenciales incorrectas indistinguibles
```