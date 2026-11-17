<!-- Copyright 2026 Phase Two, Inc. -->
<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Passwordless login by magic link — via raw Admin REST

## What "magic link" actually is

The user clicks a link, does not enter a password, and is signed in. Mechanically:
Keycloak mints a signed, time-limited action token, embeds it in a URL, and emails
that URL. Opening the link authenticates the same way any other action-token flow
does (email verification, password reset) — it is not a bespoke security model, and
that is worth saying if anyone asks whether it is "as secure" as a password: it is
exactly as strong as email delivery and the token's own lifespan.

## Set up one admin REST helper first — tokens expire after 60 seconds

Self-managed Keycloak only. This does NOT apply to Phase Two hosted deployments — there
is no self-service admin REST credential of any kind there (confirmed with Phase Two).
Without MCP, use the dashboard instead; raw REST is a dead end there, not a fallback.

`<admin-access-token>` is a `master`-realm admin access token **the developer gives you** — ask for
the token, never for their admin password (see SKILL.md "Admin token"). A `master` token
**expires after 60 seconds** by default, so it can go stale partway through the task. A `401
Unauthorized` halfway through means the token expired — **not** that the endpoint,
method, or body is wrong: ask for a fresh token and rewrite the helper with it. For a long
session, the developer can mint one from a service-account client with a longer lifespan.

```bash
cat > /tmp/kc <<'EOF'
#!/bin/sh
# kc METHOD PATH [curl args...] — one admin REST call with the developer's token.
# Body goes to stdout (pipe it to jq); "HTTP <status>" goes to stderr.
BASE=http://localhost:8080/auth   # include the relative path if one is configured
m=$1 p=$2; shift 2
exec curl -s -X "$m" "$BASE/admin/realms$p" -H "Authorization: Bearer <admin-access-token>" \
  -H "Content-Type: application/json" -w "%{stderr}HTTP %{http_code}\n" "$@"
EOF
chmod +x /tmp/kc
REALM=<realm>
```

Each step below says which status means success. Several succeed with an **empty
body** — that is the success, not a failure to retry.

## Check the provider is actually installed, first

This capability comes from the **p2-inc `keycloak-magic-link` provider**
(`io.phasetwo.keycloak.magic.auth.magic.MagicLinkAuthenticatorFactory`, provider id
`ext-magic-form`), dropped into Keycloak's `providers/` directory. It is not part of
stock Keycloak. Confirm it before doing anything else:

```bash
/tmp/kc GET "/$REALM/authentication/flows" | jq -r '.[].alias'    # HTTP 200
```

If **`magic link`** is in that list, the provider is present — proceed below. If it is
not, no amount of realm configuration will produce it: the jar has to be installed
and Keycloak restarted first, and that is outside what an Admin REST session can do.
Say so plainly rather than trying to hand-build a flow around `ext-magic-form` when
the provider offering that authenticator does not exist on this server.

