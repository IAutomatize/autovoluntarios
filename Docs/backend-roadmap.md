# Guia de Implementação Backend - AutoVoluntários

## 🎯 Objetivo
Este documento fornece um roadmap detalhado para implementar o backend do sistema AutoVoluntários, baseado no frontend já desenvolvido.

---

## 🏗️ Arquitetura Recomendada

### **Stack Tecnológico**
```bash
# Runtime & Framework
Node.js 18+ 
Express.js 4.x
TypeScript 5.x

# Banco de Dados
PostgreSQL 15+ (principal)
Redis 7+ (cache e filas)

# ORM & Migrations
Prisma 5.x (recomendado)
# ou Sequelize/TypeORM

# Autenticação
JWT + Refresh Tokens
bcrypt para hashing

# Background Jobs
Bull/BullMQ + Redis
# ou node-cron para tarefas simples

# Validação
Zod (consistente com frontend)
# ou Joi/Yup

# Testing
Jest + Supertest
@testing-library para testes

# Monitoring
Winston (logging)
Sentry (error tracking)
```

### **Estrutura de Pastas**
```
backend/
├── src/
│   ├── controllers/       # Route handlers
│   │   ├── auth.controller.ts
│   │   ├── volunteers.controller.ts
│   │   ├── shifts.controller.ts
│   │   ├── substitutions.controller.ts
│   │   └── reports.controller.ts
│   ├── services/          # Business logic
│   │   ├── auth.service.ts
│   │   ├── scoring.service.ts
│   │   ├── notification.service.ts
│   │   ├── substitution.service.ts
│   │   └── scheduling.service.ts
│   ├── models/           # Database models
│   │   ├── volunteer.model.ts
│   │   ├── shift.model.ts
│   │   ├── assignment.model.ts
│   │   └── substitution.model.ts
│   ├── middleware/       # Express middleware
│   │   ├── auth.middleware.ts
│   │   ├── validation.middleware.ts
│   │   ├── error.middleware.ts
│   │   └── rate-limit.middleware.ts
│   ├── routes/           # Route definitions
│   │   ├── auth.routes.ts
│   │   ├── volunteers.routes.ts
│   │   ├── shifts.routes.ts
│   │   └── api.routes.ts
│   ├── utils/            # Utilities
│   │   ├── database.ts
│   │   ├── logger.ts
│   │   ├── validation.ts
│   │   └── constants.ts
│   ├── types/            # Shared types
│   │   └── index.ts
│   └── jobs/             # Background jobs
│       ├── scoring.job.ts
│       ├── notifications.job.ts
│       └── cleanup.job.ts
├── tests/
│   ├── unit/
│   ├── integration/
│   └── fixtures/
├── prisma/               # Database schema
│   ├── schema.prisma
│   ├── migrations/
│   └── seeds/
├── docs/                 # API documentation
└── scripts/              # Utility scripts
```

---

## 📊 Schema do Banco de Dados

