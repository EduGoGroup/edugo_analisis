# Integración con Design Tokens Existentes

## Ubicación de Design Tokens

Los estándares visuales del proyecto están centralizados en:
```
/Users/jhoanmedina/source/Documentation/GuideDesign/Design/
```

Este análisis UI mobile **DEBE usar** los tokens definidos en esa documentación para mantener consistencia visual en todas las plataformas.

---

## Estructura de Design Tokens por Plataforma

### Kotlin Multiplatform (KMP)

#### Android MD3
- **Ubicación**: `/GuideDesign/Design/KMP/Android_MD3/`
- **Tokens**: Tokens/
- **Componentes**: Components/
- **Patrones**: Patterns/
- **Features**: Features/
- **Guidelines**: Guidelines/

#### Desktop MD3
- **Ubicación**: `/GuideDesign/Design/KMP/Desktop_MD3/`
- Misma estructura que Android MD3

#### Web WASM MD3
- **Ubicación**: `/GuideDesign/Design/KMP/Web_WASM_MD3/`
- Tokens adaptados para Compose for Web

#### Web JS MD3
- **Ubicación**: `/GuideDesign/Design/KMP/Web_JS_MD3/`
- CSS Variables para JavaScript/Kotlin Web

#### Web React MD3
- **Ubicación**: `/GuideDesign/Design/KMP/Web_React_MD3/`
- Tokens para React + TypeScript (referencia si necesario)

---

### Apple Ecosystem

#### iOS 18 (Estable)
- **Ubicación**: `/GuideDesign/Design/Apple/iOS18/`
- **Tokens**: SwiftUI system colors + custom tokens
- **Features**: Face ID, biometric auth, etc.

#### iOS 26 (Liquid Glass)
- **Ubicación**: `/GuideDesign/Design/Apple/iOS26/`
- **Estilo**: Glassmorphism effects
- **Material effects**: Ultra thin, thin, regular, thick

#### macOS 15 (Estable)
- **Ubicación**: `/GuideDesign/Design/Apple/macOS15/`
- **Características**: Sidebar, menu bar, toolbar

#### macOS 26 (Liquid Glass)
- **Ubicación**: `/GuideDesign/Design/Apple/macOS26/`
- **Características**: Window glass integration, multi-display support

---

## Tokens Comunes (Common)

**Ubicación**: `/GuideDesign/Design/Common/`

Recursos transversales a todas las plataformas:
- **TokensReference.md**: Referencia completa de tokens
- **SpacingVariants.md**: Sistema de espaciado
- **ResponsiveBreakpoints.md**: Breakpoints responsivos
- **ValidationRules.md**: Reglas de validación UI
- **ErrorHandling.md**: Manejo de errores
- **AlertSystem.md**: Sistema de alertas
- **TestingChecklist.md**: Checklist de testing
- **performance_guidelines.md**: Guidelines de performance
- **responsive_guidelines.md**: Guidelines responsive

---

## Inventario de Tokens Clave

### Colores (Material Design 3)

#### Colores Principales
```kotlin
// KMP Android/Desktop/Web
primary: "#6750A4" / "#D0BCFF" (dark)
onPrimary: "#FFFFFF" / "#381E72" (dark)
primaryContainer: "#EADDFF" / "#4F378B" (dark)
onPrimaryContainer: "#21005D" / "#EADDFF" (dark)
```

```swift
// Apple
Color.accentColor // System blue adaptable
Color.primary // Dynamic según light/dark
```

#### Superficies
```kotlin
// KMP
surface: "#FEF7FF" / "#1C1B1F" (dark)
onSurface: "#1C1B1F" / "#E6E1E5" (dark)
surfaceVariant: "#E7E0EC" / "#49454F" (dark)
```

```swift
// Apple
Color(.systemBackground) // Adaptable
Color(.secondarySystemBackground)
```

#### Error States
```kotlin
// KMP
error: "#BA1A1A" / "#F2B8B5" (dark)
errorContainer: "#FFDAD6" / "#8C1D18" (dark)
```

```swift
// Apple
Color(.systemRed)
```

---

### Espaciado (Spacing System)

#### Escala Base (8px/pt)
```css
/* Web */
--spacing-1: 4px   /* xs */
--spacing-2: 8px   /* sm */
--spacing-3: 12px  /* sm-md */
--spacing-4: 16px  /* md ⭐ base */
--spacing-5: 20px  /* md-lg */
--spacing-6: 24px  /* lg */
--spacing-8: 32px  /* xl */
--spacing-10: 40px /* xxl */
--spacing-12: 48px /* huge */
--spacing-16: 64px /* extra-huge */
--spacing-20: 80px /* maximum */
```

