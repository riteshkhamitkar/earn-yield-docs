# Anzo Backend - Official Contribution Guidelines (rules.md)

## 1. Introduction

Welcome to the Anzo Backend project! This document serves as the official set of rules and guidelines for all contributors. Its purpose is to ensure that our codebase remains clean, consistent, performant, and scalable.

Adhering to these rules is mandatory for all contributions. This guide will help you understand our architectural decisions, coding standards, and development processes. Before starting any work, please familiarize yourself with this document.

## 2. Core Architectural Principles

Our architecture is designed for long-term maintainability and performance within a monolithic structure.

### 2.1. Monolith-First Architecture

**Rule:** The Anzo backend is and will remain a **monolithic application**. We do not plan to extract modules into separate microservices. All new features must be built as modules within this single codebase.

**Rationale:** This decision simplifies development, deployment, and debugging by avoiding the complexities of distributed systems (e.g., network latency, service discovery, distributed transactions). All inter-module communication should be done via direct service injection, not HTTP or message queues.

### 2.2. The Service & Handler Pattern

**Rule:** All business logic must be organized using the **Service and Handler pattern**.

*   **Services (`*.service.ts`):** Act as a public-facing facade for the module. They orchestrate calls to one or more handlers. Controllers should **only** inject and call methods on services.
*   **Handlers (`*.handler.ts`):** Contain the actual business logic for a specific use case or workflow. They are injected into the service. A handler should be focused on a single, cohesive responsibility (e.g., `RegistrationHandler`, `SwapExecutionHandler`).

**Guideline for Creating Handlers:**

*   **New Feature:** Always create a new handler for a distinct new workflow (e.g., a `PlaidIntegrationHandler`).
*   **Existing Feature:** If your new logic shares more than 60% of its context or dependencies with an existing handler, add the logic to that handler. If it's a completely new workflow within the same module, create a new handler. Avoid creating dozens of tiny handlers; group related functionalities.

**Example Structure:**

```typescript
// anzo-backend-main/src/modules/auth/auth.service.ts
@Injectable()
export class AuthService {
  constructor(
    private registrationHandler: RegistrationHandler, // Injects Handler
    private sessionSyncHandler: SessionSyncHandler,
  ) {}

  // Service method delegates to the appropriate handler
  async initiateRegistration(dto: InitiateRegistrationDto) {
    return this.registrationHandler.initiateRegistration(dto);
  }
}

// anzo-backend-main/src/modules/auth/handlers/registration.handler.ts
@Injectable()
export class RegistrationHandler {
  constructor(private prisma: PrismaService) {}

  // Handler contains the actual business logic
  async initiateRegistration(dto: InitiateRegistrationDto) {
    // ... complex logic to check if user exists
  }
}
```

## 3. Project Structure & Naming Conventions

A predictable structure is key to a maintainable monolith.

### 3.1. Module Structure

*   All domain-specific logic resides in `src/modules/`.
*   Each module must have a clear boundary and responsibility (e.g., `auth`, `swaps`, `yield`).
*   **Module Registration:** All new modules must be registered in `AppModule` (`src/app.module.ts`) before they can be used.

### 3.2. Utility Functions (`/utils/`)

**Rule:** The placement of utility functions depends on their scope.

*   **Global Utilities:** If a function is a pure, generic helper that can be used across **multiple unrelated modules**, place it in `src/common/utils/`.
    *   **Examples:** `amount.util.ts` (for `toSmallestUnit`), `address-chain.util.ts`.
*   **Module-Specific Utilities:** If a function is tightly coupled to a single module's logic and has no use outside of it, place it in `src/modules/<module-name>/utils/`.
    *   **Examples:** `src/modules/transfers/utils/transfer-routing.util.ts`.

This prevents `src/common` from becoming a dumping ground and maintains module encapsulation.

## 4. Coding Standards & Best Practices

### 4.1. General Standards

*   **Linting & Formatting:** All code must pass ESLint and Prettier checks. Run `npm run lint` and `npm run format` before committing.
*   **Strong Typing:** Avoid using `any` wherever possible. Use explicit TypeScript types, interfaces, or DTOs.
*   **Immutability:** Avoid mutating objects and arrays directly. Use spread syntax (`...`) or functional methods (`.map`, `.filter`) to create new instances.

### 4.2. DTOs & Request Validation

