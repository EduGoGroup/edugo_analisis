# [RFC-XXX]: [Nombre del RFC]

## Información General

- **RFC Original**: `RFCs/XX-categoria/RFC-XXX-nombre.md`
- **Categoría**: [Arquitectura / Autenticación / Gestión Escolar / Materiales / Evaluaciones / Resúmenes IA / Estadísticas]
- **Prioridad**: [Alta / Media / Baja]
- **Complejidad UI**: [Alta / Media / Baja]

## Descripción

[Breve descripción del RFC y su propósito funcional]

---

## Design Tokens y Recursos

**IMPORTANTE**: Este análisis debe usar los design tokens existentes del sistema de diseño centralizado.

### Ubicación de Tokens
- **Android MD3**: `/GuideDesign/Design/KMP/Android_MD3/Tokens/`
- **Desktop MD3**: `/GuideDesign/Design/KMP/Desktop_MD3/Tokens/`
- **Web WASM**: `/GuideDesign/Design/KMP/Web_WASM_MD3/Tokens/`
- **Common**: `/GuideDesign/Design/Common/`

### Patrones Reutilizables
Consultar si existe patrón similar:
- Login: `[Platform]/Patterns/login.md`
- Forms: `[Platform]/Patterns/form_pattern.md`
- Lists: `[Platform]/Patterns/list_pattern.md`
- Navigation: `[Platform]/Patterns/navigation_pattern.md`
- Dashboard: `[Platform]/Patterns/dashboard_pattern.md`
- Modals: `[Platform]/Patterns/modal_pattern.md`
- Search: `[Platform]/Patterns/search_pattern.md`

### Tokens Clave a Usar
```kotlin
// Colores - NO usar valores hardcoded
MaterialTheme.colorScheme.primary        // #6750A4
MaterialTheme.colorScheme.surface        // #FEF7FF
MaterialTheme.colorScheme.error          // #BA1A1A

// Spacing - Sistema base 8dp
Spacing.spacingMd   // 16.dp (base)
Spacing.spacingSm   // 8.dp
Spacing.spacingLg   // 24.dp

// Typography
MaterialTheme.typography.headlineSmall   // 24sp
MaterialTheme.typography.bodyLarge       // 16sp
MaterialTheme.typography.labelMedium     // 12sp
```

**Referencia completa**: `analisis-mobile/DESIGN-TOKENS-INTEGRATION.md`

---

## Análisis por Plataforma

### Componentes Comunes (Common)

Componentes compartidos entre todas las plataformas KMP:

#### Estados de UI
```kotlin
sealed class [Feature]UiState {
    object Idle : [Feature]UiState()
    object Loading : [Feature]UiState()
    data class Success(val data: [DataType]) : [Feature]UiState()
    data class Error(val message: String) : [Feature]UiState()
}
```

#### Modelo de Datos
```kotlin
data class [Entity](
    val id: String,
    // campos según RFC...
)
```

#### ViewModel Común
```kotlin
class [Feature]ViewModel : ViewModel() {
    private val _uiState = MutableStateFlow<[Feature]UiState>(Idle)
    val uiState: StateFlow<[Feature]UiState> = _uiState.asStateFlow()

    // Lógica común...
}
```

---

### Android (Material Design 3)

#### Pantallas Principales
1. **[NombrePantalla]Screen**
   - Descripción de la pantalla
   - Componentes usados

#### Componentes Material 3
```kotlin
@Composable
fun [Component]Screen(
    viewModel: [Feature]ViewModel = viewModel(),
    onNavigate: (route: String) -> Unit
) {
    val uiState by viewModel.uiState.collectAsState()

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("[Título]") }
            )
        }
    ) { paddingValues ->
        // Contenido...
    }
}
```

#### Navegación
- **Ruta**: `[feature]/[screen]`
- **Argumentos**: `[arg1], [arg2]`
- **Destinos**: Desde/Hacia qué pantallas navega

#### Componentes Específicos
- **TopAppBar**: [Configuración]
- **FAB**: [Si aplica]
- **BottomSheet**: [Si aplica]
- **Dialogs**: [Si aplica]

#### Gestión de Estados
- Loading: Shimmer effects, CircularProgressIndicator
- Error: Snackbar, AlertDialog
- Success: Content rendering
- Empty: EmptyState illustration

---

### iOS (Material Design en iOS)

#### Consideraciones Especiales
- Mantener Material Design 3 en plataforma iOS
- Adaptar gestos iOS nativos donde sea crítico
- Safe Areas (notch, Dynamic Island)
- Permisos iOS

#### Adaptaciones Necesarias
```kotlin
@Composable
fun [Component]ScreenIOS(
    // Mismo componente pero con ajustes iOS
) {
    // Ajustes específicos:
    // - Safe area insets
    // - Gestos de swipe back
    // - Permisos (Camera, Photos, etc.)
}
```

#### Navegación iOS
- Gesture back navigation
- Modal presentations
- Deep linking

---

### Desktop (Windows/macOS/Linux)

#### Layout Adaptativo
```kotlin
@Composable
fun [Component]ScreenDesktop() {
    Row {
        // NavigationRail
        NavigationRail {
            // Items...
        }

        // Content area
        Box(modifier = Modifier.weight(1f)) {
            // Contenido principal con más espacio horizontal
        }
    }
}
```

