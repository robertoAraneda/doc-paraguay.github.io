## Infra Paraguay

### Infraestructura

El servidor se encuentra en la dirección IP: 192.168.101.107

Implementación de infraestructura para interoperabilidad

![INFRA](prod-paraguay.jpg?raw=true "infra-prod")

La infraestructura propuesta contempla la utilización de varios componentes:

1. API Gateway

Una API Gateway es un proxy reverso que permite confogurar y gestionar las rutas hacia determinados servicios (APIs).

En este proyecto se implementó [Kong Gateway](https://docs.konghq.com/gateway/3.5.x/)

Kong Gateway es una herramienta Open Source que puede ejecutarse en frente de cualquier API RESTful, pudiendo extender su funcionalidad implementando módulos o plugins (Token instrospection, oidc).

2. Servidor Oauth2/OpenID

Los datos almacenados en el servidor FHIR no pueden ser obtenidor libremente. Un servidor de autenticación  que nos permite los usuarios/clientes tengan un acceso limitado según los permisos que posea.

En este proyecto se implementó [Keycloak](https://www.keycloak.org/)

Keycloak es una herramienta Open Source que permite gestionar la identificación y el acceso a un sistema.

Keycloak nos entrega entre otras funcionalidades, federación de usuarios, autenticación, gestión de usuarios, authorización y más.

3. Servidor FHIR

Los servidores FHIR (Fast Healthcare Interoperability Resources) son esenciales para la interoperabilidad de los datos en salud, ya que como implementan el estandar FHIR, la gestión de los datos por parte de los usuarios es única.

En este proyecto de implementó [HAPI FHIR](https://hapifhir.io/).

HAPI FHIR es una herramienta Open Source que implementa de forma completa el estandar de [HL7 FHIR](https://www.hl7.org/fhir/) para interoperabilidad en salud en lenguage Java.


### Despliegue

El despliegue de las aplicaciones se realizó mediante la tecnología de `Docker`.

Las configuraciones asociadas son las siguientes:

```yaml
version: "3.3"

networks:
  kong-net:
  keycloak-net:

services:
  #######################################
  # HAPI FHIR: FHIR server
  #######################################
  hapi-fhir-server:
    image: hapiproject/hapi:v6.8.3
    restart: always
    volumes:
      - ./hapi-config:/data/hapi
    environment:
      SPRING_CONFIG_LOCATION: "file:///data/hapi/application.yaml"
      SPRING_DATASOURCE_URL: ${HAPI_SPRING_DATASOURCE_URL}
      SPRING_DATASOURCE_USERNAME: ${HAPI_SPRING_DATASOURCE_USERNAME}
      SPRING_DATASOURCE_PASSWORD: ${HAPI_SPRING_DATASOURCE_PASSWORD}
      SPRING_DATASOURCE_DRIVERCLASSNAME: ${HAPI_SPRING_DATASOURCE_DRIVERCLASSNAME}
      SPRING_JPA_PROPERTIES_HIBERNATE_DIALECT: ${HAPI_SPRING_JPA_PROPERTIES_HIBERNATE_DIALECT}
    networks:
      - kong-net
  #######################################
  # Postgres: The database used by HAPI
  #######################################
  hapi-db:
    image: "postgres:14.6"
    restart: always
    user: root
    environment:
      POSTGRES_DB: ${HAPI_DB_NAME}
      POSTGRES_USER: ${HAPI_DB_USER}
      POSTGRES_PASSWORD: ${HAPI_DB_PASSWORD}"
    networks:
      - kong-net
    ports:
      - "35432:5432"
    volumes:
      - ./postgres-hapi-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready"]
      interval: 20s
      timeout: 10s
      retries: 5
    command: -p 5432

    #######################################
  # Postgres: The database used by Kong
  #######################################
  kong-database:
    image: postgres:9.5
    networks:
      - kong-net
    environment:
      POSTGRES_USER: ${KONG_DB_USER}
      POSTGRES_DB: kong
      POSTGRES_PASSWORD: ${KONG_DB_PASSWORD}
    ports:
      - "15432:5432"
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "kong"]
      interval: 5s
      timeout: 5s
      retries: 5
    restart: on-failure
    stdin_open: true
    tty: true
    volumes:
      - kong_data:/var/lib/postgresql/data

  #######################################
  # Kong database migration
  #######################################
  kong-migration:
    image: kong
    command: "kong migrations bootstrap"
    networks:
      - kong-net
    restart: on-failure
    environment:
      KONG_PG_HOST: kong-database
      KONG_PG_PASSWORD: ${KONG_DB_PASSWORD}
    links:
      - kong-database
    depends_on:
      - kong-database

  #######################################
  # Kong Gateway
  #######################################
  kong:
    image: robertoaraneda/kong-conectaton:latest
    user: root
    restart: always
    networks:
      - kong-net
    volumes:
      - ./kong.yml:/etc/kong/kong.yml
    environment:
      KONG_PG_HOST: kong-database
      KONG_PG_PASSWORD: ${KONG_DB_PASSWORD}
      KONG_PROXY_LISTEN: 0.0.0.0:8000
      KONG_PROXY_LISTEN_SSL: 0.0.0.0:8443
      KONG_ADMIN_LISTEN: 0.0.0.0:8001
      KONG_PLUGINS: bundled, oidc, token-introspection
      KONG_PROXY_ACCESS_LOG: /dev/stdout
      KONG_ADMIN_ACCESS_LOG: /dev/stdout
      KONG_PROXY_ERROR_LOG: /dev/stderr
      KONG_ADMIN_ERROR_LOG: /dev/stderr
      KONG_LOG_LEVEL: debug

    depends_on:
      - kong-database
    healthcheck:
      test: ["CMD", "curl", "-f", "http://kong:8001"]
      interval: 5s
      timeout: 2s
      retries: 15
    ports:
      - 8000:8000 # Listener
      - 8001:8001 # Admin API
      - 8002:8002 # UI
      - 8443:8443 # Listener  (SSL)
      - 8444:8444 # Admin API (SSL)

  #######################################
  # Konga: Kong GUI
  #######################################
  konga:
    image: pantsel/konga
    container_name: konga
    networks:
      - kong-net
    ports:
      - 1337:1337
    environment:
      DB_ADAPTER: postgres
      DB_URI: postgresql://kong:kong@kong-database:5432/konga
      KONGA_LOG_LEVEL: warn
      TOKEN_SECRET: ${KONGA_TOKEN_SECRET}
      KONGA_HOOK_TIMEOUT: 120000
      NODE_ENV: development
    restart: on-failure
    depends_on:
      - kong

  #######################################
  # Keycloak: OIDC server
  #######################################
  keycloak:
    image: quay.io/keycloak/keycloak
    depends_on:
      - keycloak-db
    networks:
      - keycloak-net
    ports:
      - "8080:8080"
    command:
      - start-dev
      - --import-realm
    volumes:
      - ./keycloak-test-realm.json:/opt/keycloak/data/import/realm.json
    environment:
      JAVA_OPTS_APPEND: -Dkeycloak.profile.feature.upload_scripts=enable
      KC_DB_PASSWORD: ${KC_DB_PASSWORD}
      KC_DB_URL: jdbc:postgresql://keycloak-db:5432/keycloak
      KC_DB: postgres
      KC_DB_USERNAME: ${KC_DB_USERNAME}
      KC_HEALTH_ENABLED: "true"
      KC_HTTP_ENABLED: "true"
      KC_METRICS_ENABLED: "true"
      KEYCLOAK_ADMIN: ${KC_KEYCLOAK_ADMIN}
      KEYCLOAK_ADMIN_PASSWORD: ${KC_KEYCLOAK_ADMIN_PASSWORD}
      KC_FEATURES: token-exchange

  #######################################
  # Keycloak database
  #######################################
  keycloak-db:
    image: postgres:9.6
    volumes:
      - keycloak-datastore:/var/lib/postresql/data
    networks:
      - keycloak-net
    ports:
      - "25432:5432"
    environment:
      POSTGRES_DB: keycloak
      POSTGRES_USER: ${KC_DB_USERNAME}
      POSTGRES_PASSWORD: ${KC_DB_PASSWORD}

volumes:
  kong_data:
  keycloak-datastore:

```

Las variables de entorno:

```text
## HAPIFHIR

HAPI_SPRING_DATASOURCE_URL=jdbc:postgresql://hapi-db:5432/root
HAPI_SPRING_DATASOURCE_USERNAME=root
HAPI_SPRING_DATASOURCE_PASSWORD=hapifhir2023
HAPI_SPRING_DATASOURCE_DRIVERCLASSNAME=org.postgresql.Driver
HAPI_SPRING_JPA_PROPERTIES_HIBERNATE_DIALECT=ca.uhn.fhir.jpa.model.dialect.HapiFhirPostgres94Dialect

HAPI_DB_USER=name
HAPI_DB_NAME=name
HAPI_DB_PASSWORD=password

## Keycloak

KC_KEYCLOAK_ADMIN=admin
KC_KEYCLOAK_ADMIN_PASSWORD=secret
KC_DB_USERNAME=keycloak
KC_DB_PASSWORD=password

## Konga

KONGA_TOKEN_SECRET=some_secret_token

## Kong

KONG_DB_PASSWORD=kong
KONG_DB_USER=kong
```

### Implementación

El servidor FHIR implementa un endpoint `/fhir/metadata` el cual nos devuelve todas las funcionalidades y/o comportamientos (Capability Statement) que éste servidor posee. 

Este endpoint esta libre de mecanismos de autenticación, ya que cada aplicación debe conocer esta información.

Ejemplo de metadata:

```json
{
    "resourceType": "CapabilityStatement",
    "id": "df20b1ee-51a8-4d49-8bf9-bdc541870366",
    "name": "RestServer",
    "status": "active",
    "date": "2023-10-12T19:59:09Z",
    "publisher": "Not provided",
    "kind": "instance",
    "software": {
        "name": "HAPI FHIR Server",
        "version": "6.8.3"
    },
    "implementation": {
        "description": "HAPI FHIR R4 Server",
        "url": "http://hapi-fhir-server:8080/fhir"
    },
    "fhirVersion": "4.0.1",
    "format": [
        "application/fhir+xml",
        "xml",
        "application/fhir+json",
        "json",
        "application/x-turtle",
        "ttl",
        "html/json",
        "html/xml",
        "html/turtle"
    ],
    "patchFormat": [
        "application/fhir+json",
        "application/fhir+xml",
        "application/json-patch+json",
        "application/xml-patch+xml"
    ],
    "rest": [
        {
            "mode": "server",
            "resource": [
                {
                    "type": "Account",
...
```

Para consumir cualquier otro recurso FHIR, es necesario que el cliente este autenticado.

La secuencia de eventos se ejemplifica en el siguiente diagrama:

![Secuencia](diagrama-secuencia.jpg?raw=true "secuencia")

El servidor FHIR se encuentra protegido a través del proxy Kong Gateway. Cualquier solicitud que sea necesaria autenticar, no será procesada si es que no lleva los datos correctos.

### Casos de uso.

#### Solicitar token de autenticación.

1. Para conocer los endpoint asociados al flujo openid debemos consultar la siguiente url.

Endpoint: http://192.168.101.107:8080/realms/interoperabilidad/.well-known/openid-configuration

Ejemplo de respuesta:
```json
{
    "issuer": "http://192.168.101.107:8080/realms/interoperabilidad",
    "authorization_endpoint": "http://192.168.101.107:8080/realms/interoperabilidad/protocol/openid-connect/auth",
    "token_endpoint": "http://192.168.101.107:8080/realms/interoperabilidad/protocol/openid-connect/token",
    "introspection_endpoint": "http://192.168.101.107:8080/realms/interoperabilidad/protocol/openid-connect/token/introspect",
    "userinfo_endpoint": "http://192.168.101.107:8080/realms/interoperabilidad/protocol/openid-connect/userinfo",
    "end_session_endpoint": "http://192.168.101.107:8080/realms/interoperabilidad/protocol/openid-connect/logout",
    "frontchannel_logout_session_supported": true,
    "frontchannel_logout_supported": true,
    "jwks_uri": "http://192.168.101.107:8080/realms/interoperabilidad/protocol/openid-connect/certs",
    "check_session_iframe": "http://192.168.101.107:8080/realms/interoperabilidad/protocol/openid-connect/login-status-iframe.html",
    "grant_types_supported": [
        "authorization_code",
        "implicit",
        "refresh_token",
        "password",
        "client_credentials",
        "urn:openid:params:grant-type:ciba",
        "urn:ietf:params:oauth:grant-type:token-exchange",
        "urn:ietf:params:oauth:grant-type:device_code"
    ],
```

2. Solicitud de token mediante el flujo `Oauth2 client credentials`.

Endpoint: http://192.168.101.107:8080/realms/interoperabilidad/protocol/openid-connect/token

Metodo: POST
Content-Type: application/x-www-form-urlencoded

En este ejemplo los datos de cliente serán:

client_id: "system-client"
client_secret: "secret"
scope: "system-access"

Body:
```text
  "scope": "system-access",
  "client_id": "system-client",
  "client_secret": "secret",
  "grant_type": "client_credentials"
```

Ejemplo de respuesta exitosa:

```json
{
    "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJxcDlmM1FpQzBKX2hEX3NNXzdlNnpBbGhrM2x3TEZFdkFuZWRLaGxXREh3In0.eyJleHAiOjE3MDI0ODU3NDEsImlhdCI6MTcwMjQ4NTQ0MSwianRpIjoiZTI4MWNkMjQtMjg3ZC00NzhmLTk4YzItNjIwMjIyZDhiNzQ0IiwiaXNzIjoiaHR0cDovLzE5Mi4xNjguMTAxLjEwNzo4MDgwL3JlYWxtcy9pbnRlcm9wZXJhYmlsaWRhZCIsInN1YiI6InNlcnZpY2UtYWNjb3VudC1zeXN0ZW0tY2xpZW50IiwidHlwIjoiQmVhcmVyIiwiYXpwIjoic3lzdGVtLWNsaWVudCIsInNjb3BlIjoic3lzdGVtLWFjY2VzcyIsImNsaWVudEhvc3QiOiIxOTIuMTY4LjEuMjU0IiwiY2xpZW50QWRkcmVzcyI6IjE5Mi4xNjguMS4yNTQiLCJjbGllbnRfaWQiOiJzeXN0ZW0tY2xpZW50In0.HMWQtV76rvY3gnt9peM4GAkodv1ZhNm9NW1xt-pTf4rf7OqlnkJYJCKgRwIW-TjwLMerER-ciowNi2ahl5GONtM-eTPThsOunOQLYaBE9n_k1O2zkYoBal3ugonTgG1CLbndxla5VnFg5iwwxrbGYc-MeKWCOXNq4iUSqB56HZKAo_hR3GGZU8zZV1-AAQDZC-CGd1Tlf26nYEKTZckgEA-XjgOu7hTDlurF3FmUn4Ik9FRrfOUpapSwbP8cwp6jPBL2I0MaefQO44X6WtLqqLArnPyyQRT7xG-HkyhwkHcg-Cu63lyWP-bsWvTolTrGQp7lUXJl_cRXkCmsgMn-Cg",
    "expires_in": 300,
    "refresh_expires_in": 0,
    "token_type": "Bearer",
    "not-before-policy": 0,
    "scope": "system-access"
}
```

Ejemplo de respuesta fallida:

```json
{
    "error": "unauthorized_client",
    "error_description": "Invalid client or Invalid client credentials"
}
```

#### Obtener Resurso FHIR

1. Con el access token ahora podemos hacer una petición al servidor FHIR.

Endpoint: http://192.168.101.107:8000/fhir/Patient
Method: GET

Agregar el token obtenido en lo headers de la petición `Authorization: Bearer <access-token>`.

Ejemplo de respuesta exitosa:

```json
{
    "resourceType": "Bundle",
    "id": "e5edbb05-7880-4efb-b3af-903f59e273ed",
    "meta": {
        "lastUpdated": "2023-12-13T13:55:19.028+00:00"
    },
    "type": "searchset",
    "total": 0,
    "link": [
        {
            "relation": "self",
            "url": "http://hapi-fhir-server:8080/fhir/Patient"
        }
    ]
}
```
Ejemplo de respuesta fallida

```json
{
    "data": [],
    "error": {
        "code": 401,
        "message": "The resource owner or authorization server denied the request."
    }
}
```

### Aplicaciones

#### Administrador Keycloak

Endpoint: http://192.168.101.107:8080

usuario: admin
password: btF5qd9fzG

#### Administrador Kong

Endpoint: http://192.168.101.107:1337

usuario: administrador
password: btF5qd9fzG

#### Listener Proxy Kong Gateway

Endpoint base: http://192.168.101.107:8000

URLs implementadas:

http://192.168.101.107:8000/fhir/metadata
http://192.168.101.107:8000/fhir/*


Referencias:

FHIR: https://www.hl7.org/fhir/

Keycloak: https://www.keycloak.org/documentation

Kong Gateway: https://docs.konghq.com/gateway/3.5.x/

HAPI FHIR: https://hapifhir.io/hapi-fhir/docs/
