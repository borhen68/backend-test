setup.md — Frametale Microservices Architecture (v1)
🧠 Platform Summary
Frametale is a personalized print product platform where users can design photobooks, calendars, and cards using an interactive canvas. The backend is microservices-based, written in NestJS, uses MongoDB, and communicates via REST APIs.

🧱 Microservices Overview
Each service is deployed independently via Docker and exposes its own Swagger documentation. Communication is via REST using internal service URLs and auth tokens.

1. Auth Service
Language/Framework: NestJS + JWT

DB: users collection in MongoDB

Responsibilities:

Handle user registration/login

Manage user sessions and roles

Key Endpoints:

POST /auth/signup

POST /auth/login

GET /auth/me (JWT-protected)

Notes:

Use @nestjs/passport + @nestjs/jwt

Shared User DTO across services via shared lib package

2. Project Service
DB: projects collection

Responsibilities:

Manage user-created projects (photobooks, calendars, cards)

Store canvas state: layouts, text, image positions

Key Models:

ts
Copy
Edit
type: 'PHOTBOOK' | 'CALENDAR' | 'CARD';
pages: LayoutPage[];
title: string;
createdBy: ObjectId;
createdAt: Date;
updatedAt: Date;
Endpoints:

POST /projects – create a new project

GET /projects/:id

PATCH /projects/:id

DELETE /projects/:id

3. Product Catalog Service
DB: products, suppliers

Responsibilities:

Define base product types and sizes

Handle dynamic product metadata (binding, cover type)

Map POD supplier data

Key Models:

ts
Copy
Edit
Product {
  name: string;
  type: 'PHOTBOOK' | 'CARD' | 'CALENDAR';
  basePrice: number;
  sizes: ['A5', 'A4', 'Letter'];
  templates: TemplateRef[];
  supplierId: ObjectId;
}

Supplier {
  name: string;
  productionTimeDays: number;
  location: string;
}
Endpoints:

GET /products

GET /products/:id

POST /products

GET /suppliers

4. Pricing Service
DB: Stateless or optional pricing_rules config

Responsibilities:

Calculate user-facing price

Apply markup, dynamic pricing logic, page count, etc.

Pricing Logic:

Example: base price = €5 → sell for €20

+€1 per extra page after 20

Endpoints:

GET /pricing/estimate?productId=xxx&pages=25

Response:

json
Copy
Edit
{
  "basePrice": 5,
  "markup": 15,
  "extraPages": 5,
  "extraCost": 5,
  "finalPrice": 25
}
5. Template Service
DB: templates in MongoDB OR S3 + metadata index

Responsibilities:

Serve pre-designed templates (manual InDesign exports or Figma)

Organize templates by product type and size

Model:

ts
Copy
Edit
Template {
  _id: ObjectId;
  name: string;
  type: 'PHOTBOOK' | 'CALENDAR';
  size: 'A5' | 'A4' | 'Letter';
  pages: TemplatePage[];
}
Endpoints:

GET /templates?type=PHOTBOOK&size=A5

GET /templates/:id

Storage: JSON format stored in MongoDB or S3 (with file key)

6. Media Service
DB: media collection

Responsibilities:

Upload, store, and retrieve images

Extract metadata (orientation, EXIF, etc.)

Storage: S3 / Cloudinary + DB pointer

Model:

ts
Copy
Edit
Media {
  _id: ObjectId;
  userId: ObjectId;
  projectId: ObjectId;
  url: string;
  metadata: { orientation: string, size: number, dateTaken: string }
}
Endpoints:

POST /media/upload

GET /media/:id

7. File Generation Service
Responsibilities:

Render final project into PDF/ZIP for print

Merge layout + user media + template assets

Tools: Puppeteer / headless Chrome / PDFKit

Endpoints:

POST /render/:projectId → generates and stores file

GET /render/:projectId/download → returns ZIP/PDF

📦 Supporting Services
8. GDPR & Compliance Service
Responsibilities:

Manage user data rights (deletion, export)

Track consent for marketing and data processing

Models:

ts
Copy
Edit
GdprRequest { userId, type: 'DELETE' | 'EXPORT', status }
Consent { userId, consentType: 'marketing' | 'usage', accepted: boolean }
Endpoints:

POST /gdpr/request-delete

GET /gdpr/data-export

PATCH /gdpr/consent

9. Notification Service
Responsibilities:

Send transactional emails (order created, export ready)

Tools: Resend / SendGrid

Endpoints:

POST /notifications/email

10. Logging & Audit Trail
Responsibilities:

Store internal service logs (e.g. project edits)

Enable admin-level debugging

DB: logs collection or log shipper (e.g. Loki, Elastic)

11. Analytics Service
Responsibilities:

Track key events: project created, template used

Tools: PostHog, Segment (optional)

Events:

project.created

media.uploaded

template.applied

🧪 Local Dev Setup
Pre-reqs
Docker + Docker Compose

Node.js 18+

MongoDB (local or Docker)

1. Start All Services
bash
Copy
Edit
docker-compose up --build
2. Health Check Example
bash
Copy
Edit
GET http://localhost:3001/healthz → 200 OK
3. Swagger Endpoints
Auth: http://localhost:3001/api

Projects: http://localhost:3002/api

Media: http://localhost:3003/api

Pricing: http://localhost:3004/api

4. Mongo Shell Access
bash
Copy
Edit
docker exec -it frametale-db mongosh
5. Environment Config
.env example:

ini
Copy
Edit
JWT_SECRET=your_secret
MONGO_URI=mongodb://frametale-db:27017
MEDIA_BUCKET=s3://frametale-media
✅ TODO & Future
Real-time collaboration via WebSockets (Canvas editing)

Integration with POD supplier API for status syncing

Global CDN for faster media/template deliver