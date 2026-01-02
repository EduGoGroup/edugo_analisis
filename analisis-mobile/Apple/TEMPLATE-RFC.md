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
- **iOS 18**: `/GuideDesign/Design/Apple/iOS18/Tokens/`
- **iOS 26 (Liquid Glass)**: `/GuideDesign/Design/Apple/iOS26/Tokens/`
- **macOS 15**: `/GuideDesign/Design/Apple/macOS15/Tokens/`
- **macOS 26 (Liquid Glass)**: `/GuideDesign/Design/Apple/macOS26/Tokens/`
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

### Features Específicos Apple
- **Liquid Glass**: `[Platform]26/Features/liquid_glass.md`
- **Dark Mode**: `[Platform]/Features/dark_mode.md`
- **Biometric Auth**: `[Platform]/Features/biometric_auth.md`
- **Gestures**: `[Platform]/Features/gestures.md`
- **i18n**: `[Platform]/Features/i18n.md`

### Tokens Clave a Usar
```swift
// Colores - NO usar valores hardcoded
Color.accentColor                    // System blue
Color(.systemBackground)             // Adaptable
Color(.label)                        // Texto principal

// Spacing - Sistema base 8pt
let spacingMd: CGFloat = 16          // Base
let spacingSm: CGFloat = 8
let spacingLg: CGFloat = 24

// Typography - SF Text Styles
.font(.body)                         // ~17pt
.font(.headline)                     // ~17pt bold
.font(.title2)                       // ~22pt

// Liquid Glass Effects (iOS/macOS 26)
.background(.ultraThinMaterial)
.shadow(color: .black.opacity(0.1), radius: 10)
```

**Referencia completa**: `analisis-mobile/DESIGN-TOKENS-INTEGRATION.md`

---

## Análisis por Plataforma

### Shared (Código Compartido)

Modelos y lógica compartida entre iOS, iPadOS y macOS:

#### Estados de UI
```swift
enum [Feature]State {
    case idle
    case loading
    case success([DataType])
    case error(Error)
}
```

#### Modelo de Datos
```swift
struct [Entity]: Identifiable, Codable {
    let id: String
    // campos según RFC...
}
```

#### ViewModel (ObservableObject)
```swift
@MainActor
class [Feature]ViewModel: ObservableObject {
    @Published var state: [Feature]State = .idle

    func [action]() async {
        // Lógica común...
    }
}
```

---

### iOS (iPhone)

#### Pantallas Principales
1. **[NombrePantalla]View**
   - Descripción de la pantalla
   - Componentes usados

#### Implementación SwiftUI
```swift
struct [Component]View: View {
    @StateObject private var viewModel = [Feature]ViewModel()

    var body: some View {
        NavigationStack {
            ZStack {
                // Background con Liquid Glass
                Color.clear
                    .background(.ultraThinMaterial)

                // Contenido
                content
            }
            .navigationTitle("[Título]")
            .toolbar {
                // Toolbar items
            }
        }
    }

    @ViewBuilder
    private var content: some View {
        switch viewModel.state {
        case .idle:
            EmptyView()
        case .loading:
            ProgressView()
        case .success(let data):
            successView(data)
        case .error(let error):
            errorView(error)
        }
    }
}
```

#### Liquid Glass Effects
```swift
// Card con efecto glassmorphism
RoundedRectangle(cornerRadius: 16)
    .fill(.white.opacity(0.1))
    .overlay(
        RoundedRectangle(cornerRadius: 16)
            .stroke(.white.opacity(0.2), lineWidth: 1)
    )
    .background(.ultraThinMaterial)
    .shadow(color: .black.opacity(0.1), radius: 10, x: 0, y: 4)
```

#### Navegación
- **NavigationStack**: Para flujos jerárquicos
- **TabView**: Para navegación principal
- **Sheet**: Para modales
- **FullScreenCover**: Para pantallas completas

#### Safe Areas
```swift
.safeAreaInset(edge: .top) {
    // Contenido sobre safe area
}
.ignoresSafeArea(edges: .bottom) // Si necesario
```

#### Gestos iOS
```swift
.swipeActions(edge: .trailing) {
    Button(role: .destructive) {
        // Acción delete
    } label: {
        Label("Eliminar", systemImage: "trash")
    }
}
```

#### Integraciones Nativas
- **Face ID / Touch ID**: Autenticación biométrica
- **Share Sheet**: Compartir contenido
- **Notifications**: Push notifications
- **Widgets**: WidgetKit para Home Screen

---

### iPadOS

#### Consideraciones iPad
- **Multitasking**: Split View, Slide Over
- **Apple Pencil**: Gestos y anotaciones (si aplica)
- **Keyboard**: Shortcuts y comandos
- **Orientación**: Landscape optimizado

#### NavigationSplitView
```swift
NavigationSplitView {
    // Sidebar
    List(selection: $selectedItem) {
        // Items...
    }
    .navigationTitle("Secciones")
} detail: {
    // Detail view
    if let item = selectedItem {
        DetailView(item: item)
    } else {
        ContentUnavailableView(
            "Selecciona un ítem",
            systemImage: "sidebar.left"
        )
    }
}
```

