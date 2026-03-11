# CLAUDE.md — OpenForge

## 🎯 Qué es OpenForge

OpenForge es un **AI app builder open source con BYOK** (Bring Your Own Key).

El usuario describe lo que quiere en lenguaje natural → OpenForge genera una app completa en Next.js + Supabase → El usuario exporta el código y lo deploya donde quiera.

**Diferenciadores clave:**
- **Zero vendor lock-in** — El código generado es tuyo, te lo llevás
- **BYOK** — Usás tu propia API key (Claude, OpenAI, etc.)
- **Open source** — Self-hosteable, la comunidad contribuye
- **AI-native** — No drag & drop, interfaz conversacional

**Competencia y por qué somos diferentes:**
- Bubble/Retool → Caros, lock-in total
- Base44 → Vendor lock-in, no exportás código real
- Bolt/v0 → Generan código pero no persisten data ni hostean
- Lovable → Closed source, no BYOK

---

## 🏗️ Arquitectura General

```
openforge/
├── apps/
│   └── web/                    # App principal (Next.js 14)
│       ├── app/
│       │   ├── (auth)/         # Login, register, forgot password
│       │   ├── (dashboard)/    # Dashboard del usuario
│       │   │   ├── projects/   # Lista de proyectos
│       │   │   └── [projectId]/# Editor de proyecto específico
│       │   ├── api/
│       │   │   ├── generate/   # Endpoint principal de generación
│       │   │   ├── chat/       # Streaming chat con el agente
│       │   │   └── export/     # Exportar proyecto como ZIP
│       │   └── layout.tsx
│       ├── components/
│       │   ├── ui/             # shadcn/ui components
│       │   ├── editor/         # Editor de código (Monaco)
│       │   ├── preview/        # Preview iframe de la app
│       │   ├── chat/           # Chat interface con el agente
│       │   └── project/        # Project tree, file explorer
│       ├── lib/
│       │   ├── ai/             # AI providers (Claude, OpenAI)
│       │   ├── generators/     # Code generators por tipo
│       │   ├── templates/      # Templates base
│       │   └── supabase/       # Supabase client
│       └── ...
├── packages/
│   ├── core/                   # Lógica compartida de generación
│   │   ├── agents/             # Agentes especializados
│   │   ├── prompts/            # System prompts
│   │   ├── parsers/            # Parsers de código
│   │   └── schemas/            # Zod schemas
│   ├── cli/                    # CLI para usar desde terminal
│   └── templates/              # Templates de apps generadas
│       ├── nextjs-supabase/    # Template base Next.js + Supabase
│       └── ...
├── supabase/
│   ├── migrations/             # Migrations de la DB de OpenForge
│   └── seed.sql
├── .env.example
├── turbo.json                  # Turborepo config
└── package.json
```

---

## 🔧 Stack Técnico

### OpenForge (el builder)
- **Framework:** Next.js 14 (App Router)
- **Styling:** Tailwind CSS + shadcn/ui
- **Database:** Supabase (Postgres + Auth + Storage)
- **AI:** Anthropic Claude API / OpenAI API (BYOK)
- **Editor:** Monaco Editor (VS Code)
- **Monorepo:** Turborepo + pnpm
- **Deploy:** Vercel / Self-hosted

### Apps Generadas (output)
- **Framework:** Next.js 14 (App Router)
- **Database:** Supabase
- **Auth:** Supabase Auth
- **Styling:** Tailwind CSS + shadcn/ui
- **ORM:** Supabase JS Client (no Prisma para simplificar)

---

## 🤖 Sistema de Agentes

OpenForge usa un sistema de agentes especializados que trabajan en conjunto:

### Agente Principal (Orchestrator)
- Recibe el prompt del usuario
- Decide qué agentes invocar
- Coordina el flujo de generación
- Mantiene el contexto de la conversación

### Agentes Especializados

