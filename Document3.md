
# 📄 **Backend Developer Guide for PushPilot (Markdown Code)**


# Backend Developer Guide for PushPilot

## 1. Backend Philosophy & Project Structure

### Philosophy
- **12-factor app**: environment-driven configuration, stateless server.
- **Thin views, fat services**: views only orchestrate; business logic lives in services.
- **Multi-tenant isolation by default**: filter all data by workspace.
- **Async where required**: use Celery for heavy/IO tasks.
- **Everything logged & auditable**.

### Project Structure


pushpilot_backend/

config/

apps/

accounts/

workspaces/

integrations/

firebase/

ai/

campaigns/

notifications/

analytics/

common/

requirements.txt

Dockerfile

docker-compose.yml



---

## 2. Environment Configuration & Secrets

### Environments
- local  
- staging  
- production  

### Secrets stored in AWS/GCP Secret Manager
- DB URL  
- Redis URL  
- AI API key  
- Firebase server key  
- JWT signing keys  

Expose via environment variables.

---

## 3. Multi-Tenancy & Workspace Context

### Workspace Middleware

python
class WorkspaceMiddleware(MiddlewareMixin):
    def process_request(self, request):
        if not request.user.is_authenticated:
            request.workspace = None
            return

        workspace_id = request.headers.get("X-Workspace-ID") or request.GET.get("workspace_id")
        if not workspace_id:
            request.workspace = None
            return

        try:
            ws = Workspace.objects.get(id=workspace_id)
        except Workspace.DoesNotExist:
            request.workspace = None
            return

        is_member = WorkspaceMember.objects.filter(
            user=request.user, workspace=ws, status="active"
        ).exists()

        if not is_member:
            request.workspace = None
            return

        request.workspace = ws

        

Use in views:

python
def get_queryset(self):
    return Model.objects.filter(workspace=self.request.workspace)


---

## 4. Authentication, JWT, OAuth, Permissions

### JWT (SimpleJWT)

* Access token (short TTL)
* Refresh token (long TTL)
* Stored in frontend memory or cookies

### OAuth (Google/Apple)

Frontend sends provider `id_token`, backend verifies and returns JWT.

### Permissions

```python
class IsWorkspaceMember(BasePermission):
    def has_permission(self, request, view):
        return bool(
            request.user and request.user.is_authenticated and request.workspace
        )
```

---

## 5. API Design Conventions

### Endpoint Patterns

* List: `GET /campaigns`
* Detail: `GET /campaigns/{id}`
* Create: `POST /campaigns`
* Update: `PATCH /campaigns/{id}`
* Delete: `DELETE /campaigns/{id}`
* Actions: `POST /campaigns/{id}/schedule`

### Query Params vs Body

* **Query:** filtering / search
* **Body:** create/update JSON payloads

### Pagination

Use DRF pagination: `page`, `page_size`.

### Error Format

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid data.",
    "details": { "field": ["error"] }
  }
}
```

Custom handler:

```python
REST_FRAMEWORK = {
    "EXCEPTION_HANDLER": "apps.common.exceptions.custom_exception_handler"
}
```

---

## 6. Data Modeling, Indexes & Constraints

Use UUID PKs everywhere.

### Example Indexing

```python
class ScheduledJob(TimeStampedModel):
    class Meta:
        indexes = [
            models.Index(fields=["workspace", "run_at", "status"])
        ]
```

Soft delete with:

```python
is_deleted = models.BooleanField(default=False)
```

---

## 7. Business Logic & Algorithms

### 7.1 User Onboarding

1. User registers
2. Create default workspace
3. Add user as workspace owner
4. Seed example templates (optional)

---

### 7.2 Firebase Integration

Request:

```json
{
  "project_id": "my-app",
  "server_key": "AAAA..."
}
```

Backend:

* Validate
* Encrypt key
* Store in `FirebaseConfig`

Test message:

```python
client.send_broadcast("Test", "Firebase connected!")
```

---

### 7.3 AI Content Generation

```python
def generate_notification_content(prompt, tone):
    # 1. Build AI prompt
    # 2. Call provider
    # 3. Validate response (title, subtitle)
    return title, subtitle
