# Plan Maestro: Modulos Compartidos Mobile (iOS/Android)

## Contexto

Este documento define la estrategia para crear dos proyectos de modulos compartidos:
- edugo-ios-shared (Swift Package Manager)
- edugo-android-shared (Gradle Multi-Module)

Basado en la arquitectura probada de edugo-shared (Go).

---

## 1. Clasificacion de Modulos

### 1.1 Modulos CROSS (Genericos - Reutilizables)

| Modulo | Proposito | Prioridad |
|--------|-----------|-----------|
| common | Tipos base: UUID wrapper, Result types, AppError | ALTA |
| logger | Interfaz abstracta de logging | ALTA |
| config | Carga de configuracion | MEDIA |
| network | Cliente HTTP con interceptors, retry, auth | ALTA |
| storage | Almacenamiento local seguro y preferencias | ALTA |
| auth | JWT decode, secure token storage | ALTA |

### 1.2 Modulos ESPECIFICOS (Dominio EduGo)

| Modulo | Proposito | Prioridad |
|--------|-----------|-----------|
| edugo-models | Modelos de dominio: User, School, Material | ALTA |
| edugo-events | Definiciones de eventos CloudEvents | MEDIA |
| edugo-api | Cliente API especifico de EduGo | ALTA |
| edugo-roles | Enums de roles y permisos | ALTA |

---

## 2. Interfaces Compartidas

### Logger
- debug/info/warn/error(message, fields?)
- with(fields) -> Logger

### HTTPClient
- get/post/put/delete<T>(url, body?, headers?) -> Result<T, NetworkError>
- addInterceptor, setAuthToken

### SecureStorage
- save/get/delete/clear/exists

### TokenManager
- saveTokens, getAccessToken, getRefreshToken, clearTokens
- isAccessTokenExpired, decodeAccessToken, getUserId, getRole

### AppError
- code, message, details, statusCode, underlyingError

---

## 3. Plan por Fases (9 documentos)

1. Setup proyectos
2. Modulo Common
3. Modulo Logger
4. Modulo Storage
5. Modulo Network
6. Modulo Auth
7. Modulos Dominio EduGo
8. Modulo API Client
9. Documentacion y Publicacion

---

## 4. Preguntas Pendientes

1. Version minima iOS? (recomiendo 15+)
2. Version minima Android? (recomiendo API 24+)
3. HTTP client Kotlin? (Ktor vs Retrofit)
4. Incluir analytics?
5. DI en shared o en apps?

---
Documento: 2026-01-02
