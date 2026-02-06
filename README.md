# Keycloak Spring Boot Demo - Demonstration

A complete demonstration project showing OAuth2/OIDC integration with Keycloak, including remote debugging capabilities and custom event listeners.

## 🎯 Task Overview

This project demonstrates:
- **Identity Flow Understanding**: OIDC, SSO, Keycloak integration
- **Reproducible Development Environment**: Single command setup
- **Remote Debugging**: JDWP debugging in real runtime
- **Custom Login Hooks**: Event listeners, webhooks, and user attributes

## 🚀 Quick Start

### Prerequisites
- Docker and Docker Compose
- Java 17+ and Maven (only for local development or code modifications)

### 1. Critical Setup: Hosts File Configuration

**REQUIRED BEFORE STARTING ANY SERVICES**

Add this entry to your OS hosts file (`/etc/hosts` on Linux/Mac, `C:\Windows\System32\drivers\etc\hosts` on Windows):
```
127.0.0.1    keycloak
```

**Why this is critical:**
- OIDC issuer URL must be identical for token generation and validation
- Keycloak generates tokens with issuer: `http://keycloak:8080/realms/demo`
- Spring Boot validates tokens against the same issuer URL
- **Without this entry, the system will NOT work**

**Test the setup:**
```bash
ping keycloak  # Should resolve to 127.0.0.1
```

### 2. Start Everything

**Simple Docker approach (recommended):**
```bash
# Start all services (images already built)
docker-compose up

# Or run in detached mode
docker-compose up -d
```

**Rebuild only when needed:**
```bash
# Use --build only if you modified:
# - Dockerfile
# - Source code in app/ directory  
# - First time setup
docker-compose up --build
```

**Event Listener Modifications (only if you change code):**
```bash
# Build Keycloak event listener
cd keycloak-event-listener
mvn clean package
cp target/*.jar ../keycloak/providers/  # Copy updated JAR to providers
cd ..

# Restart only Keycloak to reload the JAR
docker-compose restart keycloak
```

### 3. Authentication Flow

### Standard Login
1. Navigate to http://localhost:8088
2. Redirect to Keycloak login page
3. Login with: `testuser` / `test`
4. Redirect back to app with user data

### Mock Identity Provider Login
1. Navigate to http://localhost:8088
2. Click "Mock Identity" on Keycloak login page
3. Login with: `mockuser` / `mock`
4. Complete identity provider flow

## 🪝 Login Hooks Implementation

When user successfully logs in, the system triggers:

1. **Keycloak Event Listener** (`DemoEventListenerProvider.java`)
   - Structured logging of login event
   - Adds `demo-attr` attribute to user
   - Sends HTTP POST webhook to Spring Boot app

2. **Spring Boot Webhook** (`UserController.java`)
   - Receives webhook at `/webhook/login`
   - Processes login event data
   - Returns success response

3. **User Data Display**
   - Shows userId, username, issuer, clientId
   - Includes custom `demo-attr` from event listener

## 🐛 Remote Debugging

### Spring Boot Application Debugging

**Port**: 8787

**IntelliJ IDEA Setup:**
1. `Run` → `Edit Configurations...`
2. Click `+` → `Remote JVM Debug`
3. Configure:
   - **Name**: `Spring Boot Debug`
   - **Host**: `localhost`
   - **Port**: `8787`
4. Click `Debug` button to attach

**VS Code Setup:**
```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "type": "java",
            "name": "Attach to Remote Spring Boot",
            "hostName": "localhost",
            "port": 8787,
            "request": "attach"
        }
    ]
}
```

### Key Debugging Points

Set breakpoints during login flow:

**UserController.java:**
- Line 28: `logger.info("USER_ACCESS: Processing user request");`
- Line 40: `logger.info("USER_DATA_RETURNED: Successfully returned user information");`
- Line 56: `logger.info("WEBHOOK_RECEIVED: {}", payload);`

**DemoEventListenerProvider.java:**
- Line 52: Structured login event logging
- Line 61: User attribute update
- Line 89: Webhook payload creation
- Line 99: Webhook response handling

### Debugging Workflow

1. Start services: `docker-compose up --build`
2. Attach debugger to port 8787
3. Set breakpoints
4. Navigate to http://localhost:8088
5. Login and observe execution flow

## 📁 Project Structure

```
├── app/                              # Spring Boot application
│   ├── src/main/java/com/demo/
│   │   ├── DemoApplication.java      # Main application class
│   │   ├── config/
│   │   │   └── SecurityConfig.java   # OAuth2 security configuration
│   │   └── controller/
│   │       └── UserController.java  # User endpoints and webhook
│   ├── src/main/resources/
│   │   └── application.yml          # OAuth2 client configuration
│   ├── pom.xml                       # Maven dependencies
│   └── target/                       # Build output
├── keycloak/                         # Keycloak configuration
│   ├── realm-demo.json              # Main realm configuration
│   ├── mock-idp-realm.json          # Mock identity provider realm
│   └── providers/                   # Custom providers directory
├── keycloak-event-listener/          # Custom Keycloak extension
│   ├── src/main/java/com/demo/keycloak/
│   │   ├── DemoEventListenerProvider.java
│   │   └── DemoEventListenerProviderFactory.java
│   ├── pom.xml
│   └── target/keycloak-event-listener-1.0.jar
├── docker-compose.yml                # Service orchestration
├── Dockerfile                        # Spring Boot build configuration
└── README.md                         # This file
```

