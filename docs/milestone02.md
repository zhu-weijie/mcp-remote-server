#### **Secure Configuration & Structured Logging**

*   **Problem:**
    1.  The service currently has no mechanism to manage configuration (e.g., port numbers, log levels) without modifying the source code. This is inflexible and prevents the same application artifact from being promoted across different environments (dev, staging, production).
    2.  Application logs are unstructured plain text. This makes them extremely difficult to search, parse, and analyze in an automated way, rendering effective monitoring and alerting in a cloud environment like AWS CloudWatch nearly impossible.

*   **Solution:**
    1.  Introduce a `Settings Manager` component, implemented using the `pydantic-settings` library. This component will load all configuration from environment variables, strictly separating configuration from code, in alignment with 12-Factor App principles.
    2.  Introduce a `Structured Logger` component. This will be implemented by configuring Python's standard `logging` module with a JSON formatter. All application logs will be written to `stdout` as machine-readable JSON objects.

*   **Trade-offs:**
    *   **Pros:**
        *   **Security & Flexibility:** Secrets and environment-specific settings are never hardcoded. The same Docker image can be deployed anywhere by providing the appropriate environment variables.
        *   **Enhanced Observability:** Structured JSON logs are the industry standard for modern cloud applications. They enable powerful querying, metrics creation, and reliable alerting in log aggregation platforms (e.g., CloudWatch, Datadog).
        *   **Improved Developer Experience:** `pydantic-settings` provides automatic type validation for configuration, catching errors early.
    *   **Cons:**
        *   Introduces one new dependency (`pydantic-settings`), which is a negligible and highly justified trade-off for the benefits gained.
        *   Requires developers to adopt the standard logging framework instead of using `print()` statements, which is a necessary discipline for building production-grade software.

#### **Design the Architecture-as-Code (AaC)**

*   **Logical View (C4 Component Diagram)**

    *This diagram shows the new logical components being introduced *inside* the existing FastAPI service.*

```mermaid
C4Component
    title Component Diagram for Secure Configuration & Logging

    Person(operator, "Operator / Developer")
    System(log_ingestion, "Log Ingestion Platform", "e.g., AWS CloudWatch Logs")

    System_Boundary(c1, "MCP Remote Server [System]") {
        Component(fastapi_service, "FastAPI Web Service", "FastAPI, Python", "Provides HTTP endpoints and orchestrates core logic.")

        System_Boundary(c2, "Observability & Configuration") {
            Component(settings_manager, "Settings Manager", "Pydantic-settings", "Loads and validates configuration from the environment.")
            Component(structured_logger, "Structured Logger", "Python Logging w/ JSON Formatter", "Formats and outputs all logs as structured JSON.")
        }
    }

    Rel(operator, settings_manager, "Provides environment variables to")
    Rel(fastapi_service, settings_manager, "Uses", "Reads config")
    Rel(fastapi_service, structured_logger, "Uses", "Writes logs")
    Rel(structured_logger, log_ingestion, "Outputs logs to", "stdout (JSON)")
```

*   **Physical View (Deployment Diagram)**

    *This diagram shows how the container's runtime is influenced by external configuration and how its output is consumed.*

```mermaid
graph TD
    subgraph "Runtime Environment (e.g., AWS App Runner or Local Docker)"
        EnvVars["fa:fa-envira Environment Variables<br/><i>(e.g., LOG_LEVEL=INFO)</i>"]

        subgraph C["Container (mcp-remote-server:latest)"]
            App["Uvicorn / FastAPI Process"]
            subgraph "Internal Components"
                direction LR
                Settings["Pydantic-settings"]
                Logger["JSON Logger"]
            end
        end
        
        Logs["fa:fa-database Structured Logs<br/><i>(stdout as JSON)</i>"]

        EnvVars -- "Injects Into" --> C
        C -- "Reads at startup by" --> Settings
        App -- "Uses" --> Settings
        App -- "Writes via" --> Logger
        Logger -- "Outputs to" --> Logs
    end
```

*   **Component-to-Resource Mapping Table**

| Logical Component | Physical Resource | Rationale (Why this choice?) |
| :--- | :--- | :--- |
| **Settings Manager** | **Pydantic-settings Library** | This is the de facto standard for configuration management in the FastAPI ecosystem. It provides type safety, validation, and strictly enforces the 12-Factor App principle of separating config from code, which is critical for secure and flexible deployments. |
| **Structured Logger** | **Python's standard `logging` module with a JSON Formatter** | Leveraging the standard library is robust and avoids unnecessary dependencies. A JSON formatter is chosen because JSON is a universal, machine-readable format that is natively supported by virtually all modern log aggregation and observability platforms, including AWS CloudWatch. |
