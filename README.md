# Guide for Simple, Maintainable, and Scalable Golang DDD

---

## Introduction

The approach detailed in this guide represents what I currently believe is the best way to design and build software using Go and Domain-Driven Design. It is based on practical experience and resources like [sklinkert/go-ddd](https://github.com/sklinkert/go-ddd). I’ve adopted what works well and adapted other elements to arrive at what I believe is an optimal approach.

---

Core Principles
---------------

Before diving into the structure, let's define the core principles that underpin my approach:

-   **Domain-Driven Design:** I believe that the most effective software is that which closely models the business domain it serves. DDD provides the tools and patterns to achieve this.

-   **Simplicity:** I favor simple solutions over complex ones. Complexity should only be introduced when it provides a clear and substantial benefit.

-   **Maintainability:** Our code should be easy to understand, modify, and extend. This requires clean architecture, clear naming conventions, and comprehensive testing.

-   **Scalability:** Our systems should be able to handle increasing load and complexity without requiring major architectural changes.

-   **Explicit Dependencies:** Dependencies between components should be clear and well-defined, minimizing coupling and promoting modularity.

-   **Layered Architecture:** I organize the code into distinct layers, each with a specific responsibility. This promotes separation of concerns and makes it easier to reason about the system.

---

## Project Structure

The project structure provides a consistent and predictable way to organize code, making it easier to navigate and understand different projects. Here's a detailed breakdown:

```
├── cmd
│   ├── desktop       # Application entry point for desktop applications
│   │   └── main.go
│   ├── web_app       # Application entry point for web application
│   │   └── main.go
│   ├── api           # Application entry point for api
│   │   └── main.go
│   └── cli           # Application entry point for command-line applications
│       └── main.go
├── config            # Configuration loading and management
│   ├── database.go     # Example: Configuration specific to database connection
│   └── whatsapp.go     # Example: Configuration specific to WhatsApp integration
├── internal          # Internal application code (not intended for external use)
│   ├── application     # Application-level services (use cases, orchestration)
│   │   ├── chatbot_service.go  # Handles chatbot workflows (message processing, responses)
│   │   ├── user_service.go   # Coordinates user-related operations (create, update, fetch)
│   │   └── auth_service.go     # Manages authentication (login, signup, token issuance)
│   ├── domain          # Domain logic (entities, value objects, interfaces)
│   │   ├── chat.go        # Example: Domain model for a Chat
│   │   ├── message.go     # Example: Domain model for a Message
│   │   ├── user.go      # Example: Domain model for a User
│   │   └── role.go        # Example: Domain model for a Role
│   ├── infrastructure    # Implementation of external dependencies (databases, APIs, etc.)
│   │   ├── ai           # Integration with AI services
│   │   │   ├── client.go  # Client for the AI service
│   │   │   ├── message.go # Handling AI-specific message structures
│   │   │   ├── persona.go # Managing AI personas
│   │   │   └── role.go    # AI-specific roles
│   │   ├── database      # Database interaction (e.g., PostgreSQL)
│   │   │   ├── client.go  # Database client
│   │   │   └── user_repository.go # Implementation of UserRepository
│   │   ├── messaging     # Generic messaging infrastructure
│   │   │   └── sender.go    # Interface and common logic for message sending
│   │   ├── telegram      # Integration with Telegram
│   │   │   ├── chat.go    # Telegram-specific chat handling
│   │   │   └── client.go  # Telegram client
│   │   └── whatsapp      # Integration with WhatsApp
│   │       ├── chat.go    # WhatsApp-specific chat handling
│   │       ├── client.go  # WhatsApp client
│   │       └── event.go   # Handling WhatsApp events
│   └── interface       # Adapters for external systems (e.g., HTTP, CLI)
│       ├── cli           # Command-line interface adapters
│       │   └── user_cli_adapter.go # Example: CLI adapter for User-related commands
│       └── http          # HTTP interface adapters
│           ├── html      # for http requests that return HTML
│           │   └── user_http_adapter.go
│           └── api       # for http requests that return JSON
│               └── user_http_adapter.go
├── pkg               # Generic, reusable code with no knowledge of this app. Think small utilities you could publish as libraries. No domain logic, no tight coupling to internal packages.
│   ├── debounce               # Time-based call coalescing (debounce.New)
│   │   └── debounce.go        # Debouncer implementation
│   └── retry                  # Retry logic with backoff strategies (retry.Do)
│       └── retry.go           # Retry helpers and policies
├── go.mod           # Go module definition
└── go.sum           # Go module dependencies
```

### Key Directories Explained

-   **cmd:** This directory contains the main entry points for the applications. Each subdirectory represents a different way of running the application (e.g., `cli` for command-line interface, `api` for a web server, `desktop` for a desktop application). This allows me to have multiple ways to run the same underlying application logic. I strive to keep this directory very thin, with minimal logic. Its primary responsibility is to initialize the necessary dependencies and start the application.

-   **config:** This directory holds the application's configuration. I typically use a library like `godotenv` to load configuration from environment variables, files, or other sources. Configuration is structured to be easily accessible and type-safe. For example, `whatsapp.go` might define a struct with fields like `APIKey`, `PhoneNumber`, and `Timeout`.

-   **internal:** This is where the core application logic resides. Code within this directory is not intended to be accessed from outside the module. This enforces a strong boundary and prevents accidental coupling.

    -   **application:** This layer contains the use cases of the application. It orchestrates the interaction between the domain layer and the infrastructure layer. Application services are responsible for handling business transactions, validation (that *isn't* domain-level), and coordinating the work of multiple domain objects.

        -   I keep the application layer as a single, flat package. This is because, in my experience, the services within this layer tend to be highly interconnected. Creating subpackages often leads to unnecessary complexity and circular dependencies. I use clear naming conventions (e.g., `ChatbotService`, `UserService`) to organize the code within this package.

    -   **domain:** This is the heart of the application. It contains the business logic, represented as entities, value objects, and interfaces. The domain layer is completely independent of any technical details, such as databases or web frameworks. It defines *what* the application does, not *how* it does it.

        -   Like the application layer, I also keep the domain layer as a single, flat package. I find that this simplifies the organization of domain concepts and reduces the likelihood of circular dependencies.

        -   **Entities:** Represent objects with a unique identity and lifecycle (e.g., a `User`, an `Order`). Entities encapsulate both data and behavior. Validation logic that enforces business rules is often placed within entity methods. For example, a `User` entity might have a `ChangeName(newName string) error` method that validates the new name before updating the entity's state.

        -   **Value Objects:** Represent immutable objects that are defined by their attributes rather than their identity (e.g., a `Date`, a `Money` amount). Value objects should be treated as interchangeable; two value objects with the same attributes are considered equal.

        -   **Interfaces:** Define contracts that are implemented by the infrastructure layer. For example, the domain layer might define a `MessageSender` interface with a `SendMessage(user User, message Message) error` method. This allows the application layer to depend on the *abstraction* of message sending, without being tied to a specific *implementation* (e.g., sending via WhatsApp or Telegram). This is a key aspect of achieving loose coupling and testability.

    -   **infrastructure:** This layer provides the technical implementation of the interfaces defined in the domain layer. It handles communication with external systems, such as databases, message queues, and third-party APIs. Code in this layer is responsible for *how* things are done.

        -   I organize infrastructure implementations into subpackages based on the external system they interact with (e.g., `database`, `messaging`, `telegram`, `whatsapp`, `ai`). This keeps the code organized and makes it easy to find the implementation for a specific dependency. Within each subpackage, I may have further organization as needed, but I avoid creating excessively deep nesting. For example, the `database` package might contain a `client.go` for establishing the database connection, `models.go` for defining database-specific data structures (if necessary), and `user_repository.go` for implementing the `UserRepository` interface defined in the domain layer. The key is to maintain a balance between organization and simplicity.

    -   **interface:** This layer acts as the entry point for external requests and translates them into commands that can be processed by the application layer. It's responsible for handling the specifics of the communication protocol (e.g., HTTP, CLI) and for presenting the application's response to the user.

        -   I structure this layer by the type of interface (e.g., `cli`, `http`). Within each interface type, I may further organize the code by specific functionality or technology.

        -   For example, the `http` package might be further divided into `html` and `api` subpackages to handle different response formats. Each of these subpackages would then contain adapters for specific resources (e.g., `user_http_adapter.go`).

        -   The `cli` package would contain adapters that handle argument parsing, command dispatching, and output formatting. For example, `user_cli_adapter.go` might define functions to handle commands related to creating, updating, or deleting `User` entities. It acts as an intermediary between the user's terminal input and the application layer.

-   **pkg:** This directory contains reusable packages that are not specific to the application's domain. These packages can be used across multiple projects. Examples include utility functions, generic data structures, and third-party library wrappers. Code in this directory should not depend on any code in the `internal` directory. 

---

### Dependency Rules

To maintain a clean architecture and prevent tight coupling, I adhere to the following dependency rules:

-   **domain:** The domain layer should not depend on any other layer. It is the most fundamental layer and should be completely self-contained. This ensures that the core business logic remains independent of any technical details.

-   **application:** The application layer can only depend on the domain layer. It orchestrates domain logic but should not be aware of the implementation details of external systems. If the application layer needs to interact with an external service, the interface for that service is defined in the domain layer, and the application layer depends on that interface.

-   **interface:** Interface layers (e.g., `cli`, `http`) can depend on both the domain layer and the application layer. They translate external requests into application commands and present the results to the user.

-   **infrastructure:** Infrastructure packages can only depend on the domain layer. They implement the interfaces defined in the domain layer and may use external libraries or frameworks to interact with external systems. They should not depend on the application or interface layers.

These rules ensure that the domain layer remains pure and that the application logic is decoupled from the infrastructure. This makes the code more modular, testable, and maintainable.

---

Layered Architecture
--------------------

The architecture follows a layered pattern, where each layer has a specific responsibility and interacts with other layers in a defined way. This approach promotes separation of concerns, making the code easier to understand, maintain, and evolve.

Here's a summary of the layers and their responsibilities:

1.  **Domain Layer:**

    -   Contains the core business logic of the application.

    -   Defines entities, value objects, and interfaces.

    -   Is completely independent of any technical details.

    -   Focuses on *what* the application does.

2.  **Application Layer:**

    -   Orchestrates the interaction between the domain layer and the infrastructure layer.

    -   Implements use cases and business transactions.

    -   Handles application-level validation and error handling.

    -   Focuses on *how* the application uses the domain logic.

3.  **Infrastructure Layer:**

    -   Provides the technical implementation of the interfaces defined in the domain layer.

    -   Handles communication with external systems (e.g., databases, APIs, message queues).

    -   Focuses on *how* the application interacts with the outside world.

4.  **Interface Layer:**

    -   Acts as the entry point for external requests.

    -   Translates requests into application commands.

    -   Presents the application's response to the user.

    -   Handles the specifics of the communication protocol (e.g., HTTP, CLI).

Each layer has a well-defined responsibility, and the layers interact with each other in a controlled manner. This layered architecture provides several benefits:

-   **Separation of Concerns:** Each layer focuses on a specific aspect of the application, making the code easier to understand and maintain.

-   **Loose Coupling:** Layers are only coupled to the layers directly below them, reducing the impact of changes in one layer on other layers.

-   **Testability:** Each layer can be tested independently, making it easier to isolate and fix bugs.

-   **Flexibility:** The implementation of one layer can be changed without affecting other layers, as long as the interface between the layers remains the same.


---

## Example: User Management

To illustrate how these principles and patterns work in practice, let's consider a simple example: user management.

### Domain Layer

```go
// internal/domain/user.go
package domain

import (
	"errors"
	"github.com/google/uuid"
)

// User is an entity that represents a user in the system.
type User struct {
    ID    string
    Name  string
    Email string
    Role  Role
}

func NewUser(name string, email string, role Role) (User, error) {
    if len(name) < 3 {
        return User{}, ErrInvalidName
    }
    return User{
        ID:    uuid.New().String(),
        Name:  name,
        Email: email,
        Role:  role,
    }, nil
}

// Role is a value object that represents the role of a user.
type Role string

const (
    RoleAdmin = "admin"
    RoleUser  = "user"
)

// ErrInvalidName is returned when a name is invalid.
var ErrInvalidName = errors.New("name must be at least 3 characters long")

// ChangeName changes the name of the user.
func (u *User) ChangeName(newName string) error {
    if len(newName) < 3 {
        return ErrInvalidName
    }
    u.Name = newName
    return nil
}

// UserRepository is an interface for persisting User entities.
type UserRepository interface {
    GetByID(id string) (User, error)
    Save(user User) error
}

```

In this example, `User` is an entity with an `ID`, `Name`, `Email`, and `Role`. The `ChangeName` method enforces a business rule: the name must be at least 3 characters long. `Role` is a value object. `UserRepository` is an interface for persisting `User` entities.

### Application Layer

```go
// internal/application/user.go
package application

import (
    "example.com/internal/domain"
    "errors"
)

// UserService provides use cases for managing users.
type UserService struct {
    userRepo domain.UserRepository
}

// NewUserService creates a new UserService.
func NewUserService(userRepo domain.UserRepository) *UserService {
    return &UserService{
        userRepo: userRepo,
    }
}

// GetUserByID retrieves a user by their ID.
func (s *UserService) GetUserByID(id string) (domain.User, error) {
    user, err := s.userRepo.GetByID(id)
    if err != nil {
        return domain.User{}, err
    }
    return user, nil
}

// UpdateUserName updates the name of a user.
func (s *UserService) UpdateUserName(id, newName string) error {
    user, err := s.GetUserByID(id)
    if err != nil {
        return err
    }

    err = user.ChangeName(newName)
    if err != nil {
        return err // Return the error from the domain layer
    }

    return s.userRepo.Save(user)
}

// CreateUser creates a new user.
func (s *UserService) CreateUser(name, email string, role domain.Role) (string, error) {
    // Application level validation
    if name == "" || email == "" || role == "" {
        return "", errors.New("name, email and role must not be empty")
    }

    user, err := domain.NewUser(name, email, role)
    if err != nil {
    	return "", err
    }

    err = s.userRepo.Save(user)
    if err != nil {
        return "", err
    }
    return user.ID, nil
}

```

The `UserService` implements use cases for managing `User` entities. It uses the `UserRepository` interface to interact with the persistence layer. It also handles application-level validation, handles errors cleanly without leaking database types, and ensures that domain logic is correctly applied."

### Infrastructure Layer

```go
// internal/infrastructure/database/user_repository.go
package database

import (
    "example.com/internal/domain"
    "database/sql"
    "errors"
)

// UserRepository is an implementation of the domain.UserRepository interface.
type UserRepository struct {
    db *sql.DB
}

// NewUserRepository creates a new UserRepository.
func NewUserRepository(db *sql.DB) *UserRepository {
    return &UserRepository{
        db: db,
    }
}

// GetByID retrieves a user from the database by their ID.
func (r *UserRepository) GetByID(id string) (domain.User, error) {
    row := r.db.QueryRow("SELECT id, name, email, role FROM users WHERE id = $1", id)
    var user domain.User
    err := row.Scan(&user.ID, &user.Name, &user.Email, &user.Role)
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return domain.User{}, domain.ErrItemNotFound // Use the error from the domain
        }
        return domain.User{}, err
    }
    return user, nil
}

// Save saves a user to the database.
func (r *UserRepository) Save(user domain.User) error {
    _, err := r.db.Exec(
        "INSERT INTO users (id, name, email, role) VALUES ($1, $2, $3, $4) "+
            "ON CONFLICT (id) DO UPDATE SET name = $2, email = $3, role = $4",
        user.ID, user.Name, user.Email, user.Role,
    )
    return err
}

```

The `UserRepository` in the `database` package implements the `domain.UserRepository` interface using a PostgreSQL database. It handles the mapping between the domain model (`domain.User`) and the database schema. Error handling is crucial here. I translate database-specific errors (e.g., `sql.ErrNoRows`) into domain-specific errors (e.g., `domain.ErrItemNotFound`) to avoid leaking implementation details.

### Interface Layer

#### CLI Example (Using Cobra framework)

The `UserCLIAdapter` handles command-line arguments, validates command arguments, and dispatches them to the appropriate `UserService` methods.

```go
// cmd/cli/main.go
package main

import (
	...
)

func main() {
	db, err := database.NewClient(config.Database)
	if err != nil {
		fmt.Println("error connecting to database:", err)
		os.Exit(1)
	}
	defer db.Close()

	userRepo := database.NewUserRepository(db)
	userService := application.NewUserService(userRepo)
	userAdapter := cli.NewUserCLIAdapter(userService)

	rootCmd := &cobra.Command{
		Use: "app",
	}

	rootCmd.AddCommand(
		userAdapter.Command(),
		// orderAdapter.Command(),
		// billingAdapter.Command(),
	)

	if err := rootCmd.Execute(); err != nil {
		fmt.Println("error:", err)
		os.Exit(1)
	}
}
```

```go
// internal/interface/cli/user_cli_adapter.go
package cli

import (
	...
)

// UserCLIAdapter handles CLI commands for managing users.
type UserCLIAdapter struct {
	userService *application.UserService
}

func NewUserCLIAdapter(userService *application.UserService) *UserCLIAdapter {
	return &UserCLIAdapter{
		userService: userService,
	}
}

// Command returns the root cobra command for this adapter.
func (a *UserCLIAdapter) Command() *cobra.Command {
	cmd := &cobra.Command{
		Use:   "user",
		Short: "Manage users",
	}

	cmd.AddCommand(
		a.getUserCmd(),
		a.updateUserCmd(),
		a.createUserCmd(),
	)

	return cmd
}

func (a *UserCLIAdapter) getUserCmd() *cobra.Command {
	return &cobra.Command{
		Use:   "get [id]",
		Short: "Get a user by ID",
		Args:  cobra.ExactArgs(1),
		RunE: func(cmd *cobra.Command, args []string) error {
			return a.GetUserByID(args[0])
		},
	}
}

func (a *UserCLIAdapter) updateUserCmd() *cobra.Command {
	return &cobra.Command{
		Use:   "update-name [id] [name]",
		Short: "Update a user's name",
		Args:  cobra.ExactArgs(2),
		RunE: func(cmd *cobra.Command, args []string) error {
			return a.UpdateUserName(args[0], args[1])
		},
	}
}

func (a *UserCLIAdapter) createUserCmd() *cobra.Command {
	return &cobra.Command{
		Use:   "create [name] [email] [role]",
		Short: "Create a new user",
		Args:  cobra.ExactArgs(3),
		RunE: func(cmd *cobra.Command, args []string) error {
			return a.CreateUser(args[0], args[1], args[2])
		},
	}
}

func (a *UserCLIAdapter) GetUserByID(id string) error {
	user, err := a.userService.GetUserByID(id)
	if err != nil {
		if errors.Is(err, domain.ErrItemNotFound) {
			return fmt.Errorf("user with ID '%s' not found", id)
		}
		return err
	}

	fmt.Printf("ID: %s, Name: %s, Email: %s, Role: %s\n",
		user.ID, user.Name, user.Email, user.Role,
	)
	return nil
}

func (a *UserCLIAdapter) UpdateUserName(id, newName string) error {
	err := a.userService.UpdateUserName(id, newName)
	if err != nil {
		if errors.Is(err, domain.ErrItemNotFound) {
			return fmt.Errorf("user with ID '%s' not found", id)
		}
		return err
	}

	fmt.Println("User name updated successfully.")
	return nil
}

func (a *UserCLIAdapter) CreateUser(name, email, role string) error {
	var roleValue domain.Role

	switch role {
	case "admin":
		roleValue = domain.RoleAdmin
	case "user":
		roleValue = domain.RoleUser
	default:
		return fmt.Errorf("invalid role: must be 'admin' or 'user'")
	}

	userID, err := a.userService.CreateUser(name, email, roleValue)
	if err != nil {
		return err
	}

	fmt.Printf("User created successfully with ID: %s\n", userID)
	return nil
}
```

#### HTTP Example (Using standard net/http)

The `UserHTTPAdapter` handles incoming HTTP requests, decodes payloads, and routes them to the appropriate `UserService` methods before returning JSON or HTML responses.

```go
// cmd/api/main.go
package main

import (
	...
)

func main() {
	db, err := database.NewClient(config.Database)
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()

	userRepo := database.NewUserRepository(db)
	userService := application.NewUserService(userRepo)

	userAdapter := api.NewUserHTTPAdapter(userService)

	mux := http.NewServeMux()
	userAdapter.RegisterRoutes(mux)

	log.Println("HTTP server running on :8080")
	log.Fatal(http.ListenAndServe(":8080", mux))
}
```

```go
// internal/interface/http/api/user_http_adapter.go
package api

import (
	...
)

type UserHTTPAdapter struct {
	userService *application.UserService
}

func NewUserHTTPAdapter(userService *application.UserService) *UserHTTPAdapter {
	return &UserHTTPAdapter{
		userService: userService,
	}
}

// RegisterRoutes wires HTTP routes to handlers.
func (a *UserHTTPAdapter) RegisterRoutes(mux *http.ServeMux) {
	mux.HandleFunc("GET /users/{id}", a.GetUserByID)
	mux.HandleFunc("POST /users", a.CreateUser)
	mux.HandleFunc("PATCH /users/{id}/name", a.UpdateUserName)
}

type UserResponse struct {
	ID    string `json:"id"`
	Name  string `json:"name"`
	Email string `json:"email"`
	Role  string `json:"role"`
}

func (a *UserHTTPAdapter) GetUserByID(w http.ResponseWriter, r *http.Request) {
	id := r.PathValue("id")

	user, err := a.userService.GetUserByID(id)
	if err != nil {
		if errors.Is(err, domain.ErrItemNotFound) {
			http.Error(w, "user not found", http.StatusNotFound)
			return
		}
		http.Error(w, "internal error", http.StatusInternalServerError)
		return
	}

	writeJSON(w, http.StatusOK, UserResponse{
		ID:    user.ID,
		Name:  user.Name,
		Email: user.Email,
		Role:  string(user.Role),
	})
}

func (a *UserHTTPAdapter) CreateUser(w http.ResponseWriter, r *http.Request) {
	var req struct {
		Name  string `json:"name"`
		Email string `json:"email"`
		Role  string `json:"role"`
	}

	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, "invalid request body", http.StatusBadRequest)
		return
	}

	var role domain.Role
	switch req.Role {
	case "admin":
		role = domain.RoleAdmin
	case "user":
		role = domain.RoleUser
	default:
		http.Error(w, "invalid role", http.StatusBadRequest)
		return
	}

	id, err := a.userService.CreateUser(req.Name, req.Email, role)
	if err != nil {
		http.Error(w, "internal error", http.StatusInternalServerError)
		return
	}

	writeJSON(w, http.StatusCreated, map[string]string{"id": id})
}

func (a *UserHTTPAdapter) UpdateUserName(w http.ResponseWriter, r *http.Request) {
	id := r.PathValue("id")

	var req struct {
		Name string `json:"name"`
	}

	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, "invalid request body", http.StatusBadRequest)
		return
	}

	err := a.userService.UpdateUserName(id, req.Name)
	if err != nil {
		if errors.Is(err, domain.ErrItemNotFound) {
			http.Error(w, "user not found", http.StatusNotFound)
			return
		}
		http.Error(w, "internal error", http.StatusInternalServerError)
		return
	}

	w.WriteHeader(http.StatusNoContent)
}

func writeJSON(w http.ResponseWriter, status int, v any) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)

	if err := json.NewEncoder(w).Encode(v); err != nil {
		http.Error(w, "failed to encode response", http.StatusInternalServerError)
	}
}
```

---

Key Considerations
------------------

### Error Handling

Error handling is crucial for building robust and reliable systems. I follow these guidelines:

-   **Domain Layer:** Errors in the domain layer should represent violations of business rules (e.g., `ErrInvalidName`).

-   **Application Layer:** Errors in the application layer should represent failures in use cases. The application layer may also wrap domain errors to provide more context.

-   **Infrastructure Layer:** Errors in the infrastructure layer should represent failures in external systems (e.g., database connection errors, API errors). These errors should be translated into domain-specific errors whenever possible to avoid leaking implementation details to the upper layers.

-   **Interface Layer:** Errors in the interface layer should be translated into appropriate responses for the given communication protocol (e.g., HTTP status codes, CLI error messages).

-   **Wrapping Errors:** Use `errors.Is` and `errors.As` for error checking.

### Logging

I use a consistent logging strategy to track the behavior of the applications, diagnose problems, and gain insights into system usage. A good logging strategy includes:

-   Using a structured logging library (e.g., `zap`, `slog`).

-   Logging at different levels (e.g., `debug`, `info`, `warn`, `error`, `fatal`).

-   Including relevant context in log messages (e.g., request ID, user ID).

-   Centralizing logs for easy analysis.

### Configuration

I typically use a `godotenv` to load configuration from a .env file in development and testing environments.

### Code Style

I adhere to the principles of clean code, including:

-   Meaningful names for variables, functions, and types.

-   Consistent formatting and indentation.

-   Short and focused functions.

`gofmt` and `golangci-lint` enforce code style and identify potential issues.

---

## Conclusion

By following these principles, I aim to create systems that are simple, scalable, and easy to understand. 

This guide is a reflection of practical patterns that work well for me, but the best engineering practices are forged through shared experiences and open debate.
If you have a different perspective, an optimization, or a question about these architectural choices, please open an issue or start a discussion. If your point of view is validated by the community and aligns with our core goals of simplicity and maintainability, it will be integrated into this living document to benefit everyone.
