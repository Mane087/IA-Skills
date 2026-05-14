---
name: flutter_form_builder
description: >
  Guide for building Flutter forms with flutter_form_builder and
  form_builder_validators in a consistent, maintainable, and testable way,
  including FormBuilder structure, field selection, declarative validation,
  form state handling, and Riverpod-friendly architecture.
  Trigger: Use when the task mentions or contains `FormBuilder`,
  `FormBuilderState`, `FormBuilderTextField`, `FormBuilderDropdown`,
  `FormBuilderValidators`, `saveAndValidate`, `didChange`, a centralized
  `GlobalKey<FormBuilderState>`, or when designing, validating, submitting,
  resetting, or refactoring Flutter forms built with flutter_form_builder.
metadata:
  stack: dart
  framework: flutter
  area: forms
---
# SKILL: Flutter Form Builder + Form Builder Validators (Project-agnostic)

Este documento define **reglas generales** y patrones recomendados para trabajar con:

- `flutter_form_builder`
- `form_builder_validators`

de forma consistente, mantenible y escalable, sin asumir arquitectura o dominio específico.

> **Meta:** que cualquier proyecto Flutter tenga formularios predecibles, validados, desacoplados del UI y fáciles de testear.

Estos paquetes permiten construir formularios complejos reduciendo boilerplate, reutilizando validaciones y facilitando la captura de datos del usuario.

---

# 0) Setup base (REQUIRED)

## Instalación

```bash
flutter pub add flutter_form_builder form_builder_validators
```

```yaml
dependencies:
  flutter_form_builder: ^latest
  form_builder_validators: ^latest
```

# 1) Principios de diseño

### 1.1 FormBuilder reemplaza boilerplate del Form clásico

- **Flutter base:** `Form` + `TextFormField` + controllers + validators manuales.
- **Con FormBuilder:** `FormBuilder` + campos tipados + validadores reutilizables.

El paquete elimina código repetitivo, facilita validación y permite recolectar valores del formulario fácilmente.

### 1.2 El formulario produce un Map tipado por nombres

Cada campo tiene un identificador único:

```dart
name: 'email'
```

Los valores se obtienen con:

```dart
_formKey.currentState!.value // Retorna Map<String, dynamic>
```

### 1.3 Validación declarativa

Regla clave: **UI define reglas, Controller decide acciones.**

La validación debe ser:

- **Explícita**
- **Reusable**
- **Aislada del widget**

# 2) Arquitectura recomendada

### 2.1 FormKey centralizado (REQUIRED)

```dart
final GlobalKey<FormBuilderState> formKey = GlobalKey<FormBuilderState>();
```

_Nunca uses múltiples keys para el mismo formulario._

### 2.2 El FormBuilder es contenedor del formulario

```dart
FormBuilder(
  key: formKey,
  child: Column(
    children: [
      // Campos aquí
    ],
  ),
)
```

El widget `FormBuilder` actúa como contenedor y controlador del estado del formulario.

### 2.3 Separar UI de lógica

- **REQUIRED:** `Widget` → `FormBuilder` → `Controller` → `UseCase`
- **FORBIDDEN:** Validaciones complejas en widgets.

# 3) Campos disponibles

FormBuilder incluye múltiples widgets listos para distintos tipos de entrada:

- `FormBuilderTextField`
- `FormBuilderDropdown`
- `FormBuilderCheckbox`
- `FormBuilderDateTimePicker`
- `FormBuilderSlider`
- `FormBuilderSwitch`
- `FormBuilderRadioGroup`
- `FormBuilderFilterChip`

# 4) Ejemplo base de formulario

```dart
final _formKey = GlobalKey<FormBuilderState>();

FormBuilder(
  key: _formKey,
  child: Column(
    children: [
      FormBuilderTextField(
        name: 'email',
        decoration: const InputDecoration(labelText: 'Email'),
        validator: FormBuilderValidators.compose([
          FormBuilderValidators.required(),
          FormBuilderValidators.email(),
        ]),
      ),
      FormBuilderDropdown<String>(
        name: 'role',
        items: ['Admin', 'User']
            .map((role) => DropdownMenuItem(value: role, child: Text(role)))
            .toList(),
      ),
      ElevatedButton(
        onPressed: () {
          if (_formKey.currentState!.saveAndValidate()) {
            final data = _formKey.currentState!.value;
            print(data);
          }
        },
        child: const Text('Submit'),
      )
    ],
  ),
);
```

# 5) Reglas de validación

### 5.1 Usar FormBuilderValidators (REQUIRED)

```dart
validator: FormBuilderValidators.compose([
  FormBuilderValidators.required(),
  FormBuilderValidators.email(),
]),
```

### 5.2 Validaciones compuestas

Permite encadenar múltiples reglas de validación de forma legible.

### 5.3 Validación condicional

No se recomienda cambiar el validador dinámicamente en el constructor. Es mejor manejar la lógica dentro de la función:

```dart
validator: (value) {
  if (someCondition) {
    return FormBuilderValidators.required()(value);
  }
  return null;
}
```

# 6) Manipulación del formulario

### 6.1 Guardar y validar

```dart
formKey.currentState!.saveAndValidate();
```

### 6.2 Obtener valores

```dart
final values = formKey.currentState!.value;
```

### 6.3 Reset

```dart
formKey.currentState!.reset();
```

### 6.4 Setear valor manualmente

```dart
formKey.currentState!.fields['email']!.didChange('new@mail.com');
```

# 7) Integración con Riverpod

### 7.1 Controller del formulario (REQUIRED)

```dart
class LoginController {
  Future<void> submit(Map<String, dynamic> data) async {
    final email = data['email'];
    final password = data['password'];
    // Lógica de negocio
  }
}
```

El widget solo envía datos: `controller.submit(formData)`.

### 7.2 Provider del controller

```dart
final loginControllerProvider = Provider<LoginController>((ref) {
  return LoginController();
});
```

# 8) Testing (REQUIRED)

### 8.1 Test del controller sin UI

```dart
test('submit parses data', () async {
  final controller = LoginController();
  await controller.submit({
    'email': 'test@mail.com',
    'password': '123456'
  });
  // expect(...)
});
```

### 8.2 Test widget con form

```dart
await tester.enterText(find.byType(FormBuilderTextField), 'test@mail.com');
await tester.tap(find.text('Submit'));
```

# 9) Naming conventions (REQUIRED)

- **Campos:** `emailField`, `passwordField`, `userRoleField`.
- **Keys:** `loginFormKey`, `registerFormKey`.
- **Providers:** `loginControllerProvider`.

# 10) Anti-patrones (FORBIDDEN)

- [ ] Usar controllers manuales (`TextEditingController`) para cada campo.
- [ ] Guardar estado del form en múltiples sitios.
- [ ] Validaciones inline gigantes.
- [ ] Lógica de negocio en el widget.
- [ ] Usar nombres de campo (`name`) inconsistentes.
- [ ] No usar validators reutilizables.

# 11) Checklist rápido

- [ ] `FormKey` único y tipado.
- [ ] `FormBuilder` como contenedor principal.
- [ ] Todos los campos tienen un `name` descriptivo.
- [ ] Validación mediante `FormBuilderValidators`.
- [ ] UI desacoplada del controller (el form envía un `Map`).
- [ ] Reset y save centralizados.
- [ ] Tests unitarios para el controller.
