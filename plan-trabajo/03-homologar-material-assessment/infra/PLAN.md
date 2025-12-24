# Plan: edugo-infrastructure - Homologar material_assessment

**Proyecto:** edugo-infrastructure
**Rama base:** dev
**Rama feature:** `feature/PT-003-homologar-material-assessment`

---

## Fase 1: Preparacion

### Paso 1.1: Crear rama

```bash
cd /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure
git fetch origin
git checkout dev
git pull origin dev
git checkout -b feature/PT-003-homologar-material-assessment
```

### Paso 1.2: Verificar archivos a eliminar

```bash
# Verificar que existen los archivos
ls -la mongodb/migrations/structure/001_material_assessment.go
ls -la mongodb/migrations/constraints/001_material_assessment_indexes.go
```

---

## Fase 2: Eliminar Migracion Duplicada

### Paso 2.1: Eliminar migracion de estructura

```bash
rm -f mongodb/migrations/structure/001_material_assessment.go
```

### Paso 2.2: Eliminar migracion de indexes

```bash
rm -f mongodb/migrations/constraints/001_material_assessment_indexes.go
```

### Paso 2.3: Eliminar seeds duplicados (si existen)

```bash
rm -f mongodb/seeds/material_assessment.js
rm -f seeds/mongodb/material_assessment.js
```

---

## Fase 3: Crear Script de Migracion de Datos

### Paso 3.1: Crear script de migracion

Crear archivo: `mongodb/scripts/migrate_material_assessment.js`

```javascript
// Script: migrate_material_assessment.js
// Description: Migrar datos de material_assessment a material_assessment_worker
// Author: [Tu nombre]
// Date: 2025-12-XX
// Ticket: PT-003

// IMPORTANTE: Ejecutar backup antes de esta operacion
// mongodump --db edugo_db --collection material_assessment --out /backup/

print("=== Migracion material_assessment -> material_assessment_worker ===");
print("Fecha: " + new Date().toISOString());

// 1. Contar documentos actuales
var oldCount = db.material_assessment.count();
var workerCount = db.material_assessment_worker.count();

print("Documentos en material_assessment: " + oldCount);
print("Documentos en material_assessment_worker: " + workerCount);

if (oldCount === 0) {
    print("No hay documentos para migrar. Saliendo.");
    quit();
}

// 2. Helper para calcular puntos totales
function calculateTotalPoints(questions) {
    if (!questions) return 0;
    return questions.reduce(function(sum, q) {
        return sum + (q.points || 1);
    }, 0);
}

// 3. Migrar documentos
var migrated = 0;
var skipped = 0;
var errors = 0;

db.material_assessment.find().forEach(function(oldDoc) {
    try {
        // Verificar si ya existe en worker
        var exists = db.material_assessment_worker.findOne({
            material_id: oldDoc.material_id
        });

        if (exists) {
            print("  SKIP: " + oldDoc.material_id + " (ya existe en worker)");
            skipped++;
            return;
        }

        // Crear nuevo documento con schema worker
        var newDoc = {
            material_id: oldDoc.material_id,
            questions: oldDoc.questions || [],
            total_questions: oldDoc.questions ? oldDoc.questions.length : 0,
            total_points: calculateTotalPoints(oldDoc.questions),
            version: oldDoc.version || 1,
            ai_model: (oldDoc.metadata && oldDoc.metadata.generated_by) ?
                      oldDoc.metadata.generated_by : "migrated",
            processing_time_ms: 0, // No disponible en schema antiguo
            metadata: oldDoc.metadata || {},
            created_at: oldDoc.created_at || new Date(),
            updated_at: new Date()
        };

        // Insertar en coleccion worker
        db.material_assessment_worker.insertOne(newDoc);
        print("  OK: " + oldDoc.material_id);
        migrated++;

    } catch (e) {
        print("  ERROR: " + oldDoc.material_id + " - " + e.message);
        errors++;
    }
});

// 4. Resumen
print("");
print("=== Resumen de Migracion ===");
print("Migrados: " + migrated);
print("Omitidos (ya existian): " + skipped);
print("Errores: " + errors);
print("Total procesados: " + (migrated + skipped + errors));

// 5. Verificacion
var newWorkerCount = db.material_assessment_worker.count();
print("");
print("Conteo final material_assessment_worker: " + newWorkerCount);

if (errors === 0 && migrated + skipped === oldCount) {
    print("");
    print("EXITO: Migracion completada correctamente");
    print("");
    print("SIGUIENTE PASO: Despues de verificar, ejecutar:");
    print("  db.material_assessment.drop()");
} else {
    print("");
    print("ADVERTENCIA: Revisar errores antes de eliminar coleccion antigua");
}
```

### Paso 3.2: Crear script de verificacion

Crear archivo: `mongodb/scripts/verify_material_assessment_migration.js`

