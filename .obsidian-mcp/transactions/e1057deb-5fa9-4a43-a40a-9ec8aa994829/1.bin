# AHS Agent Authentication Flow

The browser logs into the local Agent, which performs the authoritative credential check at AHS. Browser credentials and OAuth bearer tokens do not authenticate the browser to AHS directly.

1. `ApplicationLoginPage.vue` collects username/password, or receives an MSAL token. `ApplicationSessionStore.initApplication` POSTs to the local Agent at `/api/applications/{type}/login?id={lookupId}`. OAuth uses `Authorization: Bearer <token>`; password login sends `{ username, password, deviceId }` in the body.
2. The Agent's `ApplicationAuthenticationFilter` resolves the application from `id`, reads the client IP (`X-Forwarded-For`, then `X-Real-IP`, then remote address), and creates either a `UserPasswordAuthenticationToken` or a bearer token backed by the JWT principal.
3. `IApplicationController` maps that token to `ApplicationCredentials`, including application lookup ID, type, user principal/password or OAuth marker, device ID where supplied, and remote address. It calls `QueryService.authenticate`.
4. `QueryService` authenticates its Agent-to-AHS WebClient using the configured server-auth mode (API key or OAuth client registration), then POSTs the credential payload to AHS at `/api/v1/agents/{agentId}/applications/authenticate`.
5. AHS `AgentRestController` delegates to `ApplicationService`/`ApplicationQueryService`. It checks the configured credential record and application worker/device authorization, creates the persistent AHS application session, and returns `ApplicationAuthenticated` containing identity, device/worker context, session ID, and optional idle timeout.
6. The Agent turns that response into a local `ApplicationAuthentication` and local `ApplicationSession`; for device logins it also enforces the configured device IP authorization. The UI receives the local session ID, stores it in session storage, and sends it as `x-session-id` on subsequent Agent requests. The Agent validates the local session and its idle timeout; it does not forward that browser session ID to AHS for each request.

Runtime observation (2026-10-13): AHS started with the AHS local/dev profiles on port 8080. Agent started with `local-dev`, completed initialization, and logged its public IP through `IpAddressApplicationRunner`/`PublicIpUtil`. The Agent UI dev server started at `http://localhost:9001/`.

Caveat: the Agent reported failures for unavailable local ATOP hardware connectors during initialization, but its initialization completed and the web/API process remained up.