**Rule:** All request bodies, query parameters, and path parameters must be validated using DTOs with `class-validator` decorators.

*   **DTOs (`*.dto.ts`):** Create DTO classes for every endpoint that accepts input. Use decorators such as `@IsString()`, `@IsOptional()`, `@IsNumber()`, `@IsArray()`, `@ValidateNested()`, etc.
*   **ValidationPipe:** The application uses a global `ValidationPipe` with `whitelist: true` and `forbidNonWhitelisted: true`. Ensure DTOs only expose allowed fields.
*   **Documentation:** Use `@ApiProperty()` from `@nestjs/swagger` (if used) or JSDoc to document expected types and validation rules.

### 4.3. Performance & Optimization

**Rule:** Performance is a core feature. All contributions must be mindful of latency and resource usage.

1.  **Database Queries:**
    *   **Minimize Queries:** Never use multiple sequential database queries where a single query with relations or a `Promise.all` can achieve the same result. Avoid N+1 problems.
    *   **Efficient Queries:** Use Prisma's `findUnique` or `findFirst` with `where` clauses on indexed columns. Use `select` to only fetch the columns you need.
    *   **Parallelism:** For independent async operations (e.g., fetching data from two different external APIs), always use `Promise.all` to run them concurrently.

2.  **Algorithmic Complexity:**
    *   Be mindful of the time and space complexity of your algorithms, especially for operations that handle lists of data. Avoid nested loops (O(n²)) where a Map (O(n)) could be used for lookups.

3.  **Caching:**
    *   For expensive, frequently accessed data that changes infrequently (e.g., external API data), implement in-memory caching with a Time-To-Live (TTL). Follow the pattern in `PricingProvider`.

### 4.4. Error Handling

**Rule:** We use a hybrid approach for clear, consistent error handling.

1.  **Throw Specific Exceptions:** In handlers and services, throw specific NestJS `HttpException`s when a known error occurs (e.g., `BadRequestException`, `NotFoundException`, `ConflictException`). This makes the business logic explicit.

2.  **Implement a Global Exception Filter:** A single global `HttpExceptionFilter` must be implemented and applied in `main.ts`. This filter will catch all `HttpException`s and format them into a standardized JSON response, ensuring a consistent error structure across the entire API.

**Example `HttpExceptionFilter` (to be created in `src/common/filters/`):**

```typescript
// src/common/filters/http-exception.filter.ts
@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const status = exception.getStatus();
    const exceptionResponse = exception.getResponse();

    const message = typeof exceptionResponse === 'string'
      ? exceptionResponse
      : (exceptionResponse as any).message;

    response.status(status).json({
      statusCode: status,
      message: message,
      error: (exceptionResponse as any).error || 'Error',
      timestamp: new Date().toISOString(),
    });
  }
}
```

### 4.5. Authentication & Authorization

**Rule:** All endpoints that require an authenticated user must use the `AuthGuard`.

*   **Protected Routes:** Apply `@UseGuards(AuthGuard)` to controller methods that need user context.
*   **User Context:** Access the authenticated user via `@Request() req` and use `req.user.userId` (or equivalent) for user-scoped operations.

## 5. Database & Migrations

### 5.1. Prisma Migrations

**Rule:** All database schema changes must be done via Prisma migrations with descriptive names.

*   **Command:** Always use `npx prisma migrate dev --name <descriptive_name>`.
*   **Format:** The name must be a short, snake_cased description of the change.
    *   **Correct:** `npx prisma migrate dev --name add_user_preferences`
    *   **Incorrect:** `npx prisma migrate dev`

This creates a self-documenting migration history (e.g., `20251208091616_add_token_model`).

### 5.2. Schema Best Practices

*   **Indexing:** All foreign key columns and columns frequently used in `where` clauses must be indexed (`@@index`).
*   **Relations:** Define explicit relations using `@relation`. Use `onDelete: Cascade` or `onDelete: SetNull` appropriately to maintain data integrity.
*   **Comments:** Use `///` comments in `schema.prisma` to explain the purpose of complex fields or models.

### 5.3. Deprecating Models & Columns

**Rule:** Never drop a column or table from a production database in a single step. Follow this safe, multi-stage deprecation process:

1.  **Mark as Deprecated:** In `schema.prisma`, mark the field with `@deprecated` and a comment explaining its replacement and removal timeline.

