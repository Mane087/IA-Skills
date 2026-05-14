---
name: flutter_secure_storage
description: >
  Guide for using Flutter Secure Storage in a secure, testable, and maintainable
  way, including wrapper services, provider-based access, key management,
  session persistence, token lifecycle handling, and Riverpod integration.
  Trigger: Use when the task mentions or contains `FlutterSecureStorage`,
  secure local persistence of tokens or session data, wrapper storage services,
  provider-based storage access, key naming for sensitive values, logout/session
  cleanup, bootstrap or rehydration of auth state from secure storage, or when
  reviewing whether sensitive local storage is being used safely and outside the UI layer.
metadata:
  stack: dart
  framework: flutter
  area: security
---

# SKILL: Flutter Secure Storage (Project-agnostic)

Este documento define reglas generales y patrones recomendados para trabajar con
`flutter_secure_storage` de forma consistente, segura y mantenible, sin asumir arquitectura o dominio específico.

> **Meta**: que cualquier proyecto Flutter tenga un estándar mínimo de seguridad local: manejo correcto de tokens, claves protegidas, persistencia controlada y arquitectura testeable.

## 0) Setup base (REQUIRED)

### Instalación

```yaml
dependencies:
  flutter_secure_storage: ^latest
```

Después:

```bash
flutter pub get
```

### Configuración Android (REQUIRED)

- **REQUIRED**: `minSdkVersion >= 18`
- **REQUIRED**: desactivar backups de Google Drive para evitar errores con claves.

Esto es importante porque el plugin usa cifrado y almacenamiento seguro nativo del sistema.

## 1) Principios de diseño

### 1.1 Secure Storage NO es base de datos

Secure storage:
- Guarda datos sensibles.
- Persistencia pequeña.
- Key-value.

**NO REEMPLAZA SQLITE/ISAR/HIVE**

Se usa para:
- Tokens.
- Refresh tokens.
- Claves privadas.
- Secrets de sesión.

El almacenamiento está cifrado y usa mecanismos del sistema como Keychain o Keystore.

### 1.2 Guarda lo mínimo posible

**REQUIRED:** Solo datos críticos.

**FORBIDDEN:**
- Cache grande.
- Listas complejas.
- Datos que pueden pedirse al backend.

> Guardar menos reduce superficie de ataque.

### 1.3 No confiar ciegamente en el cliente

Secure storage protege datos, pero:
- Dispositivos rooteados pueden ser vulnerables.
- Nunca almacenes secretos permanentes críticos.
- El cliente nunca debe ser fuente de verdad de seguridad.

## 2) Arquitectura recomendada

### 2.1 Storage encapsulado en servicio (REQUIRED)

**FORBIDDEN**: Usar `FlutterSecureStorage()` directo en widgets.

**REQUIRED**: Wrapper o servicio.

```dart
class SecureStorageService {
  final FlutterSecureStorage storage;

  const SecureStorageService(this.storage);

  Future<void> write(String key, String value) =>
      storage.write(key: key, value: value);

  Future<String?> read(String key) =>
      storage.read(key: key);

  Future<void> delete(String key) =>
      storage.delete(key: key);
}
```

Esto permite:
- Testing.
- Overrides.
- Mockeo.
- Control de opciones.

### 2.2 Exponerlo vía provider (REQUIRED)

```dart
final secureStorageProvider = Provider<FlutterSecureStorage>((ref) {
  return const FlutterSecureStorage();
});

final secureStorageServiceProvider = Provider<SecureStorageService>((ref) {
  final storage = ref.watch(secureStorageProvider);
  return SecureStorageService(storage);
});
```

Arquitectura típica:
`UI → Controller → StorageService → FlutterSecureStorage`

## 3) Manejo de keys (REQUIRED)

### 3.1 Centralizar keys en enum o clase

**REQUIRED**: Evitar strings hardcodeados.

```dart
enum SecureKey {
  accessToken,
  refreshToken,
  userId,
}

extension SecureKeyX on SecureKey {
  String get value => name;
}
```

