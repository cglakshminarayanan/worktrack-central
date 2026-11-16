# WorkTrack: setup, architecture and limits

## What was built

| Area | Status |
|---|---|
| App renamed WorkTrack, 13 colour-matched logos (default plus your 12) | Done |
| Sign-in with email and password (Supabase Auth, project `worktrack-central`) | Done |
| Registration wizard: welcome, accent colour, firm details, location and geofence, hierarchy, review, Drive, done | Done |
| Designation list (15 values incl. Article Assistant, Audit Assistant, Paid Assistant, Admin Staff), working hours, Mon–Sat working days | Done |
| Hierarchy mind map: add, edit, delete, change manager, collapse, zoom | Done |
| Settings (Managing Partner only): details, hours, days, office, geofence, accent, hierarchy, Drive, backups, firm DB | Done |
| Employee logins created by the Managing Partner (email and password, no Google) | Done |
| Drive OAuth, folder creation, verify, disconnect with revocation | Server code deployed, **needs your Google credentials** |
| Backup to Drive with read-back checksum verification | Server code deployed, needs Drive connected |
| Existing sample-data pages (My Day, tasks, timesheets, leave, clients, documents, invoices, calendar, monitor, rates, log, dashboards) | Kept, still sample data |

## Supabase project (already created and configured)

- Project: `worktrack-central`, ref `ozbfdawgrfdlnrvdneoi`, region ap-south-1.
- Schema is applied (see Database → Migrations: `central_schema_v1`, `profiles_designation_nullable`). Tables: `firms`, `profiles`, `hierarchy_members`, `backup_runs`, plus `drive_secrets` and `oauth_states` that no browser can read.
- Row-level security is on everywhere. Browsers can only read their own firm's rows and cannot write directly. All changes go through checked functions (`register_firm`, `update_firm_settings`, `save_hierarchy`) or Edge Functions.
- Edge Functions deployed: `create-employee`, `drive-connect`, `drive-callback`, `drive-manage`.

## Steps you must do

1. **Supabase → Authentication → URL Configuration**: set Site URL and add a Redirect URL for your Netlify address.
2. **Supabase → Authentication → Providers → Email**: decide on "Confirm email". On the built-in mail sender, confirmation emails are heavily rate-limited, so for real use turn confirmation off or add your own SMTP.
3. **Google Cloud Console** (free): create a project, enable the Google Drive API, configure the OAuth consent screen, and create an OAuth client of type *Web application*.
   - Authorised redirect URI: `https://ozbfdawgrfdlnrvdneoi.supabase.co/functions/v1/drive-callback`
   - Scope: only `.../auth/drive.file` (plus openid and email). This scope is non-sensitive, so no Google verification is needed.
   - Set the consent screen to **In production**. In *Testing* mode Google expires refresh tokens after 7 days.
4. **Supabase → Edge Functions → Secrets**, add:
   - `GOOGLE_CLIENT_ID`
   - `GOOGLE_CLIENT_SECRET`
   - `TOKEN_ENC_KEY`: run `openssl rand -base64 32`
   - `APP_URL`: the full address where the app is hosted, including the repo folder on GitHub Pages, e.g. `https://YOUR-USER.github.io/YOUR-REPO/` (used for the redirect back after Drive and for CORS)
5. Upload `worktrack.html` to your GitHub repo as `index.html` (GitHub Pages), later to Netlify the same way.

The anon key inside the HTML is public by design. The service-role key and Google secret are never in the file.

## Supabase versus Google Drive

