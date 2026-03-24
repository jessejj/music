# Centralized Auth Integration Guide

## Overview

```
music.jjjp.ca (GitHub Pages)     jjjp.ca/auth/          jjjp.ca/music/api.php
┌─────────────────────┐     ┌──────────────────┐     ┌──────────────────────┐
│  auth.js             │     │  app_token.php    │     │  api.php             │
│                      │     │                   │     │                      │
│  1. User clicks      │────>│  2. Passkey login │     │  5. Validates token  │
│     "Authenticate"   │     │                   │     │     from APP_TOKENS  │
│                      │     │  3. Generate token│     │     (falls back to   │
│  4. Captures ?token= │<────│     in APP_TOKENS │     │     legacy API_KEYS) │
│     stores locally   │     │     redirect back │     │                      │
│                      │     └──────────────────┘     │  6. whoami endpoint  │
│  5. Uses token as    │─────────────────────────────>│     returns user info │
│     X-API-Key header │     posts.db                  └──────────────────────┘
└─────────────────────┘     ┌──────────────────┐
                             │  USERS (existing) │
                             │  APP_TOKENS (new) │
                             └──────────────────┘
```

## Files Included

| File                  | Goes to                    | Purpose                                    |
|-----------------------|----------------------------|--------------------------------------------|
| `add_app_tokens.sql`  | Run against `posts.db`     | Creates the APP_TOKENS table               |
| `app_token.php`       | `/auth/app_token.php`      | Generates tokens after login, redirects    |
| `api_auth_snippet.php`| Reference for `api.php`    | Replace auth block + add whoami endpoint   |
| `auth.js`             | `music.jjjp.ca` repo       | Client-side token handling                 |

## Step-by-step Setup

### 1. Run the migration

```bash
sqlite3 /path/to/posts.db < add_app_tokens.sql
```

### 2. Deploy app_token.php

Copy `app_token.php` to your `/auth/` directory. Make sure it can access your
`functions.php` (adjust `require` path if needed). It expects `PostDB` and
`session_start()` to be available.

### 3. Update api.php

In `jjjp.ca/music/api.php`:

**a)** Add the `AUTH_DB_PATH` constant and `validateAppToken()` function from
`api_auth_snippet.php`.

**b)** Replace the auth block (around line 72) from:
```php
if (!in_array($key, API_KEYS, true)) {
    http_response_code(401);
    ...
}
```
to:
```php
$authUserId = validateAppToken($key);
if ($authUserId === null && !in_array($key, API_KEYS, true)) {
    http_response_code(401);
    header('Content-Type: application/json');
    echo json_encode(['error' => 'Unauthorized']);
    exit;
}
```

**c)** Add the `whoami` action to the elseif chain:
```php
elseif ($action === 'whoami') handleWhoAmI();
```

**d)** Add the `handleWhoAmI()` function from the snippet.

### 4. Add auth.js to music.jjjp.ca

**a)** Add `auth.js` to your repo.

**b)** On page load, capture the callback:
```js
import { auth } from './auth.js';

// Capture token from redirect
auth.handleCallback();
```

**c)** In your settings/UI, add the authenticate button:
```js
if (auth.isAuthenticated()) {
    const user = await auth.whoami();
    // Show: "Signed in as {user.given_name}" + avatar
} else {
    // Show: "Authenticate" button
    document.getElementById('auth-btn').onclick = () => auth.login();
}
```

**d)** Replace wherever you currently read the API key with:
```js
headers: { 'X-API-Key': auth.getToken() }
```

### 5. Migrate existing users (optional)

To give your 4 existing API key users tokens without making them re-auth,
insert rows manually:

```sql
INSERT INTO APP_TOKENS (USER_ID, APP, TOKEN) VALUES
    (2,   'music', '81fec16a75cfe311dbbb1266eaad74fbe50abab70975741c8f5c0f40cb44256e'),
    (13,  'music', 'd921474d3fb0e3bbd9877e071172d2fe92d10ad91a30490f7ad802ff18fe372c');
-- Map each existing API_KEYS entry to the correct USERS.ID
```

This way the old keys work through the new APP_TOKENS path too, and you can
eventually remove the API_KEYS array entirely.

## Adding Future Apps

1. Add the app name to `$allowedApps` in `app_token.php`
2. Add the domain to `$allowedHosts` in `app_token.php`
3. Copy `auth.js` to the new app, change `AUTH_CONFIG.app` and `AUTH_CONFIG.apiUrl`
4. In that app's API, validate with:
   `SELECT USER_ID FROM APP_TOKENS WHERE APP = 'yourapp' AND TOKEN = :token`
