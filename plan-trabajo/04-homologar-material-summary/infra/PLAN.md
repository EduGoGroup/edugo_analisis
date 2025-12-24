# Plan: edugo-infrastructure - Verificar material_summary

**Proyecto:** edugo-infrastructure
**Estado:** Solo verificacion y creacion de script de migracion

---

## Fase 1: Verificacion

### Paso 1.1: Confirmar que migracion existe

```bash
cd /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure

# Verificar que existe la migracion
cat mongodb/migrations/structure/007_material_summary.go
```

### Paso 1.2: Verificar entity

```bash
# Verificar campos de la entidad
cat mongodb/entities/material_summary.go
```

**Campos esperados:**
- material_id
- summary
- key_points
- language
- word_count
- version
- ai_model
- processing_time_ms
- metadata
- created_at
- updated_at

---

## Fase 2: Crear Script de Migracion (Opcional)

Si no existe ya, crear script para migrar datos.

### Paso 2.1: Crear script de migracion

Crear archivo: `mongodb/scripts/migrate_material_summaries.js`

```javascript
// Script: migrate_material_summaries.js
// Description: Migrar datos de material_summaries (plural) a material_summary (singular)
// Date: 2025-12-XX
// Ticket: PT-004

print("=== Migracion material_summaries -> material_summary ===");
print("Fecha: " + new Date().toISOString());

// 1. Verificar colecciones
var oldExists = db.getCollectionNames().indexOf("material_summaries") !== -1;
var newExists = db.getCollectionNames().indexOf("material_summary") !== -1;

print("material_summaries existe: " + oldExists);
print("material_summary existe: " + newExists);

if (!oldExists) {
    print("No hay coleccion material_summaries para migrar. Saliendo.");
    quit();
}

// 2. Contar documentos
var oldCount = db.material_summaries.count();
var newCount = newExists ? db.material_summary.count() : 0;

print("Documentos en material_summaries: " + oldCount);
print("Documentos en material_summary: " + newCount);

// 3. Helpers
function generateSummaryFromMainIdeas(mainIdeas) {
    if (!mainIdeas || mainIdeas.length === 0) return "";
    return mainIdeas.join(". ") + ".";
}

function calculateWordCount(mainIdeas) {
    if (!mainIdeas) return 0;
    var text = mainIdeas.join(" ");
    return text.split(/\s+/).length;
}

function detectLanguage(mainIdeas) {
    if (!mainIdeas || mainIdeas.length === 0) return "es";
    var text = mainIdeas.join(" ").toLowerCase();
    // Palabras comunes en espanol
    if (text.match(/\b(el|la|los|las|de|del|y|que|en)\b/)) return "es";
    // Palabras comunes en ingles
    if (text.match(/\b(the|and|of|to|in|is|that)\b/)) return "en";
    return "es"; // Default
}

// 4. Migrar documentos
var migrated = 0;
var skipped = 0;
var errors = 0;

db.material_summaries.find().forEach(function(oldDoc) {
    try {
        // Verificar si ya existe
        var exists = db.material_summary.findOne({material_id: oldDoc.material_id});
        if (exists) {
            print("  SKIP: " + oldDoc.material_id + " (ya existe)");
            skipped++;
            return;
        }

        // Transformar a nuevo schema
        var newDoc = {
            material_id: oldDoc.material_id,
            summary: generateSummaryFromMainIdeas(oldDoc.main_ideas),
            key_points: oldDoc.main_ideas || [],
            language: detectLanguage(oldDoc.main_ideas),
            word_count: calculateWordCount(oldDoc.main_ideas),
            version: 1,
            ai_model: "migrated",
            processing_time_ms: 0,
            metadata: {
                migrated_from: "material_summaries",
                original_sections: oldDoc.sections || [],
                original_glossary: oldDoc.glossary || [],
                original_key_concepts: oldDoc.key_concepts || []
            },
            created_at: oldDoc.created_at || new Date(),
            updated_at: new Date()
        };

        db.material_summary.insertOne(newDoc);
        print("  OK: " + oldDoc.material_id);
        migrated++;

    } catch (e) {
        print("  ERROR: " + oldDoc.material_id + " - " + e.message);
        errors++;
    }
});

// 5. Resumen
print("");
print("=== Resumen de Migracion ===");
print("Migrados: " + migrated);
print("Omitidos: " + skipped);
print("Errores: " + errors);

var finalCount = db.material_summary.count();
print("Conteo final material_summary: " + finalCount);

if (errors === 0 && migrated + skipped === oldCount) {
    print("");
    print("EXITO: Migracion completada");
    print("SIGUIENTE: Despues de verificar, ejecutar:");
    print("  db.material_summaries.drop()");
}
```

### Paso 2.2: Commit (si se creo script)

```bash
git add mongodb/scripts/migrate_material_summaries.js
git commit -m "chore(mongodb): agregar script migracion material_summaries

Script para migrar datos de material_summaries (plural) a
material_summary (singular) con transformacion de schema.

Ticket: PT-004"
```

---

## Sin Cambios de Codigo Requeridos

La migracion `007_material_summary.go` ya existe y tiene el schema correcto. Solo se requiere:

1. Crear script de migracion de datos (opcional, si no existe)
2. Ejecutar migracion de datos en ambientes

---

**Siguiente paso:** [../api-mobile/PLAN.md](../api-mobile/PLAN.md)