| Supabase central (live now) | Google Drive (Managing Partner's account) |
|---|---|
| Logins, firm registry, user-to-firm mapping | Firm Documents, Employee Records, Client Documents |
| Firm settings, hours, days, office pin, geofence | Audit Working Papers, Invoices, Reports |
| Hierarchy, employee profiles and access levels | Archives and Backups (verified JSON snapshots) |
| Drive status and folder id, backup run history | Nothing else |
| Encrypted Drive refresh token (server-only table) | |

Drive is never used as a database. The app can only see files and folders it created itself (`drive.file`), so it cannot browse the rest of the Managing Partner's Drive.

## Security notes

- Settings, Drive and employee management are enforced in the database and Edge Functions, not just by hiding menu items.
- Refresh tokens are AES-GCM encrypted with a key held only as an Edge Function secret.
- OAuth state is single use and expires in 10 minutes. Disconnect revokes the token at Google and reports honestly if Google did not confirm.
- "Connected" is set only after the folder structure is read back from Drive. "Verified" for a backup is set only after the uploaded file is downloaded and its SHA-256 and size match.
- A disabled employee is also banned at the auth level.

## Not done, and why

1. **Operational data in Google Drive is decided but not built yet.** You chose to keep each firm's tasks, timesheets and invoices in the Drive of the person who registered it, with no per-firm Supabase project. Those pages still show sample data. Planned design: one server gateway function that reads and writes records as JSON files in the firm's Drive folder using the Managing Partner's stored authorisation, checking each employee's role first. Known costs of this choice: no database queries (dashboards load and total records in code), slower reads and writes than a database, Google's rate limits, a risk of lost edits if two people change the same record at once (reduced by one file per record), and the whole firm stops working if the Managing Partner's Google access is revoked or expires.
2. **Attendance, chat and notifications do not exist yet.** They were not in the uploaded file. Working hours, days and the geofence are stored but nothing enforces them yet.
3. **Backups cover central records only** (firm, people, hierarchy), not task or timesheet data, because that data is not stored anywhere real yet.
4. **Scheduled backups, archiving, restore and database usage monitoring are not implemented.** The schedule setting is saved but nothing runs on it. Restore is manual from the JSON in the Backups folder.
5. **Accent colours:** all 12 logos from your collage are built in, plus the blue-and-green default. Each accent changes the interface colour, sidebar logo, sign-in logo and browser tab icon.
6. **Work pages (tasks, timesheets, invoices, clients, documents, etc.) are empty for a signed-in firm.** Demo data now appears only in "Try with sample data" mode on the sign-in screen. Nothing is pre-filled in your Drive: registration only creates eight empty folders.
7. **Supabase free projects pause after about a week of inactivity.** A paused central project blocks every sign-in. Keep it active or upgrade before relying on it.

## What was tested

- Server logic, run in a rolled-back database transaction: firm registration, role assignment, missing-root, loop and invalid-designation rejection, hours validation, settings update.
- The page in a headless browser with Supabase and the map faked: sign-in screen, full wizard, hierarchy add and delete with re-parenting, Settings, Users, My Day, no script errors.
- **Not tested:** real sign-up and sign-in against Supabase, the four Edge Functions at runtime, the Google OAuth round trip, and real backup upload. Those need your credentials and a live run.

## Password reset (important one-time Supabase setup)

Supabase's default reset email uses a one-time link that mail scanners and double clicks can use up, which shows as "Email link is invalid or has expired". Make it reliable:

1. **Authentication → URL Configuration**: while testing on GitHub Pages, set Site URL to your Pages address, e.g. `https://YOUR-USER.github.io/YOUR-REPO/`. Under Redirect URLs add `https://YOUR-USER.github.io/**` (and `http://localhost:3000/**` only if you also test locally). When you move to Netlify, add `https://YOUR-SITE.netlify.app/**` and update Site URL and the `APP_URL` secret.
2. **Authentication → Email Templates → Reset Password**: replace the body with:

```html
<h2>Reset your WorkTrack password</h2>
<p>Click the link below to choose a new password.</p>
<p><a href="{{ .RedirectTo }}?token_hash={{ .TokenHash }}&type=recovery">Reset password</a></p>
<p>If you did not ask for this, you can ignore this email.</p>
```
3. Request only one reset email at a time and open the newest one. The app now blocks a second request for 60 seconds.

After you save the new password, the app signs you out and signs you in again with it, so you know it works.

## My Day (live) — what it needs

- Edge Function `workday` (new) saves attendance, appointments and task times into the firm's Google Drive
  (folder `Employee Records / Work Records`, one file per person per month). Every save is read back from Drive before it is shown as done.
- Drive must be connected for My Day to work. It cannot be disconnected from the app any more, and a reconnect must use the same Google account.
- Employees mark attendance only within the office geofence (Settings → location). The Managing Partner can mark from anywhere.
- Hierarchy: add each person's email (and, during registration, a temporary password). Passwords are never shown or stored in the page.
  For people added later in Settings, a random password is set and they use "Forgot password" on the sign-in page.
- Database additions: `hierarchy_members.email`, `firms.work_folder_id`, table `work_files`.
