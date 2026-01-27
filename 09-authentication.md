# Three-layer forward authentication using NGINX reverse proxy pattern:

**Forward authentication** (**forward auth**) is a method where a **reverse proxy** (~~like Traefik, Caddy,~~ NGINX) intercepts requests to an application and **forwards** them to an ~~external~~ authentication service (~~like Authentik or Auth0~~) for validation, allowing the proxy to **secure apps without built-in login features** by **adding headers** with **user details** if access is granted.
- If the **auth service** returns a successful response (2xx), the proxy lets the request through; otherwise, it sends the user to a **login page** or returns an error, **centralizing authentication logic** outside the application itself.

<img width="75%" height="75%" alt="image" src="https://github.com/user-attachments/assets/2478eda9-5ab6-47a1-973a-7bf541461d71" />

- **NGINX Ingress** intercepts all requests and makes internal subrequests to **OAuth2 Proxy** (`/oauth2/auth`) before routing to apps.
- **OAuth2 Proxy** validates **session cookies**;
- ...on failure (`401`), **NGINX** redirects browser to **OAuth2 Proxy's login endpoint** which initiates **OIDC** flow with **Dex**.
- **Dex** acts as **OIDC provider abstraction**, **delegating** actual authentication to **Azure Entra ID** via **Microsoft connector**, then issues standard **OIDC tokens** back to **OAuth2 Proxy**.
- **OAuth2 Proxy** exchanges **authorization codes** for **ID tokens**, creates **encrypted session cookies**, and on subsequent requests returns `200` with `X-Auth-Request-*` headers containing **user claims**.
- **NGINX** strips **all incoming auth headers**, injects only **OAuth2 Proxy-validated headers**, and forwards requests to **ClusterIP** apps which trust headers due to **network isolation—apps** cannot be reached except through **NGINX**, eliminating auth logic from application code entirely.

https://github.com/dexidp/dex/blob/master/connector/microsoft/microsoft.go

# Concepts
- **OAuth2-Proxy**: A gatekeeper / a guard at the door. It stops anyone who isn't logged in. It sits in front of your app and forces users to log in before they can enter.
  
- **Dex IdP**: A translator. It talks to many login systems and translates them into one standard language. It doesn't store users itself; instead, it talks to many different login systems and makes them all look like one standard.