```typescript
// packages/core/agents/index.ts

export const agents = {
  // Analiza el prompt y genera el schema de la app
  architect: {
    name: 'Architect',
    description: 'Analiza requerimientos y diseña la estructura de la app',
    output: 'AppSchema (entities, relations, pages, features)'
  },
  
  // Genera el schema de Supabase
  database: {
    name: 'Database Designer',
    description: 'Diseña el schema de Postgres y las policies de RLS',
    output: 'SQL migrations, RLS policies, seed data'
  },
  
  // Genera componentes React
  frontend: {
    name: 'Frontend Engineer',
    description: 'Genera componentes React, pages, y layouts',
    output: 'TSX components, pages, hooks'
  },
  
  // Genera API routes y server actions
  backend: {
    name: 'Backend Engineer', 
    description: 'Genera API routes, server actions, validations',
    output: 'API routes, server actions, Zod schemas'
  },
  
  // Revisa y mejora el código
  reviewer: {
    name: 'Code Reviewer',
    description: 'Revisa código, sugiere mejoras, encuentra bugs',
    output: 'Code review, fixes, improvements'
  }
}
```

### Flujo de Generación

```
Usuario: "Quiero una app para trackear mis gastos personales"
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ ORCHESTRATOR                                                 │
│ - Parsea el intent                                          │
│ - Identifica: CRUD app, entities: expenses, categories      │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ ARCHITECT                                                    │
│ Output:                                                      │
│ {                                                           │
│   entities: [                                               │
│     { name: 'expense', fields: [...] },                     │
│     { name: 'category', fields: [...] }                     │
│   ],                                                        │
│   pages: ['/', '/expenses', '/expenses/new', '/categories'],│
│   features: ['auth', 'crud', 'dashboard']                   │
│ }                                                           │
└─────────────────────────────────────────────────────────────┘
    │
    ├──────────────────┬──────────────────┐
    ▼                  ▼                  ▼
┌────────────┐  ┌────────────┐  ┌────────────┐
│ DATABASE   │  │ FRONTEND   │  │ BACKEND    │
│            │  │            │  │            │
│ - Schema   │  │ - Pages    │  │ - Actions  │
│ - RLS      │  │ - Comps    │  │ - Routes   │
│ - Seed     │  │ - Hooks    │  │ - Schemas  │
└────────────┘  └────────────┘  └────────────┘
    │                  │                  │
    └──────────────────┴──────────────────┘
                       │
                       ▼
              ┌────────────────┐
              │ REVIEWER       │
              │ - Validar      │
              │ - Corregir     │
              │ - Optimizar    │
              └────────────────┘
                       │
                       ▼
              [Código Final Listo]
```

---

## 📁 Schemas Principales

### AppSchema (output del Architect)

```typescript
// packages/core/schemas/app.ts
import { z } from 'zod'

export const FieldSchema = z.object({
  name: z.string(),
  type: z.enum(['string', 'number', 'boolean', 'date', 'datetime', 'json', 'uuid', 'text', 'email', 'url']),
  required: z.boolean().default(true),
  unique: z.boolean().default(false),
  default: z.any().optional(),
  reference: z.object({
    entity: z.string(),
    field: z.string(),
    onDelete: z.enum(['cascade', 'set-null', 'restrict']).default('cascade')
  }).optional()
})

export const EntitySchema = z.object({
  name: z.string(),
  displayName: z.string(),
  fields: z.array(FieldSchema),
  timestamps: z.boolean().default(true),
  softDelete: z.boolean().default(false),
  belongsToUser: z.boolean().default(true) // Para RLS automático
})

export const PageSchema = z.object({
  path: z.string(),
  name: z.string(),
  type: z.enum(['list', 'detail', 'form', 'dashboard', 'custom']),
  entity: z.string().optional(),
  components: z.array(z.string()).optional()
})

export const AppSchema = z.object({
  name: z.string(),
  description: z.string(),
  entities: z.array(EntitySchema),
  pages: z.array(PageSchema),
  features: z.array(z.enum([
    'auth',           // Supabase Auth
    'crud',           // CRUD automático
    'dashboard',      // Dashboard con stats
    'search',         // Full-text search
    'filters',        // Filtros avanzados
    'export',         // Export a CSV
    'file-upload',    // Subir archivos
    'realtime',       // Supabase Realtime
    'multi-tenant'    // Organizaciones/Teams
  ])),
  theme: z.object({
    primaryColor: z.string().default('blue'),
    mode: z.enum(['light', 'dark', 'system']).default('system')
  }).optional()
})

export type App = z.infer<typeof AppSchema>
export type Entity = z.infer<typeof EntitySchema>
export type Field = z.infer<typeof FieldSchema>
export type Page = z.infer<typeof PageSchema>
```