The provider itself is **[p2-inc/keycloak-magic-link](https://github.com/p2-inc/keycloak-magic-link)**
— point the developer there to build/download the jar and drop it into `providers/` (then
`kc.sh build` and restart) if it isn't installed yet. That installation step is filesystem-level and
out of scope for this reference, which picks up once the provider (and its auto-created `magic link`
flow) already exist on the server.

## The flow already exists — bind it, don't build it

The moment the provider is present, Keycloak **auto-creates a built-in `magic link`
flow on every realm**, including realms created after the provider was installed.
This is the detail that makes the task look harder than it is: there is no custom
flow to author, only a binding to make.

```bash
# Bind realm-wide...
/tmp/kc GET "/$REALM" > /tmp/realm.json                          # HTTP 200
# set "browserFlow": "magic link" in /tmp/realm.json, then PUT it back
/tmp/kc PUT "/$REALM" --data-binary @/tmp/realm.json             # HTTP 204, empty body = done

# ...or to one client only, via authenticationFlowBindingOverrides on the client,
# if only one application should go passwordless rather than the whole realm.
```

Only bind it where it belongs. Binding realm-wide switches every application's
password login off in favour of email; a client-level override is the right choice
when just one app should go passwordless while everything else keeps password login.

## Realm mail settings — required, and it fails silently without them

The authenticator sends through Keycloak's own configured mail sender. A realm ships
with **no SMTP settings at all**, and the send call catches its own failure
internally — nothing surfaces to the caller. The login page still shows its normal
"check your email" response either way, so a missing SMTP configuration looks
identical to a working one until you check whether mail actually left.

```json
{
  "smtpServer": {
    "host": "smtp.example.com",
    "port": "587",
    "from": "noreply@example.com",
    "auth": "true",
    "user": "...",
    "password": "...",
    "starttls": "true",
    "ssl": "false"
  }
}
```

PUT the whole realm representation with this set — a realm PUT that omits fields
resets them, so read first. Setting `smtpServer` and `browserFlow` in the same
`/tmp/realm.json` and doing one PUT is fine. Expect `HTTP 204` with an empty body.

**Verify by checking what actually left**, not by trusting a `204`. If you have no
real mail provider to point at (a test/sandbox setting), a minimal SMTP-protocol
capture server is enough — Keycloak does not care that nothing downstream is a real
mailbox, only that something answers on the configured host and port.

## The trap: anyone's email creates an account, by default

`ext-magic-form`'s own config carries a "create user if none exists" setting
(`ext-magic-create-nonexistent-user`), and **it defaults to `true`.** Left alone,
typing any email address on the login page — one belonging to nobody — silently
provisions a brand-new account and emails *that* address a working login link. There
is no error, no warning, and no visible difference in what the page shows.

```bash
# 1. Find the ext-magic-form execution's id, and whether it already has a config:
/tmp/kc GET "/$REALM/authentication/flows/magic%20link/executions" \
  | jq '.[] | select(.providerId=="ext-magic-form") | {id, authenticationConfig}'   # HTTP 200

# 2a. No authenticationConfig yet → attach one. POST only:
/tmp/kc POST "/$REALM/authentication/executions/$EXEC_ID/config" -d '{
    "alias": "magic-link-existing-accounts-only",
    "config": {"ext-magic-create-nonexistent-user": "false"}
  }'                                                       # HTTP 201, empty body = done

# 2b. authenticationConfig already set → update that config in place instead:
/tmp/kc PUT "/$REALM/authentication/config/$CONFIG_ID" -d '{
    "id": "'"$CONFIG_ID"'", "alias": "magic-link-existing-accounts-only",
    "config": {"ext-magic-create-nonexistent-user": "false"}
  }'                                                       # HTTP 204, empty body = done

# 3. Confirm: rerun step 1 — authenticationConfig now holds an id. Read it with
/tmp/kc GET "/$REALM/authentication/config/$CONFIG_ID"   # HTTP 200
```

Three responses here look like errors and are not:

- `GET …/executions/$EXEC_ID/config` returns **404**. That path accepts POST only; the
  404 does not mean the path or the execution id is wrong. There is also no list
  endpoint at `GET …/authentication/config` (also 404). Read an existing config only by
  its id, as in step 3.
- The POST in 2a returns **201 with an empty body**. That is success — don't repeat it.
  Each extra POST creates another config and re-links the execution to it.
- A **401** anywhere means the token expired — ask the developer for a fresh one and
  rewrite `/tmp/kc` with it, rather than exploring other endpoints.

Turn this off whenever the request implies "our existing staff/customers", not "let
anyone create an account by typing an email" — which is almost always the intent
behind "passwordless login for our users."

Other config on the same execution, all optional: `ext-magic-update-profile-action`
/ `ext-magic-update-password-action` (required actions to add on a newly created
user — only relevant if create-on-demand is intentionally left on),
`ext-magic-allow-token-reuse` (whether the link can be clicked more than once before
it expires; defaults to reusable), `ext-magic-token-life-span` (seconds, defaults to
86400 = 1 day).

## Why registered and unregistered addresses look the same — on purpose

The authenticator deliberately shows the identical "check your email" response
whichever case applies, to avoid letting a login page be used to enumerate which
addresses are registered. This means the *page response* can never be the check for
whether create-on-demand is off — it looks correct either way. The only way to tell
is checking what happened on the *other* side: was mail sent, and did a user get
created, for an address that should have neither.

## Verifying the whole thing works — drive one real login

Drive an actual login rather than trusting configuration alone. Nothing here has a status
endpoint that says "passwordless is working." Plain `curl` with a cookie jar is enough:

```bash
BASE=http://localhost:8080/auth   # same as in /tmp/kc
JAR=$(mktemp)
form_action() { grep -o 'action="[^"]*"' | head -1 | sed -e 's/^action="//' -e 's/"$//' -e 's/&amp;/\&/g'; }
login_page() {   # fresh login page for this client → prints the form's POST URL
  curl -s -G -c "$JAR" -b "$JAR" "$BASE/realms/$REALM/protocol/openid-connect/auth" \
    --data-urlencode "client_id=<client-id>" --data-urlencode "redirect_uri=<app-redirect-uri>" \
    -d response_type=code -d scope=openid \
    -d code_challenge_method=S256 -d code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM \
    | form_action   # a fixed PKCE challenge is fine: the code is never redeemed
}

# 1. A known address: expect the "check your email" page and NO password field (prints 0)
curl -s -c "$JAR" -b "$JAR" --data-urlencode "username=<an-existing-user's-email>" \
  "$(login_page)" | grep -c 'type="password"'

# 3. Follow the link from the mail: expect the last URL to be the redirect URI with ?code=
#    (curl may then fail to connect if nothing listens there; the printed URL is what counts)
curl -s -L -c "$JAR" -b "$JAR" -o /dev/null -w '%{url_effective}\n' '<link-from-the-mail>'

# 4. An address with no account: the same page as step 1 ...
curl -s -c "$JAR" -b "$JAR" --data-urlencode "username=<an-address-with-no-account>" \
  "$(login_page)" | grep -c 'type="password"'
# ... and no user was created (expect [])
/tmp/kc GET "/$REALM/users?email=<an-address-with-no-account>&exact=true"
```

The checks, and which ones actually prove something:

1. A known address on the client's login page gets no password field.
2. **Mail actually arrived** (real inbox, or your capture point) with a link containing
   `login-actions/action-token`. Check the inbox. The page says "check your email" whether
   or not anything was sent.
3. **The link reaches the application.** Opened with nothing else entered, it ends at the
   app's redirect URI with an authorization `code`.
4. An address with no account gets the *same* page as step 1. Out of band, no mail went out
   and no user was created. On the MCP path, use `findUser` instead of `/tmp/kc`.

Steps 2 and 3 are the ones that count. A check that only confirms the "check your email"
page will report success when no mail left the server. **Once all four pass, the
configuration is done — stop there** rather than re-inspecting flows and executions.
If one fails, the Troubleshooting table below says which setting to fix. Fix that one
thing and rerun the failing step.

Two client-side details, if scripting this rather than clicking through it:

- Keycloak's auth cookies are `Secure; SameSite=None`. Browsers and `curl` send them over
  `http://localhost` (loopback counts as secure), but not over plain `http://` to any other
  host, and every form POST then fails as `cookie_not_found`. Use `https` or `localhost`.
- The action-token URL is exactly what a normal login-flow redirect produces — follow
  it the same way you would any other redirect chain, including through
  `required-action` hops, until it reaches the application's `redirect_uri`.

## Troubleshooting

| Symptom | Cause |
| --- | --- |
| No "magic link" flow on the realm at all | The provider jar isn't installed on this server |
| "Check your email" shown, but no mail arrives, for anyone | Realm SMTP settings are unconfigured or wrong |
| An unregistered address gets a working login link | `ext-magic-create-nonexistent-user` left at its default `true` |
| Login page behaves differently for known vs. unknown addresses | Something downstream (a custom flow, a proxy) broke the built-in anti-enumeration behaviour — this is not stock; investigate what changed it |
| Link works but only once | `ext-magic-allow-token-reuse` is false — expected if that's the intent, otherwise flip it |
| Existing password login broke everywhere | The flow was bound realm-wide when only one client should have gone passwordless |
