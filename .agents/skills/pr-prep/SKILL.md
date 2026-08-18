---
name: pr-prep
description: Corrida completa de los quality gates locales antes de pushear y abrir PR. Úsala cuando el usuario pida "preparar PR", "checklist PR", "correr gates", "verificar antes de PR", o antes de invocar la skill create-pr.
---

# PR Prep

Ejecuta la verificación completa de calidad en el repositorio antes de hacer un push. 

## Instrucciones
1. Ejecuta linters de Python si están disponibles (ej. uff check ., andit -r .).
2. Ejecuta linters de Node/JS si están disponibles (ej. 
pm run lint o eslint .).
3. Si alguno falla, detente y muestra el output para que el usuario o el agente lo arreglen usando lint-fix-loop.
4. Si todo pasa exitosamente, indica que el código está listo para PR.
