# StockTrack Biometric Login — What was added & how to finish setup

## Why this needs a small backend piece

WebAuthn/passkeys are cryptographic — a browser can prove "the device's secure
hardware signed this challenge," but only a server can *verify* that signature
against the public key it stored earlier, and only a server should be trusted
to then issue a real Supabase session. Doing this purely in the browser (as a
localStorage flag, or "if fingerprint prompt succeeds, just log the user in")
would be a fake biometric implementation — exactly what you asked me not to
build. So this ships with two small Supabase Edge Functions that do the actual
verification. **Stocktrack.html itself never contains a service-role key.**

## What's in this delivery

- `Stocktrack.html` — your app, modified. All existing functionality
  (dashboard, transactions, inventory, reports, settings, CSV import/export,
  backup/restore, logout, PIN recovery, desktop sidebar, etc.) is untouched.
- `supabase/functions/webauthn-register/index.ts` — issues registration
  challenges and verifies new passkey registrations (stores public key only).
- `supabase/functions/webauthn-authenticate/index.ts` — issues login
  challenges, verifies the signed assertion, and mints a genuine Supabase
  session only after successful cryptographic verification.
- `supabase/migrations/0001_webauthn_biometric.sql` — the two tables these
  functions need (`fintrack_webauthn_credentials`, `fintrack_webauthn_challenges`),
  locked down with RLS so the browser can never read/write them directly.

## One-time setup (you'll need Supabase CLI access to your project)

```bash
# 1. Run the migration (SQL editor in Supabase dashboard, or:)
supabase db push

# 2. Deploy the two functions
supabase functions deploy webauthn-register
supabase functions deploy webauthn-authenticate --no-verify-jwt

# 3. Set the RP (Relying Party) secrets — must match the exact domain
#    StockTrack is served from (no scheme for RP_ID, full https URL for RP_ORIGIN)
supabase secrets set RP_ID=yourapp.example.com
supabase secrets set RP_NAME="StockTrack"
supabase secrets set RP_ORIGIN=https://yourapp.example.com
```

`SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, and `SUPABASE_ANON_KEY` are
already available to Edge Functions automatically — you don't set those.

If you serve StockTrack from a different domain, update `RP_ID`/`RP_ORIGIN`
accordingly and redeploy. WebAuthn will simply refuse to work if these don't
match the page's actual origin — that's expected, not a bug.

## How the behavior maps to your spec

| Requirement | Implementation |
|---|---|
| First login always normal | Biometric UI never appears until `checkAuthSession()` confirms a real Supabase session exists. |
| Biometric never shown on desktop/laptop | `isLikelyMobileOrTabletDevice()` requires touch + a mobile UA/narrow screen *and* a working platform authenticator; the whole settings block and login button are set to `display:none`, not just disabled. |
| Settings → Biometric Login | New card in Settings, only rendered when `isBiometricCapableDevice()` resolves true. |
| Enabling triggers real registration | `registerBiometricCredential()` calls `navigator.credentials.create()`, backed by `webauthn-register/options` + `/verify`. Toggle only turns on if verification succeeds. |
| No secrets stored client-side | Only a non-secret credential *ID* and username are kept in localStorage — the private key stays in the device's secure hardware; the server stores only the public key. |
| Login screen offers biometrics on returning mobile devices | `initBiometricLoginButton()` shows the button only if this device previously registered a credential for the account. |
| Fallback to password always available | The normal login form is never hidden or blocked; every biometric failure path shows a toast and leaves the person on the login screen. |
| Logout keeps biometric registration | `executeConfirmedLogout()` intentionally does not clear the WebAuthn credential pointer. |
| Disable removes the credential | Toggle OFF calls `webauthn-register/revoke`, deleting the stored public key server-side, then clears the local pointer. |
| Per-device / per-credential | Registration is tied to `credential.id` per browser/device; nothing assumes one device's registration applies to another. |
| Supabase session integrity | `webauthn-authenticate/verify` only issues tokens via Supabase's own admin link-generation + OTP exchange after a verified signature — never a client-side fabricated session. |

## Testing without deploying yet

Until the Edge Functions are deployed, the biometric *toggle* and *login
button* will still correctly appear/hide based on device capability, but
attempting to enable biometric login will show a network-error toast and
safely leave the toggle OFF — it will never fake success. Deploy the
functions above to make it fully functional end-to-end.
