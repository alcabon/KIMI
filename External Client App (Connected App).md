#  Brand-new External Client App 

Below is a step-by-step “recipe” that works in Summer ’25 orgs when you need a brand-new **External Client App** (Connected App) to obtain Salesforce-scoped access tokens with the **OAuth 2.0 Client-Credentials grant** (no PKCE, no refresh tokens, no human login).  The hard part—the auto-generated or region-specific URLs and the exact scopes—are called out explicitly.

---

### 1. Create the Connected App (one-time)

| Field | What to enter / why |
|---|---|
| **Enable OAuth Settings** | ☑ |
| **Enable for Client Credentials** | ☑ (appears only in Summer ’25+) |
| **Callback URL** | Put anything syntactically valid, e.g. `https://not-used.example.com` (will never be called). |
| **Scopes** | Add only the scopes you need.  For pure client-credentials you usually want **api**, **refresh_token** (yes, it is still listed even though no refresh token is issued), and any custom named-credential scopes such as `lightning.force.com`, `visualforce`, `chatter_api`, etc. |
| **IP Relaxation** | “Relax IP restrictions” unless you have a static egress range you can whitelist. |
| **Require Secret for Web Server Flow** | Leave ON (it has no effect on client-credentials). |

Save → Salesforce generates the **Consumer Key** (client_id) and **Consumer Secret** (client_secret).

---

### 2. Build the token endpoint URL (auto-generated)

1. From **Setup > Company Information** copy the **Instance Name** (e.g. `NA123`).
2. The token endpoint is **always**  
   ```
   https://<instance>.salesforce.com/services/oauth2/token
   ```
   If you are in a My Domain sandbox the host becomes  
   ```
   https://<mydomain>--<sandbox>.cs<NN>.my.salesforce.com/services/oauth2/token
   ```
   Do **not** guess the hostname—use the exact value shown in **Setup > API > OAuth Endpoints** or simply hit  
   ```
   https://<mydomain>.my.salesforce.com/.well-known/openid-configuration
   ```
   and read the `token_endpoint` field.

---

### 3. Request the token

HTTP POST (form-url-encoded):

```http
POST https://<instance>.salesforce.com/services/oauth2/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&
client_id=<ConsumerKey>&
client_secret=<ConsumerSecret>&
scope=api%20refresh_token%20chatter_api
```

- Separate scopes with **space or %20** (Salesforce accepts both).  
- No `redirect_uri`, no `code`, no PKCE parameters.  
- Salesforce returns an **opaque access token** (JWT if you also added the `openid` scope, but that is optional).

---

### 4. Consume the token

Use the token exactly as a session-id:

```http
GET https://<instance>.salesforce.com/services/data/v61.0/sobjects/Account
Authorization: Bearer 00D...!
```

---

### 5. Dealing with “auto-generated” changes

- **Instance Migration**: If Salesforce migrates your org to a new POD the hostname changes (`NA123` → `NA212`).  Your connected-app record and its credentials remain valid—only the host prefix changes.  Always read the live `.well-known/openid-configuration` at runtime if you want to be future-proof.  
- **Sandbox refresh**: After refresh the sandbox hostname changes (`cs42` → `cs57`).  When you reconnect, use the **new** `token_endpoint` but keep the **same Consumer Key/Secret** (unless you re-created the connected app in the refreshed sandbox).

---

### Quick checklist

| Item | Value / Command |
|---|---|
| Grant type | `client_credentials` |
| Token endpoint | `https://<instance>.salesforce.com/services/oauth2/token` |
| Required headers | `Content-Type: application/x-www-form-urlencoded` |
| Scope delimiter | space or `%20` |
| PKCE | Not used |
| Refresh token | Never issued |
| Rotate secret | Manual: Setup > App Manager > (app) > Manage Consumer Details |

That is the entire flow with all the “precise scopes and URLs” that Salesforce currently supports for client-credentials.

---

Below is a **concise checklist** that shows the **two-step installation difference** when you want to consume an **External Client App** (the new-generation Connected App) from **Named Credentials** for an **OAuth 2.0 Client-Credentials (no PKCE)** integration.

The instructions are valid for **Summer ’25 orgs** and later.

---

### STEP 1 – Install / subscribe to the External Client App

