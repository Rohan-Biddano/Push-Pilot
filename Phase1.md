# Phase 1 — Requirements & Scope Document

**Thursday, 4 December 2025**
**11:07 AM**

## 📘 Phase 1 — Requirements & Scope Document for PushPilot

*(Backend: Django/Python — Cloud: AWS/GCP)*

---

## 1️⃣ Product Overview

PushPilot is a SaaS platform that enables developers and companies to create, manage, and automate multi-channel notification campaigns.
It uses AI to generate push messages and integrates with Firebase Cloud Messaging (FCM) for real-time delivery.

---

## 2️⃣ Goals & Objectives

### **Primary Goals**

- Allow businesses to generate notifications automatically using AI prompts.
- Provide campaign creation, scheduling, and management in one dashboard.
- Enable developers to integrate their apps through Firebase.
- Track campaign performance and notification analytics.
- Support multi-channel delivery: App push, Web push, SMS, Email, WhatsApp (future).

### **Secondary Goals**

- Provide ready-to-use templates.
- Offer tone/style formatting.
- Support dark/light themes.
- Make the platform scalable and secure.

---

## 3️⃣ Scope of Work (Backend + Platform)

### **In-Scope**

- User authentication & authorization
- Workspace + business profiles
- AI content generation endpoints
- Campaign Builder APIs
- Notification scheduling engine
- Firebase integration module
- Campaign analytics
- Template library
- Multi-channel delivery module (Phase 2)

### **Out-of-Scope (For Now)**

- Built-in SMTP or SMS gateways
- Multi-tenant billing module
- AI fine-tuning or model hosting
  **These can be added later.**

---

## 4️⃣ Target Users

- Developers integrating notifications in their apps
- Companies with existing mobile/web apps
- Product growth teams
- CRM and marketing teams

---

## 5️⃣ Functional Requirements

### **A. User Management**

- Email/password signup
- Google login (OAuth2)
- Apple login (OAuth2)
- Forgot password
- Session token management (JWT recommended)
- Role-based access (Owner, Admin, Member)

### **B. Workspace & Configuration**

- Create workspace
- Add business details
- Connect Firebase project
- Save Project ID + Server Key (encrypted)
- Manage connected application

### **C. Campaign Builder**

- AI prompt → notification generation
- Title (4–6 words)
- Subtitle (10–15 words)
- Tone selection
- Image upload
- Save campaign draft
- Edit / regenerate / duplicate campaign
- Set schedule
- Preview notification

### **D. Notifications Engine**

- Store notification payload
- Queue jobs for scheduled sends
- Integrate with FCM to deliver push notifications
- Support retry logic
- Track delivery status

### **E. Template Library**

Predefined categories:

- Festive
- Weekend
- Health/Pharma (Glycomet, Pan D example)
- Promotions
- Ability to copy/use template

### **F. Analytics**

- Total notifications sent
- Daily schedule view
- Next notification timer
- Campaign progress (sent %, success rate)
- Active / expired campaigns

---

## 6️⃣ Non-Functional Requirements

### **Performance**

- Handle up to 10,000 notifications/min
- Low latency for AI prompt responses

### **Scalability**

- Horizontal scaling using AWS/GCP
- Background job worker for scheduling

### **Security**

- JWT auth for APIs
- Encrypt Firebase server key
- Database encryption at rest
- HTTPS only

### **Reliability**

- Auto restart workers
- Logging + monitoring
- Automated retries for failed pushes

---

## 7️⃣ Technology Choices

### **Backend**

- Python + Django (REST framework)
- Django DRF for APIs
- Celery/Redis for scheduling jobs
- Firebase Admin SDK for push delivery

### **AI**

- OpenAI / Claude / Llama API

### **Database**

- PostgreSQL (recommended)
- Redis (for background tasks + caching)

### **Cloud**

- **AWS/GCP**
  - Compute: EC2 / Cloud Run
  - DB: RDS / Cloud SQL
  - Storage: S3 / GCS
  - Monitoring: CloudWatch / Stackdriver

---

## 8️⃣ API Requirements (Summary)

*(Full API documentation will be created in Phase 2)*

### **Authentication APIs**

- /auth/register
- /auth/login
- /auth/logout
- /auth/google
- /auth/apple

### **Workspace APIs**

- /workspace/create
- /workspace/settings
- /workspace/firebase-connect

### **Campaign APIs**

- /campaign/create
- /campaign/generate-content (AI)
- /campaign/update
- /campaign/list
- /campaign/delete

### **Notification APIs**

- /notification/send-now
- /notification/schedule
- /notification/status

### **Analytics APIs**

- /analytics/summary
- /analytics/campaign/{id}

---

## 9️⃣ Database Design (High-Level)

*(Detailed ER diagram will be part of Phase 2)*

### **Core Tables**

- users
- workspaces
- workspace_members
- firebase_configs
- campaigns
- notifications
- scheduled_jobs
- templates
- analytics_logs

---

## 🔟 Constraints & Assumptions

### **Constraints**

- Firebase is mandatory for push delivery
- AI calls require an active API key
- System must handle India-centric festive notifications (Diwali, etc.)
- Must comply with GDPR-style privacy standards

### **Assumptions**

- User already has an app (iOS/Android/Web)
- User can integrate FCM SDK
- User has technical ability to create Firebase credentials

---
