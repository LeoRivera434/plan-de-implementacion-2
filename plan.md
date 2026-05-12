# PLAN DE IMPLEMENTACIÓN: "URBA & FLOW" – TIENDA DIGITAL DE ROPA PARA HOMBRE
> **Alcance:** Desarrollo exclusivo para entornos de desarrollo/staging. Sin analíticas, sin Crashlytics, sin despliegue a producción. **Multiplataforma nativa (Android + iOS) + Web + Windows**. Roles: Admin y Usuario. Estado: Provider. Backend: Firebase (Auth + Firestore + Storage). UI: Paleta azul urbana/minimalista. Sin código en este documento.

---

## 📱 1. Adaptación Multiplataforma: Android & iOS (Nativo)

### Configuración Específica por Plataforma

#### 🔵 Android (`android/`)
| Archivo | Configuración Requerida | Propósito |
|---------|------------------------|-----------|
| `AndroidManifest.xml` | Permisos: `INTERNET`, `READ_EXTERNAL_STORAGE`, `CAMERA` (opcional) | Acceso a red y galería para perfil/checkout |
| `build.gradle (app)` | `minSdkVersion 21`, `targetSdkVersion 34`, `multidex true` | Compatibilidad y soporte Firebase |
| `google-services.json` | Configurado con package name `com.urbaflow.app.dev` | Conexión con Firebase Auth/Firestore |
| `res/values/styles.xml` | Tema base `Theme.MaterialComponents.DayNight.NoActionBar` | Consistencia UI con Flutter |
| `res/mipmap-*/` | Iconos adaptativos para launcher en todas las densidades | Identidad de marca en dispositivo |

**Consideraciones Android:**
- ✅ Soporte para gestos de retroceso del sistema (`WillPopScope` / `PopScope`)
- ✅ Manejo de permisos en runtime con `permission_handler` (galería, cámara)
- ✅ Adaptación a diferentes aspect ratios (notch, punch-hole, tablets)
- ✅ Soporte para back button físico en navegación

#### 🍎 iOS (`ios/`)
| Archivo | Configuración Requerida | Propósito |
|---------|------------------------|-----------|
| `Info.plist` | Keys: `NSPhotoLibraryUsageDescription`, `NSCameraUsageDescription`, `CFBundleURLTypes` | Permisos y deep linking |
| `Runner.xcodeproj` | Deployment Target: iOS 13.0+, Signing Team ID configurado | Build y distribución en staging |
| `GoogleService-Info.plist` | Configurado con bundle ID `com.urbaflow.app.dev` | Conexión con Firebase |
| `Assets.xcassets/AppIcon.appiconset` | Iconos en todas las resoluciones requeridas por App Store Connect | Identidad visual en dispositivo |
| `LaunchScreen.storyboard` | Pantalla de lanzamiento con logo centrado y fondo azul `#1E3A8A` | Experiencia de carga nativa |

**Consideraciones iOS:**
- ✅ SafeArea handling para notch y Dynamic Island (`MediaQuery.of(context).padding`)
- ✅ Soporte para gestos de swipe para retroceder (`CupertinoBackGestureDetector`)
- ✅ Tipografía San Francisco como fallback si `Inter/Montserrat` no carga
- ✅ Respetar modo oscuro del sistema (preparado en `ThemeData`)

---

