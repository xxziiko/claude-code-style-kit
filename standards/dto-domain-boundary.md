# DTO / Domain Boundary

## Purpose

Keep external response shapes from spreading through application code.

## Rules

- Treat API responses, generated types, database rows, and third-party SDK objects as boundary types.
- Convert boundary types into application/domain types at the adapter or mapper boundary.
- Keep null/default policy in one place.
- Do not duplicate response-shape cleanup in components.
- Do not import generated or external DTO types into UI components unless the project intentionally uses DTO-first UI.

## Mapper Guidance

- Mappers should be deterministic and small.
- Mapper tests are useful when they protect policy: null handling, enum translation, unit conversion, date normalization, or backward compatibility.
- Avoid testing field-copy mappers that contain no policy.

## Review Checklist

- Where does boundary data enter the app?
- Where is it normalized?
- Are defaults/nulls handled once?
- Can UI code use domain language without knowing transport details?
