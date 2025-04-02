Guide for Simple, Maintainable, and Scalable Golang DDD
===========================================================================

Introduction
------------

This document outlines our company's philosophy and architectural approach to building software, specifically using Domain-Driven Design (DDD) in Golang. It serves as the definitive guide for how we structure our projects, manage dependencies, and approach common challenges. Our goal is to create systems that are not only functional but also maintainable, scalable, and easy to understand.

This guide is a living document, reflecting our collective experience and ongoing refinement of best practices. We encourage all engineers to contribute to its evolution.

**Our Perspective (As of March 19, 2025)**

The approach detailed in this guide represents our current best understanding of how to design and build software. It's the result of significant experience with various architectural patterns and methodologies, including a close examination of resources like the [go-ddd](https://github.com/sklinkert/go-ddd "null") template. We've adopted what works well, adapted other elements, and introduced some of our own innovations---such as our approach to the Strategy pattern and application layer structure---to arrive at what we believe is an optimal approach for our needs. This isn't to say this is the *only* way, but it's *our* way, the one we've found leads to the most consistent and successful outcomes for our team.

Core Principles
---------------

Before diving into the structure, let's define the core principles that underpin our approach:

-   **Domain-Driven Design:** We believe that the most effective software is that which closely models the business domain it serves. DDD provides the tools and patterns to achieve this.

-   **Simplicity:** We favor simple solutions over complex ones. Complexity should only be introduced when it provides a clear and substantial benefit.

-   **Maintainability:** Our code should be easy to understand, modify, and extend. This requires clean architecture, clear naming conventions, and comprehensive testing.

-   **Scalability:** Our systems should be able to handle increasing load and complexity without requiring major architectural changes.

-   **Testability:** Every component of our system should be easily testable, enabling us to build confidence in our code and facilitate rapid iteration.

-   **Explicit Dependencies:** Dependencies between components should be clear and well-defined, minimizing coupling and promoting modularity.

-   **Layered Architecture:** We organize our code into distinct layers, each with a specific responsibility. This promotes separation of concerns and makes it easier to reason about the system.

Project Structure
-----------------

Our project structure is a key element of our architectural approach. It provides a consistent and predictable way to organize our code, making it easier for developers to navigate and understand different projects. Here's a detailed breakdown:

```
├── cmd
│   ├── desktop         # Application entry point for desktop applications
│   │   └── main.go
│   ├── web             # Application entry point for web applications
│   │   └── main.go
│   └── cli             # Application entry point for command-line applications
│       └── main.go
├── config            # Configuration loading and management
│   └── whatsapp.go     # Example: Configuration specific to WhatsApp integration
├── go.mod           # Go module definition
├── go.sum           # Go module dependencies
├── internal          # Internal application code (not intended for external use)
│   ├── application     # Application-level services (use cases, orchestration)
│   │   ├── chatbot_service.go # Example: A service for managing chatbot interactions
│   │   └── person.go      # Example: Application logic related to Person
│   ├── domain          # Domain logic (entities, value objects, interfaces)
│   │   ├── chat.go        # Example: Domain model for a Chat
│   │   ├── message.go     # Example: Domain model for a Message
│   │   ├── person.go      # Example: Domain model for a Person
│   │   └── role.go        # Example: Domain model for a Role
│   ├── infrastructure    # Implementation of external dependencies (databases, APIs, etc.)
│   │   ├── ai           # Integration with AI services
│   │   │   ├── client.go  # Client for the AI service
│   │   │   ├── message.go # Handling AI-specific message structures
│   │   │   ├── persona.go # Managing AI personas
│   │   │   └── role.go    # AI-specific roles
│   │   ├── database      # Database interaction (e.g., PostgreSQL)
│   │   │   ├── client.go  # Database client
│   │   │   ├── models.go  # Database-specific models (if needed)
│   │   │   └── person_repository.go # Implementation of PersonRepository
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
│       │   └── person_cli_adapter.go # Example: CLI adapter for Person-related commands
│       └── web           # Web interface adapters
│           ├── html        # HTML rendering
│           │   ├── telegram_handler.go   # Handler for Telegram web requests (HTML)
│           │   └── whatsapp_handler.go   # Handler for WhatsApp web requests (HTML)
│           └── json        # JSON API
│               ├── telegram_handler.go   # Handler for Telegram web requests (JSON)
│               └── whatsapp_handler.go   # Handler for WhatsApp web requests (JSON)
├── pkg               # Reusable, application-independent packages
│   └── utils           # Utility functions
│       └── deboucer.go    # Example: Debouncer implementation
└── README.md         # Project documentation

```

