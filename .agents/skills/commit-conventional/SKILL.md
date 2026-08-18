---
name: commit-conventional
description: Sugiere un mensaje de commit Conventional Commits para los cambios staged. Úsala cuando el usuario pida un commit, mensaje de commit, o cómo resumir cambios; o cuando veas archivos staged en git y haya que cerrar el commit.
---

# Conventional Commit Helper

El proyecto usa **Conventional Commits**. Reglas que el mensaje debe cumplir:

- **Formato**: <tipo>: <descripción imperativa> en una sola línea, máximo 100 caracteres totales.
- **Tipos válidos**: eat, ix, docs, style, efactor, perf, 	est, uild, ci, chore, evert.
- **Ejemplo**: eat: agregar soporte para Claude Code o ix: corregir parseo de JSON en diff.

**Pasos**:
1. Analizar los cambios en git diff --staged.
2. Generar 2 o 3 opciones de mensajes válidos.
3. Pedir confirmación al usuario antes de hacer el commit con git commit -m "...".
