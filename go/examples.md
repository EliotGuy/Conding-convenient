# Go Examples

Real-world examples showing how to implement our Go conventions.

## Complete Service Example

This example shows a complete implementation of the account service following our conventions.

### types.go

```go
package account

import (
	"context"

	"github.com/j0lvera/go-budgetwise/pkg/pagination"
)

// Result objects
type (
	// ListOutput represents a paginated list of items
	ListOutput[T any, P any] struct {
		Items      []T `json:"items"`
		Pagination P   `json:"pagination"`
	}

	// GetOutput represents the response for a single account
	GetOutput struct {
		UUID        string `json:"uuid"`
		Name        string `json:"name"`
		Description string `json:"description"`
		Type        string `json:"type"`
		LedgerUUID  string `json:"ledgerUUID"`
	}

	// CreateOutput is an alias for GetOutput
	CreateOutput = GetOutput
)

// Parameter objects
type (
	// GetInput for retrieving a single account
	GetInput struct {
		UUID       string `param:"uuid" validate:"required"`
		LedgerUUID string `query:"ledgerUUID" validate:"required"`
	}

	// CreateInput for creating a new account
	CreateInput struct {
		Name        string `json:"name" validate:"required,min=1,max=255"`
		Description string `json:"description,omitempty" validate:"omitempty,max=255"`
		Type        string `json:"type" validate:"required,oneof=asset liability income equity"`
		LedgerUUID  string `json:"ledgerUUID" validate:"required"`
	}

	// UpdateInput for updating an existing account
	UpdateInput struct {
		UUID        string `param:"uuid" validate:"required"`
		Name        string `json:"name,omitempty" validate:"omitempty,min=1,max=255"`
		Description string `json:"description,omitempty" validate:"omitempty,max=255"`
		Type        string `json:"type,omitempty" validate:"omitempty,oneof=asset liability income equity"`
		LedgerUUID  string `json:"ledgerUUID" validate:"required"`
	}

	// DeleteInput for deleting an account
	DeleteInput struct {
		GetInput
	}

	// ListInput for listing accounts with pagination
	ListInput[P any] struct {
		Pagination P
		LedgerUUID string `query:"ledgerUUID" validate:"required"`
		Type       string `query:"type" validate:"required"`
	}
)

// Service defines the interface for account service operations
type Service interface {
	GetAccount(uuid string, ledgerUUID string, userID int64) (*GetOutput, error)
	CreateAccount(input CreateInput, userID int64) (*GetOutput, error)
	ListAccounts(input ListInput[pagination.Input], userID int64) (*ListOutput[GetOutput, pagination.Output], error)
	UpdateAccount(input UpdateInput, userID int64) (*GetOutput, error)
	DeleteAccount(uuid string, ledgerUUID string, userID int64) error
}

// Store defines the interface for account repository operations
type Store interface {
	GetAccount(
		ctx context.Context, uuid string, ledgerUUID string, userID int64,
	) (*GetOutput, error)

	CreateAccount(
		ctx context.Context, name, description, accountType, ledgerUUID string,
		userID int64,
	) (*GetOutput, error)

	ListAccounts(
		ctx context.Context, ledgerUUID, accountType string,
		limit, offset int, sortBy, sortDirection, filter string, userID int64,
	) ([]GetOutput, int64, error)

	UpdateAccount(
		ctx context.Context, input UpdateInput, userID int64,
	) (*GetOutput, error)

	DeleteAccount(
		ctx context.Context, uuid, ledgerUUID string, userID int64,
	) error

	VerifyLedgerOwnership(
		ctx context.Context, ledgerUUID string, userID int64,
	) (bool, error)
}
```

### service.go

