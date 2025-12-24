# Plan: edugo-worker - Fase 2 Integraciones

**Proyecto:** edugo-worker
**Rama base:** dev
**Rama feature:** `feature/PT-009-worker-fase2-integraciones`
**Duracion estimada:** 3-4 semanas

**PRERREQUISITO:** Fase 1 (PT-008) completada

---

## Fase 2.1: Cliente de Email

### Paso 2.1.1: Definir interfaz

**Archivo:** `internal/domain/service/email_service.go`

```go
package service

import "context"

type EmailService interface {
    SendEmail(ctx context.Context, req SendEmailRequest) error
    SendTemplatedEmail(ctx context.Context, req SendTemplatedEmailRequest) error
}

type SendEmailRequest struct {
    To      []string
    Subject string
    Body    string
    IsHTML  bool
}

type SendTemplatedEmailRequest struct {
    To           []string
    TemplateID   string
    TemplateData map[string]interface{}
}
```

### Paso 2.1.2: Implementar con SendGrid

**Archivo:** `internal/infrastructure/email/sendgrid_client.go`

```go
package email

import (
    "context"

    "github.com/sendgrid/sendgrid-go"
    "github.com/sendgrid/sendgrid-go/helpers/mail"
)

type sendGridClient struct {
    client    *sendgrid.Client
    fromEmail string
    fromName  string
    logger    logger.Logger
}

func NewSendGridClient(apiKey, fromEmail, fromName string, logger logger.Logger) service.EmailService {
    return &sendGridClient{
        client:    sendgrid.NewSendClient(apiKey),
        fromEmail: fromEmail,
        fromName:  fromName,
        logger:    logger,
    }
}

func (c *sendGridClient) SendEmail(ctx context.Context, req service.SendEmailRequest) error {
    from := mail.NewEmail(c.fromName, c.fromEmail)

    for _, toAddr := range req.To {
        to := mail.NewEmail("", toAddr)
        message := mail.NewSingleEmail(from, req.Subject, to, req.Body, req.Body)

        _, err := c.client.Send(message)
        if err != nil {
            c.logger.Error("failed to send email", "to", toAddr, "error", err)
            return err
        }
    }

    return nil
}

func (c *sendGridClient) SendTemplatedEmail(ctx context.Context, req service.SendTemplatedEmailRequest) error {
    // Implementar con templates de SendGrid
    return nil
}
```

### Paso 2.1.3: Templates de email

**Archivo:** `templates/emails/welcome_student.html`

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Bienvenido a EduGo</title>
</head>
<body>
    <h1>Hola {{.StudentName}}!</h1>
    <p>Has sido inscrito en <strong>{{.UnitName}}</strong>.</p>
    <p>Tu docente es {{.TeacherName}}.</p>
    <p>
        <a href="{{.AppURL}}">Ingresa a la app</a> para comenzar tu aprendizaje.
    </p>
</body>
</html>
```

**Archivo:** `templates/emails/low_score_alert.html`

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Alerta de Evaluacion</title>
</head>
<body>
    <h1>Alerta: Estudiante con puntaje bajo</h1>
    <p><strong>Estudiante:</strong> {{.StudentName}}</p>
    <p><strong>Material:</strong> {{.MaterialTitle}}</p>
    <p><strong>Puntaje:</strong> {{.Score}}%</p>
    <p><strong>Intentos:</strong> {{.AttemptNumber}}</p>
    <p>Te recomendamos contactar al estudiante para ofrecer apoyo adicional.</p>
</body>
</html>
```

---

## Fase 2.2: Push Notifications

### Paso 2.2.1: Definir interfaz

**Archivo:** `internal/domain/service/push_service.go`

```go
package service

import "context"

type PushService interface {
    SendToUser(ctx context.Context, userID string, notification Notification) error
    SendToUsers(ctx context.Context, userIDs []string, notification Notification) error
    SendToTopic(ctx context.Context, topic string, notification Notification) error
}

type Notification struct {
    Title    string
    Body     string
    Data     map[string]string
    ImageURL string
}
```