## 🗂️ 2. Arquitectura y Estructura de Carpetas (Feature-First + Platform)
```
lib/
├── core/
│   ├── config/
│   │   ├── app_flavor.dart      # Flavor: dev, staging (sin prod)
│   │   ├── platform_config.dart # Config específica Android/iOS/Web/Win
│   │   └── routes/              # Definición de rutas con go_router
│   ├── theme/
│   │   ├── app_theme.dart       # ThemeData con paleta azul
│   │   ├── app_colors.dart      # Constantes de color por plataforma
│   │   └── app_typography.dart  # Inter/Montserrat con fallbacks
│   ├── utils/
│   │   ├── platform_utils.dart  # Helpers: isAndroid, isIOS, isWeb, isWindows
│   │   ├── validators.dart
│   │   └── formatters.dart
│   └── services/
│       ├── firebase_init.dart   # Init condicional por plataforma
│       ├── storage_service.dart
│       └── cloud_functions_stub.dart # Stub para dev (sin deploy real)
├── features/
│   ├── auth/
│   │   ├── login_screen.dart    # Adaptaciones UI por plataforma
│   │   ├── register_screen.dart
│   │   └── auth_provider.dart
│   ├── catalog/
│   │   ├── product_list_screen.dart
│   │   ├── product_detail_screen.dart # Galería adaptativa móvil/desktop
│   │   └── product_provider.dart
│   ├── cart/
│   │   ├── cart_screen.dart     # BottomSheet en móvil, sidebar en desktop
│   │   └── cart_provider.dart
│   ├── checkout/
│   │   ├── checkout_flow.dart   # Stepper adaptativo por pantalla
│   │   └── checkout_provider.dart
│   ├── admin/
│   │   ├── dashboard_screen.dart # Layout responsive: rail lateral desktop
│   │   └── admin_provider.dart
│   └── profile/
│       ├── profile_screen.dart
│       └── profile_provider.dart
├── shared/
│   ├── widgets/
│   │   ├── adaptive/
│   │   │   ├── adaptive_button.dart    # Material/Cupertino según plataforma
│   │   │   ├── adaptive_scaffold.dart  # Layout móvil vs desktop
│   │   │   └── adaptive_dialog.dart    # Diálogos nativos por OS
│   │   ├── common/
│   │   │   ├── product_card.dart
│   │   │   ├── app_text_field.dart
│   │   │   └── loading_indicator.dart
│   │   └── platform/
│   │       ├── android_specific.dart
│   │       ├── ios_specific.dart
│   │       └── desktop_specific.dart
│   ├── models/
│   │   ├── product_model.dart
│   │   ├── user_model.dart
│   │   └── order_model.dart
│   └── providers/
│       ├── auth_provider.dart
│       ├── product_provider.dart
│       ├── cart_provider.dart
│       └── theme_provider.dart
├── main.dart                    # Entry point, MultiProvider, MaterialApp.router
└── flavors/
    ├── main_dev.dart           # Punto de entrada para flavor dev
    └── main_staging.dart       # Punto de entrada para flavor staging
assets/
├── fonts/
├── images/
│   ├── android/                # Assets específicos Android (splash, iconos)
│   ├── ios/                    # Assets específicos iOS (launch, appicon)
│   └── common/                 # Assets compartidos
└── icons/
```

---

## 🗄️ 3. Mapeo Relacional → Firestore (NoSQL)
| Entidad SQL | Adaptación Firestore | Estrategia |
|-------------|----------------------|------------|
| `producto` | Colección `products` | Documento con campos base. `variants` como array embebido (<20 variantes) o subcolección si escala. |
| `variante` | Array dentro de `products` o subcolección `products/{id}/variants` | Cada variante: `talla`, `color`, `codigo_barras`, `precio_extra`, `stock`, `activo`. |
| `categoria` | Colección `categories` | Jerarquía vía `parentId`. Índices por `orden` y `slug`. |
| `imagen` | Array `imageUrls` en producto o subcolección `products/{id}/images` | Solo URLs de Firebase Storage + `orden` + `principal`. |
| `inventario` | Campo `stock` dentro de cada variante + colección `inventory_movements` | `stock` se actualiza atómicamente con transactions. |
| `movimiento_inventario` | Colección `inventory_movements` | Documentos con `variantId`, `branchId`, `tipo`, `cantidad`, `motivo`, `fecha`, `employeeId`. |
| `cliente` | Colección `users` (UID de Auth como doc ID) | Perfil embebido. Direcciones en subcolección `users/{uid}/addresses`. |
| `pedido` + `detalle_pedido` | Colección `orders` | `details` como array de mapas. Campos calculados (`subtotal`, `total`) denormalizados. |
| `pago` + `envio` | Campos embebidos dentro de `orders` | `payment: { metodo, estado, referencia }`, `shipping: { paqueteria, estado, tracking }`. |
| `cupon` | Colección `coupons` | Validación de `usos_max`, `fecha_inicio/fin`, `minimo_compra`. |
| `pedido_cupon` | Campo `couponApplied` dentro de `orders` | No requiere colección separada. |
| `sucursal`, `empleado`, `proveedor` | Colecciones `branches`, `employees`, `suppliers` | `employees` vinculado a Auth con custom claims. Solo admin escribe. |

---

## 🔐 4. Autenticación y Control de Acceso (RBAC)
| Paso | Acción | Entregable |
|------|--------|------------|
| 4.1 | Registro/Login con email/password vía Firebase Auth | Flujo completo con validación de formulario |
| 4.2 | Asignación de Custom Claims (`admin: true/false`) desde backend simulado o Cloud Functions stub | Claims accesibles en cliente vía `AuthProvider` |
| 4.3 | Interceptor de rutas (`go_router redirect`) que redirige según rol y estado de sesión | Navegación protegida: `/admin` solo para admins |
| 4.4 | Reglas de Firestore: lectura pública para catálogo, escritura solo para `admin`, acceso a `users/{uid}` solo por propietario o admin | `.rules` validadas en Firebase Emulator Suite |
| 4.5 | Persistencia de sesión automática y cierre seguro con limpieza de estado local y providers | Estado coherente post-logout en todas las plataformas |