### **Schema Prisma (prisma/schema.prisma)**
```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String   @id @default(cuid())
  email     String   @unique
  password  String
  role      Role     @default(VOLUNTEER)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  
  volunteer Volunteer?
  
  @@map("users")
}

model Volunteer {
  id          String   @id @default(cuid())
  userId      String   @unique
  name        String
  phone       String?
  skills      String[] // JSON array of skills
  maxPerWeek  Int      @default(2)
  preferences Json?    // Availability preferences
  isActive    Boolean  @default(true)
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  
  user         User         @relation(fields: [userId], references: [id])
  assignments  Assignment[]
  substitutionRequests SubstitutionRequest[]
  
  @@map("volunteers")
}

model Shift {
  id          String      @id @default(cuid())
  title       String
  description String?
  startTime   DateTime
  endTime     DateTime
  requiredRole String
  maxVolunteers Int       @default(1)
  isRecurring Boolean    @default(false)
  recurringPattern Json? // Recurrence rules
  status      ShiftStatus @default(DRAFT)
  createdAt   DateTime   @default(now())
  updatedAt   DateTime   @updatedAt
  
  assignments Assignment[]
  substitutionRequests SubstitutionRequest[]
  
  @@map("shifts")
}

model Assignment {
  id          String           @id @default(cuid())
  shiftId     String
  volunteerId String
  status      AssignmentStatus @default(CONFIRMED)
  assignedAt  DateTime         @default(now())
  confirmedAt DateTime?
  
  shift     Shift     @relation(fields: [shiftId], references: [id])
  volunteer Volunteer @relation(fields: [volunteerId], references: [id])
  
  @@unique([shiftId, volunteerId])
  @@map("assignments")
}

model SubstitutionRequest {
  id              String                    @id @default(cuid())
  shiftId         String
  requesterId     String
  reason          String?
  urgencyLevel    UrgencyLevel             @default(NORMAL)
  deadline        DateTime
  status          SubstitutionRequestStatus @default(PENDING)
  selectedCandidateId String?
  
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt
  
  shift           Shift     @relation(fields: [shiftId], references: [id])
  requester       Volunteer @relation(fields: [requesterId], references: [id])
  
  @@map("substitution_requests")
}

model NotificationLog {
  id          String           @id @default(cuid())
  userId      String
  type        NotificationType
  title       String
  message     String
  isRead      Boolean          @default(false)
  sentAt      DateTime         @default(now())
  readAt      DateTime?
  
  @@map("notification_logs")
}

model AuditLog {
  id        String   @id @default(cuid())
  userId    String
  action    String
  resource  String
  resourceId String?
  changes   Json?
  timestamp DateTime @default(now())
  
  @@map("audit_logs")
}

enum Role {
  ADMIN
  LEADER
  VOLUNTEER
}

enum ShiftStatus {
  DRAFT
  OPEN
  FILLED
  CANCELLED
}

enum AssignmentStatus {
  PENDING
  CONFIRMED
  CANCELLED
}

enum SubstitutionRequestStatus {
  PENDING
  PROCESSING
  APPROVED
  REJECTED
  COMPLETED
}

enum UrgencyLevel {
  LOW
  NORMAL
  HIGH
  CRITICAL
}

enum NotificationType {
  SHIFT_ASSIGNED
  SUBSTITUTION_REQUEST
  SHIFT_REMINDER
  SYSTEM_ALERT
}
```

---

## 🔧 Implementação dos Serviços

