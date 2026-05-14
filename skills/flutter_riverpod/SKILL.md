---
name: flutter_riverpod
description: >
  Guide for using Flutter Riverpod 2.x in a consistent, maintainable, and
  testable way, including provider selection, UI consumption patterns,
  dependency composition, side-effect handling, lifecycle management,
  and testing strategy.
  Trigger: Use when the task involves `Provider`, `StateProvider`,
  `NotifierProvider`, `AsyncNotifierProvider`, `FutureProvider`,
  `StreamProvider`, `ProviderScope`, `WidgetRef`, `ref.watch`,
  `ref.read`, `ref.listen`, `AsyncValue`, provider overrides, or when
  deciding how state, dependencies, side effects, or controller logic
  should be modeled with Riverpod in Flutter.
metadata:
  stack: dart
  framework: flutter
  area: state-management
---

# SKILL: Flutter Riverpod (Project-agnostic)

Este documento define **reglas generales** y patrones recomendados para trabajar con **flutter_riverpod (2.x)** de forma consistente, mantenible y testeable, sin asumir arquitectura o dominio específico.

> **Meta:** que cualquier proyecto Flutter con Riverpod tenga un “estándar mínimo” de calidad: estado predecible, dependencias claras, UI reactiva, side-effects controlados y pruebas fáciles.

---

## 0) Setup base (REQUIRED)

### ProviderScope

```dart
// ✅ ALWAYS: Wrap the app with ProviderScope.
void main() {
  runApp(const ProviderScope(child: MyApp()));
}
```

### Un solo punto de inicialización de dependencias

* **REQUIRED:** toda dependencia global (cliente HTTP, repositorios, DB, almacenamiento, etc.) debe exponerse **desde providers**.
* **FORBIDDEN:** singletons globales “hardcodeados” accedidos desde cualquier parte.

---

## 1) Principios de diseño

### 1.1 Providers como “composición de dependencias”

* Un provider debe ser **pequeño, enfocado** y con responsabilidades claras.
* Providers deben componerse: un provider puede depender de otros, pero evita cadenas profundas e ilegibles.

### 1.2 UI declarativa, side-effects explícitos

* El `build()` **solo** debe:

  * observar estado (`ref.watch`),
  * renderizar UI,
  * delegar acciones a notifiers/services.
* Side-effects (snackbars, navegación, logging, analytics): **solo** con `ref.listen` / `ref.listenManual`.

### 1.3 Estado inmutable por defecto

* Prefiere estados inmutables (por ejemplo, con `copyWith`), para evitar bugs por mutación.
* Listas/mapas: siempre reasigna (`state = [...state, item]`).


## 2) Selección de tipo de provider (reglas)

### 2.1 Provider (dependencias sin estado)

* Use para exponer **dependencias** o valores inmutables: repositorios, clientes, configuración.

```dart
final httpClientProvider = Provider<HttpClient>((ref) {
  final client = HttpClient();
  ref.onDispose(client.close);
  return client;
});
```

### 2.2 StateProvider (estado simple, UI-driven)

* **Solo** para estado trivial: toggles, índices, filtros locales.
* **Evita** lógica de negocio dentro del widget.

```dart
final selectedTabProvider = StateProvider<int>((ref) => 0);
```

### 2.3 NotifierProvider / AsyncNotifierProvider (recomendado para lógica)

* Para estado con métodos (acciones) y reglas de negocio.
* Usa `AsyncNotifier` si el estado es asíncrono y quieres modelar `AsyncValue`.

```dart
// ✅ Preferred for sync state with actions.
class Counter extends Notifier<int> {
  @override
  int build() => 0;

  void increment() => state++;
}

final counterProvider = NotifierProvider<Counter, int>(Counter.new);
```

```dart
// ✅ Preferred for async state with actions.
class ItemsController extends AsyncNotifier<List<Item>> {
  @override
  Future<List<Item>> build() async {
    final repo = ref.watch(itemsRepositoryProvider);
    return repo.fetchItems();
  }

  Future<void> refresh() async {
    state = const AsyncLoading();
    state = await AsyncValue.guard(() async {
      final repo = ref.read(itemsRepositoryProvider);
      return repo.fetchItems();
    });
  }
}

final itemsControllerProvider =
    AsyncNotifierProvider<ItemsController, List<Item>>(ItemsController.new);
```