### ProjectSchema (para la DB de OpenForge)

```typescript
// packages/core/schemas/project.ts
export const ProjectSchema = z.object({
  id: z.string().uuid(),
  userId: z.string().uuid(),
  name: z.string(),
  description: z.string().optional(),
  appSchema: AppSchema.optional(), // El schema generado
  files: z.record(z.string()), // path -> content
  status: z.enum(['draft', 'generating', 'ready', 'exported']),
  settings: z.object({
    aiProvider: z.enum(['anthropic', 'openai']),
    // API key se guarda encriptada o en env del usuario
  }),
  createdAt: z.date(),
  updatedAt: z.date()
})
```

---

## 🎨 UI/UX del Builder

### Layout Principal

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ ┌─────────┐                                            ┌─────┐ ┌─────────┐  │
│ │OpenForge│  proyecto-x                                │ ⚙️  │ │ Export  │  │
│ └─────────┘                                            └─────┘ └─────────┘  │
├────────────────┬────────────────────────────┬───────────────────────────────┤
│                │                            │                               │
│  📁 Files      │      Editor (Monaco)       │      Preview                  │
│                │                            │                               │
│  ▼ src/        │  // app/page.tsx           │   ┌─────────────────────┐     │
│    ▼ app/      │                            │   │                     │     │
│      page.tsx  │  export default function   │   │    [Live Preview    │     │
│      layout    │  Home() {                  │   │     of generated    │     │
│    ▼ components│    return (                │   │        app]         │     │
│      ...       │      <div>                 │   │                     │     │
│    lib/        │        ...                 │   │                     │     │
│                │      </div>                │   └─────────────────────┘     │
│                │    )                       │                               │
│                │  }                         │   [Desktop] [Tablet] [Mobile] │
│                │                            │                               │
├────────────────┴────────────────────────────┴───────────────────────────────┤
│                                                                             │
│  💬 Chat with OpenForge                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Agregame un gráfico de gastos por categoría en el dashboard         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  🤖 OpenForge: Perfecto, voy a agregar un PieChart con Recharts...         │
│     [Generando: components/charts/ExpensesByCategory.tsx]                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Componentes Clave

```typescript
// Componentes principales a implementar

// 1. ProjectEditor - Layout principal del editor
// apps/web/components/editor/ProjectEditor.tsx

// 2. FileTree - Árbol de archivos del proyecto
// apps/web/components/editor/FileTree.tsx

// 3. CodeEditor - Monaco editor wrapper
// apps/web/components/editor/CodeEditor.tsx

// 4. Preview - Iframe con hot reload
// apps/web/components/editor/Preview.tsx

// 5. Chat - Interface de chat con el agente
// apps/web/components/chat/Chat.tsx

// 6. GenerationStatus - Estado de generación en tiempo real
// apps/web/components/chat/GenerationStatus.tsx
```

---

## 🔌 API Routes

### POST /api/generate
Genera una app nueva desde un prompt.

```typescript
// apps/web/app/api/generate/route.ts
import { anthropic } from '@/lib/ai/anthropic'
import { orchestrator } from '@openforge/core/agents'

export async function POST(req: Request) {
  const { prompt, projectId, apiKey, provider } = await req.json()
  
  // Validar API key
  const client = provider === 'anthropic' 
    ? createAnthropic(apiKey)
    : createOpenAI(apiKey)
  
  // Ejecutar orchestrator
  const stream = await orchestrator.run({
    prompt,
    projectId,
    client
  })
  
  // Stream response
  return new Response(stream, {
    headers: { 'Content-Type': 'text/event-stream' }
  })
}
```

