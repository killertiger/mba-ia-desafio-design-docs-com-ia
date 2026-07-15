# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an **Order Management System (OMS)** REST API built with Node.js, TypeScript, Express, and Prisma/MySQL. The application manages users, customers, products, and orders with role-based access control and state-machine-controlled order lifecycle.

The repository is a **design documentation challenge**: the deliverable is purely documental (docs/ folder), not code changes. The existing application code serves as reference and context only.

## Essential Commands

### Development
```bash
# Start development server with hot reload (watches for file changes)
npm run dev

# Build TypeScript to JavaScript
npm run build

# Start production server
npm start
```

### Database
```bash
# Run pending migrations (creates/updates schema from prisma/schema.prisma)
npm run db:migrate

# Reset database to initial state (destructive - drops all data)
npm run db:reset

# Seed database with initial data
npm run db:seed
```

### Testing & Quality
```bash
# Run tests once (all test files in tests/ matching *.test.ts)
npm run test

# Run tests in watch mode (reruns on file changes)
npm run test:watch

# Lint TypeScript code
npm lint

# Auto-format code with Prettier
npm run format
```

## Project Structure

```
.
├── src/                          # Main application code
│   ├── server.ts                 # Bootstrap and server startup
│   ├── app.ts                    # Express app factory and dependency injection
│   ├── config/                   # Configuration (env, database)
│   ├── middlewares/              # Express middlewares (logging, errors, auth, validation)
│   ├── shared/                   # Shared utilities
│   │   ├── errors/               # Error classes (AppError, HTTP errors)
│   │   ├── logger/               # Pino logger instance
│   │   └── http/                 # HTTP response utilities
│   ├── modules/                  # Feature modules (each follows same pattern)
│   │   ├── auth/                 # Authentication (JWT login)
│   │   ├── users/                # User management (CRUD, roles)
│   │   ├── customers/            # Customer management
│   │   ├── products/             # Product catalog
│   │   └── orders/               # Order lifecycle and state machine
│   └── routes/                   # Route registration
├── prisma/                       # Database schema and migrations
│   ├── schema.prisma             # Prisma schema (defines all models)
│   ├── migrations/               # Auto-generated migration files
│   └── seed.ts                   # Seed script for initial data
├── tests/                        # Test suite
│   ├── setup.ts                  # Test environment setup (Prisma connection, DB cleanup)
│   ├── auth.test.ts              # Auth and user tests
│   ├── orders.test.ts            # Order lifecycle tests
│   └── helpers/factories.ts      # Test data factories
├── docs/                         # **Design documentation (challenge deliverable)**
│   ├── PRD.md                    # Product Requirements Document
│   ├── RFC.md                    # Request for Comments (technical proposal)
│   ├── FDD.md                    # Feature Design Document
│   ├── TRACKER.md                # Traceability matrix (docs → code/transcript)
│   └── adrs/                     # Architecture Decision Records
├── .env.example                  # Environment variables template
├── docker-compose.yml            # MySQL service definition
├── package.json                  # Dependencies and scripts
├── tsconfig.json                 # TypeScript configuration
├── vitest.config.ts              # Vitest configuration
└── .eslintrc.json                # ESLint rules
```

## Architecture Patterns

### Module Structure
Each feature module (users, orders, customers, etc.) follows a consistent layered pattern:

- **`module.schemas.ts`**: Input validation schemas (Zod)
- **`module.repository.ts`**: Database access (wraps Prisma)
- **`module.service.ts`**: Business logic (state transitions, calculations, validations)
- **`module.controller.ts`**: HTTP request handlers
- **`module.routes.ts`**: Route definitions
- **`module.ts`** (sometimes): Enums or special utilities (e.g., `order.status.ts` for OrderStatus enum)

### Error Handling
- All errors are subclasses of `AppError` (src/shared/errors/app-error.ts)
- Each error has `statusCode`, `errorCode` (string like `"INVALID_ORDER_STATUS"`), and optional `details`
- Errors are caught by centralized `errorMiddleware` (src/middlewares/error.middleware.ts) which formats responses
- Error codes follow naming convention: `UPPERCASE_SNAKE_CASE`

### Authentication & Authorization
- JWT-based authentication via `authMiddleware` (src/middlewares/auth.middleware.ts)
- Role-based access control: `requireRole` middleware checks `UserRole.ADMIN` or `UserRole.OPERATOR`
- JWT secret and expiry configured via environment variables

### Order Lifecycle
- **State machine**: Order progresses through states defined in `OrderStatus` enum: `PENDING → PAID → PROCESSING → SHIPPED → DELIVERED` or `CANCELLED`
- **State transitions**: Managed in `OrderService.changeStatus()` (src/modules/orders/order.service.ts)
- **Auditability**: Every state change is logged to `OrderStatusHistory` table with timestamp and user who made the change
- **Inventory control**: Stock is checked and reserved transactionally when order is created; released if order is cancelled

