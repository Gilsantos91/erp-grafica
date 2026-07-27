# 🏗️ Arquitetura de Aplicação - ERP Gráfica

## 📋 Índice

1. [Visão Geral](#visão-geral)
2. [Stack Tecnológico](#stack-tecnológico)
3. [Arquitetura em Camadas](#arquitetura-em-camadas)
4. [Padrões de Design](#padrões-de-design)
5. [Fluxo de Dados](#fluxo-de-dados)
6. [Segurança](#segurança)
7. [Performance](#performance)
8. [Escalabilidade](#escalabilidade)

---

## 📊 Visão Geral

Arquitetura cliente-servidor em três camadas com componentes bem definidos e separação clara de responsabilidades.

```
┌─────────────────────────────────────────┐
│         CLIENT LAYER (Frontend)         │
│  Web (Next.js) | Mobile PWA | Desktop   │
└────────────────────┬────────────────────┘
                     │ HTTP/WebSocket
                     │
┌────────────────────▼────────────────────┐
│       API LAYER (Backend - NestJS)      │
│  Controllers | Services | Repositories  │
└────────────────────┬────────────────────┘
                     │ SQL/Cache
                     │
┌────────────────────▼────────────────────┐
│      DATA LAYER                         │
│  PostgreSQL | Redis | Supabase Storage  │
└─────────────────────────────────────────┘
```

---

## 🏗️ Stack Tecnológico

| Camada | Tecnologia | Razão |
|--------|-----------|-------|
| **Frontend** | Next.js 14 | SSR/SSG, routing moderno, excelente performance |
| **Backend** | NestJS | Framework robusto, TypeScript nativo, enterprise patterns |
| **Banco** | PostgreSQL | ACID completo, JSON nativo, extensível |
| **ORM** | Prisma | Type-safe, migrations automáticas, excelente DX |
| **Cache** | Redis | Cache rápido, sessões, pub/sub |
| **Linguagem** | TypeScript | Type safety, melhor manutenibilidade |
| **Styling** | Tailwind CSS | Utility-first, rápido, customizável |
| **UI Components** | Shadcn/UI | Acessíveis, sem dependências pesadas |
| **Hospedagem** | Vercel + Railway | Deploy automático, escalabilidade |
| **Storage** | Supabase | PostgreSQL gerenciado, Auth, Storage |

---

## 🏢 Arquitetura em Camadas

### 1. **Camada de Apresentação (Frontend - Next.js)**

**Responsabilidades:**
- Interface responsiva (Web, Mobile PWA)
- State management (Zustand)
- Requisições HTTP (Axios/Fetch)
- Validação de formulários (React Hook Form)
- Cache local (React Query)
- Notificações em tempo real (WebSocket)
- Modo claro/escuro
- Sincronização com backend

**Estrutura:**
```
frontend/src/
├── app/                    # Next.js App Router
│   ├── (auth)/            # Routes não autenticadas
│   ├── (dashboard)/       # Routes autenticadas
│   └── api/               # API Routes
├── components/            # React components
├── hooks/                 # Custom hooks
├── services/              # API clients
├── store/                 # Zustand stores
├── types/                 # TypeScript types
└── utils/                 # Utility functions
```

### 2. **Camada de API (Backend - NestJS)**

**Responsabilidades:**
- Roteamento de requisições
- Autenticação e autorização
- Validação de dados (class-validator)
- Lógica de negócio
- Acesso ao banco de dados
- Gerenciamento de transações
- WebSocket/Real-time
- Email e notificações
- Background jobs

**Padrão MVC:**
```
Controller     → Recebe HTTP requests
  ↓
Service        → Lógica de negócio
  ↓
Repository     → Acesso a dados (Prisma)
  ↓
Database       → PostgreSQL
```

**Módulos principais:**
- `auth/` - Autenticação JWT + 2FA
- `clients/` - Gerenciamento de clientes
- `orders/` - Ordens de serviço
- `financial/` - Gestão financeira
- `inventory/` - Controle de estoque
- `production/` - Kanban e produção
- `files/` - Gerenciamento de arquivos
- `notifications/` - Alertas e notificações
- `common/` - Guards, interceptors, pipes

### 3. **Camada de Dados**

```
PostgreSQL (Primária)
├── Dados transacionais
├── Índices otimizados
└── Triggers de auditoria

Redis (Cache)
├── Sessões de usuários
├── Cache de queries
└── Rate limiting

Supabase Storage
├── Arquivos de OS
├── Imagens e designs
└── Documentos
```

---

## 🎨 Padrões de Design

### 1. **MVC com Repository Pattern**

```typescript
// Controller - Recebe HTTP requests
@Controller('clients')
export class ClientsController {
  constructor(private readonly clientsService: ClientsService) {}

  @Post()
  create(@Body() dto: CreateClientDto) {
    return this.clientsService.create(dto);
  }
}

// Service - Business logic
@Injectable()
export class ClientsService {
  constructor(private repo: ClientsRepository) {}
  
  async create(dto: CreateClientDto) {
    // Validações de negócio
    return this.repo.create(dto);
  }
}

// Repository - Data access
@Injectable()
export class ClientsRepository {
  constructor(private prisma: PrismaService) {}
  
  async create(dto: CreateClientDto) {
    return this.prisma.clients.create({ data: dto });
  }
}
```

### 2. **DTOs para Validação**

```typescript
export class CreateClientDto {
  @IsString()
  @MinLength(3)
  name: string;

  @IsEmail()
  email: string;

  @IsPhoneNumber('BR')
  phone: string;
}
```

### 3. **Guards para Autenticação**

```typescript
@Injectable()
export class JwtAuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const token = this.extractToken(request);
    
    if (!token) return false;
    
    try {
      const payload = this.jwtService.verify(token);
      request.user = payload;
      return true;
    } catch {
      return false;
    }
  }
}

@Get()
@UseGuards(JwtAuthGuard)
findAll() { }
```

### 4. **Interceptadores para Logging**

```typescript
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler) {
    const now = Date.now();
    return next.handle().pipe(
      tap(() => console.log(`${Date.now() - now}ms`)),
    );
  }
}
```

---

## 🔄 Fluxo de Dados

### Criação de Ordem de Serviço

```
Cliente (UI)
    ↓ POST /api/orders
Controller (OrdersController)
    ↓ Recebe JSON
Pipe (ValidationPipe)
    ↓ Valida DTO
Guard (JwtAuthGuard)
    ↓ Verifica autenticação
Service (OrdersService)
    ↓ Lógica: gera número, valida dados
Repository (OrdersRepository)
    ↓ Acessa Prisma
PostgreSQL
    ↓ INSERT service_orders
    ↓ INSERT service_order_history (trigger)
    ↓ INSERT audit_logs (trigger)
Cache Invalidation (Redis)
    ↓ Limpa cache
WebSocket Event
    ↓ Notifica clientes em tempo real
Response (200 OK)
    ↓ { id, orderNumber, status, ... }
```

---

## 🔒 Segurança

### 1. **Autenticação**
- JWT com expiration (24h)
- Refresh tokens
- 2FA com TOTP
- Senhas com bcrypt (cost 10)

### 2. **Autorização**
- Role-Based Access Control (RBAC)
- Permission-based granular access
- Guards em rotas protegidas

### 3. **Validação**
- DTOs com class-validator
- Sanitização de entrada
- Proteção contra SQL Injection (Prisma)
- XSS prevention

### 4. **Criptografia**
- HTTPS/TLS
- API keys encriptadas
- End-to-end encryption para dados sensíveis

### 5. **Rate Limiting**
- Redis para tracking
- Limite por IP/usuário
- Proteção contra brute force

### 6. **CORS**
- Whitelist de domínios
- Métodos permitidos restritos

### 7. **Auditoria**
- Log de todos os acessos
- Histórico completo de alterações
- Rastreamento de quem fez o quê, quando

---

## ⚡ Performance

### 1. **Caching**
- Redis para sessões (TTL 24h)
- Cache de queries frequentes
- Cache no frontend (React Query)
- ETags para validação

### 2. **Indexação**
- Índices em colunas de busca
- Índices compostos (client_id, created_at)
- Índices em JSONB para tags

### 3. **Otimização de Queries**
- Evitar N+1 queries (Prisma include/select)
- Paginação obrigatória
- Lazy loading

### 4. **Compressão**
- Gzip em responses
- Minificação de assets
- Otimização de imagens

### 5. **CDN**
- Arquivos estáticos em CDN (Vercel Edge)
- Imagens com lazy loading
- Versionamento de assets

### 6. **Code Splitting**
- Routes dinâmicas no Next.js
- Componentes lazy loaded
- Bundles otimizados

---

## 📈 Escalabilidade

### 1. **Arquitetura Modular**
- Cada módulo independente
- Sem dependências circulares
- Fácil testar e manter

### 2. **Microserviços (Futuro)**
- Auth Service
- Orders Service
- Financial Service
- Inventory Service
- Notifications Service
- Files Service

### 3. **Load Balancing**
- Múltiplas instâncias backend
- Nginx como reverse proxy
- Sticky sessions para WebSocket

### 4. **Banco de Dados**
- Read replicas para scaling
- Particionamento por data
- Backup contínuo

### 5. **Jobs Assíncronos**
- BullMQ para background jobs
- Processamento de arquivos
- Envio de emails
- Relatórios
- Retry automático

### 6. **Monitoramento**
- Prometheus metrics
- Grafana dashboards
- Error tracking (Sentry)
- Performance monitoring

---

## ✅ Checklist de Implementação

**Fase 1 - MVP:**
- [ ] Setup inicial (repositório, dependências)
- [ ] Banco de dados e Prisma
- [ ] Autenticação JWT
- [ ] CRUD de clientes
- [ ] Ordens de serviço básico
- [ ] Dashboard simples

**Fase 2 - Intermediária:**
- [ ] Kanban de produção
- [ ] Gestão financeira
- [ ] Controle de estoque
- [ ] Upload de arquivos

**Fase 3 - Completa:**
- [ ] Alertas inteligentes
- [ ] Relatórios avançados
- [ ] IA/ML features
- [ ] Integrações externas

---

## 📊 Decisões Técnicas Justificadas

| Decisão | Razão | Alternativas |
|---------|-------|-------------|
| Next.js 14 | SSR/SSG nativo, routing moderno | Remix, SvelteKit |
| NestJS | Framework robusto, TypeScript nativo | Express, Fastify |
| PostgreSQL | ACID completo, JSON nativo | MySQL, MongoDB |
| Prisma | Type-safe, migrations automáticas | TypeORM, Sequelize |
| TypeScript | Type safety, melhor manutenibilidade | JavaScript puro |
| Tailwind CSS | Utility-first, rápido de desenvolver | Bootstrap, Material-UI |
| Shadcn/UI | Componentes acessíveis | Ant Design, Chakra UI |
| Redis | Cache rápido, pub/sub | Memcached, Varnish |
| Supabase | PostgreSQL gerenciado, Auth | Firebase, AWS Amplify |
| Vercel | Deploy automático, SSR otimizado | Netlify, Railway |

---

## 📚 Referências

- Next.js: https://nextjs.org/docs
- NestJS: https://docs.nestjs.com
- Prisma: https://www.prisma.io/docs
- PostgreSQL: https://www.postgresql.org/docs
- TypeScript: https://www.typescriptlang.org/docs
