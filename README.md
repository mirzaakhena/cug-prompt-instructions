# CUG Prompt Instructions

## Overview
CUG Prompt Instructions is a comprehensive guide for implementing Clean Architecture in Golang application development. This guide provides clear structure, patterns, and rules for building modular, testable, and maintainable systems.

## Documentation Structure

This repository contains a series of interconnected guidance documents:

1. **[00-main-rules.md](./00-main-rules.md)** - Main rules and basic terminology.
2. **[01-core-components.md](./01-core-components.md)** - Core component definitions: ActionHandler, Gateway, and Usecase.
3. **[02-usecase-test.md](./02-usecase-test.md)** - Unit testing guidelines for usecases.
4. **[03-controller.md](./03-controller.md)** - Controller implementation for HTTP, scheduler, and message subscribers.
5. **[04-middleware.md](./04-middleware.md)** - Middleware for transactions, logging, and retry.
6. **[05-wiring.md](./05-wiring.md)** - Dependency injection and component composition.

## Architecture

```
├── controller/ # Exposes usecases to the outside world
├── core/       # Contains ActionHandler definition
├── gateway/    # External interfaces and system utilities
├── middleware/ # Cross-cutting concerns (transaction, logging, retry)
├── model/      # Domain entities
├── usecase/    # Business logic
├── utils/      # Helper functions
└── wiring/     # Connects components
```

## Core Principles

### 1. Component Separation
- **Controller**: Exposes usecases via protocols (HTTP, MQTT, schedulers, etc.)
- **Usecase**: Pure business logic, orchestrates gateways, no infrastructure dependencies
- **Gateway**: Handles infrastructure and non-deterministic functions
- **Middleware**: Decorators for cross-cutting concerns (transactions, logging, etc.)
- **Wiring**: Dependency injection and component composition

### 2. ActionHandler Pattern
All components (usecases and gateways) implement a consistent ActionHandler pattern:

```go
type ActionHandler[REQUEST any, RESPONSE any] func(ctx context.Context, request REQUEST) (*RESPONSE, error)
```

### 3. Development Workflow
1. Analyze requirements thoroughly
2. Read core documentation
3. Scan existing code
4. Plan implementation strategy
5. Check module imports
6. Implement components
7. Implement unit tests
8. Run code validation

## Component Guidelines

### Usecases
- Contain pure business logic
- Never access infrastructure directly
- Use gateways for infrastructure operations
- Implement input validation
- Follow standard business logic flow

### Gateways
- Handle infrastructure operations
- One task per gateway
- Use transaction middleware for DB operations
- Wrap errors with context
- No business logic

### Controllers
- One controller calls exactly one usecase
- No business logic
- Handle protocol-specific concerns
- Implement proper error handling

### Middleware
- Implement decorator pattern
- Add cross-cutting functionality
- Apply middleware in the correct order

### Wiring
- Initialize all components
- Inject dependencies
- Apply middleware
- Register controllers

## Testing Practices
- Test each usecase in isolation
- Mock all gateway dependencies
- Test both success and error paths
- Structure tests consistently
- Use table-driven approach

## Development Principles
- Prioritize component reuse
- Follow consistent naming conventions
- Maintain strict separation between usecase and gateway
- Avoid dependency cycles
- No global state sharing
- Make usecase functionality pure functions
- Manage database transactions carefully

## Contribution
Before implementation, ensure you thoroughly understand these guidelines and follow the mandatory development workflow outlined in `00-main-rules.md`.
