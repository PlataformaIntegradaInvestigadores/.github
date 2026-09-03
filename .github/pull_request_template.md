## ¿Qué cambió?
Breve descripción del cambio.

## ¿Por qué?
Problema / requerimiento / necesidad que originó esto.

## ¿Cómo?
Solución + cambios técnicos principales (bullets).

## Notas
Consideraciones de review / deploy / ops (solo si aplica).

## Enlace
<URL del ticket (Jira / Linear / etc.)>

## Issue relacionado
Closes #<numero-de-issue>

## Checklist
- [ ] Rama nombrada `feature/*`/`chore/*` (→ develop) o `hotfix/*` (→ main)
- [ ] Issue creado en este repo y linkeado arriba (se refleja solo en el Project del org)
- [ ] Control de calidad pasa localmente
- [ ] Cambio de entidad → migración incluida en el mismo PR
- [ ] Nueva config → variables en TODOS los entornos (local + prod)
- [ ] Sin secretos, tokens, ni URLs de producción en el código
