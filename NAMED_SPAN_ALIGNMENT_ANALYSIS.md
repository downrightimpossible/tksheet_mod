# Named Span Alignment - Problemanalyse

## Zusammenfassung

Es wurden zwei Hauptprobleme mit Named Spans und Alignment identifiziert:

1. **Named Spans können nicht nachträglich mit `.align()` ausgerichtet werden**
2. **Vor Erstellung eines Named Spans gesetzte Alignments gehen bei span-updates verloren**

## Detaillierte Analyse

### Problem 1: Named Spans können nicht nachträglich aligned werden

#### Reproduktion:
```python
# Erstelle einen Named Span mit type_="highlight"
span = sheet.span("A1:B2", type_="highlight", name="my_span", bg="yellow")
sheet.named_span(span)

# Versuche, Alignment nachträglich hinzuzufügen
my_span = sheet.MT.named_spans["my_span"]
my_span.align("center")  # Funktioniert initial
```

#### Was passiert:
1. `my_span.align("center")` ruft `sheet.align(span, align="center")` auf (siehe `other_classes.py:289-290`)
2. Das Alignment wird auf die Zellen angewendet ✓
3. **ABER:** `span.kwargs` wird NICHT aktualisiert ✗
4. **ABER:** `span.type_` bleibt "highlight" (nicht "align") ✗

#### Problem:
Wenn später Zeilen/Spalten hinzugefügt/gelöscht werden, wird `create_options_from_span()` erneut aufgerufen:

```python
# In main_table.py:1680, 1922, 4640, 4703
self.PAR.create_options_from_span(span)
```

Diese Methode führt aus (siehe `sheet.py:970-975`):
```python
def create_options_from_span(self, span: Span, set_data: bool = True) -> Sheet:
    if span.type_ == "format":
        self.format(span, set_data=set_data, **span.kwargs)
    else:
        getattr(self, span.type_)(span, **span.kwargs)
    return self
```

Da `span.type_="highlight"` ist, wird aufgerufen:
```python
self.highlight(span, bg="yellow")  # span.kwargs = {"bg": "yellow"}
```

Das Alignment wird **nicht** erneut angewendet, weil:
- Es nicht in `span.kwargs` gespeichert ist
- Die Methode nur `highlight()` aufruft, nicht `align()`

#### Betroffene Code-Stellen:
- **sheet.py:970-975** - `create_options_from_span()` berücksichtigt nur `span.type_` und `span.kwargs`
- **other_classes.py:289-290** - `Span.align()` aktualisiert nicht `span.kwargs`
- **main_table.py:1680, 1922, 4640, 4703** - Re-Application nach Row/Column-Operationen

---

### Problem 2: Vor Erstellung gesetzte Alignments werden nicht übernommen

#### Reproduktion:
```python
# Setze Alignment BEVOR der Named Span erstellt wird
sheet.align("A1:B2", align="center")

# Erstelle einen Named Span für denselben Bereich
span = sheet.span("A1:B2", type_="highlight", name="my_span", bg="yellow")
sheet.named_span(span)

# Später: Füge Zeilen hinzu
sheet.insert_rows(0, 2)
```

#### Was passiert:
1. Das Alignment wird initial auf A1:B2 angewendet ✓
2. Der Named Span wird mit `type_="highlight"` erstellt
3. `create_options_from_span()` ruft `highlight(span, bg="yellow")` auf
4. Das vorherige Alignment bleibt initial erhalten ✓
5. **ABER:** Wenn Zeilen/Spalten hinzugefügt werden:
   - Die Alignment-Optionen werden für die neue Position verschoben
   - Der Named Span wird neu angewendet über `create_options_from_span()`
   - **ABER:** Der Span "weiß" nichts vom ursprünglichen Alignment
   - Das Alignment ist nicht Teil der Named Span-Definition

#### Problem:
Das ursprüngliche Alignment ist nicht Teil des Named Spans und wird bei span-updates nicht berücksichtigt.

---

## Architekturproblem: Single-Type Spans