```go
package account

import (
	"context"

	"github.com/j0lvera/go-budgetwise/pkg/pagination"
	"github.com/rs/zerolog"
)

type Config struct {
	store  Store
	logger *zerolog.Logger
}

type service struct {
	cfg Config
}

func NewService(store Store, logger *zerolog.Logger) Service {
	return &service{
		cfg: Config{
			store:  store,
			logger: logger,
		},
	}
}

func (s *service) GetAccount(
	uuid string, ledgerUUID string, userID int64,
) (*GetOutput, error) {
	ctx := context.Background()

	account, err := s.cfg.store.GetAccount(ctx, uuid, ledgerUUID, userID)
	if err != nil {
		return nil, err
	}

	return account, nil
}

func (s *service) CreateAccount(input CreateInput, userID int64) (
	*GetOutput, error,
) {
	ctx := context.Background()

	// First verify that the ledger belongs to this user
	_, err := s.cfg.store.VerifyLedgerOwnership(ctx, input.LedgerUUID, userID)
	if err != nil {
		return nil, err
	}

	account, err := s.cfg.store.CreateAccount(
		ctx,
		input.Name,
		input.Description,
		input.Type,
		input.LedgerUUID,
		userID,
	)
	if err != nil {
		return nil, err
	}

	return account, nil
}

func (s *service) ListAccounts(
	input ListInput[pagination.Input], userID int64,
) (*ListOutput[GetOutput, pagination.Output], error) {
	ctx := context.Background()

	// Set default values if not provided
	limit := input.Pagination.Limit
	if limit <= 0 {
		limit = 10
	}

	offset := input.Pagination.Offset
	if offset < 0 {
		offset = 0
	}

	sortBy := input.Pagination.SortBy
	if sortBy == "" {
		sortBy = "created_at"
	}

	sortDirection := input.Pagination.SortDirection
	if sortDirection == "" {
		sortDirection = "desc"
	}

	accounts, totalCount, err := s.cfg.store.ListAccounts(
		ctx,
		input.LedgerUUID,
		input.Type,
		limit,
		offset,
		sortBy,
		sortDirection,
		input.Pagination.Filter,
		userID,
	)
	if err != nil {
		return nil, err
	}

	return &ListOutput[GetOutput, pagination.Output]{
		Items: accounts,
		Pagination: pagination.Output{
			Limit:      limit,
			Offset:     offset,
			TotalCount: totalCount,
		},
	}, nil
}

func (s *service) UpdateAccount(input UpdateInput, userID int64) (
	*GetOutput, error,
) {
	ctx := context.Background()

	// First verify that the ledger belongs to this user if provided
	if input.LedgerUUID != "" {
		_, err := s.cfg.store.VerifyLedgerOwnership(
			ctx, input.LedgerUUID, userID,
		)
		if err != nil {
			return nil, err
		}
	}

	account, err := s.cfg.store.UpdateAccount(ctx, input, userID)
	if err != nil {
		return nil, err
	}

	return account, nil
}

func (s *service) DeleteAccount(
	uuid string, ledgerUUID string, userID int64,
) error {
	ctx := context.Background()

	// First verify that the ledger belongs to this user
	_, err := s.cfg.store.VerifyLedgerOwnership(ctx, ledgerUUID, userID)
	if err != nil {
		return err
	}

	// Now verify that the account exists and belongs to this user
	_, err = s.cfg.store.GetAccount(ctx, uuid, ledgerUUID, userID)
	if err != nil {
		return err
	}

	return s.cfg.store.DeleteAccount(ctx, uuid, ledgerUUID, userID)
}
```

### store.go

```go
package account

import (
	"context"
	"errors"

	"github.com/j0lvera/go-budgetwise/internal/db"
	dbgen "github.com/j0lvera/go-budgetwise/internal/db/gen"
	"github.com/jackc/pgx/v5/pgtype"
)

type store struct {
	db *db.Client
}

func NewStore(db *db.Client) Store {
	return &store{
		db: db,
	}
}

func (s *store) GetAccount(
	ctx context.Context, uuid string, ledgerUUID string, userID int64,
) (*GetOutput, error) {
	account, err := s.db.Queries.GetAccount(
		ctx, dbgen.GetAccountParams{
			Uuid:       uuid,
			LedgerUuid: ledgerUUID,
			UserID:     userID,
		},
	)
	if err != nil {
		return nil, err
	}
	
	return &GetOutput{
		UUID:        account.Uuid,
		Name:        account.Name,
		Description: account.Description.String,
		Type:        account.Type,
		LedgerUUID:  ledgerUUID,
	}, nil
}

func (s *store) ListAccounts(
	ctx context.Context, ledgerUUID, accountType string, 
	limit, offset int, sortBy, sortDirection, filter string, userID int64,
) ([]GetOutput, int64, error) {
	params := dbgen.ListAccountsParams{
		LedgerUuid: ledgerUUID,
		Limit:      int32(limit),
		Offset:     int32(offset),
		SortBy:     pgtype.Text{String: sortBy, Valid: true},
		SortDirection: pgtype.Text{
			String: sortDirection,
			Valid:  true,
		},
		Filter: pgtype.Text{
			String: filter,
			Valid:  filter != "",
		},
		Type:   accountType,
		UserID: userID,
	}
	
	accounts, err := s.db.Queries.ListAccounts(ctx, params)
	if err != nil {
		return nil, 0, err
	}
	
	// Handle empty results - return empty slice, not nil
	if len(accounts) == 0 {
		return []GetOutput{}, 0, nil
	}
	
	var totalCount int64
	if len(accounts) > 0 {
		totalCount = accounts[0].TotalCount
	}
	
	result := make([]GetOutput, len(accounts))
	for i, account := range accounts {
		result[i] = GetOutput{
			UUID:        account.Uuid,
			Name:        account.Name,
			Description: account.Description.String,
			Type:        account.Type,
			LedgerUUID:  account.LedgerUuid,
		}
	}
	
	return result, totalCount, nil
}

// ... other store methods
```