### Paso 2.2.2: Implementar con Firebase

**Archivo:** `internal/infrastructure/push/firebase_client.go`

```go
package push

import (
    "context"

    firebase "firebase.google.com/go/v4"
    "firebase.google.com/go/v4/messaging"
)

type firebaseClient struct {
    client *messaging.Client
    logger logger.Logger
}

func NewFirebaseClient(ctx context.Context, credentialsFile string, logger logger.Logger) (service.PushService, error) {
    app, err := firebase.NewApp(ctx, nil, option.WithCredentialsFile(credentialsFile))
    if err != nil {
        return nil, err
    }

    client, err := app.Messaging(ctx)
    if err != nil {
        return nil, err
    }

    return &firebaseClient{
        client: client,
        logger: logger,
    }, nil
}

func (c *firebaseClient) SendToUser(ctx context.Context, userID string, notification service.Notification) error {
    // Obtener token del usuario desde BD
    // Enviar notificacion
    return nil
}

func (c *firebaseClient) SendToUsers(ctx context.Context, userIDs []string, notification service.Notification) error {
    // Enviar a multiples usuarios
    return nil
}

func (c *firebaseClient) SendToTopic(ctx context.Context, topic string, notification service.Notification) error {
    message := &messaging.Message{
        Notification: &messaging.Notification{
            Title:    notification.Title,
            Body:     notification.Body,
            ImageURL: notification.ImageURL,
        },
        Data:  notification.Data,
        Topic: topic,
    }

    _, err := c.client.Send(ctx, message)
    return err
}
```

---

## Fase 2.3: AssessmentAttemptProcessor

### Paso 2.3.1: Implementar processor

**Archivo:** `internal/application/processor/assessment_attempt_processor.go`

```go
package processor

type AssessmentAttemptProcessor struct {
    attemptRepo     repository.AssessmentAttemptRepository
    materialRepo    repository.MaterialRepository
    userRepo        repository.UserRepository
    membershipRepo  repository.MembershipRepository
    emailService    service.EmailService
    pushService     service.PushService
    logger          logger.Logger
}

func (p *AssessmentAttemptProcessor) Process(ctx context.Context, event events.AssessmentAttemptCompleted) error {
    p.logger.Info("processing assessment attempt", "attempt_id", event.AttemptID)

    // 1. Obtener datos del intento
    attempt, err := p.attemptRepo.FindByID(ctx, event.AttemptID)
    if err != nil {
        return err
    }

    // 2. Calcular porcentaje
    scorePercent := float64(attempt.Score) / float64(attempt.MaxScore) * 100

    // 3. Si score < 60%, notificar al docente
    if scorePercent < 60 {
        if err := p.notifyTeacher(ctx, attempt, scorePercent); err != nil {
            p.logger.Error("failed to notify teacher", "error", err)
            // No retornar error, es notificacion secundaria
        }
    }

    // 4. Actualizar estadisticas del material
    if err := p.updateMaterialStats(ctx, attempt.MaterialID); err != nil {
        p.logger.Error("failed to update material stats", "error", err)
    }

    // 5. Registrar evento de analytics
    if err := p.recordAnalyticsEvent(ctx, attempt, scorePercent); err != nil {
        p.logger.Error("failed to record analytics", "error", err)
    }

    p.logger.Info("assessment attempt processed",
        "attempt_id", event.AttemptID,
        "score_percent", scorePercent,
    )

    return nil
}

func (p *AssessmentAttemptProcessor) notifyTeacher(ctx context.Context, attempt *entities.AssessmentAttempt, scorePercent float64) error {
    // 1. Obtener material
    material, err := p.materialRepo.FindByID(ctx, attempt.MaterialID)
    if err != nil {
        return err
    }

    // 2. Obtener docente de la unidad
    teachers, err := p.membershipRepo.FindByUnitAndRole(ctx, material.UnitID, "teacher")
    if err != nil {
        return err
    }

    // 3. Obtener datos del estudiante
    student, err := p.userRepo.FindByID(ctx, attempt.UserID)
    if err != nil {
        return err
    }

    // 4. Enviar email
    for _, teacher := range teachers {
        teacherUser, _ := p.userRepo.FindByID(ctx, teacher.UserID)

        err := p.emailService.SendTemplatedEmail(ctx, service.SendTemplatedEmailRequest{
            To:         []string{teacherUser.Email},
            TemplateID: "low_score_alert",
            TemplateData: map[string]interface{}{
                "StudentName":   student.FullName,
                "MaterialTitle": material.Title,
                "Score":         scorePercent,
                "AttemptNumber": attempt.AttemptNumber,
            },
        })
        if err != nil {
            p.logger.Error("failed to send email to teacher", "teacher", teacherUser.Email, "error", err)
        }
    }

    // 5. Enviar push notification
    for _, teacher := range teachers {
        err := p.pushService.SendToUser(ctx, teacher.UserID.String(), service.Notification{
            Title: "Alerta de evaluacion",
            Body:  fmt.Sprintf("%s obtuvo %.0f%% en %s", student.FullName, scorePercent, material.Title),
            Data: map[string]string{
                "type":       "low_score_alert",
                "attemptId":  attempt.ID.String(),
                "materialId": material.ID.String(),
            },
        })
        if err != nil {
            p.logger.Error("failed to send push to teacher", "error", err)
        }
    }

    return nil
}

func (p *AssessmentAttemptProcessor) updateMaterialStats(ctx context.Context, materialID uuid.UUID) error {
    // Calcular promedio de scores, total de intentos, etc.
    return nil
}

func (p *AssessmentAttemptProcessor) recordAnalyticsEvent(ctx context.Context, attempt *entities.AssessmentAttempt, scorePercent float64) error {
    // Registrar en tabla de analytics
    return nil
}
```