| Task | Where to do it | What to set | Special notes |
|---|---|---|---|
| 1.1  Install package (if it is **Packaged**) | Setup ➞ **External Client App Manager** | Install the 2-GP package supplied by the vendor | After install the app appears in the list with **Distribution State = Packaged** |
| 1.2  Verify ownership | Same screen | Click the ▾ next to the app name | • If you see **“Edit Settings”** ➞ your org is **owner**<br>• If you only see **“Edit Policies”** ➞ your org is **subscriber** |
| 1.3  (Only if owner) Save Consumer Secret | Setup ➞ **External Client App Manager** ➞ app ➞ **View Consumer Details** | Copy the **Consumer Key** and **Consumer Secret** | Subscriber orgs cannot see the secret; the vendor must share it out-of-band |

---

### STEP 2 – Create the Named Credential (new objects)

Named Credentials for External Client Apps now rely on two **new metadata types** that did not exist for classic Connected Apps:

- **External Credential** – holds the secret(s)  
- **Named Credential** – holds the endpoint & links to the credential

| Task | Where to do it | What to set | Special notes |
|---|---|---|---|
| 2.1  Create **External Credential** | Setup ➞ **Named Credentials** ➞ **New External Credential** | • Protocol: **OAuth 2.0**<br>• Identity Type: **Named Principal** (single service account)<br>• Authentication Parameters:<br>  – `ClientId` = Consumer Key from 1.3<br>  – `ClientSecret` = Consumer Secret from 1.3<br>  – `Scope` = e.g. `api refresh_token`<br>  – `GrantType` = `client_credentials` | The new UI exposes these as key/value pairs instead of the old “Custom Headers” trick |
| 2.2  Create **Named Credential** | Setup ➞ **Named Credentials** ➞ **New Named Credential** | • URL = `https://<yourInstance>.salesforce.com` (or the partner’s endpoint)<br>• External Credential = pick the one created in 2.1<br>• **Generate Authorization Header** = ☑<br>• Allow Formulas in HTTP Header = ☐ (not needed for client-credentials) | The URL must be the **token host**, not the token path; Salesforce will append `/services/oauth2/token` automatically |
| 2.3  Authorize | Save & **Start Authentication Flow on Save** | One-time server-to-server handshake | Because `client_credentials` does not involve a browser, the UI simply validates the secret and marks the credential **Authenticated** |

---

### Using it in Apex / Flow

```apex
HttpRequest req = new HttpRequest();
req.setEndpoint('callout:My_Named_Credential/services/data/v61.0/sobjects/Account');
req.setMethod('GET');
HttpResponse res = new Http().send(req);
```

---

### Quick visual recap of the “two-step” difference

| Classic Connected App | New External Client App |
|---|---|
| Single file (`ConnectedApp`) | **Header file + Global OAuth file + OAuth Settings file** |
| Owner & subscriber share same UI | **Owner edits Settings, subscriber edits Policies only** |
| Secret stored inside Connected App | **Secret stored in External Credential object** |
| Named Credential uses “Custom Headers” | **Named Credential references External Credential** |

That’s all you need to wire an **External Client App** into a **Named Credential** for **OAuth 2.0 Client-Credentials** without PKCE.

---

For External Client Apps (the new-generation Connected Apps) in Summer ’25, Salesforce auto-generates the **standard OAuth 2.0 endpoints** based on the org’s My Domain / instance host.  
They are **not editable** inside the app once it is packaged, and they are the same paths you already know from classic Connected Apps:

| Endpoint | Default URL pattern |
|---|---|
| **Authorization (login)** | `https://<myDomain>/.my.salesforce.com/services/oauth2/authorize` |
| **Token** | `https://<myDomain>/.my.salesforce.com/services/oauth2/token` |
| **User-Info** (if `openid` scope is used) | `https://<myDomain>/.my.salesforce.com/services/oauth2/userinfo` |
| **Revoke / Logout** | `https://<myDomain>/.my.salesforce.com/services/oauth2/revoke` |

* `<myDomain>` is automatically replaced with the org’s actual My Domain host at runtime (e.g., `mycompany.my.salesforce.com` or `mycompany--sandbox.cs42.my.salesforce.com`).*

Because External Client Apps are designed for **server-to-server** or **headless** flows (especially **client-credentials**), the **Start URL** and **Logout URL** fields are **optional**; if you do populate them, they behave exactly like the classic Connected App fields—i.e., Salesforce redirects the user there after interactive login or logout when `redirect_uri` or `logout_uri` is supplied in the request .

---

With **External Client Apps** and the **Client-Credentials grant** the callback-URL story is actually **much simpler**, because **no user browser is ever involved**.  