### Logging
- Uses **Pino** logger (src/shared/logger/index.js)
- Structured JSON logging: each log entry is an object with fields
- Request logging middleware logs HTTP method, URL, response time, status
- Logger instance is injected or imported globally

### Validation
- Input validation via **Zod** schemas in module schema files
- Validated in `validateMiddleware` or manually in controllers/services

## Key Integration Points for Webhooks Feature

The webhook notification feature needs to integrate with these existing components:

1. **Order state changes** (`src/modules/orders/order.service.ts`, `changeStatus()` method)
   - Hook here to publish webhook events when order status changes

2. **Error handling patterns** (`src/shared/errors/`)
   - Reuse `AppError` class and error code conventions (e.g., `WEBHOOK_*` codes)

3. **Database models** (`prisma/schema.prisma`)
   - Add new models for webhooks (endpoints, event outbox, retries)
   - Use existing patterns (UUIDs, timestamps, enums)

4. **Request/response patterns** (`src/shared/http/response.ts`)
   - Follow existing HTTP response formatting for new webhook endpoints

5. **Logging** (`src/shared/logger/index.js`)
   - Use Pino for webhook processing, retries, failures

6. **Middleware** (src/middlewares/)
   - Possible new middleware for HMAC signature verification on webhook endpoints

7. **Module pattern** (src/modules/)
   - Webhooks should follow same module structure: schemas, repository, service, controller, routes

## Testing

- **Framework**: Vitest (configured in vitest.config.ts)
- **Environment**: Node, single-forked process (deterministic, no parallelism)
- **Database**: Uses real MySQL (no mocks) via Prisma
- **Setup**: tests/setup.ts connects to DB before tests, cleans tables before each test, disconnects after all tests
- **Pattern**: Test files are colocated in tests/ folder, importing from src/

Run a single test:
```bash
npm run test -- tests/orders.test.ts
```

Run test in watch mode for rapid iteration:
```bash
npm run test:watch
```

## Environment Variables

See `.env.example` for template. Key variables:

- `NODE_ENV`: "development" or "production"
- `PORT`: Server port (default 3000)
- `LOG_LEVEL`: Pino log level (trace, debug, info, warn, error, fatal)
- `DATABASE_URL`: MySQL connection string
- `SHADOW_DATABASE_URL`: Secondary DB for Prisma migrations (copy of DATABASE_URL)
- `JWT_SECRET`: Signing key for auth tokens (change in production)
- `JWT_EXPIRES_IN`: Token expiry duration (e.g., "8h")

## Code Style & Quality

- **TypeScript**: Strict mode enabled (`strict: true`). No implicit any; prefer type imports.
- **Linting**: ESLint via `npm run lint`
  - Enforces no unused vars (except those prefixed `_`)
  - Requires explicit equality checks (`===`)
  - Consistent type imports
  - No console.log (only console.warn/error allowed)
- **Formatting**: Prettier via `npm run format`
  - Auto-formats code on save if IDE is configured
  - Run before committing to avoid lint failures

## Database

- **ORM**: Prisma
- **Database**: MySQL 8.0
- **Migrations**: Stored in `prisma/migrations/`, auto-generated by Prisma
- **Schema**: Single source of truth in `prisma/schema.prisma`
- **Local development**: Docker Compose starts MySQL automatically (`docker-compose up`)
  - Runs on localhost:3306
  - Credentials in `.env` (root password, oms_user, oms_password)

Common Prisma commands:
```bash
# Create a new migration after schema change
npx prisma migrate dev --name <migration_name>

# View/explore data
npx prisma studio

# Generate Prisma client (run after pulling prisma/schema.prisma changes)
npx prisma generate
```

## Important Constraints for This Challenge

⚠️ **Do not modify code in src/, prisma/, or tests/ directories.** The deliverable is purely documentation in the docs/ folder.

The challenge requires:
1. Reading existing code to understand patterns and architecture
2. Reading `TRANSCRICAO.md` (meeting transcript) to extract requirements and decisions
3. Creating docs that reference actual code files, classes, and methods
4. Ensuring 100% traceability: every requirement or decision must link to transcript or code

Use grep or code exploration to verify:
- That files/classes referenced in docs actually exist
- That method names and signatures match reality
- That error codes are consistent with patterns

## Useful Commands for Documentation Work

List all error codes:
```bash
grep -r "errorCode:" src/ --include="*.ts"
```

Find order status references:
```bash
grep -r "OrderStatus" src/ --include="*.ts"
```

List all API routes:
```bash
grep -r "@" src/modules/ --include="*.routes.ts" -A 5
```

Find middleware usage:
```bash
grep -r "middleware" src/ --include="*.ts"
```