### **1. Serviço de Pontuação (src/services/scoring.service.ts)**
```typescript
import { Volunteer, Shift } from '@prisma/client';

interface ScoringWeights {
  availability: number;
  preference: number;
  fairRotation: number;
  workloadPenalty: number;
  recentPenalty: number;
  certificationBonus: number;
}

export class ScoringService {
  private weights: ScoringWeights = {
    availability: 40,
    preference: 15,
    fairRotation: 20,
    workloadPenalty: 15,
    recentPenalty: 10,
    certificationBonus: 25,
  };

  async calculateCandidateScore(
    volunteer: Volunteer,
    shift: Shift
  ): Promise<number> {
    let totalScore = 0;

    // 1. Verificar disponibilidade básica
    const availabilityScore = await this.calculateAvailabilityScore(volunteer, shift);
    totalScore += availabilityScore * (this.weights.availability / 100);

    // 2. Preferência do voluntário
    const preferenceScore = this.calculatePreferenceScore(volunteer, shift);
    totalScore += preferenceScore * (this.weights.preference / 100);

    // 3. Rotação justa
    const rotationScore = await this.calculateRotationScore(volunteer);
    totalScore += rotationScore * (this.weights.fairRotation / 100);

    // 4. Penalidade por carga de trabalho
    const workloadPenalty = await this.calculateWorkloadPenalty(volunteer);
    totalScore -= workloadPenalty * (this.weights.workloadPenalty / 100);

    // 5. Penalidade por trabalho recente
    const recentPenalty = await this.calculateRecentPenalty(volunteer);
    totalScore -= recentPenalty * (this.weights.recentPenalty / 100);

    // 6. Bônus por certificação
    const certificationBonus = this.calculateCertificationBonus(volunteer, shift);
    totalScore += certificationBonus * (this.weights.certificationBonus / 100);

    return Math.max(0, Math.min(100, totalScore));
  }

  private async calculateAvailabilityScore(
    volunteer: Volunteer,
    shift: Shift
  ): Promise<number> {
    // Implementar verificação de disponibilidade
    // Retorna 0-100 baseado na disponibilidade do voluntário
    return 80; // Placeholder
  }

  private calculatePreferenceScore(
    volunteer: Volunteer,
    shift: Shift
  ): number {
    // Implementar baseado nas preferências do voluntário
    return 60; // Placeholder
  }

  private async calculateRotationScore(volunteer: Volunteer): Promise<number> {
    // Calcular baseado na frequência de participação
    return 70; // Placeholder
  }

  private async calculateWorkloadPenalty(volunteer: Volunteer): Promise<number> {
    // Penalidade baseada na carga atual de trabalho
    return 10; // Placeholder
  }

  private async calculateRecentPenalty(volunteer: Volunteer): Promise<number> {
    // Penalidade por ter trabalhado recentemente
    return 5; // Placeholder
  }

  private calculateCertificationBonus(
    volunteer: Volunteer,
    shift: Shift
  ): number {
    // Bônus baseado em certificações/habilidades
    return 20; // Placeholder
  }
}
```

### **2. Serviço de Substituição (src/services/substitution.service.ts)**
```typescript
import { PrismaClient } from '@prisma/client';
import { ScoringService } from './scoring.service';
import { NotificationService } from './notification.service';

export class SubstitutionService {
  constructor(
    private prisma: PrismaClient,
    private scoringService: ScoringService,
    private notificationService: NotificationService
  ) {}

  async createSubstitutionRequest(data: {
    shiftId: string;
    requesterId: string;
    reason?: string;
    urgencyLevel?: 'LOW' | 'NORMAL' | 'HIGH' | 'CRITICAL';
    deadline: Date;
  }) {
    const request = await this.prisma.substitutionRequest.create({
      data: {
        ...data,
        status: 'PENDING',
      },
    });

    // Iniciar processamento em background
    await this.processSubstitutionRequest(request.id);

    return request;
  }

  async processSubstitutionRequest(requestId: string) {
    const request = await this.prisma.substitutionRequest.findUnique({
      where: { id: requestId },
      include: { shift: true, requester: true },
    });

    if (!request) throw new Error('Substitution request not found');

    // Buscar candidatos elegíveis
    const candidates = await this.findEligibleCandidates(request.shift);

    // Calcular pontuação para cada candidato
    const scoredCandidates = await Promise.all(
      candidates.map(async (candidate) => ({
        volunteer: candidate,
        score: await this.scoringService.calculateCandidateScore(
          candidate,
          request.shift
        ),
      }))
    );

    // Ordenar por pontuação
    scoredCandidates.sort((a, b) => b.score - a.score);

    // Auto-aprovar se o melhor candidato tem score alto o suficiente
    const threshold = 80; // Configurável
    if (scoredCandidates[0]?.score >= threshold) {
      await this.autoApproveSubstitution(requestId, scoredCandidates[0].volunteer.id);
    } else {
      // Notificar líderes para aprovação manual
      await this.notificationService.notifyLeadersForApproval(requestId, scoredCandidates);
    }

    return scoredCandidates;
  }

  private async findEligibleCandidates(shift: any) {
    // Buscar voluntários disponíveis no horário do turno
    return this.prisma.volunteer.findMany({
      where: {
        isActive: true,
        // Adicionar filtros de disponibilidade
      },
    });
  }

  private async autoApproveSubstitution(requestId: string, candidateId: string) {
    await this.prisma.substitutionRequest.update({
      where: { id: requestId },
      data: {
        status: 'APPROVED',
        selectedCandidateId: candidateId,
      },
    });

    // Criar nova assignment
    await this.prisma.assignment.create({
      data: {
        shiftId: (await this.prisma.substitutionRequest.findUnique({
          where: { id: requestId }
        }))!.shiftId,
        volunteerId: candidateId,
        status: 'CONFIRMED',
      },
    });
  }
}
```