### POST /api/chat
Chat iterativo para modificar la app.

```typescript
// apps/web/app/api/chat/route.ts
export async function POST(req: Request) {
  const { message, projectId, context, apiKey, provider } = await req.json()
  
  // Cargar proyecto actual
  const project = await getProject(projectId)
  
  // Determinar qué agente usar basado en el mensaje
  const agent = await orchestrator.route(message, project)
  
  // Ejecutar agente con contexto
  const stream = await agent.run({
    message,
    currentFiles: project.files,
    appSchema: project.appSchema,
    client: createClient(provider, apiKey)
  })
  
  return new Response(stream, {
    headers: { 'Content-Type': 'text/event-stream' }
  })
}
```

### POST /api/export
Exporta el proyecto como ZIP.

```typescript
// apps/web/app/api/export/route.ts
import JSZip from 'jszip'

export async function POST(req: Request) {
  const { projectId } = await req.json()
  
  const project = await getProject(projectId)
  
  const zip = new JSZip()
  
  // Agregar todos los archivos
  for (const [path, content] of Object.entries(project.files)) {
    zip.file(path, content)
  }
  
  // Agregar README con instrucciones
  zip.file('README.md', generateReadme(project))
  
  // Agregar .env.example
  zip.file('.env.example', generateEnvExample(project))
  
  const blob = await zip.generateAsync({ type: 'blob' })
  
  return new Response(blob, {
    headers: {
      'Content-Type': 'application/zip',
      'Content-Disposition': `attachment; filename="${project.name}.zip"`
    }
  })
}
```

---

## 📝 System Prompts

### Orchestrator Prompt

```typescript
// packages/core/prompts/orchestrator.ts
export const ORCHESTRATOR_PROMPT = `Sos el orquestador de OpenForge, un AI app builder.

Tu rol es:
1. Entender qué quiere construir el usuario
2. Dividir el trabajo en tareas para agentes especializados
3. Coordinar la generación de código
4. Mantener coherencia entre todas las partes

AGENTES DISPONIBLES:
- architect: Diseña la estructura de la app (entities, pages, features)
- database: Genera SQL, migrations, RLS policies
- frontend: Genera componentes React, pages, hooks
- backend: Genera server actions, API routes, validations
- reviewer: Revisa y mejora el código generado

FLUJO:
1. Si es un proyecto nuevo → architect primero
2. Con el schema → database + frontend + backend en paralelo
3. Al final → reviewer para validar

OUTPUT FORMAT:
Respondé siempre con un JSON:
{
  "thinking": "Tu razonamiento interno",
  "nextAgent": "architect" | "database" | "frontend" | "backend" | "reviewer",
  "taskForAgent": "Descripción específica de qué debe hacer",
  "contextForAgent": { ... } // Datos relevantes
}

Si ya está todo listo, respondé:
{
  "thinking": "...",
  "complete": true,
  "summary": "Resumen de lo generado"
}`
```

### Architect Prompt

```typescript
// packages/core/prompts/architect.ts
export const ARCHITECT_PROMPT = `Sos el arquitecto de OpenForge.

Tu rol es analizar lo que quiere el usuario y diseñar la estructura de la app.

INPUT: Descripción en lenguaje natural de la app
OUTPUT: AppSchema (JSON válido)

REGLAS:
1. Siempre incluir 'auth' en features
2. Cada entity debe tener un campo 'id' (uuid) implícito
3. Si el usuario menciona usuarios/cuentas, usar belongsToUser: true para RLS
4. Inferir relaciones lógicas (expense -> category, post -> author)
5. Nombrar entities en singular, lowercase (expense, not Expenses)
6. Paths de pages siempre empiezan con /

