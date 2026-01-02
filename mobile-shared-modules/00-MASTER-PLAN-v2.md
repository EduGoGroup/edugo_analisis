# Master Plan: Mobile Shared Modules v2

## Resumen Ejecutivo

Plan para crear dos repositorios de módulos compartidos equivalentes a `edugo-shared` de Go:

| Proyecto | Plataforma | Stack |
|----------|------------|-------|
| **edugo-swift-shared** | iOS 26+, iPadOS 26+, macOS 26+ | Swift 6.0, SPM |
| **edugo-kmp-shared** | Android, Desktop, Web | Kotlin 2.1.0, KMP |

---

## Decisiones Confirmadas

### Plataformas
- **Apple**: iOS/iPadOS/macOS versión 26+ (WWDC 2025)
- **KMP es CENTRAL**: Android, JVM Desktop, Kotlin/JS (Web)

### Tecnologías
- **HTTP Client**: Ktor 3.0.2 (KMP), URLSession nativo (Swift)
- **Logging**: Kermit (KMP), os.Logger (Swift)
- **Storage**: multiplatform-settings (KMP), Keychain nativo (Swift)
- **Telemetría**: Módulo independiente en ambos

### Arquitectura
- **DI**: NO en shared (responsabilidad del consumidor)
- **Sin dependencias externas en Swift** - todo nativo
- **Dependencias mínimas en KMP** - solo ecosistema Kotlin

---

## Estructura de Módulos

### Cross Modules (Reutilizables)

| Módulo | Swift | KMP | Propósito |
|--------|-------|-----|-----------|
| Common | EduGoCommon | modules/common | Errores, extensiones, tipos base |
| Logger | EduGoLogger | modules/logger | Logging multiplataforma |
| Network | EduGoNetwork | modules/network | HTTP client |
| Storage | EduGoStorage | modules/storage | Almacenamiento seguro |
| Auth | EduGoAuth | modules/auth | Gestión de tokens, JWT |
| Analytics | EduGoAnalytics | modules/analytics | Telemetría |

### Specific Modules (Dominio EduGo)

| Módulo | Swift | KMP | Propósito |
|--------|-------|-----|-----------|
| Roles | EduGoRoles | modules/roles | SystemRole, permisos |
| Models | EduGoModels | modules/models | User, Material, etc. |
| API | EduGoAPI | modules/api | Cliente API EduGo |

---

## Fases de Implementación

### Fase 1: Setup Inicial
- [ ] Crear repositorio `edugo-swift-shared`
- [ ] Crear repositorio `edugo-kmp-shared`
- [ ] Configurar CI/CD básico

### Fase 2: Módulos Base
- [ ] common (errores, tipos)
- [ ] logger
- [ ] storage

### Fase 3: Módulos de Red
- [ ] network
- [ ] auth (tokens, JWT)

### Fase 4: Dominio EduGo
- [ ] roles
- [ ] models
- [ ] api

### Fase 5: Telemetría
- [ ] analytics

---

## Documentos de Referencia

| Documento | Descripción |
|-----------|-------------|
| [01-SWIFT-SETUP-PLAN.md](./01-SWIFT-SETUP-PLAN.md) | Plan completo para Swift |
| [01-KMP-SETUP-PLAN.md](./01-KMP-SETUP-PLAN.md) | Plan completo para KMP |

---

## Consistencia con edugo-shared (Go)

Los módulos deben mantener consistencia semántica con Go:

```
edugo-shared (Go)           → edugo-swift-shared      → edugo-kmp-shared
─────────────────────────────────────────────────────────────────────────
common/errors               → EduGoCommon/Errors      → modules/common/errors
common/types/enum/role      → EduGoRoles/SystemRole   → modules/roles
auth/                       → EduGoAuth               → modules/auth
logger/                     → EduGoLogger             → modules/logger
```

---

**Estado**: Plan aprobado, listo para implementación
**Última actualización**: 2026-01-02