### **3. Controllers (src/controllers/substitutions.controller.ts)**
```typescript
import { Request, Response } from 'express';
import { SubstitutionService } from '../services/substitution.service';
import { z } from 'zod';

const createSubstitutionSchema = z.object({
  shiftId: z.string().cuid(),
  reason: z.string().optional(),
  urgencyLevel: z.enum(['LOW', 'NORMAL', 'HIGH', 'CRITICAL']).default('NORMAL'),
  deadline: z.string().datetime(),
});

export class SubstitutionsController {
  constructor(private substitutionService: SubstitutionService) {}

  async create(req: Request, res: Response) {
    try {
      const data = createSubstitutionSchema.parse(req.body);
      const userId = req.user.id; // Vem do middleware de auth

      const volunteer = await prisma.volunteer.findUnique({
        where: { userId },
      });

      if (!volunteer) {
        return res.status(404).json({ error: 'Volunteer not found' });
      }

      const request = await this.substitutionService.createSubstitutionRequest({
        ...data,
        requesterId: volunteer.id,
        deadline: new Date(data.deadline),
      });

      res.status(201).json(request);
    } catch (error) {
      res.status(400).json({ error: error.message });
    }
  }

  async getCandidates(req: Request, res: Response) {
    try {
      const { id } = req.params;
      const candidates = await this.substitutionService.processSubstitutionRequest(id);
      res.json(candidates);
    } catch (error) {
      res.status(400).json({ error: error.message });
    }
  }

  async approve(req: Request, res: Response) {
    try {
      const { id } = req.params;
      const { candidateId } = req.body;

      await this.substitutionService.autoApproveSubstitution(id, candidateId);
      res.json({ message: 'Substitution approved' });
    } catch (error) {
      res.status(400).json({ error: error.message });
    }
  }
}
```

---

## 🚀 APIs Prioritárias

### **1. Autenticação**
```bash
POST /api/auth/login
POST /api/auth/register
POST /api/auth/refresh
POST /api/auth/logout
GET  /api/auth/me
```

### **2. Voluntários**
```bash
GET    /api/volunteers              # Listar todos
POST   /api/volunteers              # Criar novo
GET    /api/volunteers/:id          # Buscar por ID
PUT    /api/volunteers/:id          # Atualizar
DELETE /api/volunteers/:id          # Remover
GET    /api/volunteers/:id/availability # Disponibilidade
POST   /api/volunteers/:id/availability # Atualizar disponibilidade
```

### **3. Turnos**
```bash
GET    /api/shifts                  # Listar turnos
POST   /api/shifts                  # Criar turno
GET    /api/shifts/:id              # Buscar turno
PUT    /api/shifts/:id              # Atualizar turno
DELETE /api/shifts/:id              # Remover turno
GET    /api/shifts/schedule/:date   # Agenda por data
PUT    /api/shifts/:id/assign       # Atribuir voluntário
```

### **4. Substituições**
```bash
GET    /api/substitutions           # Listar solicitações
POST   /api/substitutions           # Criar solicitação
GET    /api/substitutions/:id       # Buscar solicitação
PUT    /api/substitutions/:id/approve # Aprovar
PUT    /api/substitutions/:id/reject  # Rejeitar
GET    /api/substitutions/:id/candidates # Obter candidatos
```

### **5. Relatórios**
```bash
GET    /api/reports/metrics         # Métricas gerais
GET    /api/reports/attendance      # Relatório de presença
GET    /api/reports/workload        # Carga de trabalho
POST   /api/reports/export          # Exportar relatório
```