EJEMPLO:
Input: "App para trackear gastos"
Output:
{
  "name": "expense-tracker",
  "description": "App para trackear gastos personales por categoría",
  "entities": [
    {
      "name": "category",
      "displayName": "Categoría",
      "fields": [
        { "name": "name", "type": "string", "required": true },
        { "name": "color", "type": "string", "required": false },
        { "name": "icon", "type": "string", "required": false }
      ],
      "belongsToUser": true
    },
    {
      "name": "expense",
      "displayName": "Gasto",
      "fields": [
        { "name": "amount", "type": "number", "required": true },
        { "name": "description", "type": "text", "required": false },
        { "name": "date", "type": "date", "required": true },
        { "name": "categoryId", "type": "uuid", "reference": { "entity": "category", "field": "id" } }
      ],
      "belongsToUser": true
    }
  ],
  "pages": [
    { "path": "/", "name": "Dashboard", "type": "dashboard" },
    { "path": "/expenses", "name": "Gastos", "type": "list", "entity": "expense" },
    { "path": "/expenses/new", "name": "Nuevo Gasto", "type": "form", "entity": "expense" },
    { "path": "/categories", "name": "Categorías", "type": "list", "entity": "category" }
  ],
  "features": ["auth", "crud", "dashboard"]
}`
```

### Database Prompt

```typescript
// packages/core/prompts/database.ts
export const DATABASE_PROMPT = `Sos el diseñador de base de datos de OpenForge.

INPUT: AppSchema
OUTPUT: SQL migrations para Supabase

REGLAS:
1. Usar uuid para PKs: id uuid primary key default gen_random_uuid()
2. Siempre agregar created_at y updated_at si timestamps: true
3. Crear RLS policies para cada tabla con belongsToUser: true
4. Foreign keys con ON DELETE según schema
5. Crear índices para campos con búsqueda frecuente
6. Usar nomenclatura snake_case para tablas y columnas

OUTPUT FORMAT:
{
  "migrations": [
    {
      "name": "001_initial_schema",
      "sql": "-- SQL completo"
    }
  ],
  "seed": "-- SQL para datos de ejemplo (opcional)",
  "types": "// TypeScript types generados de las tablas"
}`
```

### Frontend Prompt

```typescript
// packages/core/prompts/frontend.ts
export const FRONTEND_PROMPT = `Sos el frontend engineer de OpenForge.

INPUT: AppSchema
OUTPUT: Componentes React (Next.js 14 App Router)

STACK:
- Next.js 14 con App Router
- TypeScript
- Tailwind CSS
- shadcn/ui components
- Supabase para data fetching

REGLAS:
1. Usar Server Components por default
2. 'use client' solo cuando necesario (interactividad)
3. Usar Server Actions para mutations
4. Loading states con Suspense
5. Error boundaries
6. Responsive design (mobile-first)
7. Accesibilidad (ARIA labels, semántica)

ESTRUCTURA DE ARCHIVOS:
- app/(dashboard)/page.tsx → Dashboard
- app/(dashboard)/[entity]/page.tsx → List view
- app/(dashboard)/[entity]/[id]/page.tsx → Detail view
- app/(dashboard)/[entity]/new/page.tsx → Create form
- components/[entity]/[Entity]List.tsx
- components/[entity]/[Entity]Form.tsx
- components/[entity]/[Entity]Card.tsx

OUTPUT FORMAT:
{
  "files": {
    "app/(dashboard)/expenses/page.tsx": "// código...",
    "components/expenses/ExpenseList.tsx": "// código...",
    ...
  }
}`
```

---

## 🗃️ Database Schema (OpenForge)