- **OAuth flows that need a callback** (authorization-code, implicit, PKCE) use the **Callback URL** you register in the app so that the provider can redirect the browser back with an `authorization_code` or `access_token`.  
- **Client-credentials flow does NOT use a callback**; it is a **direct back-channel call** from your server to the token endpoint.  
- Therefore, **when you configure a Named Credential that is pointing to an External Client App and you select “Client Credentials”**, Salesforce **ignores any Callback URL** you may have entered.  
- During the **“Start Authentication Flow on Save”** step in the Named Credential wizard, Salesforce simply performs a **server-to-server POST** to the token endpoint—no browser redirect, no callback required, and you will see the status flip to **Authenticated** without ever hitting a redirect URI.

So, for client-credentials:

| Field | What to enter | Why |
|---|---|---|
| **Callback URL** in the External Client App | Any syntactically valid URL, e.g. `https://unused.example.com` | Required by the metadata schema but **never used** |
| **Callback URL** in the Named Credential | Leave blank or keep placeholder | **Not referenced** during the credential validation |

In short, the **callback URL is only relevant for interactive flows**; with client-credentials you can safely ignore it.

---

Exactly — **leave the *Callback URL* field on the *Named Credential* itself blank (or put a harmless placeholder)**.

If you populate anything in that field, the **Named Credential validation wizard** will expect to perform an **authorization-code redirect** and will throw:

- “Callback URL mismatch”  
- “Missing parameter: redirect_uri”  

… which of course can’t succeed with the **client-credentials** grant.

So the **key takeaway** (Summer ’25):

- External Client App: any placeholder callback is fine — **not used** during client-credentials.  
- Named Credential: **leave Callback URL blank** → validation becomes a single back-channel call and completes successfully.

---

**Yes—Salesforce still requires you to designate a concrete “Run-As” user even when you use the OAuth 2.0 client-credentials flow**, because every API call ultimately executes inside a Salesforce user context.

### Where you set it
1. In **Setup ➞ App Manager**, open the **External Client App** (or classic Connected App).  
2. Click **Manage ➞ Edit Policies**.  
3. In the **Client Credentials Flow** section, pick a **Run-As User**.  
   - This user becomes the *effective* user whose CRUD, FLS, and sharing rules are applied to every token issued by the flow .  
   - Best practice: create a dedicated **integration user** (with the **Salesforce Integration** license) and scope its permissions down to the minimum set of objects and fields .  
   - Starting Summer ’23 the user no longer has to be **API-Only**, but keeping it API-Only is still recommended for least privilege .  

### Impact
- All inbound REST/SOAP calls that present the client-credentials token run as this single user, so audit logs will show that user as the actor.  
- If you need multiple integrations with different privilege levels, create **one External Client App per integration**, each with its own Run-As user .

---

Running-User rules for **External Client App + Named Credential + OAuth 2.0 Client-Credentials**

1. **During the one-time “Start Authentication Flow on Save”**  
   • No browser pop-up is shown.  
   • Salesforce performs a **server-to-server POST** to the token endpoint **as the *current admin user who is pressing the “Save & Authenticate” button***.  
   • That admin must therefore have:  
     – API Enabled permission  
     – “Manage Named Credentials” permission (or the new permission set license “External Services User” if you are using a scratch-org with restricted admin model)  
   • The admin does **not** need to be the same user who will later call the Named Credential from Apex/Flow; this step is only to exchange the secret and cache the access-token in the **Named Principal** context.

2. **At run-time (Apex, Flow, OmniScript, etc.)**  
   • The **Named Credential** uses **Named Principal** identity type, so every execution runs under the cached token that was obtained in step 1.  
   • **No per-user refresh token is stored**, therefore the **running user of the transaction does not need any OAuth or refresh-token permission**; they only need:  
     – “Apex REST Services” or “Run Flows” (whatever entry point is in use)  
     – (Optional) “API Enabled” if you want to let them do other API work.  

3. **Sandbox refresh / Org split**  
   • After sandbox refresh you must redo step 1 with a sandbox admin—the cached token is wiped.  
   • Production tokens are never shared with sandboxes.

In short:  
- One-time setup: **any admin with “Manage Named Credentials”** can validate.  
- Day-to-day usage: **any user or automated process** can call `callout:My_NC/...`; the token is stored at the **Named Principal** level, so the individual running user has **no OAuth requirement**.

---

Running-User rules for **External Client App + Named Credential + OAuth 2.0 Client-Credentials**