```javascript
// Script: verify_material_assessment_migration.js
// Description: Verificar que la migracion fue exitosa
// Ticket: PT-003

print("=== Verificacion de Migracion ===");

// 1. Conteo
var oldCount = db.material_assessment.count();
var newCount = db.material_assessment_worker.count();

print("material_assessment: " + oldCount);
print("material_assessment_worker: " + newCount);

// 2. Verificar que todos los material_ids fueron migrados
var missing = [];
db.material_assessment.find({}, {material_id: 1}).forEach(function(doc) {
    var exists = db.material_assessment_worker.findOne({material_id: doc.material_id});
    if (!exists) {
        missing.push(doc.material_id);
    }
});

if (missing.length > 0) {
    print("");
    print("ERROR: Los siguientes material_ids no fueron migrados:");
    missing.forEach(function(id) {
        print("  - " + id);
    });
} else {
    print("");
    print("OK: Todos los documentos fueron migrados correctamente");
}

// 3. Verificar schema de documentos migrados
print("");
print("=== Sample de documento migrado ===");
printjson(db.material_assessment_worker.findOne({ai_model: "migrated"}));
```

---

## Fase 4: Compilar y Test

### Paso 4.1: Compilar

```bash
cd /Users/jhoanmedina/source/EduGo/repos-separados/edugo-infrastructure
go build ./...
```

### Paso 4.2: Ejecutar tests

```bash
go test ./... -v
go test ./... -cover
```

### Paso 4.3: Lint

```bash
golangci-lint run ./...
```

---

## Fase 5: Documentar

### Paso 5.1: Actualizar CHANGELOG.md

```markdown
## [0.12.1] - 2025-12-XX

### Removed
- Migracion duplicada `material_assessment` (001)
- Indexes duplicados para `material_assessment`

### Changed
- `material_assessment_worker` es ahora la unica coleccion de assessments
- Script de migracion de datos agregado

### Migration
- Ejecutar `mongodb/scripts/migrate_material_assessment.js` antes de eliminar coleccion antigua
```

### Paso 5.2: Commit

```bash
git add .
git commit -m "feat(mongodb): homologar colecciones material_assessment

- Eliminar migracion duplicada 001_material_assessment.go
- Eliminar indexes duplicados
- Agregar script de migracion de datos
- Agregar script de verificacion

material_assessment_worker es ahora la coleccion canonica

Ticket: PT-003"
```

---

## Fase 6: Pull Request a Dev

### Paso 6.1: Push rama

```bash
git push -u origin feature/PT-003-homologar-material-assessment
```

### Paso 6.2: Crear PR

**Titulo:** `feat(mongodb): Homologar colecciones material_assessment`

**Descripcion:**

```markdown
## Descripcion

Unifica las colecciones `material_assessment` y `material_assessment_worker` usando `material_assessment_worker` como version canonica.

## Justificacion

- `material_assessment_worker` tiene schema mas completo
- Incluye metadata de IA (ai_model, processing_time_ms)
- Validaciones estrictas
- Worker es la fuente de verdad

## Cambios

- [x] Eliminada migracion `001_material_assessment.go`
- [x] Eliminados indexes correspondientes
- [x] Script de migracion de datos creado
- [x] Script de verificacion creado

## Checklist

- [x] `go build ./...` exitoso
- [x] `go test ./...` exitoso
- [x] `golangci-lint run` limpio

## Migracion de Datos

ANTES de eliminar `material_assessment`:

```bash
# 1. Backup
mongodump --db edugo_db --collection material_assessment --out /backup/

# 2. Migrar datos
mongo edugo_db mongodb/scripts/migrate_material_assessment.js

# 3. Verificar
mongo edugo_db mongodb/scripts/verify_material_assessment_migration.js

# 4. Eliminar coleccion antigua (solo si verificacion OK)
mongo edugo_db --eval "db.material_assessment.drop()"
```
```

### Paso 6.3: Review y merge

---

## Fase 7: Release

### Paso 7.1: PR de dev a main

### Paso 7.2: Merge a main

### Paso 7.3: Crear GitHub Release

```bash
git checkout main
git pull origin main
git tag mongodb/v0.12.1
git push origin mongodb/v0.12.1
```

**Titulo:** `edugo-infrastructure/mongodb v0.12.1`

---

## Checklist Final

- [ ] Rama feature creada
- [ ] Migracion duplicada eliminada
- [ ] Indexes eliminados
- [ ] Scripts de migracion creados
- [ ] Build exitoso
- [ ] Tests exitosos
- [ ] Lint limpio
- [ ] PR a dev mergeado
- [ ] PR a main mergeado
- [ ] Release publicado

---

**Siguiente paso:** [../worker/PLAN.md](../worker/PLAN.md) - Verificar worker