```

Retry on:

* timeouts
* 5xx

---

### 7.4 Campaign State Machine

States:

`draft → scheduled → running → completed`

or

`draft → scheduled → cancelled`

Validation:

```python
VALID_TRANSITIONS = {
    "draft": ["scheduled"],
    "scheduled": ["running", "cancelled"],
    "running": ["completed"],
}
```

---

### 7.5 Scheduling Algorithm

When user schedules:

```python
ScheduledJob.objects.create(
    workspace=campaign.workspace,
    campaign=campaign,
    run_at=scheduled_at,
    status="pending"
)
```

Celery Beat (every minute):

```python
jobs = ScheduledJob.objects.select_for_update(
    skip_locked=True
).filter(
    run_at__lte=timezone.now(), status="pending"
)[:100]

for job in jobs:
    run_campaign_job.delay(job.id)
```

Recurring campaigns:

* Parse RRULE
* Compute next run
* Create new job

---

### 7.6 Notification Sending Algorithm

Inside `run_campaign_job`:

1. Mark job → running
2. For each content:
   * Create Notification row
   * Send via Firebase topic

```python
message_id = client.send_broadcast(title, subtitle)
```

3. Mark Notification → sent
4. Mark job → completed
5. Mark campaign → completed

Retry:

```python
raise self.retry(exc=exc, countdown=60)
```

---

### 7.7 Analytics

Tables:

* Notification
* AnalyticsEvent

Query example:

```python
Campaign.objects.filter(workspace=ws).annotate(
    sent_count=Count(
        "notifications",
        filter=Q(notifications__status="sent")
    )
)
```

---

## 8. Celery Architecture

### Components:

* Celery worker
* Celery beat
* Django API server

### Queue Routing:

```python
CELERY_TASK_ROUTES = {
    "apps.notifications.tasks.run_campaign_job": {"queue": "notifications"}
}
```

---

## 9. Observability & Logging

### JSON Structured Logs:

Fields:

* timestamp
* request_id
* workspace_id
* user_id
* route
* metadata

### Metrics:

* notifications_sent_total
* notifications_failed_total
* campaign_created_total

### Error Tracking:

* Sentry
* Cloud Error Reporting

---

## 10. Security & Rate Limiting

### Policies:

* Workspace membership required
* Encrypt Firebase keys
* JWT secrets in secure vault

### Rate Limit:

```python
DEFAULT_THROTTLE_RATES = {
    "user": "1000/day",
    "ai": "60/min"
}
```

---

## 11. Performance & Scalability

### Patterns:

* Always use `select_related`, `prefetch_related`
* Redis caching for dashboards
* Use FCM topics (not individual device sends)
* Split Celery queues:
  * high priority → notifications
  * default → other tasks

---

## 12. Testing Strategy

### Unit Tests:

* Campaign state machine
* AI content service (mocked AI)
* Firebase client (mock send)
* Scheduling logic

### Integration Tests:

* DRF API tests
* Celery eager mode

### E2E:

* create workspace
* connect Firebase
* create + schedule campaign
* run tasks

---

## 13. Local Dev & CI/CD

### Local Environment (docker-compose)

```
docker-compose up --build
```

Includes:

* Django
* Celery worker
* Celery beat
* Redis
* Postgres

### CI/CD Pipeline

1. Lint (flake8, black, isort)
2. Test suite
3. Build Docker image
4. Deploy to staging
5. Smoke tests
6. Promote to production

---

# End of Backend Developer Guide for PushPilot

```

---

If you want, I can also generate:

📌 A **Front-End Developer Guide**  
📌 A **DevOps Deployment Guide**  
📌 A **Complete API Reference (OpenAPI/Swagger YAML)**  
📌 A **single combined documentation `.md` file** with all sections  

Just tell me!
```