---

## 🔄 5. Gestión de Estado con Provider
| Ámbito | Responsabilidad | Alcance | Patrón de Respuesta |
|--------|-----------------|---------|-------------------|
| `AuthProvider` | Login, registro, claims, perfil, logout, refresh token | Raíz (`MultiProvider`) | `ResultState<User>` |
| `ProductProvider` | Listado, filtros, búsqueda, detalle, paginación, favoritos | Feature `catalog` o raíz | `ResultState<List<Product>>` |
| `CartProvider` | Agregar, modificar, eliminar items, cálculo de totales, persistencia local (SharedPreferences) | Raíz (acceso global) | `ResultState<Cart>` |
| `CheckoutProvider` | Dirección seleccionada, cupón, método de pago, validación, creación de pedido | Feature `checkout` | `ResultState<Order>` |
| `AdminProvider` | CRUD productos/variantes/categorías, actualización de stock, gestión de pedidos | Feature `admin` | `ResultState<void>` |
| `ThemeStateProvider` | Cambio de tonalidades azules, modo claro/oscuro, preferencia de plataforma | Raíz | `ResultState<ThemeMode>` |

> **Patrón `ResultState<T>` conceptual:**
```dart
abstract class ResultState<T> {
  const ResultState();
  factory ResultState.idle() => _Idle<T>();
  factory ResultState.loading() => _Loading<T>();
  factory ResultState.success(T data) => _Success<T>(data);
  factory ResultState.error(String message) => _Error<T>(message);
}
```
> Todos los providers exponen listeners eficientes con `notifyListeners()` solo cuando el estado cambia realmente, evitando rebuilds innecesarios.

---

## 🎨 6. UI/UX: Tonalidades Azules y Adaptabilidad Multiplataforma
### Paleta de Colores
| Token | Hex | Uso |
|-------|-----|-----|
| `primaryDark` | `#0A2540` | AppBar, footer, textos sobre fondo claro |
| `primary` | `#1E3A8A` | Botones principales, activos, enlaces |
| `primaryLight` | `#3B82F6` | Hover, acentos, iconos interactivos |
| `primarySoft` | `#60A5FA` | Fondos suaves, estados focus, loaders |
| `primaryPale` | `#93C5FD` | Bordes sutiles, placeholders, separadores |
| `background` | `#FFFFFF` | Fondo principal (modo claro) |
| `surface` | `#F8FAFC` | Tarjetas, contenedores secundarios |
| `textPrimary` | `#0F172A` | Títulos, texto principal |
| `textSecondary` | `#475569` | Subtítulos, descripciones, metadata |

### Adaptaciones por Plataforma
| Elemento | Android | iOS | Web | Windows |
|----------|---------|-----|-----|---------|
| **AppBar** | Material 3, elevation 2, color `primaryDark` | Cupertino-style opcional, blur effect | Sticky con shadow, URL visible | Title bar integrado, menú en barra superior |
| **Navegación** | BottomNavigationBar (5 ítems) + Drawer | TabBar inferior + swipe gestures | NavigationRail lateral o top menu | NavigationRail + soporte teclado (Ctrl+Tab) |
| **Botones** | Ripple effect Material, radius 12 | Highlight effect Cupertino, radius 12 | Hover state visible, cursor pointer | Focus ring visible, teclado navegable |
| **Inputs** | Underline o outlined Material | Rounded Cupertino style | Focus border `primaryLight`, label flotante | Focus indicator accesible, tab order lógico |
| **Diálogos** | Material AlertDialog | CupertinoActionSheet en móvil, alert en tablet | Modal centrado con backdrop blur | Window modal nativo-style |
| **Scroll** | Overscroll glow azul | Bounce effect iOS | Scrollbar visible al hover | Scrollbar siempre visible, wheel support |

### Responsividad con `LayoutBuilder` + `AdaptiveLayout`
```dart
// Concepto de widget adaptativo
class AdaptiveProductGrid extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final breakpoint = AdaptiveLayout.of(context).breakpoint;
    return GridView.count(
      crossAxisCount: breakpoint.crossAxisCount, // 2 mobile, 3 tablet, 4-6 desktop
      childAspectRatio: breakpoint.aspectRatio,   // 0.75 mobile, 0.85 desktop
      children: products.map((p) => ProductCard(p)).toList(),
    );
  }
}
```

