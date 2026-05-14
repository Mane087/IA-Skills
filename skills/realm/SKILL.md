---
name: realm
description: >
  Guide for using Realm in Flutter as a consistent, testable, and maintainable
  persistence layer, including initialization, schema design, transactions,
  queries, migrations, sync considerations, error handling, and repository-based
  architecture.
  Trigger: Use when the task mentions or contains `Realm`, `Configuration.local`,
  Realm schemas/models, write transactions, migrations, query encapsulation,
  Realm providers or DI setup, live Realm objects, sync/session handling, or
  when deciding how local persistence, reactivity, and data-layer responsibilities
  should be modeled with Realm in a Flutter app.
compatibility: opencode
metadata:
  stack: dart
  framework: flutter
  area: persistence
---

# SKILL: Realm para Flutter (agnóstico al proyecto)

Este documento define reglas y patrones recomendados para usar **Realm** como capa de persistencia en Flutter, priorizando: **consistencia**, **seguridad**, **rendimiento**, **mantenibilidad** y **testabilidad**.

> **Meta:** que el acceso a datos sea predecible: una única fuente de verdad, transacciones claras, modelos bien definidos, migraciones controladas y mínimos “efectos colaterales” en UI.

---

## 0) Principios base

### 0.1 Realm es una base orientada a objetos (OODB)

* Los objetos de Realm suelen ser **vivos** (live objects): reflejan cambios automáticamente.
* No asumas que un objeto es “inmutable” o que puedes usarlo como DTO fuera de su contexto.

### 0.2 Single responsibility

* La UI **no debe** contener lógica de persistencia.
* El acceso a Realm debe pasar por una capa de **data source / repository** (aunque sea minimalista).

---

## 1) Inicialización y ciclo de vida (REQUIRED)

### 1.1 Un solo punto de construcción de Realm

* **REQUIRED:** construye la instancia de Realm en un solo lugar.
* **REQUIRED:** expón esa instancia vía un provider (por ejemplo, Riverpod) o un contenedor de DI.
* **FORBIDDEN:** crear múltiples instancias “porque sí” en widgets o services.

```dart
// ✅ Create and manage Realm lifecycle in one place.
final realmProvider = Provider<Realm>((ref) {
  final config = Configuration.local(
    [User.schema, Todo.schema],
    schemaVersion: 1,
    migrationCallback: (migration, oldVersion) {
      // English comments preferred.
      // Handle migrations here when schemaVersion changes.
    },
  );

  final realm = Realm(config);

  ref.onDispose(() {
    // ✅ Always close the Realm instance.
    realm.close();
  });

  return realm;
});
```

### 1.2 No compartas Realm entre isolates

* **REQUIRED:** cada isolate debe abrir su propia instancia.
* Si usas background work, pasa **IDs/valores primitivos**, no instancias de Realm ni objetos Realm.

---

## 2) Modelado de datos (schemas) (REQUIRED)

### 2.1 Convenciones para models

* **REQUIRED:** define un `primaryKey` si el modelo tiene identidad estable.
* **REQUIRED:** define defaults razonables.
* **REQUIRED:** evita campos opcionales si el dominio los requiere (prefiere validación en capa de dominio).

```dart
@RealmModel()
class _Todo {
  @PrimaryKey()
  late String id;

  late String title;
  bool isDone = false;

  // Use DateTime for timestamps.
  DateTime createdAt = DateTime.now();
}
```

### 2.2 Separación “domain vs persistence” (recomendado)

* Si tu proyecto requiere independencia de base de datos, considera:

  * Entidades de dominio (inmutables)
  * Mappers hacia/desde Realm models
* Si es un proyecto simple, puedes usar directamente Realm models, pero mantén la lógica fuera de UI.

---

## 3) Escrituras y transacciones (REQUIRED)

### 3.1 Toda escritura debe ir en una transacción

* **REQUIRED:** `realm.write(() { ... })` (o la variante disponible en tu versión).
* **FORBIDDEN:** mutar objetos fuera de `write`.

```dart
void toggleTodo(Realm realm, String todoId) {
  final todo = realm.find<Todo>(todoId);
  if (todo == null) return;

  realm.write(() {
    // ✅ Mutations must happen inside a write transaction.
    todo.isDone = !todo.isDone;
  });
}
```

### 3.2 Evita transacciones anidadas

* **REQUIRED:** estructura tus repositorios para que una operación haga una sola transacción.
* Si un método interno necesita escribir, que reciba una función o se asuma que el caller ya está en write.

### 3.3 Escrituras por lotes

* Si insertas/actualizas muchos elementos:

  * usa una sola transacción,
  * evita loops con transacciones repetidas.

---

## 4) Lecturas, queries y reactividad

### 4.1 Queries deben ser “reproducibles” y declarativas

* **REQUIRED:** encapsula queries en repositorios o providers.
* **FORBIDDEN:** duplicar strings de query por toda la app.

