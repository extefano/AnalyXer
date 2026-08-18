---
name: lint-fix-loop
description: Loop iterativo de diagnóstico y arreglo de errores de lint, typecheck o test. Úsala cuando un check falló y el usuario pida "arreglar", "fix lint", "iterar".
---

# Lint Fix Loop

Cuando un linter (Ruff, ESLint, Bandit, etc.) falla, entra en este loop:

1. **Diagnosticar**: Corre el comando que falló (ej. uff check . o eslint).
2. **Analizar**: Lee los errores reportados, enfócate en el archivo con más errores primero.
3. **Arreglar**: Modifica el archivo usando las herramientas de edición de código.
4. **Verificar**: Vuelve a correr el linter. Repite el loop hasta que el archivo pase. Continúa con el siguiente archivo.