- **IdP**: The ID card office. The place that actually stores your username and password (like Google). The source of truth for who a user is (e.g., Google, Okta, or your company's LDAP)
  
- **Upstream IdP**: The external login system that Dex connects to (e.g., GitHub or Active Directory)
  
- **Connectors**: The cables Dex uses to plug into different IdPs. The plugins in Dex used to talk to different Upstream IdPs (e.g., the "GitHub Connector" or "LDAP Connector")
  
- **Federation**: The linking of different identity systems so a user can log in once and access many apps across different networks.
  
- **OIDC (OpenID Connect)**: The modern language used for login. It’s a standard way for an app to ask an IdP, "Who is this person?" and get a "This is me" answer.

- **Token**: A digital pass. After you log in, you get a "Token" (a string of text) that proves your identity to the app. Your digital wristband. You show it to apps to prove you already logged in.

- **Ingress** vs. **Controller**: Ingress is the map; the Controller (Nginx/Traefik) is the driver who follows the map.
  
- **NodePort**: A side door on a server. Opens a specific port (like 30001) on every server in your cluster to let traffic in. Simple but not very secure or scalable
  
- **LoadBalancer**: A main lobby with a public address. A public IP provided by a cloud (like AWS or GCP) that sends traffic into your cluster

- **Ingress**: A smart hallway that sends people to different rooms based on the name on their ticket. A smart router that uses one LoadBalancer to handle many different apps based on their URL (e.g., app1.com vs app2.com).

- An [**ingress controller**](https://ngrok.com/blog/ingress-controller-vs-api-gateway) is a Kubernetes-native component designed to seamlessly route HTTP/HTTPS and other TCP traffic from the outside world (also known as north-south traffic) to the correct backend service running inside the Kubernetes cluster. Since external components lack the context to determine which pod or container should handle a request, an internal component—the ingress controller—is necessary. Essentially, an ingress controller translates Ingress resources into routing rules that reverse proxies can recognize and implement.

<img width="75%" height="75%" alt="image" src="https://github.com/user-attachments/assets/3aa5bf6a-c4c6-4feb-b3e5-7aa4615a579f" />


<img width="1280" height="720" alt="image" src="https://github.com/user-attachments/assets/a08b54f0-b363-422a-b66c-44e5bafe2463" />


- **CRD**: A custom plugin. It lets you add new types of "stuff" to Kubernetes that it didn't have before. A way to teach Kubernetes new tricks. It lets you create your own custom objects (like a DexClient) that K8s doesn't know by default. 
  
- **OAuth2 Proxy Config**: The instructions for the guard (Who to trust? Where is the secret key?). The instructions for the guard (Who to trust? Where is the secret key?).

- **NGINX annotations** in Kubernetes are configuration directives used to customize the behavior of an NGINX Ingress Controller for specific Ingress resources

# Scenario 1: First-Time Unauthenticated User

### Step 1: Initial Request
- User browser → `https://devailab.work/gemini-rl`
- What happens:
    - DNS resolves to **Cloudflare Tunnel**
    - Cloudflare forwards to **NGINX Ingress Controller IP** 
    - Request hits **NGINX** pod in **ingress-nginx namespace**

### Step 2: NGINX Processes Ingress Rules
- NGINX reads **ingress annotations**:

```
nginx.ingress.kubernetes.io/auth-url: "http://oauth2-proxy.default.svc.cluster.local:4180/oauth2/auth"
nginx.ingress.kubernetes.io/auth-signin: "https://devailab.work/oauth2/start?rd=$escaped_request_uri"
nginx.ingress.kubernetes.io/auth-response-headers: "X-Auth-Request-User,X-Auth-Request-Email,..."
```

- NGINX sees `auth-url` **annotation** exists
- ... Pauses main request (do NOT forward to app yet)
- ... Makes **internal subrequest** to **OAuth2 Proxy**

### Step 3: NGINX → OAuth2 Proxy (Auth Subrequest)
NGINX sends:
```
GET http://oauth2-proxy.default.svc.cluster.local:4180/oauth2/auth HTTP/1.1
  Host: oauth2-proxy.default.svc.cluster.local
  X-Forwarded-Host: devailab.work
  X-Forwarded-Uri: /gemini-rl
  X-Forwarded-Proto: https
  Cookie: (forwards all cookies from original request)
```

- This happens entirely **within** Kubernetes cluster by using **ClusterIP service DNS**
- User's browser **never** sees this request
- OAuth2 Proxy listens on **port `4180`** for these auth checks

### Step 4: OAuth2 Proxy Validates Session
OAuth2 Proxy checks:
- **Cookie present?** → Look for `_oauth2_proxy` cookie
- **Cookie valid?** → Check signature, expiration
- **Session exists?** → Look up session in memory/storage

- Result: No valid session found
- OAuth2 Proxy responds to NGINX:
```
HTTP/1.1 401 Unauthorized
X-Auth-Request-Redirect: https://devailab.work/oauth2/start?rd=/gemini-rl
```

### Step 5: NGINX Handles 401 Response

- NGINX sees `401` from **auth subrequest** and:
- Does NOT forward request to app
- Reads `auth-signin` annotation
- Redirects user's browser:

```
HTTP/1.1 302 Found
Location: https://devailab.work/oauth2/start?rd=/gemini-rl
```

- User's browser follows redirect.

### Step 6: User Lands on OAuth2 Proxy Start Page

```
Browser → https://devailab.work/oauth2/start?rd=/gemini-rl
```

This request matches **OAuth2 Proxy ingress**:
```
path: /oauth2
  pathType: Prefix
  backend:
    service:
      name: oauth2-proxy
      port: 4180
```

NGINX routes directly to **OAuth2 Proxy** (no auth check for `/oauth2/` paths).

### Step 7: OAuth2 Proxy Initiates OAuth2 Flow

OAuth2 Proxy:
1. Generates random `state` parameter (**CSRF protection**)
2. Stores `state` + original redirect URL (`/gemini-rl`) in memory
3. Constructs **authorization URL** to Dex:

```
https://devailab.work/dex/auth?
  client_id=oauth2-proxy&
  response_type=code&
  scope=openid+email+profile&
  redirect_uri=https://devailab.work/oauth2/callback&
  state=RANDOM_STATE_STRING
```
Redirects browser:

```
HTTP/1.1 302 Found
Location: https://devailab.work/dex/auth?client_id=...
```

### Step 8: Browser Hits Dex

Browser → `https://devailab.work/dex/auth?client_id=oauth2-proxy&...`

This matches **Dex ingress**:
```
path: /dex
  pathType: Prefix
  backend:
    service:
      name: dex
      port: 5556
```

NGINX routes to Dex (Dex paths also bypass auth).

### Step 9: Dex Processes Authorization Request
Dex validates `client_id` matches static **client config**:

```
staticClients:
- id: oauth2-proxy
  redirectURIs:
  - 'https://devailab.work/oauth2/callback'
```

- Checks if user has **active session** with Dex → No
- Reads **connector config**:

```
connectors:
- type: microsoft
  id: azure
  config:
    clientID: $AZURE_CLIENT_ID
    tenant: $AZURE_TENANT_ID
```
Constructs **Azure AD authorization URL**

### Step 10: Dex → Azure AD
Dex redirects browser to Azure:

```
HTTP/1.1 302 Found
Location: https://login.microsoftonline.com/{TENANT_ID}/oauth2/v2.0/authorize?
  client_id={AZURE_CLIENT_ID}&
  response_type=code&
  redirect_uri=https://devailab.work/dex/callback&
  scope=openid+email+profile&
  state=DEX_STATE_STRING
```
**User leaves your infrastructure** → authenticates with Microsoft.

### Step 11: User Authenticates at Azure

User:
1. Sees Microsoft login page
2. Enters Azure credentials / MFA
3. Azure validates identity
4. Azure redirects back to Dex:

```
Browser → https://devailab.work/dex/callback?
  code=AZURE_AUTH_CODE&
  state=DEX_STATE_STRING
```

### Step 12: Dex Exchanges Code for Tokens
- Dex makes **server-to-server request** to Azure:
```
POST https://login.microsoftonline.com/{TENANT_ID}/oauth2/v2.0/token
Content-Type: application/x-www-form-urlencoded
grant_type=authorization_code&
code=AZURE_AUTH_CODE&
client_id={AZURE_CLIENT_ID}&
client_secret={AZURE_CLIENT_SECRET}&
redirect_uri=https://devailab.work/dex/callback
```

Azure responds:
```
{
  "access_token": "...",
  "id_token": "JWT_WITH_USER_CLAIMS",
  "expires_in": 3600
}
```
Dex decodes `id_token` JWT:
```
{
  "sub": "user-unique-id",
  "email": "user@company.com",
  "name": "John Doe",
  "preferred_username": "john.doe"
}
```

### Step 13: Dex Creates Session & Redirects to OAuth2 Proxy

Dex:
- Stores **authorization code** in Kubernetes CRD (`authcodes.dex.coreos.com`)
- Generates authorization code for OAuth2 Proxy
- Redirects browser back to OAuth2 Proxy callback:

```
HTTP/1.1 302 Found
Location: https://devailab.work/oauth2/callback?
  code=DEX_AUTH_CODE&
  state=ORIGINAL_OAUTH2_PROXY_STATE
```

### Step 14: OAuth2 Proxy Exchanges Code with Dex

Browser hits:
```
https://devailab.work/oauth2/callback?code=DEX_AUTH_CODE&state=...
```
OAuth2 Proxy:
- Validates state matches stored value
- Makes server-to-server request to Dex:

```
POST https://devailab.work/dex/token
Content-Type: application/x-www-form-urlencoded
grant_type=authorization_code&
code=DEX_AUTH_CODE&
client_id=oauth2-proxy&
client_secret=OAUTH2_PROXY_CLIENT_SECRET&
redirect_uri=https://devailab.work/oauth2/callback
```

Dex responds:
```
{
  "access_token": "...",
  "id_token": "JWT_WITH_USER_CLAIMS",
  "refresh_token": "..."
}
```

### Step 15: OAuth2 Proxy Creates Session

1. OAuth2 Proxy decodes Dex's `id_token` JWT:
```
{
  "email": "user@company.com",
  "name": "John Doe",
  "preferred_username": "john.doe"
}
```
2. Creates **session in memory** with **user claims**
3. Generates **session cookie**:
```
_oauth2_proxy=ENCRYPTED_SESSION_ID; 
  Domain=devailab.work; 
  Path=/; 
  Secure; 
  HttpOnly; 
  SameSite=Lax
```

Retrieves original destination from state → /gemini-rl
Redirects browser:

```
HTTP/1.1 302 Found
Set-Cookie: _oauth2_proxy=...; Domain=devailab.work; ...
Location: https://devailab.work/gemini-rl
```

### Step 16: Browser Retries Original Request (With Cookie)
```
Browser → https://devailab.work/gemini-rl
Cookie: _oauth2_proxy=ENCRYPTED_SESSION_ID
```
Request hits NGINX again.

### Step 17: NGINX Auth Subrequest (Second Time)

NGINX makes internal subrequest:
```
GET http://oauth2-proxy.default.svc.cluster.local:4180/oauth2/auth
Cookie: _oauth2_proxy=ENCRYPTED_SESSION_ID
X-Forwarded-Host: devailab.work
X-Forwarded-Uri: /gemini-rl
```

### Step 18: OAuth2 Proxy Validates Session (Success)

OAuth2 Proxy:
- Finds _oauth2_proxy cookie
- Decrypts session ID
- Looks up session → Found!
- Extracts user claims from session
- Responds to NGINX:

```
HTTP/1.1 200 OK
X-Auth-Request-User: john.doe
X-Auth-Request-Email: user@company.com
X-Auth-Request-Preferred-Username: john.doe
```

### Step 19: NGINX Forwards to Application
NGINX:
- Sees 200 OK from auth subrequest
- Strips ALL incoming X-Auth-Request-* headers from original request
- Injects headers from OAuth2 Proxy response
- Forwards to app:

```
GET /gemini-rl HTTP/1.1
Host: geminirealtime.apps.svc.cluster.local:8000
X-Auth-Request-User: john.doe
X-Auth-Request-Email: user@company.com
X-Auth-Request-Preferred-Username: john.doe
X-Forwarded-For: ...
X-Real-IP: ...
```

### Step 20: Application Processes Request
Our FastAPI app:

```
@app.get("/gemini-rl")
def index(
    x_auth_request_email: str = Header(None)
):
    # x_auth_request_email = "user@company.com"
    return {"user": x_auth_request_email}
```

App returns response → NGINX → Browser.

# Scenario 2: Returning User (Has Valid Cookie)
```
Browser → https://devailab.work/helloworld
Cookie: _oauth2_proxy=VALID_SESSION_ID
```

**Flow:**
1. NGINX receives request
2. NGINX → OAuth2 Proxy auth subrequest (with cookie)
3. OAuth2 Proxy validates session → `200 OK` + headers
4. NGINX → App (with identity headers)
5. App responds
6. **No redirects, instant access**


# Configuration Deep Dive

### NGINX Ingress Annotations

```
nginx.ingress.kubernetes.io/auth-url: "http://oauth2-proxy.default.svc.cluster.local:4180/oauth2/auth"
```
- Tells NGINX to make subrequest before forwarding
- Uses cluster-internal DNS
- Port 4180 = OAuth2 Proxy's auth endpoint

```
nginx.ingress.kubernetes.io/auth-signin: "https://devailab.work/oauth2/start?rd=$escaped_request_uri"
```
- Where to redirect on 401
- `$escaped_request_uri` = original path user requested
- Must be public URL (browser follows this)

```
nginx.ingress.kubernetes.io/auth-response-headers: "X-Auth-Request-User,X-Auth-Request-Email,..."
```

- Which headers from OAuth2 Proxy to inject into app request
- Only injected after 200 OK response

### OAuth2 Proxy Config
```
provider = "oidc"
oidc_issuer_url = "https://devailab.work/dex"
```
- **OAuth2 Proxy** acts as **OIDC client**
- **Dex** is the **OIDC provider**

```
redirect_url = "https://devailab.work/oauth2/callback"
```
- Where Dex sends user after authentication
- Must match Dex's `staticClients.redirectURIs`

```
cookie_domains = [ "devailab.work" ]
```
- Session cookie works for this domain
- Must match request `Host header` exactly

```
set_xauthrequest = true
```
- Return user claims as `X-Auth-Request-*` headers
- NGINX reads these and injects into app request

### Dex Config
```
issuer: https://devailab.work/dex
```
- Dex's public identity
- Appears in issued **JWT tokens**
- **OAuth2 Proxy** validates tokens against this issuer

```
staticClients:
- id: oauth2-proxy
  redirectURIs: ['https://devailab.work/oauth2/callback']
  secretEnv: OAUTH2_PROXY_CLIENT_SECRET
```
- Pre-registered OAuth2 client (OAuth2 Proxy)
- `secretEnv` reads from environment variable (from Kubernetes Secret)

```
connectors:
- type: microsoft
  config:
    clientID: $AZURE_CLIENT_ID
    redirectURI: https://devailab.work/dex/callback
```

- Dex acts as **Azure AD client**
- Translates **Azure's tokens** to **standard OIDC**

# Bibliography

- https://aws.amazon.com/blogs/containers/authenticate-to-amazon-eks-using-google-workspace/

<img width="50%" height="50%" alt="image" src="https://github.com/user-attachments/assets/04926f27-4870-473f-bdda-6d47d07d4749" />

- https://blog.thisisaditya.com/authentication-authorization-in-kubernetes-oauth2-proxy-with-dex-idp
- https://datastrophic.io/secure-kubeflow-ingress-and-authentication/
- https://www.kubeblogs.com/authentication-with-oauth2-proxy-and-nginx-ingress-on-kubernetes/

- [What's OIDC protocol?](https://www.authlete.com/developers/tutorial/idp/)

<img width="80%" height="80%" alt="image" src="https://github.com/user-attachments/assets/2c7ad2db-8a36-495f-b36f-193bb593d0c2" />


- https://platform9.com/blog/ultimate-guide-to-kubernetes-ingress-controllers/







































