Named Spans können nur **einen** Typ haben (siehe `constants.py:17-25`):
```python
named_span_types: set[str] = {
    "format",
    "highlight",
    "dropdown",
    "checkbox",
    "readonly",
    "align",
    "note",
}
```

Ein Named Span kann **nicht** gleichzeitig "highlight" UND "align" sein.

### Konsequenzen:
- Ein Highlight-Span kann kein persistentes Alignment haben
- Ein Align-Span kann kein Highlight haben
- Optionen können nicht kombiniert werden

---

## Lösungsansätze

### Ansatz 1: Multi-Type Named Spans (Breaking Change)
**Idee:** Named Spans können mehrere Typen gleichzeitig haben

**Änderungen:**
- `span.type_` wird zu `span.types_` (Liste oder Set)
- `create_options_from_span()` wendet alle Typen an
- `span.kwargs` wird in typ-spezifische Untergruppen aufgeteilt

**Vorteile:**
- Vollständige Flexibilität
- Alignment kann mit anderen Optionen kombiniert werden

**Nachteile:**
- Breaking Change für bestehenden Code
- Komplexere Implementierung
- Migration erforderlich

---

### Ansatz 2: Implicit Options Storage (Empfohlen)
**Idee:** Named Spans speichern ALLE angewendeten Optionen, nicht nur die vom Haupttyp

**Änderungen:**
1. Wenn `.align()` auf einem Named Span aufgerufen wird, aktualisiere `span.kwargs`:
   ```python
   # In sheet.py:align()
   if isinstance(span, Span) and span.name and span.name in self.MT.named_spans:
       span.kwargs["align"] = align
   ```

2. `create_options_from_span()` wendet zusätzliche Optionen an:
   ```python
   def create_options_from_span(self, span: Span, set_data: bool = True) -> Sheet:
       # Haupttyp anwenden
       if span.type_ == "format":
           self.format(span, set_data=set_data, **span.kwargs)
       else:
           getattr(self, span.type_)(span, **span.kwargs)

       # Zusätzliche Optionen anwenden (falls vorhanden und nicht Haupttyp)
       if span.type_ != "align" and "align" in span.kwargs:
           self.align(span, align=span.kwargs["align"], redraw=False)

       return self
   ```

**Vorteile:**
- Kein Breaking Change
- Alignment wird persistent
- Einfache Implementierung
- Abwärtskompatibel

**Nachteile:**
- Implizites Verhalten (Optionen außerhalb des Haupttyps)
- `span.kwargs` kann Optionen verschiedener Typen enthalten

---

### Ansatz 3: Options Capture on Creation
**Idee:** Beim Erstellen eines Named Spans werden alle existierenden Optionen erfasst

**Änderungen:**
```python
def named_span(self, span: Span) -> Span:
    # ... existing validation ...

    # Erfasse existierende Optionen für den Span-Bereich
    rows, cols = self.ranges_from_span(span)

    # Erfasse Alignment (falls vorhanden und nicht im Span definiert)
    if span.type_ != "align" and "align" not in span.kwargs:
        # Prüfe ob Zellen Alignment haben
        for r in rows:
            for c in cols:
                align = self.MT.get_cell_kwargs(r, c, key="align")
                if align:
                    span.kwargs["align"] = align
                    break

    self.MT.named_spans[span.name] = span
    self.create_options_from_span(span)
    return span
```

**Vorteile:**
- Vorhandene Optionen werden übernommen
- Intuitives Verhalten

**Nachteile:**
- Nur beim Erstellen, nicht bei nachträglichen Änderungen
- Komplexität bei gemischten Alignments im Bereich

---

## Empfohlene Lösung: Kombination von Ansatz 2 + 3

1. **Bei Span-Erstellung:** Erfasse vorhandenes Alignment (Ansatz 3)
2. **Bei nachträglichen Änderungen:** Update `span.kwargs` (Ansatz 2)
3. **Bei Re-Application:** Wende alle in `span.kwargs` gespeicherten Optionen an (Ansatz 2)

### Implementierung:

#### 1. Update `sheet.py:align()` - Persist alignment in span.kwargs
```python
def align(
    self,
    *key: CreateSpanTypes,
    align: str | None = None,
    redraw: bool = True,
) -> Span:
    span = self.span_from_key(*key)
    rows, cols = self.ranges_from_span(span)
    table, index, header = span.table, span.index, span.header
    align = convert_align(align)

    # ... existing code to apply alignment ...

    # NEW: If this is a named span, persist the alignment
    if span.name and span.name in self.MT.named_spans:
        span.kwargs["align"] = align

    self.set_refresh_timer(redraw)
    return span
```

#### 2. Update `sheet.py:create_options_from_span()` - Apply all options
```python
def create_options_from_span(self, span: Span, set_data: bool = True) -> Sheet:
    # Apply primary type
    if span.type_ == "format":
        self.format(span, set_data=set_data, **span.kwargs)
    else:
        getattr(self, span.type_)(span, **span.kwargs)

    # Apply additional options that are not the primary type
    additional_options = {
        "align": lambda: self.align(span, align=span.kwargs["align"], redraw=False),
        "highlight": lambda: self.highlight(span,
                                            bg=span.kwargs.get("bg", False),
                                            fg=span.kwargs.get("fg", False),
                                            redraw=False),
        "readonly": lambda: self.readonly(span, readonly=span.kwargs.get("readonly", True)),
    }

    for opt_type, apply_func in additional_options.items():
        if span.type_ != opt_type and opt_type in span.kwargs:
            apply_func()

    return self
```

#### 3. Update `sheet.py:named_span()` - Capture existing alignment
```python
def named_span(self, span: Span) -> Span:
    if span.name in self.MT.named_spans:
        raise ValueError(f"Span '{span.name}' already exists.")
    if not span.name:
        raise ValueError("Span must have a name.")
    if span.type_ not in named_span_types:
        raise ValueError(f"Span 'type_' must be one of the following: {', '.join(named_span_types)}.")

    # NEW: Capture existing alignment if not already specified
    if span.type_ != "align" and "align" not in span.kwargs:
        rows, cols = self.ranges_from_span(span)
        if rows and cols and span.table:
            # Check first cell for alignment
            first_align = self.MT.get_cell_kwargs(rows[0], cols[0], key="align")
            if first_align:
                span.kwargs["align"] = first_align

    self.MT.named_spans[span.name] = span
    self.create_options_from_span(span)
    return span
```

---

## Betroffene Dateien

- **tksheet/sheet.py:970-975** - `create_options_from_span()` (Hauptänderung)
- **tksheet/sheet.py:2796-2829** - `align()` (kwargs update)
- **tksheet/sheet.py:959-968** - `named_span()` (capture existing options)
- **tksheet/other_classes.py:289-290** - `Span.align()` (optional: update für Konsistenz)

---

## Test-Cases

### Test 1: Nachträgliches Alignment auf Named Span
```python
span = sheet.span("A1:B2", type_="highlight", name="test", bg="yellow")
sheet.named_span(span)
sheet.MT.named_spans["test"].align("center")

# Nach Row-Insert sollte Alignment erhalten bleiben
sheet.insert_rows(0, 1)
assert sheet.MT.get_cell_kwargs(1, 0, key="align") == "n"  # center
```

### Test 2: Vorhandenes Alignment wird übernommen
```python
sheet.align("A1:B2", align="right")
span = sheet.span("A1:B2", type_="highlight", name="test", bg="yellow")
sheet.named_span(span)

# Nach Row-Insert sollte Alignment erhalten bleiben
sheet.insert_rows(0, 1)
assert sheet.MT.get_cell_kwargs(1, 0, key="align") == "e"  # right
```

### Test 3: Align-Typ Named Span
```python
span = sheet.span("A1:B2", type_="align", name="test", align="center")
sheet.named_span(span)

# Nach Column-Insert sollte Alignment erhalten bleiben
sheet.insert_columns(0, 1)
assert sheet.MT.get_cell_kwargs(0, 1, key="align") == "n"  # center
```