---

## Fase 2.4: StudentEnrolledProcessor

### Paso 2.4.1: Implementar processor

**Archivo:** `internal/application/processor/student_enrolled_processor.go`

```go
package processor

type StudentEnrolledProcessor struct {
    userRepo        repository.UserRepository
    unitRepo        repository.AcademicUnitRepository
    membershipRepo  repository.MembershipRepository
    progressRepo    repository.ProgressRepository
    emailService    service.EmailService
    pushService     service.PushService
    logger          logger.Logger
}

func (p *StudentEnrolledProcessor) Process(ctx context.Context, event events.StudentEnrolled) error {
    p.logger.Info("processing student enrollment", "student_id", event.StudentID, "unit_id", event.UnitID)

    // 1. Obtener datos del estudiante
    student, err := p.userRepo.FindByID(ctx, event.StudentID)
    if err != nil {
        return err
    }

    // 2. Obtener datos de la unidad
    unit, err := p.unitRepo.FindByID(ctx, event.UnitID)
    if err != nil {
        return err
    }

    // 3. Obtener docente principal
    teachers, err := p.membershipRepo.FindByUnitAndRole(ctx, event.UnitID, "teacher")
    if err != nil {
        return err
    }

    var teacherName string
    if len(teachers) > 0 {
        teacher, _ := p.userRepo.FindByID(ctx, teachers[0].UserID)
        teacherName = teacher.FullName
    }

    // 4. Enviar email de bienvenida
    err = p.emailService.SendTemplatedEmail(ctx, service.SendTemplatedEmailRequest{
        To:         []string{student.Email},
        TemplateID: "welcome_student",
        TemplateData: map[string]interface{}{
            "StudentName": student.FullName,
            "UnitName":    unit.Name,
            "TeacherName": teacherName,
            "AppURL":      "https://app.edugo.com",
        },
    })
    if err != nil {
        p.logger.Error("failed to send welcome email", "error", err)
    }

    // 5. Inicializar progreso del estudiante
    if err := p.initializeProgress(ctx, event.StudentID, event.UnitID); err != nil {
        p.logger.Error("failed to initialize progress", "error", err)
    }

    // 6. Notificar al docente
    if len(teachers) > 0 {
        err = p.pushService.SendToUser(ctx, teachers[0].UserID.String(), service.Notification{
            Title: "Nuevo estudiante inscrito",
            Body:  fmt.Sprintf("%s se ha unido a %s", student.FullName, unit.Name),
            Data: map[string]string{
                "type":      "new_student",
                "studentId": event.StudentID.String(),
                "unitId":    event.UnitID.String(),
            },
        })
        if err != nil {
            p.logger.Error("failed to notify teacher", "error", err)
        }
    }

    p.logger.Info("student enrollment processed", "student_id", event.StudentID)

    return nil
}

func (p *StudentEnrolledProcessor) initializeProgress(ctx context.Context, studentID, unitID uuid.UUID) error {
    // Crear registro de progreso inicial
    progress := &entities.Progress{
        UserID:    studentID,
        UnitID:    unitID,
        Status:    "not_started",
        Progress:  0,
        CreatedAt: time.Now(),
        UpdatedAt: time.Now(),
    }

    return p.progressRepo.Create(ctx, progress)
}
```