1. **During the one-time “Start Authentication Flow on Save”**  
   • No browser pop-up is shown.  
   • Salesforce performs a **server-to-server POST** to the token endpoint **as the *current admin user who is pressing the “Save & Authenticate” button***.  
   • That admin must therefore have:  
     – API Enabled permission  
     – “Manage Named Credentials” permission (or the new permission set license “External Services User” if you are using a scratch-org with restricted admin model)  
   • The admin does **not** need to be the same user who will later call the Named Credential from Apex/Flow; this step is only to exchange the secret and cache the access-token in the **Named Principal** context.

2. **At run-time (Apex, Flow, OmniScript, etc.)**  
   • The **Named Credential** uses **Named Principal** identity type, so every execution runs under the cached token that was obtained in step 1.  
   • **No per-user refresh token is stored**, therefore the **running user of the transaction does not need any OAuth or refresh-token permission**; they only need:  
     – “Apex REST Services” or “Run Flows” (whatever entry point is in use)  
     – (Optional) “API Enabled” if you want to let them do other API work.  

3. **Sandbox refresh / Org split**  
   • After sandbox refresh you must redo step 1 with a sandbox admin—the cached token is wiped.  
   • Production tokens are never shared with sandboxes.

In short:  
- One-time setup: **any admin with “Manage Named Credentials”** can validate.  
- Day-to-day usage: **any user or automated process** can call `callout:My_NC/...`; the token is stored at the **Named Principal** level, so the individual running user has **no OAuth requirement**.

---

**Yes—Salesforce still requires you to designate a concrete “Run-As” user even when you use the OAuth 2.0 client-credentials flow**, because every API call ultimately executes inside a Salesforce user context.

### Where you set it
1. In **Setup ➞ App Manager**, open the **External Client App** (or classic Connected App).  
2. Click **Manage ➞ Edit Policies**.  
3. In the **Client Credentials Flow** section, pick a **Run-As User**.  
   - This user becomes the *effective* user whose CRUD, FLS, and sharing rules are applied to every token issued by the flow .  
   - Best practice: create a dedicated **integration user** (with the **Salesforce Integration** license) and scope its permissions down to the minimum set of objects and fields .  
   - Starting Summer ’23 the user no longer has to be **API-Only**, but keeping it API-Only is still recommended for least privilege .  

### Impact
- All inbound REST/SOAP calls that present the client-credentials token run as this single user, so audit logs will show that user as the actor.  
- If you need multiple integrations with different privilege levels, create **one External Client App per integration**, each with its own Run-As user .

---

You have spotted the **real-world catch**:  

- **Client-credentials flow** requires a *Run-As* user who is **API-only / cannot log in interactively**.  
- **Named-Credential validation** (the “Start Authentication Flow on Save” button) is **always executed in the browser of the admin who presses the button** and therefore needs an **interactive, non-API-only user** to complete the handshake.

Summer ’25 gives you **three practical patterns** to reconcile the two:

---

### 1. Temporary switch of Run-As user (fastest)
1. In the External Client App, set the **Run-As User** to **your admin** (interactive).  
2. Validate the Named Credential → token is cached.  
3. Immediately change the **Run-As User** back to the API-only integration user.  
   - The cached token is not invalidated by the change; it continues to work until expiry.  
   - No re-validation is required unless the token is revoked or the org is refreshed.

---

### 2. Create a **“Validation” user** (cleaner governance)
- Create a **separate user** (Salesforce license) whose **Login Hours** are 1-minute wide and whose **Profile** has *only* “API Enabled”, “Manage Named Credentials”, and no UI tabs.  
- Use that account solely for the validation step, then disable it or set API-only again.  
- Keeps the real integration user untouched.

---

### 3. Use **SFDX / Metadata API** for headless validation (CI/CD pipelines)
- Generate the **JWT Bearer** assertion yourself (or use `sfdx force:org:display --verbose`) and POST it to the token endpoint **outside the UI**.  
- After the token is received, insert the `NamedCredential` metadata with `NamedPrincipal` and the cached token; Salesforce marks it **Authenticated** without ever invoking the browser wizard.  
- This is the only way to automate the validation in scratch-org or sandbox pipelines where the integration user is strictly API-only.

---

### Quick decision matrix

| Scenario | Recommended pattern |
|---|---|
| One-off validation in prod | Pattern 1 (swap Run-As for 30 s) |
| Regular sandbox refresh | Pattern 2 (dedicated validation user) |
| CI/CD or scratch-org creation | Pattern 3 (headless JWT / Metadata API) |

So the logical problem **does exist**, but Salesforce has left these loopholes to let you finish the validation without ever forcing the API-only integration user to log in interactively.

---