---

## 📋 Checklist de Implementação

### **Fase 1: Setup Inicial** (1 semana)
- [ ] Inicializar projeto Node.js + TypeScript
- [ ] Configurar Express.js + middleware básico
- [ ] Setup PostgreSQL + Redis
- [ ] Configurar Prisma ORM
- [ ] Implementar schema inicial
- [ ] Setup de testes (Jest)

### **Fase 2: Autenticação** (1 semana)
- [ ] Sistema de registro/login
- [ ] JWT tokens + refresh
- [ ] Middleware de autenticação
- [ ] Sistema de roles/permissões
- [ ] Validação de entrada (Zod)

### **Fase 3: APIs Básicas** (2 semanas)
- [ ] CRUD de voluntários
- [ ] CRUD de turnos
- [ ] Sistema de assignments
- [ ] Validações de negócio
- [ ] Testes de integração

### **Fase 4: Sistema de Substituições** (2 semanas)
- [ ] Implementar algoritmo de pontuação
- [ ] Serviço de substituições
- [ ] Background jobs para processamento
- [ ] Notificações básicas
- [ ] Testes do algoritmo

### **Fase 5: Features Avançadas** (1-2 semanas)
- [ ] WebSockets para real-time
- [ ] Sistema de relatórios
- [ ] Exportação de dados
- [ ] Performance optimization
- [ ] Monitoramento e logs

### **Fase 6: Produção** (1 semana)
- [ ] Configuração de deploy
- [ ] Variáveis de ambiente
- [ ] CI/CD pipeline
- [ ] Documentação da API
- [ ] Testes E2E

---

## 🔍 Comandos para Começar

### **1. Setup do projeto**
```bash
# Criar pasta e inicializar
mkdir backend && cd backend
npm init -y

# Instalar dependências principais
npm install express prisma @prisma/client
npm install jsonwebtoken bcryptjs cors helmet morgan
npm install zod date-fns bull redis

# Dependências de desenvolvimento
npm install -D typescript @types/node @types/express
npm install -D @types/jsonwebtoken @types/bcryptjs
npm install -D jest @types/jest supertest @types/supertest
npm install -D nodemon ts-node

# Configurar TypeScript
npx tsc --init

# Inicializar Prisma
npx prisma init
```

### **2. Configurar ambiente**
```bash
# .env
DATABASE_URL="postgresql://user:password@localhost:5432/autovoluntarios"
REDIS_URL="redis://localhost:6379"
JWT_SECRET="your-super-secret-key"
JWT_REFRESH_SECRET="your-refresh-secret-key"
PORT=3000
NODE_ENV=development
```

### **3. Setup do banco**
```bash
# Aplicar schema
npx prisma db push

# Gerar cliente
npx prisma generate

# Seed inicial (opcional)
npx prisma db seed
```

### **4. Scripts do package.json**
```json
{
  "scripts": {
    "dev": "nodemon src/server.ts",
    "build": "tsc",
    "start": "node dist/server.js",
    "test": "jest",
    "test:watch": "jest --watch",
    "db:migrate": "npx prisma migrate dev",
    "db:seed": "npx prisma db seed",
    "db:studio": "npx prisma studio"
  }
}
```

---

## 🎯 Próximos Passos

1. **Escolha a stack**: Confirme se Node.js + Express + Prisma atende suas necessidades
2. **Setup inicial**: Configure o ambiente de desenvolvimento
3. **Implemente autenticação**: Base para todas as outras funcionalidades
4. **APIs básicas**: CRUD simples para validar a arquitetura
5. **Sistema de pontuação**: Core business logic
6. **Integração frontend**: Conecte com o frontend já desenvolvido

O frontend está **100% pronto** para receber dados reais das APIs. Assim que o backend estiver implementado, o sistema estará completamente funcional! 🚀