#### Consideraciones Desktop
- **Ventanas**: Tamaño mínimo, redimensionamiento
- **Keyboard**: Shortcuts (Ctrl/Cmd + tecla)
- **Mouse**: Hover states, right-click menus
- **Menu Bar**: Acciones de aplicación (si aplica)

#### Componentes Específicos Desktop
- NavigationRail en lugar de BottomNavigation
- Tooltips en hover
- Context menus (right-click)
- Múltiples ventanas (si aplica)

---

### Web (Kotlin/Wasm)

#### Consideraciones Web
- **Responsive**: Mobile-first approach
- **Performance**: Lazy loading, code splitting
- **Accesibilidad**: ARIA labels, keyboard navigation
- **SEO**: Meta tags (si aplica)

#### Limitaciones Web
- [Listar funcionalidades no disponibles en web]
- [Alternativas propuestas]

#### Layout Web
```kotlin
@Composable
fun [Component]ScreenWeb() {
    // Responsive breakpoints
    val windowWidth = [getWindowWidth]

    if (windowWidth < 600.dp) {
        // Mobile layout
    } else if (windowWidth < 1024.dp) {
        // Tablet layout
    } else {
        // Desktop layout
    }
}
```

---

## Componentes Reutilizables

### Listado de Componentes Custom
1. **[ComponentName]**
   ```kotlin
   @Composable
   fun [ComponentName](
       // parámetros...
   ) {
       // implementación...
   }
   ```

---

## Navegación y Flujos

### Diagrama de Flujo
```
[PantallaA] --> [PantallaB] --> [PantallaC]
     |              |
     v              v
[Dialog]      [BottomSheet]
```

### Deep Links
- `edugo://[feature]/[action]?[params]`

---

## Estados y Manejo de Errores

### Estados Posibles
| Estado | UI | Acción Usuario |
|--------|-----|----------------|
| Idle | - | - |
| Loading | Shimmer/Progress | Bloqueado |
| Success | Contenido | Interacción normal |
| Error | Mensaje error | Retry |
| Empty | Ilustración vacío | Acción para crear |

### Errores Comunes
1. **Network Error**: "Sin conexión a internet"
   - Acción: Reintentar
2. **Validation Error**: "Datos inválidos"
   - Acción: Corregir campos
3. **Server Error**: "Error del servidor"
   - Acción: Contactar soporte

---

## Offline Support

### Datos en Caché
- [Qué datos se guardan localmente]
- [Estrategia de sincronización]

### UI Offline
- Indicador de modo offline
- Acciones disponibles sin conexión
- Sincronización al reconectar

---

## Accesibilidad

### Content Descriptions
```kotlin
Icon(
    imageVector = Icons.Default.[Icon],
    contentDescription = "[Descripción para screen readers]"
)
```

### Navegación por Teclado
- Tab order lógico
- Shortcuts documentados

### Contraste y Tamaños
- Respetar Dynamic Type / Font Scaling
- Contraste mínimo WCAG AA

---

## Performance

### Optimizaciones
- LazyColumn/LazyRow para listas largas
- remember/derivedStateOf para cálculos
- Evitar recomposiciones innecesarias

### Métricas Objetivo
- Time to Interactive: < 2s
- Smooth scrolling: 60fps
- Bundle size: [estimado]

---

## Testing

### Unit Tests
```kotlin
@Test
fun `test [funcionalidad]`() {
    // Given
    // When
    // Then
}
```

### UI Tests
```kotlin
@Test
fun `test [pantalla] displays correctly`() {
    composeTestRule.setContent {
        [Component]Screen()
    }

    composeTestRule.onNodeWithText("[texto]").assertIsDisplayed()
}
```

---

## Mockups Conceptuales

### Android
```
┌─────────────────────────┐
│  ☰  [Título]        ⋮   │ <- TopAppBar
├─────────────────────────┤
│                         │
│   [Contenido principal] │
│                         │
│                         │
└─────────────────────────┘
│ 🏠  📚  📊  👤          │ <- BottomNavigation
└─────────────────────────┘
```

### Desktop
```
┌──┬──────────────────────────────┐
│🏠│  [Título]                    │
│📚│                              │
│📊│  [Contenido con más espacio  │
│👤│   horizontal aprovechado]    │
│  │                              │
└──┴──────────────────────────────┘
   NavigationRail     Content
```

---

## Decisiones de Diseño

### Trade-offs
1. **[Decisión]**: [Opción elegida]
   - Pros: [Ventajas]
   - Cons: [Desventajas]
   - Alternativa: [Opción descartada]

---

## Dependencias

### Bibliotecas KMP
- `org.jetbrains.compose.material3:material3:[version]`
- `androidx.navigation:navigation-compose:[version]`
- [Otras dependencias específicas]

---

## Notas de Implementación

### Consideraciones Técnicas
- [Notas importantes para developers]
- [Patrones recomendados]
- [Anti-patrones a evitar]

### Referencias
- [Links a documentación relevante]
- [Ejemplos de código similares]

---

## Checklist de Completitud

- [ ] Estados de UI definidos
- [ ] Componentes Material 3 especificados
- [ ] Navegación documentada
- [ ] Manejo de errores definido
- [ ] Offline support considerado
- [ ] Accesibilidad documentada
- [ ] Tests planificados
- [ ] Mockups conceptuales creados
- [ ] Adaptaciones por plataforma documentadas (Android/iOS/Desktop/Web)