---

## Fase 2.5: Tests

### Paso 2.5.1: Tests de processors

```go
func TestAssessmentAttemptProcessor_LowScore(t *testing.T) {
    // Test notificacion cuando score < 60%
}

func TestStudentEnrolledProcessor_WelcomeEmail(t *testing.T) {
    // Test envio de email de bienvenida
}
```

---

## Fase 2.6: Commit y PR

### Paso 2.6.1: Commits

```bash
git commit -m "feat(email): implementar cliente de email con SendGrid

- Agregar EmailService interface
- Implementar SendGrid client
- Agregar templates de email

Ticket: PT-009"

git commit -m "feat(push): implementar push notifications con Firebase

- Agregar PushService interface
- Implementar Firebase Cloud Messaging client
- Soporte para notificaciones a usuarios y topics

Ticket: PT-009"

git commit -m "feat(processor): implementar AssessmentAttemptProcessor

- Notificar docente si score < 60%
- Actualizar estadisticas de material
- Registrar evento de analytics

Ticket: PT-009"

git commit -m "feat(processor): implementar StudentEnrolledProcessor

- Enviar email de bienvenida
- Inicializar progreso del estudiante
- Notificar al docente

Ticket: PT-009"
```

---

## Checklist Final

- [ ] EmailService implementado (SendGrid)
- [ ] PushService implementado (Firebase)
- [ ] Templates de email creados
- [ ] AssessmentAttemptProcessor implementado
- [ ] StudentEnrolledProcessor implementado
- [ ] Analytics events registrados
- [ ] Tests de integracion
- [ ] Configuracion de ambiente
- [ ] Build exitoso
- [ ] Lint limpio
- [ ] PR a dev creado
- [ ] PR a dev mergeado

---

## Variables de Entorno Fase 2

```bash
# SendGrid
SENDGRID_API_KEY=SG.xxx
EMAIL_FROM=noreply@edugo.com
EMAIL_FROM_NAME=EduGo

# Firebase
FIREBASE_CREDENTIALS_FILE=/path/to/firebase-credentials.json
```

---

## Notas de Produccion

### Rate Limiting

- SendGrid: 100 emails/segundo (plan Pro)
- Firebase: 500 mensajes/segundo

### Retry Policy

- Emails: 3 reintentos con backoff exponencial
- Push: 2 reintentos, luego log y continuar

### Monitoring

- Metricas de emails enviados/fallidos
- Metricas de push notifications
- Alertas en errores criticos

---

**FIN DEL PLAN DE TRABAJO**
