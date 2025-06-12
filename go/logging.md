# Go Logging Conventions

How we handle structured logging in Go applications.

## Library

We use **zerolog** for all structured logging:

```go
import "github.com/rs/zerolog"
```

## Logger Setup

Create a logger configuration in a dedicated package:

```go
package log

import (
	"os"
	"time"

	"github.com/rs/zerolog"
)

// NewLogger creates a configured zerolog.Logger instance
func NewLogger() zerolog.Logger {
	level := zerolog.InfoLevel
	if os.Getenv("DEBUG") == "true" {
		level = zerolog.DebugLevel

		logWriter := zerolog.ConsoleWriter{
			Out:        os.Stdout,
			TimeFormat: time.RFC3339,
		}

		return zerolog.New(logWriter).
			Level(level).
			With().
			Timestamp().
			Caller().
			Logger()
	}

	return zerolog.New(os.Stdout).
		Level(level).
		With().
		Timestamp().
		Logger()
}
```

### Environment-Based Configuration

- **Production**: JSON output to stdout, Info level
- **Development**: Pretty console output with caller info, Debug level
- **Control**: Use `DEBUG=true` environment variable

### Usage in Main

```go
func main() {
    logger := log.NewLogger()
    
    // Pass to handlers
    handler := NewHandler(service, &logger)
}
```

### Key Features

- **Timestamps**: Always included for log correlation
- **Caller info**: Only in debug mode (performance consideration)
- **Console writer**: Human-readable format for development
- **JSON output**: Machine-readable format for production

## Logging Location

Logging should primarily happen in the handler layer, closest to the user. This ensures:

- Consistent logging across all endpoints
- Proper context about user requests
- Clear separation of concerns
- Easy debugging of user-facing issues

## Handler Pattern

Always inject logger into handlers via constructor:

```go
type Handler struct {
    service Service
    logger  *zerolog.Logger
}

func NewHandler(service Service, logger *zerolog.Logger) *Handler {
    return &Handler{
        service: service,
        logger:  logger,
    }
}
```

## Log Levels

Use these levels consistently:

- **Debug**: Operation start with parameters
- **Info**: Successful operations with key identifiers
- **Error**: Failures with error details

## Message Patterns

### Debug Messages
Always log before executing an action using resource name + action in noun form:

```go
h.logger.Debug().
    Interface("params", input).
    Msg("ledger creation")
```

Common patterns:
- "resource creation"
- "resource updating" 
- "resource deletion"
- "resource retrieval"
- "resources listing"

### Info Messages
Log successful operations with resource identifiers:

```go
h.logger.Info().Str("uuid", account.UUID).Msg("account created")
h.logger.Info().Msg("accounts listed")
```

### Error Messages
Use consistent format: "unable to <verb> <resource> with ID <resourceID>"

```go
h.logger.Error().Err(err).Msg("unable to update ledger with UUID " + input.UUID)
```

## Field Naming

- Use `uuid` for resource identifiers
- Use `params` for user/request data
- Use `Err(err)` for error objects
- Use `Interface()` for complex objects
- Use `Str()` for string values

## Message Format

Keep messages:
- Lowercase
- Action-focused ("account created", "unable to create account")
- Consistent across similar operations
- Descriptive but concise

## CRUD Operations

Follow this pattern for standard CRUD operations:

```go
// CREATE
h.logger.Debug().Interface("params", input).Msg("account creation")
h.logger.Info().Str("uuid", account.UUID).Msg("account created")

// GET
h.logger.Debug().Interface("params", input).Msg("account retrieval")
h.logger.Info().Str("uuid", account.UUID).Msg("account found")

// LIST
h.logger.Debug().Interface("params", input).Msg("accounts listing")
h.logger.Info().Msg("accounts listed")

// UPDATE
h.logger.Debug().Interface("params", input).Msg("account updating")
h.logger.Info().Str("uuid", account.UUID).Msg("account updated")

// DELETE
h.logger.Debug().Interface("params", input).Msg("account deletion")
h.logger.Info().Str("uuid", input.UUID).Msg("account deleted")
```

## Error Handling

Always log errors before returning HTTP responses:

```go
account, err := h.service.GetAccount(input.UUID, input.LedgerUUID, userID)
if err != nil {
    h.logger.Error().Err(err).Msg("unable to get account")
    // handle HTTP response
}
```

## Architecture Pattern

We use the store -> service -> handler pattern:

- **Store**: Data access, returns domain errors
- **Service**: Business logic, passes through or wraps errors  
- **Handler**: HTTP layer, logs errors and converts to HTTP responses

### Error Flow Example

```go
// Service layer - business logic, minimal logging
func (s *AccountService) GetAccount(uuid, ledgerUUID, userID string) (*Account, error) {
    account, err := s.store.GetAccount(uuid, ledgerUUID)
    if err != nil {
        return nil, fmt.Errorf("failed to retrieve account: %w", err)
    }
    
    if account.UserID != userID {
        return nil, ErrUnauthorized
    }
    
    return account, nil
}

// Handler layer - logs and converts to HTTP
func (h *Handler) GetAccount(w http.ResponseWriter, r *http.Request) {
    account, err := h.service.GetAccount(input.UUID, input.LedgerUUID, userID)
    if err != nil {
        h.logger.Error().Err(err).Msg("unable to get account")
        
        if errors.Is(err, ErrUnauthorized) {
            http.Error(w, "Forbidden", http.StatusForbidden)
            return
        }
        
        http.Error(w, "Internal Server Error", http.StatusInternalServerError)
        return
    }
    
    h.logger.Info().Str("uuid", account.UUID).Msg("account found")
    // return success response
}
```

### Why This Works

- **Single responsibility**: Each layer has clear logging duties
- **Error context**: Service adds business context, handler adds operational context
- **Clean separation**: Business logic stays separate from HTTP concerns
- **Testability**: Easy to test each layer independently

## What NOT to Log

- Sensitive data (passwords, tokens)
- Large payloads in production
- Redundant success messages