```prisma
model Swap {
  /// @deprecated This model is replaced by Transaction. Plan removal in Q3 2025.
  // ...
}
```

2.  **Remove Write Operations:** In a new pull request, remove all code that writes to the deprecated field/model. Deploy this change.
3.  **Data Migration (Optional):** If data needs to be preserved, write and run a migration script to move it from the old column to the new one.
4.  **Remove Read Operations:** In a separate pull request, remove all code that reads from the deprecated field/model. Deploy this change.
5.  **Drop from Schema:** Finally, in a new pull request, create a Prisma migration to drop the column/table from the database.

## 6. Configuration & Secrets

**Rule:** All configuration and secrets must be managed securely and consistently.

1.  **No Hardcoding:** Never hardcode secrets, API keys, or environment-specific values in the code.
2.  **Use `ConfigService`:** All environment variables must be accessed exclusively through NestJS's `ConfigService`.
3.  **Define in `.env.example`:** Every new environment variable must be added to `.env.example` with a placeholder value.
4.  **Validate on Startup:** Every new environment variable must be added to the Joi schema in `src/config/validation.schema.ts` to ensure the application fails fast if a required variable is missing.

## 7. Testing Philosophy

**Rule: No Regressions.**

While there is no strict test coverage percentage required for pull requests, every developer is responsible for ensuring their changes do not break existing functionality.

*   **Manual Testing:** The absolute minimum expectation is to manually test the feature you've built *and* any related user flows that could be impacted by your changes.
*   **Automated Testing:** Adding unit tests (`.spec.ts`) for complex business logic in handlers and E2E tests (`.e2e-spec.ts`) for critical API endpoints is **highly encouraged**. This protects your feature from being broken by future changes.

## 8. API Design & Documentation

**Rule:** Our documentation is a product. It must be as high-quality as our code.

1.  **Update Documentation with Code:** Any pull request that adds or modifies an API endpoint **must** include corresponding updates to the relevant documentation file in the `docs/` directory.
2.  **New Module, New Doc:** If you create a new module with public-facing endpoints, you must create a new, detailed `*.md` file in the `docs/` directory explaining its usage (e.g., `SAVINGS_MODULE_API.md`).
3.  **Follow Existing Format:** Follow the structure and detail level of existing documents like `BRIDGE_INTEGRATION.md` and `LIFI_BITCOIN_INTEGRATION.md`. Include API endpoints, request/response formats, flow diagrams, and frontend integration examples.

### 8.1. API Conventions

*   **Base Path:** All API routes are prefixed with `api/v1`. Controllers define routes relative to this prefix.
*   **RESTful Design:** Use standard HTTP methods (GET, POST, PUT, PATCH, DELETE) and status codes appropriately.

## 9. Version Control (Git)

**Rule:** Follow a structured Git workflow to maintain a clean and understandable history.

1.  **Branching:**
    *   All new work must be done on a feature or bugfix branch.
    *   Branch names should be descriptive and prefixed.
        *   Features: `feature/add-yield-savings`
        *   Bug Fixes: `bugfix/fix-swap-quote-error`
        *   Documentation: `docs/update-bridge-guide`

2.  **Commit Messages:**
    *   Use the [Conventional Commits](https://www.conventionalcommits.org/) specification.
    *   **Format:** `<type>: <subject>`
    *   **Types:** `feat` (new feature), `fix` (bug fix), `docs` (documentation), `style` (formatting), `refactor`, `test`, `chore` (build process, etc.).

    **Examples:**
    *   `feat(bridge): add support for SEPA external accounts`
    *   `fix(swaps): correct fee calculation for direct btc transfers`
    *   `docs(auth): update frontend guide for refresh tokens`

3.  **Pull Requests (PRs):**
    *   PRs should be small and focused on a single feature or fix.
    *   The PR description must clearly explain *what* was changed and *why*.
    *   Link the PR to the relevant issue or task.
    *   Ensure all automated checks (linting, building) pass before requesting a review.

## 10. Pre-Commit Checklist

Before committing or opening a PR, ensure:

*   [ ] `npm run lint` passes
*   [ ] `npm run format` has been run
*   [ ] `npm run build` succeeds
*   [ ] New environment variables are in `.env.example` and `src/config/validation.schema.ts`
*   [ ] Documentation is updated if API changes were made
