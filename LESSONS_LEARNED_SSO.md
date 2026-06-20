# Lessons Learned — Microsoft Entra ID SSO with MSAL v5
### SharePoint AI Chatbot Project | June 2026

---

## Overview

Integrating Microsoft Entra ID Single Sign-On into a React SPA using `@azure/msal-browser` v5 and `@azure/msal-react` v5 was significantly harder than documented. This document captures every issue encountered, the root cause, and the exact fix — so the next person on this team doesn't spend days on what should take hours.

---

## Issue 1 — Infinite Redirect Loop After Login

**Symptom:** User clicks "Sign In", completes login on Microsoft's page, gets redirected back to `localhost:3000` — and the login page appears again. URL briefly shows `?code=...` then resets.

**Root Cause:** `handleRedirectPromise()` was never called. When MSAL does a redirect login, the app reloads at the redirect URI with an auth code in the URL hash/query. `handleRedirectPromise()` must be called to consume that code and exchange it for tokens. Without it, MSAL never processes the code, sees no session, and re-triggers login.

**Fix:**
```typescript
// index.tsx — call BEFORE React renders anything
msalInstance.initialize().then(() => {
  return msalInstance.handleRedirectPromise();
}).then((response) => {
  if (response?.account) {
    msalInstance.setActiveAccount(response.account);
  } else {
    const accounts = msalInstance.getAllAccounts();
    if (accounts.length > 0) msalInstance.setActiveAccount(accounts[0]);
  }
  renderApp();
});
```

**Key rule:** `handleRedirectPromise()` must be called exactly once, in `index.tsx` (the entry point), before React renders. Never call it again inside `App.tsx` or any component — calling it twice causes `interaction_in_progress` BrowserAuthError.

---

## Issue 2 — Blank Screen After Adding React Router

**Symptom:** After adding `react-router-dom` and `useNavigate()` in `App.tsx`, the app showed a completely blank screen with no errors in the React compiler output.

**Root Cause:** `BrowserRouter` was missing from `index.tsx`. React Router's `useNavigate()` hook silently crashes when there is no Router context above it in the tree, producing a blank screen rather than a visible error.

**Fix:**
```tsx
// index.tsx — BrowserRouter MUST wrap MsalProvider
root.render(
  <React.StrictMode>
    <BrowserRouter>           {/* ← was missing */}
      <MsalProvider instance={msalInstance}>
        <App />
      </MsalProvider>
    </BrowserRouter>
  </React.StrictMode>
);
```

**Key rule:** Router context must be the outermost wrapper. Order matters: `BrowserRouter → MsalProvider → App`.

---

## Issue 3 — AADSTS9002326: Cross-Origin Token Redemption Blocked

**Symptom:** After redirect from Microsoft login, console showed:
```
AADSTS9002326: Cross-origin token redemption is permitted only for the 
'Single-Page Application' client-type. Request origin: 'http://localhost:3000'
```

**Root Cause:** The Azure App Registration had `http://localhost:3000` registered as a **Web** platform redirect URI. The Web platform uses server-side auth code flow which doesn't support cross-origin token exchange from a browser. MSAL v5 uses PKCE which requires the **Single-page application (SPA)** platform type.

**Fix — Azure Portal:**
1. App Registration → Authentication
2. Find the redirect URI under "Web" platform — **delete it**
3. Click "Add a platform" → choose **Single-page application**
4. Add `http://localhost:3000` (no trailing slash)
5. Save

**Key rule:** `http://localhost:3000` and `http://localhost:3000/` are treated as different URIs. No trailing slash. Must be registered under SPA platform, not Web.

---

## Issue 4 — TypeScript Compilation Errors with MSAL v5

**Symptom:** Multiple TypeScript errors after copy-pasting MSAL v4/v2 code:
- `Property 'LOGIN_FAILURE' does not exist on type EventType`
- `navigateToLoginRequestUrl does not exist in type BrowserAuthOptions`
- `storeAuthStateInCookie does not exist in type CacheOptions`

**Root Cause:** MSAL v5 (`@azure/msal-browser@5.x`) removed several APIs that existed in v2/v3:

| Property | Status in v5 |
|----------|-------------|
| `EventType.LOGIN_FAILURE` | Removed — use `EventType.ACQUIRE_TOKEN_FAILURE` |
| `navigateToLoginRequestUrl` | Removed from `BrowserAuthOptions` |
| `storeAuthStateInCookie` | Removed — IE11 support dropped |

**Fix:**
```typescript
// authConfig.ts — v5 compatible, no removed properties
export const msalConfig: Configuration = {
  auth: {
    clientId: "...",
    authority: "https://login.microsoftonline.com/TENANT_ID",
    redirectUri: "http://localhost:3000",
    postLogoutRedirectUri: "http://localhost:3000",
    // navigateToLoginRequestUrl ← REMOVE, does not exist in v5
  },
  cache: {
    cacheLocation: "sessionStorage",
    // storeAuthStateInCookie ← REMOVE, does not exist in v5
  },
};

// App.tsx — use ACQUIRE_TOKEN_FAILURE not LOGIN_FAILURE
if (message.eventType === EventType.ACQUIRE_TOKEN_FAILURE) {
  console.error("Auth failure:", message.error);
}
```

---

## Issue 5 — JWT Signature Verification Failure (401 on Every Request)