### Accesibilidad (WCAG AA)
- [x] Contraste mínimo 4.5:1 para texto normal, 3:1 para texto grande
- [x] `Semantics` widgets para lectores de pantalla (TalkBack/VoiceOver)
- [x] Tamaños de texto escalables con `MediaQuery.textScaler`
- [x] Navegación completa por teclado en Web/Windows (tab index lógico)
- [x] Labels descriptivos en iconos e imágenes (`alt` text en Web)

---

## 👥 7. Flujos de Usuario por Rol

### Cliente/Usuario
1. **Registro/Login** → Validación en tiempo real → Redirección a Home según plataforma
2. **Exploración** → Navegación por categorías, búsqueda con debounce, filtros (talla, precio, color, disponibilidad)
3. **Detalle de Producto** → Selector de variante (talla/color), galería swipeable, zoom en imágenes, agregar al carrito con feedback visual
4. **Carrito** → Persistencia local con `shared_preferences` + sincronización con Firestore si autenticado; edición de cantidades en tiempo real
5. **Checkout** → Selección de dirección (o nueva), aplicación de cupón con validación instantánea, resumen con cálculo de impuestos, confirmación (simulación de pago en dev)
6. **Perfil** → Historial de pedidos con estado visual, gestión de direcciones (CRUD), edición de datos personales, preferencias de notificación

### Administrador
1. **Login Admin** → Credenciales validadas + claims verificados → Dashboard con métricas clave (simuladas)
2. **CRUD Productos** → Formulario con validación en tiempo real: nombre, slug, descripción rica, precio, precio oferta, estado, destacado, categorías
3. **Gestión de Variantes** → Tabla editable: talla, color, código de barras, precio extra, stock, activo; bulk edit para múltiples variantes
4. **Inventario** → Registro de entradas/salidas/ajustes con motivo; alertas visuales de stock mínimo; historial de movimientos
5. **Pedidos** → Lista filtrable por estado; cambio de estado con transición visual; asignación de paquetería y tracking; cancelación con confirmación
6. **Cupones** → Creación con validación de fechas y límites; visualización de usos actuales vs máximos; activación/desactivación rápida
7. **Sucursales/Proveedores** → CRUD básico con campos esenciales; solo lectura para usuarios no-admin
8. **UX Admin** → Validación estricta en formularios, confirmaciones modales para acciones destructivas, snackbars de feedback con iconos y colores de estado

---

## 📦 8. Dependencias `pubspec.yaml` (Conceptual + Platform)
```yaml
dependencies:
  flutter:
    sdk: flutter

  # Firebase Core + Servicios (multiplataforma)
  firebase_core: ^3.6.0
  firebase_auth: ^5.3.1
  cloud_firestore: ^5.4.4
  firebase_storage: ^12.3.2

  # Estado & Enrutamiento
  provider: ^6.1.2
  go_router: ^14.2.7

  # UI & Assets
  cached_network_image: ^3.4.1      # Carga eficiente en móvil y web
  flutter_svg: ^2.0.10              # Iconos escalables
  google_fonts: ^6.2.1              # Inter + Montserrat con fallbacks
  intl: ^0.19.0                     # Formateo de moneda y fecha por locale

  # Utilidades & Persistencia Local
  shared_preferences: ^2.3.2        # Carrito local y preferencias
  shared_preferences_web: ^2.4.0    # Soporte web explícito
  shared_preferences_windows: ^2.4.0 # Soporte Windows explícito
  uuid: ^4.5.0                      # IDs únicos para offline-first
  formz: ^0.7.0                     # Validación de formularios tipada
  equatable: ^2.0.5                 # Comparación eficiente de modelos

  # Platform-specific utilities (opcionales pero recomendadas)
  url_launcher: ^6.2.5              # Abrir enlaces externos (políticas, contacto)
  url_launcher_android: ^6.3.0
  url_launcher_ios: ^6.2.5
  url_launcher_web: ^2.3.0
  url_launcher_windows: ^3.1.0

dev_dependencies:
  flutter_test:
    sdk: flutter
  integration_test:
    sdk: flutter
  flutter_lints: ^5.0.0
  mocktail: ^1.0.4
```

> **Excluido explícitamente:** `firebase_analytics`, `firebase_crashlytics`, `firebase_remote_config`, `flutter_facebook_auth`, `google_sign_in`, herramientas de despliegue (`flutter distribute`, CI/CD de producción).

