# SSO / OIDC

TLSentinel supports single sign-on via OpenID Connect. OIDC is enabled automatically when the four required environment variables are set — see [Configuration](../getting-started/configuration.md#oidc-optional) for the full variable reference.

The **Sign in with SSO** button appears on the login page as soon as OIDC is configured. On first SSO login, TLSentinel creates a local user record with the `viewer` role. An admin must promote the user in **Settings → Users** if elevated access is required.

---

## Microsoft Entra ID

### Prerequisites

- An Entra ID tenant with permission to register applications
- TLSentinel server running and accessible at a known URL (e.g. `https://tlsentinel.example.com`)

### 1. Create an App Registration

1. Sign in to the [Entra admin centre](https://entra.microsoft.com)
2. Navigate to **Identity → Applications → App registrations**
3. Click **New registration**

<!-- screenshot: app registrations list with "New registration" button highlighted -->

4. Fill in the registration form:
    - **Name** — something descriptive, e.g. `TLSentinel`
    - **Supported account types** — *Accounts in this organizational directory only* (single tenant)
    - **Redirect URI** — select **Web**, enter:
      ```
      https://your-tlsentinel-url/api/v1/auth/oidc/callback
      ```

5. Click **Register**

<!-- screenshot: completed registration form before clicking Register -->

### 2. Note the Application Details

After registration, you will land on the app overview page. Copy:

- **Application (client) ID** → `TLSENTINEL_OIDC_CLIENT_ID`
- **Directory (tenant) ID** → used to construct the issuer URL below

<!-- screenshot: app overview page with Application ID and Directory ID highlighted -->

The issuer URL follows the pattern:
```
https://login.microsoftonline.com/{tenant-id}/v2.0
```

Set this as `TLSENTINEL_OIDC_ISSUER`.

### 3. Create a Client Secret

1. In the left menu, go to **Certificates & secrets**
2. Click **New client secret**
3. Set a description and expiry, then click **Add**

!!! warning
    Copy the secret **Value** immediately — it is only shown once.

<!-- screenshot: client secret value immediately after creation -->

Set this as `TLSENTINEL_OIDC_CLIENT_SECRET`.

!!! tip
    Track the secret's expiry by adding it as an endpoint in TLSentinel so you get alerted before it expires.

### 4. Configure Token Claims

By default, Entra does not include the user's email in the ID token. To enable it:

1. Go to **Token configuration**
2. Click **Add optional claim**
3. Select **ID token**
4. Check **email**, **given_name**, **family_name**
5. Click **Add**

<!-- screenshot: optional claims selection with email, given_name, family_name checked -->

When prompted to add the Microsoft Graph `email` permission, accept it.

### 5. Set Environment Variables

Add the following to your `.env`:

```env
TLSENTINEL_OIDC_ISSUER=https://login.microsoftonline.com/{tenant-id}/v2.0
TLSENTINEL_OIDC_CLIENT_ID={application-client-id}
TLSENTINEL_OIDC_CLIENT_SECRET={client-secret-value}
TLSENTINEL_OIDC_REDIRECT_URL=https://your-tlsentinel-url/api/v1/auth/oidc/callback
TLSENTINEL_OIDC_SCOPES=openid,profile,email
TLSENTINEL_OIDC_USERNAME_CLAIM=preferred_username
```

Restart the server to apply the changes.

### Troubleshooting

| Symptom | Likely cause |
|---|---|
| Redirect URI mismatch error | The redirect URI in Entra does not exactly match `TLSENTINEL_OIDC_REDIRECT_URL` |
| Email not populated after login | Optional claims not configured — see step 4 |
| SSO button not appearing | One or more OIDC env vars missing or server not restarted |
| Login succeeds but user not created | Check server logs for OIDC callback errors |