## 🔧 Configuration Details

### Keycloak Setup
- **Database**: PostgreSQL
- **Main Realm**: `demo`
- **Mock IDP Realm**: `mock-idp`
- **Client**: `demo-app` (public client)
- **Test Users**: `testuser/test` (local), `mockuser/mock` (IDP)

### Spring Boot OAuth2
- **Provider**: Keycloak
- **Client ID**: `demo-app`
- **Scopes**: `openid,profile,email`
- **Redirect URI**: `http://localhost:8088/login/oauth2/code/keycloak`

### Docker Networking

The setup uses a custom Docker network `demo-network` with hostname configuration:
- **Internal Container Communication**: `keycloak:8080` (OIDC issuer URL)
- **External Browser Access**: `localhost:8080` (host port mapping)
- **Spring Boot → Keycloak**: Uses service name `keycloak:8080`
- **Browser → Keycloak**: Uses `localhost:8080` (redirect URLs)
- **Cross-Platform Compatibility**: Works on Docker Desktop, Colima, Lima, and Linux Docker

### Event Listener Features
- **Login Event Processing**: Captures successful logins
- **User Attributes**: Adds custom attributes to user profile
- **Webhook Integration**: HTTP POST to Spring Boot application
- **Structured Logging**: JSON-formatted log entries

## 🧪 Testing

### Test Login Flow

```bash
# Test standard login
curl -L http://localhost:8088

# Test webhook directly
curl -X POST http://localhost:8088/webhook/login \
  -H "Content-Type: application/json" \
  -d '{"event":"LOGIN_SUCCESS","userId":"test","username":"testuser"}'
```

### Verify Event Listener

```bash
# Check Keycloak logs for event listener activity
docker-compose logs -f keycloak | grep "LOGIN_EVENT\|WEBHOOK"

# Check Spring Boot logs
docker-compose logs -f java-app | grep "USER_ACCESS\|WEBHOOK"
```

## 🔍 Troubleshooting

### Common Issues

**"Container keycloak is unhealthy"**
- Check port 8080 availability
- Verify Keycloak startup logs: `docker-compose logs keycloak`

**"Unable to resolve Configuration with the provided Issuer"**
- Ensure Keycloak is fully started before Spring Boot
- Check network connectivity between containers
- **Critical**: Verify hosts file contains `127.0.0.1 keycloak`
- **Test**: `ping keycloak` should resolve to 127.0.0.1
- **Issuer Consistency**: Ensure all requests use the same hostname (`keycloak:8080`)

**Login redirects to wrong URL**
- Verify Keycloak frontend URL in realm-demo.json
- Check docker-compose.yml hostname configuration
- **Critical**: Ensure `KC_HOSTNAME_URL` matches internal Docker network (`keycloak:8080`)
- **Note**: Browser uses `localhost:8080` but containers use `keycloak:8080` for OIDC communication

**Webhook not received**
- Check event listener logs: `docker-compose logs keycloak`
- Test webhook manually from Keycloak container:
```bash
docker exec keycloak curl -X POST http://java-app:8088/webhook/login \
  -H "Content-Type: application/json" \
  -d '{"event":"test","userId":"test"}'
```

### Debug Commands

```bash
# View all logs
docker-compose logs -f

# Check container status
docker-compose ps

# Restart specific service
docker-compose restart keycloak
docker-compose restart java-app

# Rebuild and restart
docker-compose up --build java-app
```

## 🛠️ Development

### Local Development

```bash
# Option 1: Run everything in Docker (recommended)
docker-compose up --build

# Option 2: Run databases only, Spring Boot locally
docker-compose up postgres keycloak -d
cd app
mvn spring-boot:run  # Local build and run

# Build event listener only (needed if you modified the code)
cd keycloak-event-listener
mvn clean package
cp target/*.jar ../keycloak/providers/
docker-compose restart keycloak
```

### Modify Event Listener

**Note**: The event listener JAR is already included in `keycloak/providers/` folder. You only need to rebuild if you modify the source code.

1. Edit files in `keycloak-event-listener/src/main/java/com/demo/keycloak/`
2. Rebuild: `cd keycloak-event-listener && mvn clean package`
3. Copy JAR: `cp target/*.jar ../keycloak/providers/`
4. Restart Keycloak: `docker-compose restart keycloak`

## 🔒 Security Notes

- This demo uses HTTP for simplicity. Use HTTPS in production.
- Test credentials are for demonstration only.
- Keycloak admin credentials should be changed in production.
- Consider proper secrets management for production deployments.

## 📝 Task Completion Checklist

- [x] **Environment Setup**: Single `docker-compose up` command
- [x] **Login Flow**: Standard and mock identity provider
- [x] **Login Hooks**: Event listener, webhook, user attributes
- [x] **Remote Debugging**: JDWP on port 8787
- [x] **Documentation**: Complete setup and usage guide

## 📄 License

This project is for demonstration purposes. Feel free to use and modify as needed.
