---
name: auto_size_text
description: >
  Guide for using Flutter AutoSizeText in a consistent, responsive, and
  maintainable way, including sizing constraints, scaling rules,
  AutoSizeGroup usage, Theme integration, and testing considerations.
  Trigger: Use when the task involves `AutoSizeText`, `AutoSizeGroup`,
  responsive text sizing, preventing text overflow in constrained layouts,
  adapting typography for long or translated strings, or reviewing whether
  auto-resizing text is being used correctly without abusing it for layout
  logic or global styling decisions.
metadata:
  stack: dart
  framework: flutter
  area: library auto_size_text
---

# SKILL: Auto Size Text (Project-agnostic)

Este documento define **reglas generales** y patrones recomendados para trabajar con `auto_size_text` de forma consistente, mantenible y responsive, sin asumir arquitectura o dominio especifico.

> **Meta:** que cualquier proyecto Flutter tenga tipografia adaptable, accesible y predecible, sin hacks manuales ni calculos de tamano.

El widget `AutoSizeText` funciona igual que `Text`, pero ajusta automaticamente el tamano de fuente para que el contenido encaje dentro del espacio disponible.

---

## 0) Setup base (**REQUIRED**)

### Instalacion

```bash
flutter pub add auto_size_text
```

```yaml
dependencies:
  auto_size_text: ^latest
```

---

## 1) Principios de diseno

### 1.1 `AutoSizeText` reemplaza `Text` cuando el tamano es dinamico

Problema tipico:

```dart
Text(
  'Titulo largo',
  style: const TextStyle(fontSize: 20),
)
```

Esto falla cuando hay:

- pantallas pequenas
- textos largos
- idiomas con mas caracteres
- layouts responsivos

`AutoSizeText` resuelve esto escalando la fuente automaticamente para ajustarse al contenedor.

### 1.2 El widget necesita limites del layout

**REQUIRED:** `AutoSizeText` solo funciona si el widget tiene restricciones de tamano.

Ejemplos validos:

- `SizedBox`
- `Expanded`
- `Container` con `width`/`height`
- `Flexible`

Sin limites, no puede calcular el tamano correcto.

### 1.3 Siempre definir reglas de escala

**FORBIDDEN:** dejar el tamano totalmente libre.

Define al menos:

- `maxLines`
- `minFontSize`

Esto evita textos ilegibles o micro-fonts.

---

## 2) Uso basico

```dart
AutoSizeText(
  'Hello world',
  style: const TextStyle(fontSize: 20),
  maxLines: 2,
)
```

El widget inicia con el `fontSize` indicado y lo reduce hasta que el texto encaja en su contenedor.

---

## 3) Propiedades clave

### 3.1 `maxLines` (**REQUIRED**)

Controla cuantas lineas puede ocupar el texto.

```dart
AutoSizeText(
  'Very long text',
  maxLines: 2,
)
```

Si no se define, el ajuste depende solo del espacio disponible.

### 3.2 `minFontSize` y `maxFontSize`

Definen el rango de escala permitido.

```dart
AutoSizeText(
  'Title',
  style: const TextStyle(fontSize: 40),
  minFontSize: 18,
  maxLines: 2,
)
```

Por defecto, `minFontSize = 12`. Si no cabe ni en ese tamano, se aplica overflow.

### 3.3 `overflow`

```dart
AutoSizeText(
  'Very long text',
  maxLines: 1,
  overflow: TextOverflow.ellipsis,
)
```

Define como manejar texto que no cabe incluso despues de escalar.

### 3.4 `stepGranularity`

Define cuanto reduce el tamano en cada iteracion.

```dart
AutoSizeText(
  'Example',
  stepGranularity: 1,
)
```

- Valores bajos: mas precision, peor rendimiento.
- Valores altos: mejor performance, menos precision.

Generalmente no se recomienda usar valores menores a `1` por rendimiento.

### 3.5 `presetFontSizes`

