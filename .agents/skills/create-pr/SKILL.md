---
name: create-pr
description: Crea un pull request con gh CLI siguiendo la convención del repo. Úsala cuando el usuario pida abrir / crear / submit un PR, o cuando haya commits sin pushear en una rama.
---

# Create Pull Request

## Cuándo usarla
Después de commitear cambios en una rama de trabajo (eature/*, ix/*, etc.) y el usuario pide abrir o crear un PR.

## Instrucciones
1. Verifica que la rama actual esté pusheada (git push -u origin HEAD).
2. Recopila un resumen de los commits en la rama.
3. Usa gh pr create --title "<título>" --body "<body>" para crear la PR.
4. Asegúrate de que el título siga Conventional Commits.