```sql
-- Usuarios (manejado por Supabase Auth)

-- Proyectos
create table projects (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references auth.users(id) on delete cascade not null,
  name text not null,
  description text,
  app_schema jsonb, -- El AppSchema generado
  status text default 'draft' check (status in ('draft', 'generating', 'ready', 'exported')),
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

-- Archivos del proyecto
create table project_files (
  id uuid primary key default gen_random_uuid(),
  project_id uuid references projects(id) on delete cascade not null,
  path text not null, -- ej: "src/app/page.tsx"
  content text not null,
  created_at timestamptz default now(),
  updated_at timestamptz default now(),
  unique(project_id, path)
);

-- Conversaciones (historial de chat)
create table conversations (
  id uuid primary key default gen_random_uuid(),
  project_id uuid references projects(id) on delete cascade not null,
  messages jsonb not null default '[]',
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

-- Settings del usuario (API keys encriptadas)
create table user_settings (
  user_id uuid primary key references auth.users(id) on delete cascade,
  ai_provider text default 'anthropic',
  encrypted_api_key text, -- Encriptada con clave del server
  preferences jsonb default '{}',
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

-- RLS Policies
alter table projects enable row level security;
alter table project_files enable row level security;
alter table conversations enable row level security;
alter table user_settings enable row level security;

create policy "Users can CRUD own projects" on projects
  for all using (auth.uid() = user_id);

create policy "Users can CRUD own project files" on project_files
  for all using (
    project_id in (select id from projects where user_id = auth.uid())
  );

create policy "Users can CRUD own conversations" on conversations
  for all using (
    project_id in (select id from projects where user_id = auth.uid())
  );

create policy "Users can CRUD own settings" on user_settings
  for all using (auth.uid() = user_id);
```

---

## 🚀 Roadmap

### v0.1 — MVP (Semanas 1-4)
- [ ] Setup monorepo (Turborepo + pnpm)
- [ ] Auth con Supabase
- [ ] BYOK settings (guardar API key)
- [ ] Generación básica: prompt → AppSchema → código
- [ ] Editor con Monaco
- [ ] Preview básico
- [ ] Export a ZIP

### v0.2 — Iteración (Semanas 5-8)
- [ ] Chat para modificar la app
- [ ] Mejor preview (hot reload)
- [ ] Templates predefinidos
- [ ] Historial de versiones
- [ ] Mejorar prompts con feedback

### v0.3 — Polish (Semanas 9-12)
- [ ] Deploy one-click (Vercel)
- [ ] CLI (`npx openforge create`)
- [ ] Documentación
- [ ] Comunidad (Discord)
- [ ] Landing page

### Future
- [ ] Más stacks (Remix, Astro, etc.)
- [ ] Más DBs (Planetscale, Neon)
- [ ] Marketplace de templates
- [ ] Plugins/extensiones
- [ ] Mobile app preview
- [ ] Colaboración en tiempo real

---

## 📋 Comandos de Desarrollo

```bash
# Setup inicial
pnpm install

# Desarrollo
pnpm dev              # Corre todo
pnpm dev --filter web # Solo la web app

# Build
pnpm build

# Lint
pnpm lint

# Tests
pnpm test

# DB
pnpm db:migrate       # Correr migrations
pnpm db:reset         # Reset DB
pnpm db:studio        # Abrir Supabase Studio

# Generate
pnpm generate:types   # Generar tipos de Supabase
```

---

## 🔐 Variables de Entorno

```bash
# .env.local

# Supabase (OpenForge's own database)
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=

# Encryption key for storing user API keys
ENCRYPTION_KEY=

# Optional: Default API keys for testing
ANTHROPIC_API_KEY=
OPENAI_API_KEY=
```

---

## 🧪 Testing

```typescript
// Tests importantes a implementar

// 1. Unit tests para los parsers
// packages/core/parsers/__tests__/

// 2. Integration tests para los agentes
// packages/core/agents/__tests__/

// 3. E2E tests para el flujo completo
// apps/web/e2e/
```

---

## 📚 Referencias

- [Next.js App Router](https://nextjs.org/docs/app)
- [Supabase Docs](https://supabase.com/docs)
- [shadcn/ui](https://ui.shadcn.com/)
- [Anthropic API](https://docs.anthropic.com/)
- [Turborepo](https://turbo.build/repo/docs)

---

## 🤝 Contribuir

1. Fork el repo
2. Crear branch: `git checkout -b feature/nueva-feature`
3. Commit: `git commit -m 'Add nueva feature'`
4. Push: `git push origin feature/nueva-feature`
5. Abrir PR

---

## 📄 Licencia

MIT — Usalo como quieras.