#### Layout Adaptativo
```swift
@Environment(\.horizontalSizeClass) var horizontalSizeClass

var body: some View {
    if horizontalSizeClass == .regular {
        // Layout para iPad (landscape)
        iPadLayout
    } else {
        // Layout para iPad (portrait) o iPhone
        compactLayout
    }
}
```

#### Drag & Drop (si aplica)
```swift
.onDrag {
    // Proveer item para drag
}
.onDrop(of: [.item]) { providers in
    // Manejar drop
}
```

---

### macOS

#### Consideraciones macOS
- **Window Management**: Múltiples ventanas
- **Menu Bar**: Menús de aplicación completos
- **Toolbar**: Botones de acción rápida
- **Keyboard**: Shortcuts completos (Cmd+)

#### Window Structure
```swift
@main
struct [App]App: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .commands {
            // Custom menu commands
            CommandMenu("Acciones") {
                Button("Nueva...") {
                    // Acción
                }
                .keyboardShortcut("n", modifiers: .command)
            }
        }

        Settings {
            SettingsView()
        }
    }
}
```

#### Toolbar macOS
```swift
.toolbar {
    ToolbarItemGroup(placement: .automatic) {
        Button {
            // Acción
        } label: {
            Label("Nuevo", systemImage: "plus")
        }
    }
}
```

#### NavigationSplitView (3 columnas)
```swift
NavigationSplitView {
    // Sidebar
    SidebarView()
} content: {
    // Content list
    ContentListView()
} detail: {
    // Detail view
    DetailView()
}
.navigationSplitViewStyle(.balanced)
```

#### Keyboard Shortcuts
```swift
.keyboardShortcut("s", modifiers: .command)
.keyboardShortcut(.delete, modifiers: .command)
```

---

## Componentes Reutilizables

### Listado de Componentes Custom
1. **[ComponentName]**
   ```swift
   struct [ComponentName]: View {
       // parámetros...

       var body: some View {
           // implementación...
       }
   }
   ```

---

## Navegación y Flujos

### Diagrama de Flujo
```
[ViewA] --> [ViewB] --> [ViewC]
   |            |
   v            v
[Sheet]    [FullScreen]
```

### Deep Links
```swift
.onOpenURL { url in
    // Manejar: edugo://[feature]/[action]?[params]
}
```

### NavigationPath
```swift
@State private var path = NavigationPath()

// Navegar
path.append([Destination].detail(id: item.id))

// Volver
path.removeLast()
```

---

## Estados y Manejo de Errores

### Estados Posibles
| Estado | UI | Acción Usuario |
|--------|-----|----------------|
| Idle | - | - |
| Loading | ProgressView | Bloqueado |
| Success | Contenido | Interacción normal |
| Error | Alert/Banner | Retry |
| Empty | ContentUnavailableView | Acción para crear |

### ContentUnavailableView (iOS 17+)
```swift
ContentUnavailableView(
    "Sin elementos",
    systemImage: "tray",
    description: Text("No hay elementos para mostrar")
)
```

### Errores Comunes
```swift
enum [Feature]Error: LocalizedError {
    case networkError
    case validationError(String)
    case serverError

    var errorDescription: String? {
        switch self {
        case .networkError:
            return "Sin conexión a internet"
        case .validationError(let message):
            return message
        case .serverError:
            return "Error del servidor"
        }
    }
}
```

---

## Offline Support

### Datos en Caché
- [Qué datos se guardan localmente con UserDefaults/CoreData/SwiftData]
- [Estrategia de sincronización]

### UI Offline
```swift
@Environment(\.networkStatus) var networkStatus

if networkStatus == .offline {
    Label("Sin conexión", systemImage: "wifi.slash")
        .foregroundStyle(.secondary)
}
```

---

## Accesibilidad

### VoiceOver
```swift
.accessibilityLabel("[Descripción del elemento]")
.accessibilityHint("[Sugerencia de acción]")
.accessibilityValue("[Valor actual]")
```

### Dynamic Type
```swift
Text("[Texto]")
    .font(.body)
    .dynamicTypeSize(...(.xxxLarge)) // Limitar si necesario
```

### Reduce Motion
```swift
@Environment(\.accessibilityReduceMotion) var reduceMotion

if reduceMotion {
    // Sin animaciones
} else {
    // Con animaciones
}
```

### Color Contrast
```swift
@Environment(\.colorScheme) var colorScheme

// Adaptar colores según light/dark mode
```

---

## Temas y Estilos

### Color Scheme
```swift
// Assets.xcassets colors
Color("AccentColor")
Color("BackgroundPrimary")
Color("BackgroundSecondary")

// System colors
Color.primary
Color.secondary
Color.accentColor
```

### Custom Styles
```swift
struct LiquidGlassCardStyle: ViewModifier {
    func body(content: Content) -> some View {
        content
            .padding()
            .background(.ultraThinMaterial)
            .clipShape(RoundedRectangle(cornerRadius: 16))
            .shadow(color: .black.opacity(0.1), radius: 10)
    }
}

extension View {
    func liquidGlassCard() -> some View {
        modifier(LiquidGlassCardStyle())
    }
}
```