```kotlin
// KMP Compose
Spacing.spacingXs  // 4.dp
Spacing.spacingSm  // 8.dp
Spacing.spacingMd  // 16.dp ⭐
Spacing.spacingLg  // 24.dp
Spacing.spacingXl  // 32.dp
Spacing.spacingXxl // 48.dp
```

```swift
// SwiftUI (usar mismos valores en pt)
let spacingSm: CGFloat = 8
let spacingMd: CGFloat = 16
let spacingLg: CGFloat = 24
```

---

### Tipografía

#### Material Design 3 Type Scale
```css
/* Display */
--display-large: 57px / 64px line-height
--display-medium: 45px / 52px
--display-small: 36px / 44px

/* Headline */
--headline-large: 32px / 40px
--headline-medium: 28px / 36px
--headline-small: 24px / 32px

/* Title */
--title-large: 22px / 28px
--title-medium: 16px / 24px ⭐
--title-small: 14px / 20px

/* Body */
--body-large: 16px / 24px ⭐
--body-medium: 14px / 20px ⭐
--body-small: 12px / 16px ⭐

/* Label */
--label-large: 14px / 20px
--label-medium: 12px / 16px
--label-small: 11px / 16px
```

```kotlin
// KMP Compose
MaterialTheme.typography.displayLarge
MaterialTheme.typography.bodyLarge
MaterialTheme.typography.labelMedium
```

```swift
// SwiftUI (usar SF Text Styles)
.font(.largeTitle)  // ~34pt
.font(.title)       // ~28pt
.font(.title2)      // ~22pt
.font(.headline)    // ~17pt bold
.font(.body)        // ~17pt ⭐
.font(.callout)     // ~16pt
.font(.subheadline) // ~15pt
.font(.footnote)    // ~13pt
.font(.caption)     // ~12pt
.font(.caption2)    // ~11pt
```

---

### Elevation / Shadows

```css
/* Material Design 3 */
--elevation-0: none
--elevation-1: 0px 1px 3px rgba(0,0,0,0.12)
--elevation-2: 0px 4px 8px rgba(0,0,0,0.15)
--elevation-3: 0px 6px 10px rgba(0,0,0,0.15)
--elevation-4: 0px 8px 12px rgba(0,0,0,0.15)
```

```swift
// SwiftUI
.shadow(color: .black.opacity(0.1), radius: 10, x: 0, y: 4)
```

---

### Responsive Breakpoints

```css
--breakpoint-mobile: 599px    /* < 600px */
--breakpoint-tablet: 899px    /* 600-899px */
--breakpoint-desktop: 900px   /* >= 900px */
```

```kotlin
// Compose
val WindowSizeClass.COMPACT    // < 600dp
val WindowSizeClass.MEDIUM     // 600-839dp
val WindowSizeClass.EXPANDED   // >= 840dp
```

```swift
// SwiftUI Size Classes
@Environment(\.horizontalSizeClass) var sizeClass
// .compact (iPhone portrait)
// .regular (iPhone landscape, iPad)
```

---

### Border Radius

```css
/* Material Design 3 */
--radius-none: 0px
--radius-xs: 4px
--radius-sm: 8px
--radius-md: 12px
--radius-lg: 16px
--radius-xl: 20px
--radius-xxl: 28px
--radius-full: 999px
```

```swift
// SwiftUI
.clipShape(RoundedRectangle(cornerRadius: 12))
```

---

## Patrones de Diseño Disponibles

### Autenticación
- **Login**: `/Apple/[platform]/Patterns/login.md`, `/KMP/[platform]/Patterns/login.md`
- **Biometric Auth**: `/Apple/[platform]/Features/biometric_auth.md`

### Navegación
- **Navigation**: `Patterns/navigation_pattern.md` en cada plataforma
- **Tab Navigation**: Bottom Nav (Android), Tab Bar (iOS), Tab Row (Desktop/Web)
- **Drawer/Sidebar**: Nav Drawer (Android/Web), Sidebar (macOS)

### Formularios
- **Form Pattern**: `Patterns/form_pattern.md` en cada plataforma
- **Validation**: `/Common/ValidationRules.md`

### Visualización de Datos
- **List Pattern**: `Patterns/list_pattern.md`
- **Detail View**: `Patterns/detail_view.md`
- **Dashboard**: `Patterns/dashboard_pattern.md`

### Modales e Interacciones
- **Modal Pattern**: `Patterns/modal_pattern.md`
- **Alert System**: `/Common/AlertSystem.md`
- **Empty States**: `Patterns/empty_states.md`

### Búsqueda
- **Search Pattern**: `Patterns/search_pattern.md`

