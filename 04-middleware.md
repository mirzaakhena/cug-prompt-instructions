# Middleware Implementation

## Overview
Middleware provides a way to wrap usecases with additional functionality such as transaction management, logging, and retry logic. Middleware acts as a decorator pattern, enhancing behavior without changing the usecase's core functionality.

## ✅ REQUIRED
- Implement middleware as higher-order functions
- Accept and return `core.ActionHandler[R, S]`
- Execute the usecase with the same parameters as the middleware
- Maintain the usecase's functional signature
- Use generic type parameters for flexibility

## ❌ FORBIDDEN
- NEVER create middleware that depends on specific usecase implementations
- NEVER create permanent side effects from middleware
- NEVER fundamentally change the usecase's behavior

## 💡 IMPORTANT TO REMEMBER
- Middleware is a wrapper around usecases that adds cross-cutting functionality
- The core transaction middleware is the most important and should be applied to usecases that perform multiple database write operations
- When applying multiple middleware, consider the order of application

## Common Middleware Types

### 1. Transaction Middleware
Ensures database operations are performed within a transaction, allowing for atomic operations and automatic rollback on errors. See `01-core-components.md` for detailed implementation guidance and examples for transaction usage.

```go
// In middleware/transaction.go - Implementation overview
func Transaction[R any, S any](actionHandler core.ActionHandler[R, S], db *gorm.DB) core.ActionHandler[R, S] {
    return func(ctx context.Context, request R) (*S, error) {
        // Executes the provided actionHandler within a database transaction context
        // Automatically rolls back on errors or commits on success
        // Transaction context is passed to actionHandler via ctx parameter
    }
}
```

### 2. Logging Middleware
Provides logging before and after usecase execution, capturing request details, response, errors, and timing information.

```go
func Logging[R any, S any](actionHandler core.ActionHandler[R, S], logger *log.Logger) core.ActionHandler[R, S] {
    return func(ctx context.Context, request R) (*S, error) {
        // Pre-execution logging
        requestID := uuid.New().String()
        logger.Printf("[%s] Starting usecase with request: %+v", requestID, request)
        startTime := time.Now()

        // Execute the usecase
        result, err := actionHandler(ctx, request)

        // Post-execution logging
        duration := time.Since(startTime)
        if err != nil {
            logger.Printf("[%s] Usecase failed after %v: %v", requestID, duration, err)
        } else {
            logger.Printf("[%s] Usecase completed in %v with result: %+v", requestID, duration, result)
        }

        return result, err
    }
}
```

### 3. Retry Middleware
Implements automatic retry logic for operations that might fail due to transient issues.

```go
func Retry[R any, S any](actionHandler core.ActionHandler[R, S], maxRetries int, retryDelay time.Duration) core.ActionHandler[R, S] {
    return func(ctx context.Context, request R) (*S, error) {
        var result *S
        var err error
        
        for attempt := 0; attempt <= maxRetries; attempt++ {
            if attempt > 0 {
                // Wait before retrying
                time.Sleep(retryDelay * time.Duration(attempt))
            }
            
            result, err = actionHandler(ctx, request)
            
            // If no error or not a retryable error, return immediately
            if err == nil || !isRetryableError(err) {
                return result, err
            }
        }
        
        return result, err
    }
}

func isRetryableError(err error) bool {
    // Implement logic to determine if an error is retryable
    // Examples of retryable errors: temporary network issues, database deadlocks
    // Examples of non-retryable errors: validation errors, not found errors
    return false // Default implementation
}
```

## Middleware Application Order

When applying multiple middleware to a usecase, the order matters. The general rule is:

```
OutermostMiddleware(
    NextMiddleware(
        InnermostMiddleware(usecase)
    )
)
```

Recommended order from outermost to innermost:
1. Logging (outermost)
2. Timing
3. Authorization/Authentication
4. Validation
5. Retry
6. Transaction (innermost)

This order ensures that:
- Logging captures everything, including other middleware behavior
- Transactions are properly managed with minimal overhead
- Retries happen within the context of proper authentication but before transaction boundaries

## Example Usage in Wiring

```go
// In wiring/setup.go
func SetupWiring(mux *http.ServeMux, db *gorm.DB, logger *log.Logger) {
    // Initialize gateways
    userFindByEmail := gateway.ImplUserFindByEmail(db)
    userCreate := gateway.ImplUserCreate(db)
    systemGenerateUUID := gateway.ImplSystemGenerateUUID()
    
    // Initialize base usecase
    createUserUsecase := usecase.ImplCreateUser(
        userFindByEmail,
        userCreate,
        systemGenerateUUID,
    )
    
    // Apply middleware in the correct order
    createUserWithLogging := middleware.Logging(createUserUsecase, logger)
    createUserWithTransaction := middleware.Transaction(createUserWithLogging, db)
    
    // Register with controller
    controller.CreateUserController(mux, createUserWithTransaction)
}
```