### handler.go

```go
package account

import (
	"net/http"

	"github.com/j0lvera/go-budgetwise/internal/apierr"
	"github.com/j0lvera/go-budgetwise/internal/auth"
	"github.com/j0lvera/go-budgetwise/internal/ctxutil"
	"github.com/j0lvera/go-budgetwise/pkg/pagination"
	"github.com/labstack/echo/v4"
	"github.com/rs/zerolog"
)

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

func (h *Handler) Get() echo.HandlerFunc {
	return func(c echo.Context) error {
		input := ctxutil.GetInput[GetInput](c)
		userData := ctxutil.GetUser[auth.UserData](c)
		userID := userData.ID

		h.logger.Debug().
			Interface("params", input).
			Msg("account retrieval")

		account, err := h.service.GetAccount(input.UUID, input.LedgerUUID, userID)
		if err != nil {
			if err.Error() == "no rows in result set" {
				h.logger.Debug().Str("uuid", input.UUID).Msg("account not found")
				return c.JSON(http.StatusNotFound, apierr.ErrNotFound)
			}
			h.logger.Error().Err(err).Msg("unable to get account with UUID " + input.UUID)
			return c.JSON(http.StatusInternalServerError, apierr.Internal)
		}

		h.logger.Debug().Str("uuid", input.UUID).Msg("account found")
		return c.JSON(http.StatusOK, account)
	}
}

func (h *Handler) Create() echo.HandlerFunc {
	return func(c echo.Context) error {
		input := ctxutil.GetInput[CreateInput](c)
		userData := ctxutil.GetUser[auth.UserData](c)
		userID := userData.ID

		h.logger.Debug().
			Interface("params", input).
			Msg("account creation")

		account, err := h.service.CreateAccount(input, userID)
		if err != nil {
			if err.Error() == "ledger not found" {
				h.logger.Debug().Str("ledgerUUID", input.LedgerUUID).Msg("ledger not found")
				return c.JSON(http.StatusNotFound, apierr.ErrNotFound)
			}
			h.logger.Error().Err(err).Msg("unable to create account")
			return c.JSON(http.StatusInternalServerError, apierr.Internal)
		}

		h.logger.Debug().Str("uuid", account.UUID).Msg("account creation completed")
		return c.JSON(http.StatusCreated, account)
	}
}

func (h *Handler) List() echo.HandlerFunc {
	return func(c echo.Context) error {
		input := ctxutil.GetInput[ListInput[pagination.Input]](c)
		userData := ctxutil.GetUser[auth.UserData](c)
		userID := userData.ID

		h.logger.Debug().
			Interface("params", input).
			Msg("accounts listing")

		result, err := h.service.ListAccounts(input, userID)
		if err != nil {
			h.logger.Error().Err(err).Msg("unable to list accounts")
			return c.JSON(http.StatusInternalServerError, apierr.Internal)
		}

		h.logger.Debug().Msg("accounts listing completed")
		return c.JSON(http.StatusOK, result)
	}
}
```

## Key Patterns Demonstrated

### 1. **Proper Error Handling**
- Handle each error only once
- Use specific error messages with resource IDs
- Log at appropriate levels (Debug for expected cases, Error for unexpected)

### 2. **Logging Conventions**
- Debug logging before operations with input params
- Debug logging after successful operations
- Error logging with specific resource identifiers
- Use noun form for action descriptions

### 3. **Return Type Patterns**
- Single objects return pointers (`*GetOutput`)
- Collections return slices of values (`[]GetOutput`)
- Empty collections return `[]Type{}`, not `nil`

### 4. **Architecture Separation**
- Handler: HTTP concerns only
- Service: Business logic and validation
- Store: Data access, returns domain types
- Types: Interfaces and data structures

### 5. **Builder Pattern**
- Unexported implementation structs
- Constructors return interface types
- Config struct for dependencies
- Always include logger in all layers

This example shows how all the conventions work together in a real-world service implementation.
