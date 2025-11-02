# Keycloak deployment

## Docker Compose file

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
      KC_HOSTNAME: 10.1.1.39
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

### Volumes

if you run keycloak with start command, it requires ssl/tls encryption.

so we make and specify a crt and key file :

```bash
openssl req -newkey rsa:2048 -nodes -keyout server.key -x509 -days 365 -out server.crt
```

You'll be prompted to enter certificate information:

Common name : yourdomain( in my case i set localhost )

### Environments

we need to setup a database indside a postgres instance dedicated for keycloak

`KC_HOSTNAME` your keycloak domain 
`KC_HTTP_PORT` & `KC_HTTPS_PORT` http and https ports
`KC_HTTPS_CERTIFICATE_FILE` & `KC_HTTPS_CERTIFICATE_FILE` cert and key files path

### using keycloak as an IDP for Minio service

You can deploy Minio with this instruction : https://github.com/ParsaImi/Minio-deployment

**Configure Keycloak to work with MinIO**
- Create a new realm called 'myrealm'
- Create a Client for Minio service
Client Type: OpenID Connect
Client ID: minio
- Create a Client Scope for Minio service
Name: minio
- Navigate to Mappers tab and configure a new Mapper
select User Attributes
Name: policy
add policy as User Attribute
Token Claim Name: policy
Claim JSON Type: String
Toggle Add to ID token
Toggle Add to access token
Toggle Add to userinfo
- Add minio client scope to minio client
 select minio client and navigate to `Client scopes` tab
 add minio scope as Default scope
- Create a new User
Username: keycloakuser
Navigate to `Credentials` tab and create a new password ( keycloakpass for my case )
- Create a new Group
Name: admins
Navigate to Attributes tab and add a new attribute
Key: policy
Value: consoleAdmin ( or readwrite )
Go to Members tab and add keycloakuser as a member

**and we are done with keycloak configuration**

#### Minio shell

exec into minio container to access minio shell

- create a new alias using your minio credentials
```bash
mc alias set develop http://localhost:9000 minioadmin minioadmin123
```

- Add keycloak as an IDP for minio
```bash
mc admin config set develop identity_openid \
vendor="keycloak" \
keycloak_realm="minio" scopes="minio-authorization" \
keycloak_admin_url="http://10.1.1.39:8080/admin \
client_id=minio client_secret=fvIoYZinI9DoEtDF1juOiaW5Ac1MOpIK \
config_url="http://10.1.1.39:8080/realms/minio/.well-known/openid-configuration" display_name="Minio OpenID Login" \
redirect_uri_dynamic="on" \
tls_skip_verify="on"
```