### Key Directories Explained

-   **cmd:** This directory contains the main entry points for our applications. Each subdirectory represents a different way of running the application (e.g., `cli` for command-line interface, `web` for a web server, `desktop` for a desktop application). This allows us to have multiple ways to run the same underlying application logic. We strive to keep this directory very thin, with minimal logic. Its primary responsibility is to initialize the necessary dependencies and start the application.

-   **config:** This directory holds our application's configuration. We typically use a library like `viper` or `envconfig` to load configuration from environment variables, files, or other sources. Configuration is structured to be easily accessible and type-safe. For example, `whatsapp.go` might define a struct with fields like `APIKey`, `PhoneNumber`, and `Timeout`.

-   **internal:** This is where the core application logic resides. Code within this directory is not intended to be accessed from outside the module. This enforces a strong boundary and prevents accidental coupling.

    -   **application:** This layer contains the use cases of our application. It orchestrates the interaction between the domain layer and the infrastructure layer. Application services are responsible for handling business transactions, validation (that *isn't* domain-level), and coordinating the work of multiple domain objects.

        -   We keep the application layer as a single, flat package. This is because, in our experience, the services within this layer tend to be highly interconnected. Creating subpackages often leads to unnecessary complexity and circular dependencies. We use clear naming conventions (e.g., `ChatbotService`, `PersonService`) to organize the code within this package.

    -   **domain:** This is the heart of our application. It contains the business logic, represented as entities, value objects, and interfaces. The domain layer is completely independent of any technical details, such as databases or web frameworks. It defines *what* the application does, not *how* it does it.

        -   Like the application layer, we also keep the domain layer as a single, flat package. We find that this simplifies the organization of domain concepts and reduces the likelihood of circular dependencies. We use naming conventions (e.g., `PersonEntity`, `ChatService`, `MessageValueObject`) to distinguish between different types of domain objects.

        -   **Entities:** Represent objects with a unique identity and lifecycle (e.g., a `Person`, an `Order`). Entities encapsulate both data and behavior. Validation logic that enforces business rules is often placed within entity methods. For example, a `Person` entity might have a `ChangeName(newName string) error` method that validates the new name before updating the entity's state.

        -   **Value Objects:** Represent immutable objects that are defined by their attributes rather than their identity (e.g., a `Date`, a `Money` amount). Value objects should be treated as interchangeable; two value objects with the same attributes are considered equal.

        -   **Interfaces:** Define contracts that are implemented by the infrastructure layer. For example, the domain layer might define a `MessageSender` interface with a `SendMessage(user Person, message Message) error` method. This allows the application layer to depend on the *abstraction* of message sending, without being tied to a specific *implementation* (e.g., sending via WhatsApp or Telegram). This is a key aspect of achieving loose coupling and testability. We might use the Strategy pattern here, where the specific implementation of `MessageSender` is chosen at runtime. For instance, a `User` entity might embed a `MessageSender` interface, and the application layer would inject the appropriate implementation (e.g., a `WhatsAppMessageSender` or a `TelegramMessageSender`). Alternatively, we might use decorators to add functionality, such as validation, before calling the underlying implementation.

    -   **infrastructure:** This layer provides the technical implementation of the interfaces defined in the domain layer. It handles communication with external systems, such as databases, message queues, and third-party APIs. Code in this layer is responsible for *how* things are done.

        -   We organize infrastructure implementations into subpackages based on the external system they interact with (e.g., `database`, `messaging`, `telegram`, `whatsapp`, `ai`). This keeps the code organized and makes it easy to find the implementation for a specific dependency. Within each subpackage, we may have further organization as needed, but we avoid creating excessively deep nesting. For example, the `database` package might contain a `client.go` for establishing the database connection, `models.go` for defining database-specific data structures (if necessary), and `person_repository.go` for implementing the `PersonRepository` interface defined in the domain layer. The key is to maintain a balance between organization and simplicity.

    -   **interface:** This layer acts as the entry point for external requests and translates them into commands that can be processed by the application layer. It's responsible for handling the specifics of the communication protocol (e.g., HTTP, CLI) and for presenting the application's response to the user.

        -   We structure this layer by the type of interface (e.g., `cli`, `web`). Within each interface type, we may further organize the code by specific functionality or technology.

        -   For example, the `web` package might be further divided into `html` and `json` subpackages to handle different response formats. Each of these subpackages would then contain handlers for specific resources (e.g., `telegram_handler.go`, `whatsapp_handler.go`).

        -   The `cli` package would contain adapters that handle argument parsing, command dispatching, and output formatting. For example, `person_cli_adapter.go` might define functions to handle commands related to creating, updating, or deleting `Person` entities. It acts as an intermediary between the user's terminal input and the application layer.

-   **pkg:** This directory contains reusable packages that are not specific to our application's domain. These packages can be used across multiple projects. Examples include utility functions, generic data structures, and third-party library wrappers. Code in this directory should not depend on any code in the `internal` directory. For example, `utils/deboucer.go` provides a utility function for debouncing events, which can be used in various parts of the application without introducing dependencies on the core domain logic.

-   **README.md:** This file provides an overview of the project, including its purpose, how to build and run it, and any other relevant information.

Dependency Management
---------------------

We use Go modules to manage dependencies. The `go.mod` and `go.sum` files at the root of the project define the project's dependencies and their versions.

### Dependency Rules

To maintain a clean architecture and prevent tight coupling, we adhere to the following dependency rules:

-   **domain:** The domain layer should not depend on any other layer. It is the most fundamental layer and should be completely self-contained. This ensures that our core business logic remains independent of any technical details.

-   **application:** The application layer can only depend on the domain layer. It orchestrates domain logic but should not be aware of the implementation details of external systems. If the application layer needs to interact with an external service, the interface for that service is defined in the domain layer, and the application layer depends on that interface.

-   **interface:** Interface layers (e.g., `cli`, `web`) can depend on both the domain layer and the application layer. They translate external requests into application commands and present the results to the user.

-   **infrastructure:** Infrastructure packages can only depend on the domain layer. They implement the interfaces defined in the domain layer and may use external libraries or frameworks to interact with external systems. They should not depend on the application or interface layers.

These rules ensure that the domain layer remains pure and that the application logic is decoupled from the infrastructure. This makes our code more modular, testable, and maintainable.

Layered Architecture
--------------------

Our architecture follows a layered pattern, where each layer has a specific responsibility and interacts with other layers in a defined way. This approach promotes separation of concerns, making our code easier to understand, maintain, and evolve.

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

Example: User Management
------------------------

To illustrate how these principles and patterns work in practice, let's consider a simple example: user management.

### Domain Layer

```go
// internal/domain/person.go
package domain

import "errors"

// Person is an entity that represents a user in the system.
type Person struct {
    ID    string
    Name  string
    Email string
    Role  Role
}

// Role is a value object that represents the role of a user.
type Role string

const (
    RoleAdmin = "admin"
    RoleUser  = "user"
)

// ErrInvalidName is returned when a name is invalid.
var ErrInvalidName = errors.New("name must be at least 3 characters long")

// ChangeName changes the name of the person.
func (p *Person) ChangeName(newName string) error {
    if len(newName) < 3 {
        return ErrInvalidName
    }
    p.Name = newName
    return nil
}

// PersonRepository is an interface for persisting Person entities.
type PersonRepository interface {
    GetByID(id string) (Person, error)
    Save(person Person) error
}

```

In this example, `Person` is an entity with an `ID`, `Name`, `Email`, and `Role`. The `ChangeName` method enforces a business rule: the name must be at least 3 characters long. `Role` is a value object. `PersonRepository` is an interface for persisting `Person` entities.

### Application Layer

```go
// internal/application/person.go
package application

import (
    "example.com/internal/domain" // Corrected import path
    "errors"
)

// ErrPersonNotFound is returned when a person is not found.
var ErrPersonNotFound = errors.New("person not found")

// PersonService provides use cases for managing persons.
type PersonService struct {
    personRepo domain.PersonRepository
}

// NewPersonService creates a new PersonService.
func NewPersonService(personRepo domain.PersonRepository) *PersonService {
    return &PersonService{
        personRepo: personRepo,
    }
}

// GetPersonByID retrieves a person by their ID.
func (s *PersonService) GetPersonByID(id string) (domain.Person, error) {
    person, err := s.personRepo.GetByID(id)
    if err != nil {
        // Wrap the error to provide more context.
        if errors.Is(err, domain.ErrPersonNotFound) {
            return domain.Person{}, ErrPersonNotFound // Use the error defined in application layer
        }
        return domain.Person{}, err
    }
    return person, nil
}

// UpdatePersonName updates the name of a person.
func (s *PersonService) UpdatePersonName(id, newName string) error {
    person, err := s.personRepo.GetByID(id)
    if err != nil {
        if errors.Is(err, domain.ErrPersonNotFound){
             return ErrPersonNotFound
        }
        return err
    }

    err = person.ChangeName(newName)
    if err != nil {
        return err // Return the error from the domain layer
    }

    return s.personRepo.Save(person)
}

// CreatePerson creates a new person.
func (s *PersonService) CreatePerson(id, name, email string, role domain.Role) (string, error) {
    // Application level validation
    if id == "" || name == "" || email == "" {
        return "", errors.New("id, name and email must not be empty")
    }
    // No domain level validation in constructor, the entity is responsible for validating itself
    person := domain.Person{
        ID:    id,
        Name:  name,
        Email: email,
        Role:  role,
    }

    err := s.personRepo.Save(person)
    if err != nil {
        return "", err
    }
    return person.ID, nil
}

```

The `PersonService` implements use cases for managing `Person` entities. It uses the `PersonRepository` interface to interact with the persistence layer. It also handles application-level errors, such as `ErrPersonNotFound`, and ensures that domain logic is correctly applied.

### Infrastructure Layer

```go
// internal/infrastructure/database/person_repository.go
package database

import (
    "example.com/internal/domain" // Corrected import path
    "database/sql"
    "errors"
)

// PersonRepository is an implementation of the domain.PersonRepository interface.
type PersonRepository struct {
    db *sql.DB
}

// NewPersonRepository creates a new PersonRepository.
func NewPersonRepository(db *sql.DB) *PersonRepository {
    return &PersonRepository{
        db: db,
    }
}

// GetByID retrieves a person from the database by their ID.
func (r *PersonRepository) GetByID(id string) (domain.Person, error) {
    row := r.db.QueryRow("SELECT id, name, email, role FROM persons WHERE id = $1", id)
    var person domain.Person
    err := row.Scan(&person.ID, &person.Name, &person.Email, &person.Role)
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return domain.Person{}, domain.ErrPersonNotFound // Use the error from the domain
        }
        return domain.Person{}, err
    }
    return person, nil
}

// Save saves a person to the database.
func (r *PersonRepository) Save(person domain.Person) error {
    _, err := r.db.Exec(
        "INSERT INTO persons (id, name, email, role) VALUES ($1, $2, $3, $4) "+
            "ON CONFLICT (id) DO UPDATE SET name = $2, email = $3, role = $4",
        person.ID, person.Name, person.Email, person.Role,
    )
    return err
}

```

The `PersonRepository` in the `database` package implements the `domain.PersonRepository` interface using a PostgreSQL database. It handles the mapping between the domain model (`domain.Person`) and the database schema. Error handling is crucial here. We translate database-specific errors (e.g., `sql.ErrNoRows`) into domain-specific errors (e.g., `domain.ErrPersonNotFound`) to avoid leaking implementation details.

### Interface Layer

```go
// internal/interface/cli/person_cli_adapter.go
package cli

import (
	"example.com/internal/application" // Corrected import path
	"example.com/internal/domain"    // Corrected import path
	"fmt"
	"os"
)

// PersonCLIAdapter handles CLI commands for managing persons.
type PersonCLIAdapter struct {
	personService *application.PersonService
}

// NewPersonCLIAdapter creates a new PersonCLIAdapter.
func NewPersonCLIAdapter(personService *application.PersonService) *PersonCLIAdapter {
	return &PersonCLIAdapter{
		personService: personService,
	}
}

// GetPersonByID handles the "get-person" command.
func (a *PersonCLIAdapter) GetPersonByID(id string) {
	person, err := a.personService.GetPersonByID(id)
	if err != nil {
		if errors.Is(err, application.ErrPersonNotFound) {
			fmt.Printf("Error: Person with ID '%s' not found.\n", id)
			return
		}
		fmt.Printf("Error: %v\n", err)
		os.Exit(1) // Use os.Exit for consistent error handling in CLI
	}
	fmt.Printf("ID: %s, Name: %s, Email: %s, Role: %s\n", person.ID, person.Name, person.Email, person.Role)
}

// UpdatePersonName handles the "update-person-name" command.
func (a *PersonCLIAdapter) UpdatePersonName(id, newName string) {
	err := a.personService.UpdatePersonName(id, newName)
	if err != nil {
		if errors.Is(err, application.ErrPersonNotFound) {
			fmt.Printf("Error: Person with ID '%s' not found.\n", id)
			return
		}
		fmt.Printf("Error: %v\n", err)
		os.Exit(1)
	}
	fmt.Println("Person name updated successfully.")
}

// CreatePerson handles the "create-person" command.
func (a *PersonCLIAdapter) CreatePerson(id, name, email, role string) {
	var roleValue domain.Role
	switch role {
	case "admin":
		roleValue = domain.RoleAdmin
	case "user":
		roleValue = domain.RoleUser
	default:
		fmt.Println("Error: Invalid role.  Must be 'admin' or 'user'.")
		os.Exit(1)
	}
	personID, err := a.personService.CreatePerson(id, name, email, roleValue)
	if err != nil {
		fmt.Printf("Error: %v\n", err)
		os.Exit(1)
	}
	fmt.Printf("Person created successfully with ID: %s\n", personID)
}

func main() {
	// Example usage in a CLI application (usually in cmd/cli/main.go)
	// In a real application, you would use a library like cobra or flag to handle command-line arguments.
	db, err := sql.Open("postgres", "your_postgres_connection_string") // Replace with your connection string.
	if err != nil {
		fmt.Printf("Error connecting to database: %v\n", err)
		os.Exit(1)
	}
	defer db.Close()

	personRepo := database.NewPersonRepository(db)
	personService := application.NewPersonService(personRepo)
	personCLIAdapter := NewPersonCLIAdapter(personService)

	// Example: Handling a command-line argument
	if len(os.Args) > 1 {
		switch os.Args[1] {
		case "get-person":
			if len(os.Args) > 2 {
personCLIAdapter.GetPersonByID(os.Args[2])
			} else {
				fmt.Println("Usage: get-person <id>")
				os.Exit(1)
			}
		case "update-person-name":
			if len(os.Args) > 3 {
				personCLIAdapter.UpdatePersonName(os.Args[2], os.Args[3])
			} else {
				fmt.Println("Usage: update-person-name <id> <new_name>")
				os.Exit(1)
			}
		case "create-person":
			if len(os.Args) > 5 {
				personCLIAdapter.CreatePerson(os.Args[2], os.Args[3], os.Args[4], os.Args[5])
			} else {
				fmt.Println("Usage: create-person <id> <name> <email> <role>")
				os.Exit(1)
			}
		default:
			fmt.Println("Usage: get-person | update-person-name | create-person")
			os.Exit(1)
		}
	} else {
		fmt.Println("Usage: get-person | update-person-name | create-person")
		os.Exit(1)
	}
}

```

The `PersonCLIAdapter` handles command-line arguments and calls the appropriate methods on the `PersonService`. It then formats the output and displays it to the user. This is a simplified example; in a real-world application, you would likely use a library like `cobra` or `flag` to handle command-line argument parsing and command dispatching. Error handling in the CLI should be consistent and user-friendly. We use `os.Exit` to terminate the program with an appropriate exit code when an error occurs.

Key Considerations
------------------

### Error Handling

Error handling is crucial for building robust and reliable systems. We follow these guidelines:

-   **Domain Layer:** Errors in the domain layer should represent violations of business rules (e.g., `ErrInvalidName`).

-   **Application Layer:** Errors in the application layer should represent failures in use cases (e.g., `ErrPersonNotFound`). The application layer may also wrap domain errors to provide more context.

-   **Infrastructure Layer:** Errors in the infrastructure layer should represent failures in external systems (e.g., database connection errors, API errors). These errors should be translated into domain-specific errors whenever possible to avoid leaking implementation details to the upper layers.

-   **Interface Layer:** Errors in the interface layer should be translated into appropriate responses for the given communication protocol (e.g., HTTP status codes, CLI error messages).

-   **Wrapping Errors:** Use `errors.Is` and `errors.As` for error checking.

### Testing

We believe that comprehensive testing is essential for building high-quality software. We strive to write tests at all levels:

-   **Unit Tests:** Test individual components in isolation (e.g., domain entities, application services, infrastructure adapters). Use mocking and stubbing to isolate the component under test.

-   **Integration Tests:** Test the interaction between different components or layers (e.g., testing the interaction between the application layer and the infrastructure layer).

-   **End-to-End Tests:** Test the entire system from end to end, simulating real user interactions.

### Logging

We use a consistent logging strategy to track the behavior of our applications, diagnose problems, and gain insights into system usage. A good logging strategy includes:

-   Using a structured logging library (e.g., `zap`, `logrus`).

-   Logging at different levels (e.g., `debug`, `info`, `warn`, `error`, `fatal`).

-   Including relevant context in log messages (e.g., request ID, user ID).

-   Centralizing logs for easy analysis.

### Configuration

We use a consistent approach to configuration management, typically using a library like `viper` or `envconfig`. This allows us to:

-   Load configuration from various sources (e.g., environment variables, files).

-   Define configuration in a structured and type-safe manner.

-   Easily manage different configurations for different environments (e.g., development, testing, production).

### Code Style

We adhere to the principles of clean code, including:

-   Meaningful names for variables, functions, and types.

-   Consistent formatting and indentation.

-   Short and focused functions.

-   Clear and concise comments.

We use `gofmt` and `golint` to enforce code style and identify potential issues.

Conclusion
----------

This guide provides a comprehensive overview of our company's philosophy and architectural approach to building software with Golang and DDD. By following these principles and patterns, we aim to create systems that are simple, maintainable, scalable, and easy to understand. Remember that this is a living document, and we encourage all engineers to contribute to its evolution as we continue to learn and grow.
