# hr-system-ms

## HR System with Microservices

Microservices-based HR System built with **Java & Spring Boot**.  
Implements **service discovery**, **API Gateway**, **asynchronous processing (RabbitMQ)**, and **independent persistence per microservice**, aiming for a scalable and maintainable HR back-end solution.

---

## Architecture and Technical Decisions

### Microservices Communication Strategy

All external requests first reach the **API Gateway**, which forwards them to the appropriate microservice registered in **Eureka Service Registry**.

- **Synchronous communication (REST)**: used when a microservice requires an immediate response from another service.
- **Asynchronous communication (RabbitMQ)**: used for background or long-running processes, publishing events to queues that are later consumed by the same or other microservices.

---

## Chosen Technologies

- **Java 17**
- **Spring Boot 3.x**
- **PostgreSQL**
- **RabbitMQ**
- **Redis**

---

## Database Separation Strategy

Each microservice owns its **own database**, ensuring:
- Data isolation  
- Independent evolution  
- Reduced coupling between domains  

This approach improves resilience and aligns with microservices best practices.

---

## Service Analysis: Pros, Cons, and Alternatives

### Registry Service

- Uses **Eureka** to allow services to discover each other without relying on fixed network addresses.

---

### Gateway Service

- Central entry point for all external requests.
- User **role validation** is handled at the gateway level, preventing unauthorized requests from reaching downstream services.
- Implements **Redis caching** for user information to avoid repeated role validation calls.
- Some user information is propagated through request headers.
- When a user is updated, the **User Service publishes an event**, which is consumed by the Gateway to invalidate the Redis cache.

**Negative point**:
- Single point of entry — if the Gateway fails, the entire system becomes inaccessible.

**Possible improvement**:
- Deploy multiple Gateway instances for high availability.

**Future improvements**:
- Implement authentication using **JWT or OAuth2**.  
  Currently, the system supports authorization (roles) only.

---

### Employee Service

- Work Schedule registration is handled separately to avoid duplicating data for each employee.
- Employee onboarding endpoint includes:
  - Employee data  
  - User creation data  
  - Reference to a pre-registered Work Schedule
- User creation is done **synchronously**:
  - The employee is saved
  - The User Service is called
  - If any error occurs, the transaction is rolled back

**Negative point 1**:
- Potential inconsistency if an unexpected failure occurs after user creation but before employee persistence.

**Alternative 1**:
- Make user creation asynchronous with a compensation mechanism (rollback employee creation on failure).

**Negative point 2**:
- Downstream services do not handle authentication and authorization.

**Alternative 2**:
- Keep only the API Gateway publicly accessible and place all other services inside a private network.

---

### Time Entry Service

- Allows employees to register time entries using the current date and time.
- Supports editing and querying time entries.
- Uses a **composite index (employee + date)** due to:
  - High expected data volume
  - Frequent filtering by these fields

**Positive point**:
- Query optimization for large tables from the beginning of the project.

**Future suggestions**:
- If the table grows excessively, create an auxiliary table with reduced data
  (e.g., employee + month), referencing the main table to optimize payroll processing.

---

### User Service

- User creation endpoint invoked by the Employee Service.
- Stores only user-related data (role, password).
- Publishes an event whenever a user is updated, allowing other services to invalidate cached user data.

**Positive point**:
- Maintains single responsibility and integrates with messaging for cache consistency.

---

### Payroll Service

- Payroll calculation is started **asynchronously** by publishing an event to RabbitMQ.
- Since processing is asynchronous:
  - The request returns immediately
  - A payroll processing entity is created and returned with **HTTP 202 (Accepted)**
  - The entity ID can be used later to check the processing status
- The service consumes its own queue to process payroll.
- Retrieves data from Employee and Time Entry services as calculation inputs.
- Stores execution history including:
  - Status
  - Requesting user
  - Start and end timestamps
- Publishes an event upon completion to trigger email notifications.

**Positive point**:
- Avoids client blocking and provides full process traceability.

**Negative point**:
- Relies on asynchronous processing, requiring queue monitoring and observability.

**Future suggestions**:
- Split persistence into:
  - A generic table for processing history
  - A dedicated table for actual payroll results

---

### Email Service

- Consumes email request events from RabbitMQ.
- Responsible solely for sending notifications.

**Positive point**:
- Decouples email delivery from business services, improving scalability and responsibility separation.

---

### Common Module

- Shared library used across microservices.
- Contains:
  - Role definitions
  - Request header constants
  - Custom exceptions
  - Global exception handler (Controller Advice)

---

## Project Diagram

<img width="3600" height="1960" alt="Fluxograma Projeto Entra21 - Página 1" src="https://github.com/user-attachments/assets/e2f52fb7-14ce-491b-bd8e-32d44818741f" />

Architecture diagram available at:  
https://lucid.app/lucidchart/4f39c075-11e3-4779-96b1-34c879567a57/edit