Limita los tamanos posibles.

```dart
AutoSizeText(
  'Title',
  presetFontSizes: const [40, 24, 18],
)
```

Si se usa, ignora:

- `minFontSize`
- `maxFontSize`
- `stepGranularity`

### 3.6 `overflowReplacement`

Permite renderizar un widget alternativo si no cabe.

```dart
AutoSizeText(
  'Very long text',
  maxLines: 1,
  overflowReplacement: const Text('Too long'),
)
```

Sirve para evitar textos demasiado pequenos.

---

## 4) `AutoSizeGroup` (**REQUIRED** para layouts consistentes)

Sincroniza tamano entre multiples textos.

```dart
final group = AutoSizeGroup();

AutoSizeText('Title', group: group);
AutoSizeText('Subtitle', group: group);
```

Todos los textos del grupo adoptan el tamano del mas pequeno.

- **REQUIRED:** almacenar el grupo en estado persistente.
- **FORBIDDEN:** crear el grupo dentro de `build()`.

---

## 5) Uso con `RichText`

```dart
AutoSizeText.rich(
  TextSpan(
    text: 'Hello',
    children: [
      TextSpan(
        text: ' World',
        style: const TextStyle(fontWeight: FontWeight.bold),
      ),
    ],
  ),
)
```

Funciona igual que `Text.rich`.

---

## 6) Arquitectura recomendada

### 6.1 Usar `AutoSizeText` solo en UI

**FORBIDDEN:** usarlo para logica o calculos de layout.

Debe:

- renderizar texto adaptable.

No debe:

- decidir estilos globales.
- controlar comportamiento de negocio.

### 6.2 Integrarlo con `Theme`

```dart
AutoSizeText(
  'Title',
  style: Theme.of(context).textTheme.titleLarge,
)
```

Evita hardcodear tamanos.

### 6.3 Usarlo solo cuando haya riesgo real de overflow

No todo texto necesita auto scaling.

Usarlo en:

- titulos variables
- cards con ancho fijo
- botones con texto dinamico
- textos traducidos
- layouts responsivos

---

## 7) Rendimiento y buenas practicas

### 7.1 Evitar uso masivo en listas gigantes

Cada `AutoSizeText` puede medir multiples tamanos antes de decidir el final.

En listas largas puede impactar rendimiento.

### 7.2 Preferir tamanos iniciales razonables

Evita valores extremos como:

```dart
TextStyle(fontSize: 100)
```

y esperar que siempre lo ajuste de forma correcta.

### 7.3 Probar con idiomas largos

Casos recomendados:

- aleman
- ruso
- textos legales
- localizacion multilenguaje

---

## 8) Testing (**REQUIRED**)

### 8.1 Widget test basico

```dart
await tester.pumpWidget(
  const MaterialApp(
    home: AutoSizeText('Hello', maxLines: 1),
  ),
);
```

### 8.2 Test de layout

Verifica que no haya overflow visual y que el widget renderice:

```dart
expect(find.byType(AutoSizeText), findsOneWidget);
```

---

## 9) Naming conventions

Nombres sugeridos:

- `ResponsiveTitleText`
- `AdaptiveButtonText`
- `CardHeaderText`

Evitar:

- `text1`
- `label`
- `titleWidget`

---

## 10) Anti-patrones (**FORBIDDEN**)

- usar `AutoSizeText` sin limites de tamano
- usarlo dentro de widgets sin constraints
- usarlo para todos los textos
- crear `AutoSizeGroup` en `build()`
- usar `minFontSize` extremadamente bajo
- hardcodear tamanos fuera del `Theme`

---

## 11) Checklist rapido

- [ ] Solo usado donde haya riesgo de overflow
- [ ] Tiene `maxLines`
- [ ] Tiene rango de `fontSize`
- [ ] Tiene constraints de layout
- [ ] No se usa en listas gigantes
- [ ] Integrado con `Theme`