### 4.2 Colecciones reactivas (streams/notificaciones)

* Cuando necesites UI reactiva:

  * expón la colección (Results/RealmList) y/o un stream de cambios.
  * asegúrate de no filtrar/transformar de forma costosa en cada rebuild.

> Regla práctica: filtra en la query, no en el widget, cuando el dataset sea mediano/grande.

### 4.3 Congelar / copiar para “UI segura”

* Como los objetos son vivos, si necesitas:

  * pasar datos a otra capa,
  * evitar que cambien durante un frame,
  * serializar,
    entonces considera:
  * `freeze()` (si tu SDK lo soporta), o
  * mapear a DTO inmutable.

---

## 5) Manejo de errores (REQUIRED)

### 5.1 No ocultes excepciones de persistencia

* Si falla una escritura o query, debe:

  * propagarse como error controlado (por ejemplo, `Result`/`Either`), o
  * mapearse a una excepción de dominio.

### 5.2 Errores comunes a estandarizar

* Violación de primary key / duplicados
* Migración requerida
* Permisos / sync (si aplica)
* Escritura fuera de transacción

---

## 6) Migraciones y versionado (REQUIRED)

### 6.1 Siempre incrementa `schemaVersion` ante cambios de esquema

* **REQUIRED:** cualquier cambio a models (campos, tipos, renombres) implica revisar migración.

### 6.2 Migraciones idempotentes

* **REQUIRED:** tu `migrationCallback` debe soportar:

  * usuarios que saltan versiones,
  * ejecución desde `oldVersion` menor a la actual.

```dart
final config = Configuration.local(
  [Todo.schema],
  schemaVersion: 3,
  migrationCallback: (migration, oldVersion) {
    // English comments preferred.
    if (oldVersion < 2) {
      // Apply changes for v2.
    }
    if (oldVersion < 3) {
      // Apply changes for v3.
    }
  },
);
```

### 6.3 Cambios destructivos

* Si un cambio rompe compatibilidad y no hay migración razonable:

  * documenta el “reset” (borrar realm),
  * y asegúrate de que el usuario no pierda datos críticos (backup/export si aplica).

---

## 7) Sincronización (si aplica)

> Esta sección aplica si usas Realm Sync / App Services. Si no, ignórala.

### 7.1 Autenticación y sesión

* **REQUIRED:** encapsula login/logout y el `app.currentUser` en un `AuthRepository`.
* **FORBIDDEN:** lógica de auth dispersa en pantallas.

### 7.2 Reglas de acceso y permisos

* No confíes solo en el cliente.
* Define permisos en backend (App Services) y trata errores como parte del flujo.

### 7.3 Modo offline y conflictos

* Define explícitamente:

  * qué pasa si no hay red,
  * cómo se reintenta,
  * cómo se comunican conflictos a UI.

---

## 8) Rendimiento y buenas prácticas

### 8.1 Evita lecturas masivas innecesarias

* Pagina o limita resultados cuando aplique.
* Prefiere queries específicas sobre “traer todo”.

### 8.2 Índices (si están disponibles en tu SDK)

* Para campos muy consultados, considera indexado (según soporte del SDK).

### 8.3 No hagas trabajo pesado en el hilo UI

* Transformaciones grandes, exportaciones o cálculos sobre colecciones grandes:

  * muévelos a un isolate,
  * o reduce el dataset antes de mapear.

---

## 9) Testing (REQUIRED)

### 9.1 Realm aislado por test

* **REQUIRED:** cada test debe usar un Realm dedicado (path temporal) para evitar “state leakage”.
* **REQUIRED:** cerrar realm al final.

```dart
Future<Realm> openTestRealm() async {
  // English comments preferred.
  // Use a temporary directory path in tests.
  final config = Configuration.local([Todo.schema]);
  return Realm(config);
}
```

### 9.2 Repository tests antes que widget tests

* Prueba:

  * queries,
  * migraciones básicas,
  * operaciones CRUD,
  * reglas de integridad.

---

## 10) Anti-patrones (FORBIDDEN)

* **FORBIDDEN:** abrir/cerrar Realm repetidamente en widgets.
* **FORBIDDEN:** mutar objetos Realm fuera de una transacción.
* **FORBIDDEN:** pasar objetos Realm vivos por toda la app como si fueran DTOs.
* **FORBIDDEN:** queries duplicadas y sin encapsulación.
* **FORBIDDEN:** migraciones “a mano” sin `schemaVersion`.

---

## 11) Checklist rápido

* [ ] Una sola construcción de Realm y `close()` en dispose.
* [ ] Models con `PrimaryKey` cuando aplique.
* [ ] Escrituras siempre dentro de `write`.
* [ ] Queries encapsuladas (repositorio/providers).
* [ ] Estrategia clara para objetos vivos: freeze o DTO.
* [ ] Migraciones versionadas e idempotentes.
* [ ] Tests con Realm temporal y teardown.

