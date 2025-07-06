
# **Separation Architecture of Web and Mobile Applications with Microservices Utilization**

## **1. Benefits of Separating Web and Mobile Applications**

Designing web and mobile applications as separate systems offers the following benefits:

### **1.1. Optimized UI/UX**
- UI/UX requirements differ for web and mobile apps, allowing for individual optimization.
- Web apps can be designed for desktop screens, while mobile apps can be optimized for touch interfaces.

### **1.2. Performance and Scalability**
- Different traffic characteristics enable platform-specific optimization.
- Mobile can prioritize API request efficiency via batching and caching.

### **1.3. Independent Development and Deployment**
- Web and mobile teams can develop in parallel.
- Different tech stacks can be used (e.g., React for Web, Flutter/Swift/Android for Mobile).

### **1.4. Separate Security and Authentication**
- Web apps can use session-based auth; mobile apps can use token-based auth.
- Platform-appropriate security measures can be implemented.

## **2. Authentication and Microservice Integration via API Gateway**

Use separate API Gateways for web and mobile to handle authentication and delegate tasks to microservices.

### **2.1. Architecture Diagram**

```mermaid
graph TD;
    A[Web App] -->|Request| B[Web API Gateway<br>（Session Auth）]
    C[Mobile App] -->|Request| D[Mobile API Gateway<br>（Token Auth）]

    B -->|Uses| E[Auth Info Manager]
    D -->|Uses| E[Auth Info Manager]

    B -->|Authenticated Request| F[App Service<br>（Use Case Execution）]
    D -->|Authenticated Request| F

    F -->|Uses| G[User Info Service]
    F -->|Uses| H[Content Management]
    F -->|Uses| I[Payment Service]
    F -->|Uses| J[Log Management]

    subgraph API Gateway
        B
        D
        E
    end

    subgraph Microservices
        G
        H
        I
        J
    end

    subgraph Application
        F
    end
```

### **2.2. Component Roles**

| **Component**            | **Role** |
|--------------------------|------------------------------------------------|
| **Web App**              | Uses session-based auth and serves HTML/JS     |
| **Mobile App**           | Uses token-based auth and consumes APIs        |
| **Web API Gateway**      | Performs session auth and forwards to services |
| **Mobile API Gateway**   | Performs token auth and forwards to services   |
| **Auth Info Manager**    | Verifies OIDC/JWT tokens and issues credentials|
| **Application Service**  | Executes application use cases                 |
| **User Info Service**    | Manages user profiles/settings                 |
| **Content Management**   | Manages and delivers app content               |
| **Payment Service**      | Handles payments and subscriptions             |
| **Log Management**       | Collects and analyzes logs                     |

---

## **3. Benefits of Authentication via API Gateway**

### **3.1. Centralized Authentication**
- Unified auth: sessions for Web, tokens for Mobile.
- Offloads auth responsibility from microservices.

### **3.2. Scalability**
- Separate scaling for Web/Mobile Gateways.
- Auth services can be scaled independently.

### **3.3. Enhanced Security**
- Only authenticated requests are passed to services.
- JWTs enable control over expiration and scopes.

### **3.4. Flexible Tech Choices**
- Frontend: React, Vue.js, Flutter, Swift, Kotlin.
- Backend: Node.js, Rails, Go, Java, Python, etc.

---

## **4. Summary**

| **Aspect**           | **Web-Mobile Separation**        | **API Gateway Auth Delegation** |
|----------------------|----------------------------------|----------------------------------|
| **Development**      | Independent teams and stacks     | Central auth, delegated services |
| **UI/UX Optimization** | Platform-specific design        | Platform-specific data formatting |
| **Scalability**      | Manage load per platform         | Scale per gateway                |
| **Auth Management**  | Sessions (Web), Tokens (Mobile)  | Unified in API Gateway           |
| **Security**         | Platform-appropriate auth        | Gateway-based access control     |

This setup enables **efficient web/mobile development with a shared backend, secure authentication, and optimized data delivery.**

# Authenticated Request Specification

In this architecture, requests from Web/Mobile apps carry auth info. API Gateway verifies and forwards to application services.

## **1. Adding Authentication Info**

### **(1) From Web App**
- **Auth Method:** Session-based
- **Headers/Format:**
  - `Cookie: session_id=<session_id>` (Secure & HttpOnly)

#### **Example Request**
```http
GET /user/profile HTTP/1.1
Host: api.example.com
Cookie: session_id=abc123;
```

### **(2) From Mobile App**
- **Auth Method:** Token-based (JWT, OIDC)
- **Headers/Format:**
  - `Authorization: Bearer <JWT>`

#### **Example Request**
```http
GET /user/profile HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5...
```

## **2. Auth Management and Gateway Role**

### **(1) Auth Info Manager**
- Verifies session/token from API Gateway
- Returns `401 Unauthorized` if invalid

### **(2) API Gateway Processing**

| API Gateway            | Auth Type            | Validation     | Injected Headers               |
|------------------------|----------------------|----------------|-------------------------------|
| **Web API Gateway**    | Session-based        | Auth Manager   | `X-Authenticated-User`        |
| **Mobile API Gateway** | Token-based (JWT)    | Auth Manager   | `X-Authenticated-User`        |

## **3. Authenticated Request Format**

Post-authentication, user identity is forwarded to services.

### **(1) Web API Gateway → App Service**
```http
GET /user/profile HTTP/1.1
Host: api.example.com
X-Authenticated-User: 12345
X-Authenticated-Scopes: read_profile, edit_profile
```

### **(2) Mobile API Gateway → App Service**
```http
GET /user/profile HTTP/1.1
Host: api.example.com
X-Authenticated-User: 12345
X-Authenticated-Scopes: read_profile, edit_profile
```

## **4. JWT Token Structure**

Mobile API Gateway uses the following JWT format:

### (1) Header
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

### (2) Payload
```json
{
  "sub": "12345",
  "name": "John Doe",
  "email": "john.doe@example.com",
  "iat": 1712345678,
  "exp": 1712355678,
  "scope": ["read_profile", "edit_profile"]
}
```

- `sub`: User ID (12345)
- `email`: User email
- `iat`: Issued at
- `exp`: Expiration
- `scope`: Granted permissions

---

## 4. API Gateway Roles

- **Web API Gateway**
  - Manages cookie sessions
  - Adds `X-Authenticated-User` header
  - Returns 401 if session invalid

- **Mobile API Gateway**
  - Verifies `Authorization: Bearer <JWT>`
  - Adds `X-Authenticated-User` header
  - Returns 401 if token invalid

---

## 5. Benefits of Authenticated Requests

✅ **App services handle requests without concern for authentication**  
✅ **Unified treatment of Web and Mobile APIs**  
✅ **Microservices rely on Gateway for access control**  
✅ **Fine-grained permission control via scopes**  

This architecture ensures proper authentication handling and unified request format for both Web and Mobile apps.