Esto evita typos y mejora refactorización.

### 3.2 Naming consistente

- `tokens` → `accessToken`
- `refresh` → `refreshToken`
- `flags` → `isLoggedIn`

Nunca uses nombres ambiguos como: `token1`, `data`, `value`.

## 4) Operaciones básicas

### 4.1 Write
```dart
await storage.write(
  key: SecureKey.accessToken.value,
  value: token,
);
```

### 4.2 Read
```dart
final token = await storage.read(
  key: SecureKey.accessToken.value,
);
```

### 4.3 Delete
```dart
await storage.delete(
  key: SecureKey.accessToken.value,
);
```

### 4.4 Delete All
```dart
await storage.deleteAll();
```
Usar solo en logout o reset total.

## 5) Reglas de seguridad

### 5.1 Nunca guardar datos sin cifrar antes

**FORBIDDEN:**
```dart
token = base64.encode(...)
storage.write(token)
```
Base64 NO es cifrado.

### 5.2 Rotación de tokens

**REQUIRED:**
- Actualizar tokens expirados.
- Borrar tokens inválidos.
- Limpiar storage en logout.

### 5.3 No persistir datos innecesarios

Si puedes pedirlo al backend: **NO lo guardes local.**

## 6) Ciclo de vida y rendimiento

### 6.1 Instancia única (REQUIRED)

No crees storage en cada lectura.
- ✔ Usar provider.
- ✔ Singleton controlado.

### 6.2 Evitar lecturas en build()

**FORBIDDEN:**
```dart
build() async {
  await storage.read(...)
}
```

Lecturas deben hacerse:
- En controller.
- En init.
- En bootstrap de sesión.

### 6.3 Rehidratación de sesión

Patrón típico:
```dart
Future<AuthState> loadSession() async {
  final token = await storage.read(key: 'accessToken');

  if (token == null) return AuthState.loggedOut();

  return AuthState.loggedIn(token);
}
```

## 7) Integración con Riverpod

### 7.1 TokenStorage abstraction (REQUIRED)

```dart
abstract class TokenStorage {
  Future<String?> readToken();
  Future<void> writeToken(String token);
  Future<void> deleteToken();
}

class SecureTokenStorage implements TokenStorage {
  final SecureStorageService storage;

  SecureTokenStorage(this.storage);

  @override
  Future<String?> readToken() =>
      storage.read('accessToken');

  @override
  Future<void> writeToken(String token) =>
      storage.write('accessToken', token);

  @override
  Future<void> deleteToken() =>
      storage.delete('accessToken');
}
```

### 7.2 Provider del storage abstracto

```dart
final tokenStorageProvider = Provider<TokenStorage>((ref) {
  final service = ref.watch(secureStorageServiceProvider);
  return SecureTokenStorage(service);
});
```

Esto permite:
- Tests sin plugin real.
- Web fallback.
- Mock storage.

## 8) Testing (REQUIRED)

### 8.1 Fake storage

```dart
class FakeTokenStorage implements TokenStorage {
  String? token;

  @override
  Future<String?> readToken() async => token;

  @override
  Future<void> writeToken(String t) async => token = t;

  @override
  Future<void> deleteToken() async => token = null;
}
```

### 8.2 Override provider

```dart
ProviderScope(
  overrides: [
    tokenStorageProvider.overrideWithValue(FakeTokenStorage()),
  ],
  child: MyApp(),
);
```

## 9) Anti-patrones (FORBIDDEN)

- Usar secure storage en widgets.
- Guardar listas JSON grandes.
- Usar base64 como “cifrado”.
- Persistir secrets permanentes.
- Crear instancias múltiples.
- Leer storage en build().

## 10) Checklist rápido

- [ ] Servicio wrapper implementado.
- [ ] Keys centralizadas.
- [ ] Storage expuesto por provider.
- [ ] Solo datos sensibles guardados.
- [ ] Limpieza en logout.
- [ ] Fake storage para tests.