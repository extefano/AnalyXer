---
name: branch-check
description: Valida la rama actual antes de pushear: prefijo permitido, sincronizada con main, sin cambios sin stagear. Úsala cuando el usuario pida "validar rama", "revisar rama", "check branch".
---

# Branch Check

Valida que la rama actual cumple con las convenciones del repositorio.

## Instrucciones
1. El prefijo de la rama debe ser uno de: eature/, ix/, chore/, docs/, efactor/.
2. Comprueba que no haya cambios sin commitear (si los hay, avisa al usuario).
3. Verifica que la rama esté actualizada respecto a main (usa git fetch origin main y compara).