**Symptom:** Frontend successfully authenticated. Backend received the Bearer token. But every API call returned 401 with `Signature verification failed` — even after the correct Microsoft public key was found by `kid` matching.

**Root Cause:** Microsoft Graph access tokens (issued for `User.Read` scope) are **opaque tokens**. Microsoft explicitly does not support third-party verification of Graph token signatures. The RSA public key from `https://login.microsoftonline.com/TENANT_ID/discovery/v2.0/keys` is the correct key, but the signature still fails because Graph tokens contain internal Microsoft claims that make external verification intentionally impossible.

This is by design. From Microsoft's documentation: *"Access tokens for Microsoft APIs (including Microsoft Graph) are opaque tokens and should not be parsed or validated by client applications."*

**Fix — Short term (development):**
Decode without signature verification, validate only expiry and tenant:
```python
payload = jwt.decode(
    token,
    options={
        "verify_signature": False,  # Graph tokens are opaque
        "verify_exp": True,         # Still check expiry
        "verify_aud": False,
    },
    algorithms=["RS256"],
)
# Manual tenant check
if settings.TENANT_ID not in payload.get("iss", ""):
    raise HTTPException(401, "Wrong tenant")
```

**Fix — Production (correct approach):**
Expose your own API scope in Azure and request it from the frontend:

1. Azure Portal → App Registration → **Expose an API**
2. Set Application ID URI: `api://YOUR_CLIENT_ID`
3. Add scope: `Chat.Access`
4. Frontend `authConfig.ts`:
   ```typescript
   export const loginRequest = {
     scopes: ["User.Read", "api://YOUR_CLIENT_ID/Chat.Access"]
   };
   ```
5. Backend — verify normally with your Client ID as audience:
   ```python
   payload = jwt.decode(token, public_key, algorithms=["RS256"],
                        audience=f"api://{settings.CLIENT_ID}")
   ```

Tokens issued for your own API scope ARE fully verifiable with the public keys.

---

## Issue 6 — Wrong Signing Keys (Common vs Tenant-Specific Endpoint)

**Symptom:** Even after resolving the opaque token issue for testing, using `https://login.microsoftonline.com/common/discovery/v2.0/keys` returned keys that didn't match tokens issued by our specific tenant.

**Root Cause:** The `/common/` endpoint returns general Microsoft keys. Your tenant may use tenant-specific keys for some token types. The keys are cached at startup — a stale cache after key rotation also causes this.

**Fix:**
```python
# Use tenant-specific endpoint
MICROSOFT_KEYS_URL = f"https://login.microsoftonline.com/{TENANT_ID}/discovery/v2.0/keys"

# Refresh on key ID miss (handles key rotation)
key_data = next((k for k in azure_keys if k["kid"] == kid), None)
if not key_data:
    azure_keys = fetch_azure_keys()  # refresh
    key_data = next((k for k in azure_keys if k["kid"] == kid), None)
```

---

## Issue 7 — Token Audience Mismatch

**Symptom:** `jwt.InvalidAudienceError` — token was valid but audience check failed.

**Root Cause:** The original backend validated against `api://CLIENT_ID` but the frontend only requested `User.Read` scope. These produce tokens with completely different audiences:

| Frontend Scope | Token Audience |
|----------------|----------------|
| `User.Read` | `00000003-0000-0000-c000-000000000000` (Microsoft Graph) |
| `api://CLIENT_ID/Chat.Access` | `api://CLIENT_ID` (your app) |

**Fix:** Either request your own API scope (production path) or accept both audiences (development path). See Issue 5 above for full details.

---

## Issue 8 — Cached Old Build Ignoring Code Changes

**Symptom:** Changed the API URL in `ChatContainer.tsx` from `/api/chat/stream` to `/chat/stream`. Browser DevTools still showed the old URL in requests. Hard refresh (`Ctrl+F5`) didn't help.

**Root Cause:** React dev server serves a `bundle.js` that was cached by the browser. After stopping and restarting `npm start`, the old bundle was still being served until the process fully restarted.

**Fix:**
1. Stop `npm start` completely (Ctrl+C in terminal)
2. Check the file actually changed: `Select-String -Path "src\components\ChatContainer.tsx" -Pattern "localhost:8000"`
3. Restart: `npm start`
4. Open DevTools → Network tab → check **Disable cache** checkbox before testing

---

## Summary — SSO Checklist

Use this checklist for any future MSAL v5 integration:

- [ ] Azure App Registration platform = **Single-page application** (not Web)
- [ ] Redirect URI = exact match, no trailing slash
- [ ] `handleRedirectPromise()` called once in `index.tsx` before `renderApp()`
- [ ] `BrowserRouter` wraps `MsalProvider` in `index.tsx`
- [ ] No `navigateToLoginRequestUrl`, `storeAuthStateInCookie`, `LOGIN_FAILURE` in v5 code
- [ ] Backend uses tenant-specific key endpoint, not `/common/`
- [ ] Frontend `loginRequest.scopes` matches what backend expects as audience
- [ ] For production: expose `Chat.Access` scope, request it from frontend
- [ ] Never call `handleRedirectPromise()` more than once per page load
- [ ] Test in private/incognito window to avoid cached session state

---

*Document maintained by: Pravin Kumar Sundge | CONNX 2026 | June 2026*
