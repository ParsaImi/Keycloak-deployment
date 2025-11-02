# Keycloak Deployment

## Docker Compose File

```yml
version: "3.8"
services:
  keycloak:
    container_name: keycloak
    image: quay.io/keycloak/keycloak:26.3.5
    ports:
      - 8080:8080
      - 8443:8443
    environment:
      KC_BOOTSTRAP_ADMIN_USERNAME: admin
      KC_BOOTSTRAP_ADMIN_PASSWORD: admin
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://postgres:5432/keycloak_db
      KC_DB_USERNAME: keycloak_user
      KC_DB_PASSWORD: keycloakpass
      KC_HOSTNAME: 10.1.1.39 # replace with your ip address
      KC_HTTP_PORT: 8080
      KC_HTTPS_PORT: 8443
      KC_HTTP_ENABLED: "true"
      KC_HOSTNAME_STRICT_HTTPS: "false"
    networks:
      - keycloak_network
    command:
      - "start"
networks:
  keycloak_network:
    external: true
```

## Environment Variables

### Database Setup
You need to set up a dedicated PostgreSQL database instance for Keycloak with the credentials specified in the environment variables.

### Key Environment Variables
- `KC_HOSTNAME`: Your Keycloak domain or IP address
- `KC_HTTP_PORT` & `KC_HTTPS_PORT`: HTTP and HTTPS ports

**I am not going to set ssl/tls encryption**
## Using Keycloak as an Identity Provider for MinIO

You can deploy MinIO using these instructions: https://github.com/ParsaImi/Minio-deployment

### Configure Keycloak to Work with MinIO

#### 1. Create a Realm
- Create a new realm called `myrealm`

#### 2. Create a Client for MinIO
- **Client Type**: OpenID Connect
- **Client ID**: `minio`

#### 3. Create a Client Scope
- **Name**: `minio`

#### 4. Configure a Mapper
Navigate to the **Mappers** tab and create a new mapper:
- **Mapper Type**: User Attribute
- **Name**: `policy`
- **User Attribute**: `policy`
- **Token Claim Name**: `policy`
- **Claim JSON Type**: String
- Toggle **ON**:
  - Add to ID token
  - Add to access token
  - Add to userinfo

#### 5. Add Client Scope to Client
- Select the `minio` client
- Navigate to the **Client scopes** tab
- Add `minio` scope as a **Default** scope

#### 6. Create a User
- **Username**: `keycloakuser`
- Navigate to the **Credentials** tab and set a password (e.g., `keycloakpass`)

#### 7. Create a Group
- **Name**: `admins`
- Navigate to the **Attributes** tab and add:
  - **Key**: `policy`
  - **Value**: `consoleAdmin` (or `readwrite`)
- Go to the **Members** tab and add `keycloakuser` as a member

**Keycloak configuration is now complete!**

## MinIO Configuration

### Access MinIO Shell
Execute into the MinIO container to access the MinIO shell:

#### 1. Create a New Alias
Set up an alias using your MinIO credentials:

```bash
mc alias set develop http://localhost:9000 minioadmin minioadmin123
```

#### 2. Add Keycloak as an Identity Provider
Configure MinIO to use Keycloak for authentication:

```bash
mc admin config set develop identity_openid \
  vendor="keycloak" \
  keycloak_realm="minio" \
  scopes="minio-authorization" \
  keycloak_admin_url="http://10.1.1.39:8080/admin" \
  client_id="minio" \
  client_secret="YOUR-CLIENT-SECRET" \
  config_url="http://10.1.1.39:8080/realms/minio/.well-known/openid-configuration" \
  display_name="Minio OpenID Login" \
  redirect_uri_dynamic="on" \
  tls_skip_verify="on"
```

#### 3. Restart MinIO
Restart the MinIO service to apply the configuration:

```bash
mc admin service restart develop
```

---

**Note**: Replace IP addresses, passwords, and client secrets with your actual values.