---

## Animaciones

### Transiciones Fluidas
```swift
.animation(.spring(response: 0.3, dampingFraction: 0.8), value: state)

// O más control
withAnimation(.spring()) {
    // Cambios de estado
}
```

### Custom Transitions
```swift
.transition(.asymmetric(
    insertion: .move(edge: .trailing).combined(with: .opacity),
    removal: .move(edge: .leading).combined(with: .opacity)
))
```

---

## Performance

### Lazy Loading
```swift
LazyVStack(spacing: 16) {
    ForEach(items) { item in
        ItemRow(item: item)
    }
}
```

### AsyncImage
```swift
AsyncImage(url: URL(string: imageUrl)) { phase in
    switch phase {
    case .empty:
        ProgressView()
    case .success(let image):
        image
            .resizable()
            .aspectRatio(contentMode: .fill)
    case .failure:
        Image(systemName: "photo")
    @unknown default:
        EmptyView()
    }
}
```

### Task Management
```swift
.task {
    await viewModel.loadData()
}
.refreshable {
    await viewModel.refresh()
}
```

---

## Testing

### Unit Tests
```swift
@Test func testViewModel() async {
    let viewModel = [Feature]ViewModel()

    await viewModel.[action]()

    #expect(viewModel.state == .success)
}
```

### UI Tests (XCTest)
```swift
func testScreenDisplaysCorrectly() {
    let app = XCUIApplication()
    app.launch()

    let element = app.staticTexts["[Identificador]"]
    XCTAssertTrue(element.exists)
}
```

### Preview Tests
```swift
#Preview("[Nombre]") {
    [Component]View()
}

#Preview("[Nombre] - Dark") {
    [Component]View()
        .preferredColorScheme(.dark)
}
```

---

## Mockups Conceptuales

### iOS
```
┌─────────────────────────┐
│  [← Título]         ⋮   │ <- NavigationBar
├─────────────────────────┤
│  ╭───────────────────╮  │
│  │ Liquid Glass Card │  │ <- glassmorphism effect
│  ╰───────────────────╯  │
│                         │
│  [Contenido]            │
└─────────────────────────┘
│ 🏠  📚  📊  👤          │ <- TabView
└─────────────────────────┘
```

### iPad (Landscape)
```
┌────────┬──────────────────────────┐
│ 📚 Sec1│  [Título]          ⋮     │
│ 📊 Sec2│                          │
│ 👤 Sec3│  ╭──────────────────╮    │
│        │  │  Liquid Glass    │    │
│        │  │  Expanded Card   │    │
│        │  ╰──────────────────╯    │
└────────┴──────────────────────────┘
 Sidebar        Detail
```

### macOS
```
┌─────────────────────────────────────┐
│ File  Edit  View  Window  Help      │ <- Menu Bar
├──────┬──────────────────────────────┤
│ 🏠   │  [Título]            ⋮       │ <- Toolbar
├──────┼──────────────────────────────┤
│ 📚   │                              │
│ 📊   │   [Contenido aprovechando    │
│ 👤   │    espacio horizontal]       │
│      │                              │
└──────┴──────────────────────────────┘
Sidebar        Content Area
```

---

## Decisiones de Diseño

### Trade-offs
1. **[Decisión]**: [Opción elegida]
   - Pros: [Ventajas]
   - Cons: [Desventajas]
   - Alternativa: [Opción descartada]

### Plataforma-Specific Choices
- **iOS**: [Decisiones específicas iPhone]
- **iPadOS**: [Decisiones específicas iPad]
- **macOS**: [Decisiones específicas Mac]

---

## Dependencias

### Swift Packages
```swift
dependencies: [
    .package(url: "[URL del paquete]", from: "[version]"),
]
```

### Frameworks Apple
- SwiftUI
- Combine
- CoreData / SwiftData
- [Otros frameworks necesarios]

---

## Notas de Implementación

### Consideraciones Técnicas
- [Notas importantes para developers]
- [Patrones recomendados]
- [Anti-patrones a evitar]

### Apple HIG Compliance
- [Guidelines seguidas]
- [Excepciones justificadas]

### Referencias
- [Apple Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/)
- [SwiftUI Documentation](https://developer.apple.com/documentation/swiftui/)

---

## Checklist de Completitud

- [ ] Estados de UI definidos
- [ ] Componentes SwiftUI especificados
- [ ] Liquid Glass effects aplicados
- [ ] Navegación documentada (iOS/iPadOS/macOS)
- [ ] Manejo de errores definido
- [ ] Offline support considerado
- [ ] Accesibilidad documentada (VoiceOver, Dynamic Type, etc.)
- [ ] Animaciones fluidas especificadas
- [ ] Tests planificados
- [ ] Mockups conceptuales creados
- [ ] Adaptaciones por plataforma documentadas (iOS/iPadOS/macOS)
- [ ] Apple HIG compliance verificado