### Configuración
- **Settings Pattern**: `Patterns/settings_pattern.md`

### Primera Experiencia
- **Onboarding**: `Patterns/onboarding_pattern.md`

---

## Features Específicas Apple

### Liquid Glass (iOS 26, macOS 26)
- **Documentación**: `/Apple/[platform]26/Features/liquid_glass.md`
- **Características**:
  - `.background(.ultraThinMaterial)`
  - Glassmorphism effects
  - Blur y translucency
  - Shadow layering

### Dark Mode
- **Documentación**: `/Apple/[platform]/Features/dark_mode.md`
- **Implementación**: Color adaptables automáticos

### Internacionalización
- **Documentación**: `/Apple/[platform]/Features/i18n.md`

### Gestos
- **Documentación**: `/Apple/[platform]/Features/gestures.md`

---

## Directrices de Uso en Análisis

### Al Analizar cada RFC:

1. **Consultar Tokens Primero**
   - Revisar `/GuideDesign/Design/[Platform]/Tokens/`
   - Usar tokens existentes, NO valores hardcoded

2. **Aplicar Patrones Existentes**
   - Revisar si existe patrón similar en `Patterns/`
   - Adaptar patrón existente antes de crear uno nuevo

3. **Usar Componentes Documentados**
   - Revisar `Components/` de la plataforma
   - Reutilizar componentes existentes

4. **Validar con Guidelines**
   - Revisar `Guidelines/` de la plataforma
   - Asegurar compliance con HIG (Apple) o Material Design (KMP)

5. **Considerar Features Transversales**
   - Revisar `/Common/` para features compartidas
   - Aplicar `ValidationRules.md`, `ErrorHandling.md`, etc.

---

## Ejemplo de Referencia en Análisis

### Incorrecto ❌
```kotlin
// NO hacer esto - valor hardcoded
Text(
    "Login",
    fontSize = 24.sp,
    color = Color(0xFF6750A4)
)
```

### Correcto ✅
```kotlin
// Usar tokens del design system
Text(
    "Login",
    style = MaterialTheme.typography.headlineSmall, // 24sp
    color = MaterialTheme.colorScheme.primary        // #6750A4
)
```

### Referencia en Documentación
Al documentar un componente, agregar:
```markdown
## Design Tokens Usados

- **Color**: `MaterialTheme.colorScheme.primary`
  (ver `/GuideDesign/Design/KMP/Android_MD3/Tokens/colors.md`)
- **Typography**: `MaterialTheme.typography.headlineSmall`
  (ver `/GuideDesign/Design/KMP/Android_MD3/Tokens/typography.md`)
- **Spacing**: `Spacing.spacingMd` (16dp)
  (ver `/GuideDesign/Design/Common/SpacingVariants.md`)
```

---

## Validación de Consistencia

### Checklist al Documentar UI:
- [ ] Colores usan tokens del design system
- [ ] Tipografía usa type scales definidas
- [ ] Spacing usa sistema de espaciado (múltiplos de 8)
- [ ] Border radius usa valores estándar
- [ ] Elevations usan sistema de elevation
- [ ] Responsive breakpoints usa valores definidos
- [ ] Patrones siguen documentación existente
- [ ] Componentes referencian biblioteca existente

---

## Recursos Principales a Consultar

1. **Índice Maestro**: `/GuideDesign/Design/Readme.md`
2. **Arquitectura**: `/GuideDesign/Design/ARCHITECTURE.md`
3. **Features Cross-Platform**: `/GuideDesign/Design/CROSS_FEATURES.md`
4. **Inventario de Tokens**: `/GuideDesign/Design/TOKEN_INVENTORY_MAPPING.md`
5. **Referencia de Tokens**: `/GuideDesign/Design/Common/TokensReference.md`

---

## Notas Importantes

### Reutilización de Tokens
- **70% de valores** ya tienen tokens definidos
- **30% restante** puede requerir crear nuevos tokens
- **Siempre consultar** antes de crear un nuevo valor hardcoded

### Consistencia Visual
- Todas las plataformas deben sentirse cohesivas
- KMP mantiene Material Design 3 en todas sus variantes
- Apple mantiene HIG pero con estilo Liquid Glass unificado

### Actualización de Tokens
Si durante el análisis se identifica necesidad de nuevos tokens:
1. Documentar en el análisis del RFC
2. Proponer el token en la sección "Nuevos Tokens Necesarios"
3. Sugerir valor y naming según convención existente
4. Reportar al equipo de Design System para centralización

---

**Estado**: ✅ Documentación integrada
**Próximo Paso**: Usar estos tokens al analizar RFCs en cada fase
