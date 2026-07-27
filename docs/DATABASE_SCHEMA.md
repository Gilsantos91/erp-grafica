# 🗄️ Schema do Banco de Dados - ERP Gráfica

## 📋 Índice

1. [Tabelas de Usuários e Autenticação](#usuários-e-autenticação)
2. [Tabelas de Clientes](#clientes)
3. [Tabelas de Ordens de Serviço](#ordens-de-serviço)
4. [Tabelas de Produção](#produção)
5. [Tabelas Financeiras](#financeiro)
6. [Tabelas de Estoque](#estoque)
7. [Tabelas de Arquivos](#arquivos)
8. [Tabelas de Auditoria](#auditoria)

---

## 👥 USUÁRIOS E AUTENTICAÇÃO

### Tabela: `users`
Armazena informações dos usuários do sistema.

```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  full_name VARCHAR(255) NOT NULL,
  avatar_url TEXT,
  phone VARCHAR(20),
  role user_role NOT NULL,
  department VARCHAR(100),
  is_active BOOLEAN DEFAULT true,
  two_factor_enabled BOOLEAN DEFAULT false,
  two_factor_secret VARCHAR(255),
  last_login TIMESTAMP,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role ON users(role);
CREATE INDEX idx_users_is_active ON users(is_active);
```

### Tipo Enum: `user_role`
```sql
CREATE TYPE user_role AS ENUM (
  'admin',
  'financeiro',
  'producao',
  'designer',
  'atendimento',
  'vendedor'
);
```

### Tabela: `user_permissions`
Controle granular de permissões por usuário.

```sql
CREATE TABLE user_permissions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  module VARCHAR(100) NOT NULL,
  permission VARCHAR(50) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  UNIQUE(user_id, module, permission)
);

CREATE INDEX idx_user_permissions_user_id ON user_permissions(user_id);
```

### Tabela: `user_sessions`
Rastreamento de sessões de usuários.

```sql
CREATE TABLE user_sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  token_hash VARCHAR(255) UNIQUE NOT NULL,
  ip_address INET,
  user_agent TEXT,
  expires_at TIMESTAMP NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  revoked_at TIMESTAMP
);

CREATE INDEX idx_user_sessions_user_id ON user_sessions(user_id);
CREATE INDEX idx_user_sessions_expires_at ON user_sessions(expires_at);
```

---

## 👥 CLIENTES

### Tabela: `clients`
Cadastro completo de clientes.

```sql
CREATE TABLE clients (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  client_type VARCHAR(20) NOT NULL,
  name VARCHAR(255) NOT NULL,
  company_name VARCHAR(255),
  document_number VARCHAR(20) NOT NULL UNIQUE,
  state_registration VARCHAR(20),
  email VARCHAR(255),
  phone VARCHAR(20),
  whatsapp VARCHAR(20),
  street VARCHAR(255),
  street_number VARCHAR(20),
  complement VARCHAR(255),
  city VARCHAR(100),
  state VARCHAR(2),
  postal_code VARCHAR(10),
  country VARCHAR(100) DEFAULT 'Brasil',
  
  responsible_user_id UUID REFERENCES users(id),
  credit_limit DECIMAL(12, 2) DEFAULT 0,
  payment_terms VARCHAR(100),
  preferred_payment_method VARCHAR(50),
  is_active BOOLEAN DEFAULT true,
  is_inadimplent BOOLEAN DEFAULT false,
  
  total_spent DECIMAL(12, 2) DEFAULT 0,
  last_purchase_date DATE,
  total_orders INTEGER DEFAULT 0,
  
  notes TEXT,
  tags JSONB DEFAULT '[]',
  is_favorite BOOLEAN DEFAULT false,
  
  created_by UUID REFERENCES users(id),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP
);

CREATE INDEX idx_clients_document ON clients(document_number);
CREATE INDEX idx_clients_email ON clients(email);
CREATE INDEX idx_clients_responsible_user ON clients(responsible_user_id);
CREATE INDEX idx_clients_is_active ON clients(is_active);
CREATE INDEX idx_clients_tags ON clients USING GIN(tags);
```

---

## 📋 ORDENS DE SERVIÇO

### Tabela: `service_orders`
Cadastro completo de Ordens de Serviço (OS).

```sql
CREATE TABLE service_orders (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_number BIGINT UNIQUE NOT NULL,
  client_id UUID NOT NULL REFERENCES clients(id),
  
  title VARCHAR(255) NOT NULL,
  description TEXT,
  category VARCHAR(100) NOT NULL,
  
  entry_date DATE NOT NULL,
  due_date DATE NOT NULL,
  delivery_date DATE,
  
  priority VARCHAR(20) NOT NULL DEFAULT 'normal',
  responsible_user_id UUID REFERENCES users(id),
  assigned_team JSONB,
  
  production_status VARCHAR(50) NOT NULL DEFAULT 'novo_pedido',
  payment_status VARCHAR(50) NOT NULL DEFAULT 'pendente',
  approval_status VARCHAR(50) NOT NULL DEFAULT 'pendente',
  
  base_value DECIMAL(12, 2) NOT NULL,
  discount DECIMAL(12, 2) DEFAULT 0,
  tax DECIMAL(12, 2) DEFAULT 0,
  final_value DECIMAL(12, 2) NOT NULL,
  payment_method VARCHAR(50),
  installments INTEGER DEFAULT 1,
  installment_value DECIMAL(12, 2),
  
  notes TEXT,
  internal_notes TEXT,
  tags JSONB DEFAULT '[]',
  
  created_by UUID REFERENCES users(id),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP
);

CREATE INDEX idx_service_orders_number ON service_orders(order_number);
CREATE INDEX idx_service_orders_client_id ON service_orders(client_id);
CREATE INDEX idx_service_orders_status ON service_orders(production_status);
CREATE INDEX idx_service_orders_payment_status ON service_orders(payment_status);
CREATE INDEX idx_service_orders_due_date ON service_orders(due_date);
CREATE INDEX idx_service_orders_responsible ON service_orders(responsible_user_id);
CREATE INDEX idx_service_orders_created_at ON service_orders(created_at);
```

### Tabela: `service_order_history`
Histórico completo de todas as alterações em OS.

```sql
CREATE TABLE service_order_history (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id UUID NOT NULL REFERENCES service_orders(id) ON DELETE CASCADE,
  changed_by UUID NOT NULL REFERENCES users(id),
  field_name VARCHAR(100) NOT NULL,
  old_value TEXT,
  new_value TEXT,
  comments TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_order_history_order_id ON service_order_history(order_id);
CREATE INDEX idx_order_history_created_at ON service_order_history(created_at);
```

---

## 🏭 PRODUÇÃO

### Tabela: `production_stages`
Estágios de produção customizáveis.

```sql
CREATE TABLE production_stages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id UUID NOT NULL REFERENCES service_orders(id) ON DELETE CASCADE,
  stage_name VARCHAR(100) NOT NULL,
  position INTEGER NOT NULL,
  status VARCHAR(50) NOT NULL,
  responsible_user_id UUID REFERENCES users(id),
  estimated_hours DECIMAL(10, 2),
  actual_hours DECIMAL(10, 2),
  started_at TIMESTAMP,
  completed_at TIMESTAMP,
  notes TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_production_stages_order_id ON production_stages(order_id);
CREATE INDEX idx_production_stages_status ON production_stages(status);
```

---

## 💰 FINANCEIRO

### Tabela: `financial_transactions`
Registro de todas as transações financeiras.

```sql
CREATE TABLE financial_transactions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  transaction_type VARCHAR(50) NOT NULL,
  category VARCHAR(100) NOT NULL,
  description VARCHAR(255),
  amount DECIMAL(12, 2) NOT NULL,
  
  order_id UUID REFERENCES service_orders(id),
  client_id UUID REFERENCES clients(id),
  
  payment_method VARCHAR(50) NOT NULL,
  payment_date DATE NOT NULL,
  due_date DATE,
  status VARCHAR(50) NOT NULL DEFAULT 'pendente',
  
  is_installment BOOLEAN DEFAULT false,
  installment_number INTEGER,
  total_installments INTEGER,
  
  notes TEXT,
  proof_url TEXT,
  
  created_by UUID REFERENCES users(id),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP
);

CREATE INDEX idx_transactions_type ON financial_transactions(transaction_type);
CREATE INDEX idx_transactions_payment_date ON financial_transactions(payment_date);
CREATE INDEX idx_transactions_due_date ON financial_transactions(due_date);
CREATE INDEX idx_transactions_status ON financial_transactions(status);
CREATE INDEX idx_transactions_client_id ON financial_transactions(client_id);
CREATE INDEX idx_transactions_order_id ON financial_transactions(order_id);
```

### Tabela: `financial_summary`
Resumo diário de finanças.

```sql
CREATE TABLE financial_summary (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  summary_date DATE UNIQUE NOT NULL,
  total_revenue DECIMAL(12, 2) DEFAULT 0,
  total_expenses DECIMAL(12, 2) DEFAULT 0,
  total_received DECIMAL(12, 2) DEFAULT 0,
  total_pending DECIMAL(12, 2) DEFAULT 0,
  total_overdue DECIMAL(12, 2) DEFAULT 0,
  cash_flow DECIMAL(12, 2) DEFAULT 0,
  profit DECIMAL(12, 2) DEFAULT 0,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_financial_summary_date ON financial_summary(summary_date);
```

---

## 📦 ESTOQUE

### Tabela: `products`
Cadastro de produtos.

```sql
CREATE TABLE products (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  product_code VARCHAR(50) UNIQUE NOT NULL,
  name VARCHAR(255) NOT NULL,
  description TEXT,
  category VARCHAR(100) NOT NULL,
  unit VARCHAR(50) NOT NULL,
  
  price DECIMAL(12, 2) NOT NULL,
  cost DECIMAL(12, 2),
  markup_percentage DECIMAL(5, 2),
  
  stock_quantity DECIMAL(10, 2) DEFAULT 0,
  minimum_stock DECIMAL(10, 2),
  maximum_stock DECIMAL(10, 2),
  reorder_quantity DECIMAL(10, 2),
  
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_products_code ON products(product_code);
CREATE INDEX idx_products_category ON products(category);
```

### Tabela: `stock_movements`
Histórico de movimentações de estoque.

```sql
CREATE TABLE stock_movements (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  product_id UUID NOT NULL REFERENCES products(id),
  order_id UUID REFERENCES service_orders(id),
  
  movement_type VARCHAR(50) NOT NULL,
  quantity DECIMAL(10, 2) NOT NULL,
  unit_cost DECIMAL(12, 2),
  
  reason TEXT,
  notes TEXT,
  
  recorded_by UUID NOT NULL REFERENCES users(id),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_stock_movements_product ON stock_movements(product_id);
CREATE INDEX idx_stock_movements_type ON stock_movements(movement_type);
CREATE INDEX idx_stock_movements_created_at ON stock_movements(created_at);
```

---

## 📁 ARQUIVOS

### Tabela: `file_attachments`
Arquivos anexados a OS.

```sql
CREATE TABLE file_attachments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id UUID NOT NULL REFERENCES service_orders(id) ON DELETE CASCADE,
  
  file_name VARCHAR(255) NOT NULL,
  file_type VARCHAR(50) NOT NULL,
  file_size BIGINT,
  file_url TEXT NOT NULL,
  file_hash VARCHAR(64),
  
  version INTEGER DEFAULT 1,
  is_latest BOOLEAN DEFAULT true,
  
  approved BOOLEAN DEFAULT false,
  approved_by UUID REFERENCES users(id),
  approved_at TIMESTAMP,
  
  uploaded_by UUID NOT NULL REFERENCES users(id),
  uploaded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  deleted_at TIMESTAMP
);

CREATE INDEX idx_file_attachments_order_id ON file_attachments(order_id);
CREATE INDEX idx_file_attachments_type ON file_attachments(file_type);
CREATE INDEX idx_file_attachments_version ON file_attachments(version);
```

### Tabela: `file_versions`
Controle de versões de arquivos.

```sql
CREATE TABLE file_versions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  attachment_id UUID NOT NULL REFERENCES file_attachments(id) ON DELETE CASCADE,
  
  version_number INTEGER NOT NULL,
  file_url TEXT NOT NULL,
  file_hash VARCHAR(64),
  file_size BIGINT,
  
  description TEXT,
  created_by UUID NOT NULL REFERENCES users(id),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  UNIQUE(attachment_id, version_number)
);

CREATE INDEX idx_file_versions_attachment ON file_versions(attachment_id);
```

---

## 📅 AGENDA

### Tabela: `calendar_events`
Eventos de calendário.

```sql
CREATE TABLE calendar_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  title VARCHAR(255) NOT NULL,
  description TEXT,
  event_type VARCHAR(50) NOT NULL,
  
  start_date DATE NOT NULL,
  start_time TIME,
  end_date DATE,
  end_time TIME,
  
  order_id UUID REFERENCES service_orders(id),
  client_id UUID REFERENCES clients(id),
  assigned_to UUID REFERENCES users(id),
  
  location VARCHAR(255),
  color_hex VARCHAR(7),
  
  is_completed BOOLEAN DEFAULT false,
  completed_at TIMESTAMP,
  
  created_by UUID REFERENCES users(id),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_calendar_events_start_date ON calendar_events(start_date);
CREATE INDEX idx_calendar_events_order_id ON calendar_events(order_id);
CREATE INDEX idx_calendar_events_assigned_to ON calendar_events(assigned_to);
```

---

## 🔔 ALERTAS

### Tabela: `alerts_and_notifications`
Sistema de alertas.

```sql
CREATE TABLE alerts_and_notifications (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  
  alert_type VARCHAR(100) NOT NULL,
  title VARCHAR(255) NOT NULL,
  message TEXT NOT NULL,
  severity VARCHAR(20) NOT NULL DEFAULT 'info',
  
  related_entity_type VARCHAR(50),
  related_entity_id UUID,
  
  is_read BOOLEAN DEFAULT false,
  read_at TIMESTAMP,
  
  action_url TEXT,
  
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  expires_at TIMESTAMP
);

CREATE INDEX idx_alerts_user_id ON alerts_and_notifications(user_id);
CREATE INDEX idx_alerts_is_read ON alerts_and_notifications(is_read);
CREATE INDEX idx_alerts_created_at ON alerts_and_notifications(created_at);
```

---

## 🔐 AUDITORIA

### Tabela: `audit_logs`
Log completo de auditoria.

```sql
CREATE TABLE audit_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id),
  
  action VARCHAR(100) NOT NULL,
  entity_type VARCHAR(100) NOT NULL,
  entity_id UUID NOT NULL,
  
  old_values JSONB,
  new_values JSONB,
  
  ip_address INET,
  user_agent TEXT,
  
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_audit_logs_user_id ON audit_logs(user_id);
CREATE INDEX idx_audit_logs_entity ON audit_logs(entity_type, entity_id);
CREATE INDEX idx_audit_logs_created_at ON audit_logs(created_at);
```

---

## 📊 Resumo de Tabelas

| Tabela | Finalidade | Registros Esperados |
|--------|-----------|-------------------|
| users | Usuários do sistema | 10-100 |
| clients | Clientes cadastrados | 1k-10k |
| service_orders | Ordens de serviço | 10k-100k |
| service_order_history | Histórico de OS | 100k-1M |
| financial_transactions | Transações financeiras | 10k-100k |
| products | Produtos/serviços | 100-1k |
| stock_movements | Movimentações de estoque | 10k-100k |
| file_attachments | Arquivos anexados | 10k-100k |
| audit_logs | Log de auditoria | 100k-10M |

---

## 🔧 Notas Importantes

1. **Soft Deletes**: Use `deleted_at` para manter histórico
2. **Timestamps**: Sempre use `CURRENT_TIMESTAMP` para rastreabilidade
3. **UUIDs**: Use `gen_random_uuid()` como ID primária
4. **Índices**: Criar índices em colunas frequentemente consultadas
5. **Foreign Keys**: Use `ON DELETE CASCADE` ou `SET NULL` apropriadamente
6. **JSONB**: Para dados semi-estruturados (tags, especificações)
7. **Performance**: Desnormalizar quando necessário para read-heavy queries
8. **Particionamento**: Particionar `financial_transactions` e `audit_logs` por data em produção