---

## 🧪 9. Pruebas y Validación (Entorno Dev/Staging)
| Tipo | Alcance | Herramienta | Platform Coverage |
|------|---------|-------------|------------------|
| **Unitarias** | Modelos, lógica de cálculo, validadores, providers | `flutter test` | ✅ Todas (lógica pura) |
| **Widget** | Componentes reutilizables, estados UI, formularios | `testWidgets` | ✅ Android + iOS (emuladores) |
| **Integración** | Flujo completo: Login → Catálogo → Carrito → Checkout (simulado) | `integration_test` | ✅ Android + iOS + Web (headless) |
| **Reglas Firestore** | Simulación de lecturas/escrituras por rol y escenario | Firebase Emulator Suite | ✅ Backend-agnóstico |
| **Platform-Specific** | Gestos, navegación, permisos, UI nativa | Testing manual en emuladores/dispositivos | ✅ Android + iOS + Web + Windows |
| **Performance** | Rebuilds innecesarios, carga de imágenes, memory leaks | Flutter DevTools (memory, frames, CPU) | ✅ Perfilado por plataforma |
| **Accesibilidad** | Lectores de pantalla, contraste, navegación por teclado | TalkBack, VoiceOver, axe-core (web) | ✅ Android + iOS + Web |

---

## 🗓️ 10. Roadmap de Desarrollo (Dev/Staging)
| Semana | Foco | Entregable |
|--------|------|------------|
| **1** | Setup, estructura feature-first, tema azul, routing base, flavors dev/staging | Proyecto inicial ejecutable en Android + iOS emuladores |
| **2** | Firebase Auth + RBAC + Providers base + platform config | Login funcional, claims, `AuthProvider`, rutas protegidas |
| **3** | Firestore mapping + Catálogo + Filtros + Adaptive Layout | `ProductProvider`, listado paginado, búsqueda, grid responsive |
| **4** | Carrito + Checkout simulado + Cupones + persistencia local | `CartProvider`, flujo de pago dev, validaciones cross-platform |
| **5** | Panel Admin (CRUD productos/variantes/stock) + UI adaptativa desktop | Formularios, tablas, alertas, reglas de seguridad, layout responsive |
| **6** | Pedidos + Direcciones + Perfil + Pruebas de integración | Flujo completo end-to-end, tests, emulador Firestore configurado |
| **7** | Optimización, accesibilidad, responsive Web/Windows, platform tweaks | `AdaptiveLayout`, DevTools profiling, validación cross-platform completa |
| **8** | Documentación, empaquetado dev, revisión final, checklist QA | README técnico, guía de flavors, checklist de validación, build de staging |

---

## ✅ Checklist de Validación Pre-Implementación
- [ ] Esquema Firestore alineado con requerimientos relacionales y escalabilidad
- [ ] Custom claims definidos y probados para `admin` vs `user`
- [ ] Reglas de seguridad Firestore probadas en emulador con casos de rol
- [ ] Providers con `ResultState<T>` y listeners eficientes (sin rebuilds innecesarios)
- [ ] Rutas protegidas por `go_router` según rol y plataforma
- [ ] Paleta azul aplicada a `ThemeData` y componentes con contraste WCAG AA
- [ ] Dependencias limpias: sin analíticas, sin crashlytics, sin prod
- [ ] Estructura de carpetas feature-first + platform adapters lista para escalabilidad
- [ ] Plan de pruebas unitarias/widget/integration definido por plataforma
- [ ] Documentación de decisiones técnicas y arquitectura iniciada (`/docs`)
- [ ] Flavors `dev` y `staging` configurados con diferentes bundle IDs/package names
- [ ] Assets específicos por plataforma organizados (`assets/android/`, `assets/ios/`)
- [ ] Soporte para modo oscuro preparado en `ThemeData` (aunque no se active inicialmente)
- [ ] Configuración de `platform_config.dart` para detectar y adaptar comportamiento por OS

---

> **Nota Final:** Este plan está diseñado exclusivamente para **entornos de desarrollo y staging**. No incluye configuración para producción, monitoreo, analíticas, ni despliegue automatizado. La arquitectura multiplataforma prioriza consistencia visual y de experiencia, respetando las convenciones nativas de Android e iOS cuando aporta valor al usuario.

¿Te gustaría que genere ahora las **reglas de seguridad de Firestore**, la **configuración de emuladores**, o la **estructura base de los Providers con ResultState** para comenzar la implementación? 🚀