> Nota: `StateNotifier` sigue siendo válido, pero en Riverpod 2.x suele preferirse `Notifier/AsyncNotifier` por ergonomía y consistencia.

### 2.4 FutureProvider / StreamProvider (lectura reactiva sin acciones)

* Útiles cuando solo necesitas exponer un stream/future y no un “controller”.
* Para casos con acciones (create/update/delete), prefiere `AsyncNotifier`.

---

## 3) Reglas de consumo en UI

### 3.1 ConsumerWidget / ConsumerStatefulWidget (REQUIRED)

```dart
class ItemsScreen extends ConsumerWidget {
  const ItemsScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final itemsAsync = ref.watch(itemsControllerProvider);

    return itemsAsync.when(
      data: (items) => ListView(
        children: [for (final i in items) ListTile(title: Text(i.title))],
      ),
      loading: () => const Center(child: CircularProgressIndicator()),
      error: (e, st) => Center(child: Text('Error: $e')),
    );
  }
}
```

### 3.2 watch vs read vs listen

* `ref.watch(provider)`

  * **REQUIRED** dentro de `build()` para UI reactiva.
* `ref.read(provider)`

  * **REQUIRED** para handlers (onPressed, callbacks) o lógica puntual.
  * **FORBIDDEN** como fuente de UI en `build()`.
* `ref.listen(provider, ...)`

  * **REQUIRED** para side-effects (snackbars, navegación, analytics).

```dart
// ✅ Side-effect example.
class SaveScreen extends ConsumerStatefulWidget {
  const SaveScreen({super.key});

  @override
  ConsumerState<SaveScreen> createState() => _SaveScreenState();
}

class _SaveScreenState extends ConsumerState<SaveScreen> {
  @override
  void initState() {
    super.initState();

    ref.listen<AsyncValue<void>>(saveControllerProvider, (prev, next) {
      next.whenOrNull(
        data: (_) {
          // ✅ Side-effect in response to state.
          ScaffoldMessenger.of(context)
              .showSnackBar(const SnackBar(content: Text('Saved')));
        },
        error: (e, _) {
          ScaffoldMessenger.of(context)
              .showSnackBar(SnackBar(content: Text('Error: $e')));
        },
      );
    });
  }

  @override
  Widget build(BuildContext context) {
    return const Placeholder();
  }
}
```

### 3.3 Optimización: select

* Usa `select` para evitar rebuilds por cambios irrelevantes.

```dart
final isLoading = ref.watch(itemsControllerProvider.select((v) => v.isLoading));
```

---

## 4) Reglas para AsyncValue y errores

### 4.1 No “tragues” errores

* Si una operación puede fallar, **debe** producir un error observable (`AsyncValue.error`) o un estado de error explícito.
* Loguea errores de manera consistente (por ejemplo, con un provider de logger).

### 4.2 Usa AsyncValue.guard

```dart
state = await AsyncValue.guard(() async {
  return ref.read(repoProvider).fetch();
});
```

### 4.3 UI siempre maneja loading/error/data

* **REQUIRED:** para `AsyncValue`, siempre usa `.when(...)` o patrones equivalentes.

---

## 5) autoDispose, cache y ciclo de vida

### 5.1 autoDispose por defecto en estado temporal

* Búsquedas, formularios, filtros temporales, pantallas efímeras.

```dart
final searchQueryProvider = StateProvider.autoDispose<String>((ref) => '');
```

### 5.2 Mantener estado cuando aplique

* Si necesitas cache en background o persistencia entre pantallas:

  * usa `keepAlive()` (o `ref.keepAlive()`) según el tipo de provider,
  * o diseña un “scope” claro (por ejemplo, provider a nivel feature).

> Regla: **prefiere liberar** antes de “acumular estado global”. Mantén vivo solo lo que tenga justificación.

