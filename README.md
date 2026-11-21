#!/usr/bin/env bash
set -e

ROOT="lost-device-reports"
FRONTEND="$ROOT/frontend"
BACKEND="$ROOT/backend"

mkdir -p "$ROOT" "$FRONTEND" "$BACKEND"

echo "Creating project in: $ROOT"

# -------------------------
# docker-compose.yml
# -------------------------
cat > "$ROOT/docker-compose.yml" <<'YAML'
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: lost_reports
    ports: ["5432:5432"]
    volumes: [pgdata:/var/lib/postgresql/data]
volumes:
  pgdata:
YAML

# -------------------------
# .gitignore
# -------------------------
cat > "$ROOT/.gitignore" <<'GIT'
node_modules
.env
.env.local
.next
backend/uploads/*
!backend/uploads/.gitkeep
GIT

# -------------------------
# README
# -------------------------
cat > "$ROOT/README.md" <<'README'
# نظام إدارة بلاغات الأجهزة (Lost/Found Reports)

تنبيه داخل التطبيق: "للتعاون مع الجهات الرسمية فقط"
النظام لإدارة البلاغات فقط ولا يوفر تتبّع مباشر.

## تشغيل محلي
1) تشغيل PostgreSQL:
   docker compose up -d

2) Backend:
   cd backend
   cp .env.example .env
   npm install
   npx prisma migrate dev --name init
   npm run seed
   npm run dev

3) Frontend:
   cd ../frontend
   cp .env.local.example .env.local
   npm install
   npm run dev

الحسابات الافتراضية (بكلمة مرور Password123!):
- user@example.com  (USER)
- staff@example.com (STAFF)
- admin@example.com (ADMIN)
README
GITIGNORE

# -------------------------
# Backend files
# -------------------------
cat > "$BACKEND/package.json" <<'JSON'
{
  "name": "lost-device-backend",
  "version": "1.0.0",
  "main": "src/server.js",
  "scripts": {
    "dev": "nodemon src/server.js",
    "start": "node src/server.js",
    "migrate": "prisma migrate dev",
    "seed": "node prisma/seed.js"
  },
  "prisma": { "seed": "node prisma/seed.js" },
  "dependencies": {
    "@prisma/client": "^5.20.0",
    "bcryptjs": "^2.4.3",
    "cors": "^2.8.5",
    "dotenv": "^16.4.5",
    "express": "^4.19.2",
    "express-async-errors": "^3.1.1",
    "express-rate-limit": "^7.4.0",
    "helmet": "^7.1.0",
    "jsonwebtoken": "^9.0.2",
    "multer": "^1.4.5-lts.1",
    "zod": "^3.23.8"
  },
  "devDependencies": {
    "nodemon": "^3.1.4",
    "prisma": "^5.20.0"
  }
}
JSON

cat > "$BACKEND/.env.example" <<'ENV'
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/lost_reports?schema=public"
JWT_SECRET="change-me-to-a-long-random-string"
JWT_EXPIRES_IN="8h"
PORT=4000
FRONTEND_ORIGIN="http://localhost:3000"
UPLOAD_DIR="./uploads"
MAX_UPLOAD_MB=5
ENV

mkdir -p "$BACKEND/uploads" && touch "$BACKEND/uploads/.gitkeep"

mkdir -p "$BACKEND/prisma" "$BACKEND/src/middleware" "$BACKEND/src/routes" "$BACKEND/src/utils"

cat > "$BACKEND/prisma/schema.prisma" <<'PRISMA'
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

enum Role {
  USER
  STAFF
  ADMIN
}

enum ReportStatus {
  NEW
  IN_REVIEW
  CLOSED
}

model User {
  id           String   @id @default(uuid())
  email        String   @unique
  passwordHash String
  name         String
  role         Role     @default(USER)
  createdAt    DateTime @default(now())

  reports          Report[]
  auditLogs        AuditLog[]
  statusChanges    ReportStatusHistory[]
}

model Report {
  id              String       @id @default(uuid())
  reporterId      String
  phoneNumber     String
  phoneNormalized String
  deviceModel     String?
  lastSeenAt      DateTime?
  description     String?
  status          ReportStatus @default(NEW)
  createdAt       DateTime     @default(now())
  updatedAt       DateTime     @updatedAt

  reporter       User               @relation(fields: [reporterId], references: [id])
  location       ReportLocation?
  attachments    Attachment[]
  statusHistory  ReportStatusHistory[]
  @@index([phoneNormalized])
  @@index([status])
  @@index([createdAt])
}

model ReportLocation {
  id     String @id @default(uuid())
  reportId String @unique
  lat    Float
  lng    Float
  address String
  geocodeComponents Json?

  report Report @relation(fields: [reportId], references: [id], onDelete: Cascade)
  @@index([lat, lng])
}

model Attachment {
  id          String @id @default(uuid())
  reportId    String
  fileName    String
  storagePath String
  mimeType    String
  sizeBytes   Int
  createdAt   DateTime @default(now())

  report Report @relation(fields: [reportId], references: [id], onDelete: Cascade)
}

model ReportStatusHistory {
  id          String @id @default(uuid())
  reportId    String
  fromStatus  ReportStatus?
  toStatus    ReportStatus
  changedById String
  note        String?
  changedAt   DateTime @default(now())

  report    Report @relation(fields: [reportId], references: [id], onDelete: Cascade)
  changedBy User   @relation(fields: [changedById], references: [id])
}

model AuditLog {
  id        String @id @default(uuid())
  actorId   String
  action    String
  entity    String
  entityId  String
  changes   Json?
  ip        String?
  createdAt DateTime @default(now())

  actor User @relation(fields: [actorId], references: [id])

  @@index([entity, entityId])
  @@index([createdAt])
}
PRISMA

cat > "$BACKEND/prisma/seed.js" <<'SEED'
require('dotenv').config();
const bcrypt = require('bcryptjs');
const { PrismaClient } = require('@prisma/client');
const prisma = new PrismaClient();

async function main() {
  const pw = await bcrypt.hash('Password123!', 12);

  const admin = await prisma.user.upsert({
    where: { email: 'admin@example.com' },
    update: {},
    create: { email: 'admin@example.com', name: 'مدير النظام', passwordHash: pw, role: 'ADMIN' },
  });

  await prisma.user.upsert({
    where: { email: 'staff@example.com' },
    update: {},
    create: { email: 'staff@example.com', name: 'موظف مراجعة', passwordHash: pw, role: 'STAFF' },
  });

  const user = await prisma.user.upsert({
    where: { email: 'user@example.com' },
    update: {},
    create: { email: 'user@example.com', name: 'مستخدم عادي', passwordHash: pw, role: 'USER' },
  });

  const samples = [
    { phone: '0512345678', model: 'iPhone 14', lat: 24.7136, lng: 46.6753, address: 'الرياض، حي العليا', status: 'NEW' },
    { phone: '0551234567', model: 'Galaxy S23', lat: 21.4858, lng: 39.1925, address: 'جدة، حي الروضة', status: 'IN_REVIEW' },
    { phone: '0561112233', model: null, lat: 26.4207, lng: 50.0888, address: 'الدمام، الكورنيش', status: 'CLOSED' },
  ];

  for (const s of samples) {
    await prisma.report.create({
      data: {
        reporterId: user.id,
        phoneNumber: s.phone,
        phoneNormalized: s.phone,
        deviceModel: s.model,
        lastSeenAt: null,
        description: 'بلاغ تجريبي',
        status: s.status,
        location: { create: { lat: s.lat, lng: s.lng, address: s.address } },
        statusHistory: {
          create: { fromStatus: null, toStatus: s.status, changedById: admin.id, note: 'بيانات تجريبية' },
        },
      },
    });
  }

  console.log('Seed done');
}

main().finally(() => prisma.$disconnect());
SEED

cat > "$BACKEND/src/config.js" <<'JS'
require('dotenv').config();

module.exports = {
  port: process.env.PORT || 4000,
  jwtSecret: process.env.JWT_SECRET,
  jwtExpiresIn: process.env.JWT_EXPIRES_IN || '8h',
  frontendOrigin: process.env.FRONTEND_ORIGIN || 'http://localhost:3000',
  uploadDir: process.env.UPLOAD_DIR || './uploads',
  maxUploadBytes: (parseInt(process.env.MAX_UPLOAD_MB, 10) || 5) * 1024 * 1024,
};

if (!module.exports.jwtSecret) throw new Error('JWT_SECRET is required in .env');
JS

cat > "$BACKEND/src/db.js" <<'JS'
const { PrismaClient } = require('@prisma/client');
module.exports = new PrismaClient();
JS

cat > "$BACKEND/src/utils/phone.js" <<'JS'
const PHONE_REGEX = /^(?:\+|00)?\d{9,15}$|^0\d{9}$/;

function normalizePhone(raw) {
  let p = String(raw).replace(/[\s\-()]/g, '');
  if (p.startsWith('00')) p = '+' + p.slice(2);
  return p;
}

function isValidPhone(raw) {
  return PHONE_REGEX.test(normalizePhone(raw));
}

function maskPhone(phone) {
  const p = String(phone);
  if (p.length <= 6) return '*'.repeat(p.length);
  return p.slice(0, 2) + '*'.repeat(p.length - 6) + p.slice(-4);
}

module.exports = { normalizePhone, isValidPhone, maskPhone };
JS

cat > "$BACKEND/src/utils/audit.js" <<'JS'
const prisma = require('../db');

async function audit({ actorId, action, entity, entityId, changes, ip }, tx = prisma) {
  return tx.auditLog.create({
    data: { actorId, action, entity, entityId, changes: changes ?? undefined, ip },
  });
}

module.exports = { audit };
JS

cat > "$BACKEND/src/middleware/auth.js" <<'JS'
const jwt = require('jsonwebtoken');
const { jwtSecret } = require('../config');

function authenticate(req, res, next) {
  const header = req.headers.authorization || '';
  const token = header.startsWith('Bearer ') ? header.slice(7) : null;
  if (!token) return res.status(401).json({ error: 'غير مصرح' });

  try {
    req.user = jwt.verify(token, jwtSecret);
    next();
  } catch {
    res.status(401).json({ error: 'رمز غير صالح أو منتهي' });
  }
}

function requireRole(...roles) {
  return (req, res, next) => {
    if (!req.user || !roles.includes(req.user.role)) {
      return res.status(403).json({ error: 'صلاحيات غير كافية' });
    }
    next();
  };
}

const isStaff = (user) => !!user && ['STAFF', 'ADMIN'].includes(user.role);

module.exports = { authenticate, requireRole, isStaff };
JS

cat > "$BACKEND/src/middleware/rateLimit.js" <<'JS'
const rateLimit = require('express-rate-limit');

const globalLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 300,
  standardHeaders: true,
  legacyHeaders: false,
});

const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 10,
  message: { error: 'محاولات كثيرة، حاول لاحقًا' },
});

const createReportLimiter = rateLimit({
  windowMs: 60 * 60 * 1000,
  max: 10,
  message: { error: 'تجاوزت الحد المسموح من البلاغات في الساعة' },
});

module.exports = { globalLimiter, loginLimiter, createReportLimiter };
JS

cat > "$BACKEND/src/routes/auth.js" <<'JS'
const router = require('express').Router();
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const { z } = require('zod');
const prisma = require('../db');
const { jwtSecret, jwtExpiresIn } = require('../config');
const { loginLimiter } = require('../middleware/rateLimit');

const registerSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
  name: z.string().min(2),
});

const sign = (u) =>
  jwt.sign({ id: u.id, role: u.role, name: u.name }, jwtSecret, { expiresIn: jwtExpiresIn });

router.post('/register', loginLimiter, async (req, res) => {
  const parsed = registerSchema.safeParse(req.body);
  if (!parsed.success) return res.status(400).json({ error: 'بيانات غير صالحة' });

  const { email, password, name } = parsed.data;
  if (await prisma.user.findUnique({ where: { email } })) {
    return res.status(409).json({ error: 'البريد مستخدم' });
  }

  const user = await prisma.user.create({
    data: { email, name, passwordHash: await bcrypt.hash(password, 12), role: 'USER' },
  });

  res.status(201).json({ token: sign(user), user: { id: user.id, name: user.name, role: user.role } });
});

router.post('/login', loginLimiter, async (req, res) => {
  const { email, password } = req.body || {};
  const user = await prisma.user.findUnique({ where: { email: String(email) } });
  if (!user || !(await bcrypt.compare(String(password), user.passwordHash))) {
    return res.status(401).json({ error: 'بيانات الدخول غير صحيحة' });
  }
  res.json({ token: sign(user), user: { id: user.id, name: user.name, role: user.role } });
});

module.exports = router;
JS

cat > "$BACKEND/src/routes/reports.js" <<'JS'
const router = require('express').Router();
const { z } = require('zod');
const prisma = require('../db');
const { authenticate, requireRole, isStaff } = require('../middleware/auth');
const { createReportLimiter } = require('../middleware/rateLimit');
const { normalizePhone, isValidPhone, maskPhone } = require('../utils/phone');
const { audit } = require('../utils/audit');

const createSchema = z.object({
  phoneNumber: z.string().refine(isValidPhone, 'صيغة رقم الهاتف غير صحيحة'),
  deviceModel: z.string().max(100).optional().nullable(),
  lastSeenAt: z.string().datetime().optional().nullable(),
  description: z.string().max(2000).optional().nullable(),
  location: z.object({
    lat: z.number().min(-90).max(90),
    lng: z.number().min(-180).max(180),
    address: z.string().min(3).max(500),
    geocodeComponents: z.any().optional(),
  }),
});

const statusSchema = z.object({
  status: z.enum(['NEW', 'IN_REVIEW', 'CLOSED']),
  note: z.string().max(500).optional(),
});

const ALLOWED = { NEW: ['IN_REVIEW'], IN_REVIEW: ['CLOSED'], CLOSED: [] };

function serialize(report, user) {
  const staff = isStaff(user);
  const base = {
    id: report.id,
    status: report.status,
    deviceModel: report.deviceModel,
    lastSeenAt: report.lastSeenAt,
    createdAt: report.createdAt,
    description: report.description,
    location: report.location && {
      lat: report.location.lat,
      lng: report.location.lng,
      address: report.location.address,
    },
  };
  if (staff) return { ...base, phoneNumber: report.phoneNumber, reporterId: report.reporterId };
  return { ...base, maskedPhone: maskPhone(report.phoneNumber) };
}

router.post('/', authenticate, createReportLimiter, async (req, res) => {
  const parsed = createSchema.safeParse(req.body);
  if (!parsed.success) {
    return res.status(400).json({ error: parsed.error.issues.map(i => i.message).join(' | ') });
  }
  const d = parsed.data;
  const phone = normalizePhone(d.phoneNumber);

  const report = await prisma.$transaction(async (tx) => {
    const r = await tx.report.create({
      data: {
        reporterId: req.user.id,
        phoneNumber: phone,
        phoneNormalized: phone,
        deviceModel: d.deviceModel ?? null,
        lastSeenAt: d.lastSeenAt ? new Date(d.lastSeenAt) : null,
        description: d.description ?? null,
        location: {
          create: { lat: d.location.lat, lng: d.location.lng, address: d.location.address, geocodeComponents: d.location.geocodeComponents ?? undefined },
        },
        statusHistory: {
          create: { fromStatus: null, toStatus: 'NEW', changedById: req.user.id, note: 'إنشاء البلاغ' },
        },
      },
      include: { location: true },
    });

    await audit({
      actorId: req.user.id,
      action: 'REPORT_CREATE',
      entity: 'Report',
      entityId: r.id,
      changes: { status: 'NEW' },
      ip: req.ip,
    }, tx);

    return r;
  });

  res.status(201).json(serialize(report, req.user));
});

router.get('/map', authenticate, async (req, res) => {
  const