### 5.3 Limpieza de recursos

* **REQUIRED:** `ref.onDispose` para cerrar sockets/DB/streams.

---

## 6) Providers parametrizados (.family)

### 6.1 family para dependencias con parámetros

```dart
final itemByIdProvider = Provider.family<Item?, String>((ref, id) {
  final items = ref.watch(itemsControllerProvider).valueOrNull ?? const [];
  return items.where((x) => x.id == id).cast<Item?>().firstOrNull;
});
```

### 6.2 No abuses de family para “todo”

* Si el parámetro es parte de un flujo de navegación o feature, un controller dedicado puede ser mejor.

---

## 7) Convenciones de naming (REQUIRED)

* Providers terminan con `Provider`:

  * `itemsRepositoryProvider`, `itemsControllerProvider`, `authStateProvider`.
* Notifiers/Controllers:

  * `ItemsController`, `AuthController`.
* Archivos:

  * `items_providers.dart`, `auth_controller.dart`.

---

## 8) Arquitectura: separar UI, estado y dominio

### 8.1 Widgets “tontos”, controllers “listos”

* Widgets deberían contener **mínima lógica**.
* La lógica de negocio vive en controllers/notifiers o en use-cases (domain).

### 8.2 Repositorios a través de providers

```dart
final itemsRepositoryProvider = Provider<ItemsRepository>((ref) {
  final client = ref.watch(httpClientProvider);
  return ItemsRepositoryImpl(client);
});
```

---

## 9) Testing (REQUIRED)

### 9.1 Override de providers

```dart
testWidgets('renders items', (tester) async {
  await tester.pumpWidget(
    ProviderScope(
      overrides: [
        itemsRepositoryProvider.overrideWithValue(FakeItemsRepository()),
      ],
      child: const MaterialApp(home: ItemsScreen()),
    ),
  );

  expect(find.byType(ListTile), findsWidgets);
});
```

### 9.2 Test de controllers (sin UI)

* Prueba notifiers con `ProviderContainer`.

```dart
test('refresh updates state', () async {
  final container = ProviderContainer(overrides: [
    itemsRepositoryProvider.overrideWithValue(FakeItemsRepository()),
  ]);
  addTearDown(container.dispose);

  final notifier = container.read(itemsControllerProvider.notifier);
  await notifier.refresh();

  final state = container.read(itemsControllerProvider);
  expect(state.hasValue, true);
});
```

---

## 10) Observabilidad y debugging (recomendado)

### 10.1 ProviderObserver para logging

```dart
class AppObserver extends ProviderObserver {
  @override
  void didUpdateProvider(
    ProviderBase provider,
    Object? previousValue,
    Object? newValue,
    ProviderContainer container,
  ) {
    // ✅ Keep logs concise; avoid leaking secrets.
    // debugPrint('Provider ${provider.name ?? provider.runtimeType} updated');
  }
}

void main() {
  runApp(
    ProviderScope(
      observers: [AppObserver()],
      child: const MyApp(),
    ),
  );
}
```

---

## 11) Anti-patrones (FORBIDDEN)

* **FORBIDDEN:** `ref.read()` en `build()` para pintar UI.
* **FORBIDDEN:** side-effects dentro de `build()` (snackbars/navegación).
* **FORBIDDEN:** providers gigantes que hacen “todo”.
* **FORBIDDEN:** mutar listas/objetos directamente sin reasignar `state`.
* **FORBIDDEN:** dependencia a servicios globales fuera de providers.

---

## 12) Checklist rápido

* [ ] `ProviderScope` en `main()`.
* [ ] Dependencias expuestas por providers con `onDispose` cuando aplique.
* [ ] UI usa `watch` + `.when()` para `AsyncValue`.
* [ ] Side-effects con `listen`.
* [ ] Notifiers/Controllers contienen lógica; widgets quedan ligeros.
* [ ] autoDispose para estado temporal; keepAlive solo con justificación.
* [ ] Tests con `ProviderScope` overrides y/o `ProviderContainer`.
