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

## Clients (live)

- Edge Function `clients` keeps the whole client list in the firm's Drive (`Client Documents / wt-clients.json`).
  Add, edit and delete are each read back from Drive before it is shown as saved; simultaneous adds are queued with a short database lock (`drive_locks`).
- Nothing about clients is stored in Supabase. A deleted client is moved to a "deleted" list inside the same Drive file, so it can be recovered. Partners and managers can onboard; staff see only clients where they are in-charge.
- The country dropdown uses flag images from flagcdn.com, so the page needs internet access (it already does for Supabase).

## Masters: business sector / description and compliance list

- Defaults ship inside the page (494 business descriptions in 26 sectors from `business_description.docx`, and the 39 compliances in 7 categories).
  A firm's edited copy is kept in Drive, inside `wt-clients.json` under `masters`. Edit it from Clients → "Business master" / "Compliance list" (Managing Partner and partners).
- Clients store IDs only (`bd…` for descriptions, e.g. `gst-monthly` for compliances), so renaming or moving an entry never changes a client.
  Entries can be deactivated but never removed; the server refuses a save that drops an existing entry.
- Older client records that stored compliance names are converted to IDs when loaded, and saved in the new form the next time that client is edited.

## My Tasks module (live)

- New Edge Function `tasks` (deployed; nothing else was changed, no existing record touched). All task data lives in the Managing Partner's Drive, in `Employee Records / Work Records`:
  - `wt-tasks-<FY start year>.json`: one file per financial year of creation (tasks, per-person progress, hours log, history).
  - `wt-eod-<YYYY-MM>.json`: End-of-Day summaries, check-outs and early check-out authorisations for the whole firm, one file per month.
  - `wt-taskcfg.json`: End-of-Day mandatory/optional, delegations, task permissions, firm holidays, approved leave.
  - Every write is read back from Drive before it is reported as saved; writes are serialised with the same database lock the Clients function uses.
- Nothing in Supabase changes. Task times use the firm clock (India time, Asia/Kolkata), which decides "today", the closing time and the check-out window.
- Visibility: own tasks plus everyone below you in the hierarchy; the Managing Partner sees all. This includes people who have never logged in. Seeing is not changing: Task Settings (Managing Partner) grants, per person, assign / update team progress / edit-reassign-cancel / delete. People can always update their own tasks, and edit tasks they created themselves within their team.
- Financial year: the title shows the current FY with a dropdown for earlier years. A task completed or cancelled in an earlier FY is locked. A task still open, or finished after the current FY began, stays editable from the current FY. Previous-year views are read-only.
- My Day: task deadlines show as blue appointments (exact due time, otherwise the firm's start time). They are worked out from the task itself, so editing, moving, cancelling or deleting a task updates My Day automatically and duplicates cannot exist. Managers also see their team's (with names, can be switched off).
- Attendance records are only read, never changed. After attendance is marked, the attendance card shows Check Out. It is unavailable before the closing time (or an authorised earlier time), and while a mandatory summary is pending, even after a refresh or sign-out. The server enforces this on every call. The Managing Partner is not held to the time or summary rule.
- Workload: load = estimated remaining hours of open tasks due in the chosen period (plus overdue), capacity = firm hours on working days less holidays and approved leave. Overloaded above 100%, Moderate 60 to 100%, Available below 60%. Open tasks not updated for 3+ days (adjustable) show as "Not updated", never as available. Reallocation is suggestion-only.

First real test (not possible from here, needs your Google Drive): sign in as Managing Partner, My Tasks, assign a task, update it, open My Day, then check that `wt-tasks-<year>.json` appears in Work Records.

## Team Chat (live)
- Menu: Team Chat. All Employees (everyone with a login), personal chats and groups. Messages, reactions, polls and read receipts are kept only in the Managing Partner's Drive (Employee Records / Work Records, files `wt-chat-*`, `wt-chatcfg.json`, `wt-chat-rc.json`). They are encrypted with the existing `TOKEN_ENC_KEY` secret, so they cannot be read by opening Drive.
- Function: `chat` (verify_jwt on). The server decides who sees what: selective messages in All Employees are invisible to everyone not chosen (including the Managing Partner), poll results are for the creator only, and no files can be sent (links are fine).
- Online status uses `profiles.last_seen_at`, refreshed every minute while the app is open (`touch_presence`). Availability icons: green tick online, gold clock away from the app but present today, red cross on leave (appears once the Leave menu exists), question mark otherwise.
- Message deletion timer (1 month / 3 months / 6 months / 1 year): set in Chat settings by people given that right. A database scheduler (`chat-retention-sweep`, hourly, pg_cron + pg_net) calls `chat` with the private key in table `chat_cron`, so old messages go even when nobody is signed in. Drive's old file versions are removed too.
- Joiners and leavers appear as notices in All Employees.
- Everyone picks their own avatar colour (13 starburst colours) under Chat settings.

## Users & Roles: access rights (live)
- Function: `rbac`. Rights are set by designation and by employee (several people at once, any place in the hierarchy). Order: Managing Partner, then the employee's own setting, then the designation's, then the built-in default.
- Three-state toggle: green tick allow, blue question mark default, red cross deny.
- Firm details and Employees rights are stored in Supabase table `rbac_grants` and checked by the database (`has_right`). All other rights (clients, leave, tasks, team chat) and the full change log are kept in Drive (`wt-rbac.json`).
- The Managing Partner can name people who may grant and revoke; they can pass on only rights they hold and never change their own or the creator's.
- Not yet enforced: Clients, Tasks and Leave rights are saved but those menus still use the old rules (next step). Payslips and salary rows stay locked until those menus exist. "Viewing firm hierarchy" hides the screen; the hierarchy data itself stays readable inside the app because tasks, clients and chat depend on it.
- Deleting an employee's credentials (function `create-employee`, action `delete`): login blocked for good and email freed, the person leaves the hierarchy (a "Removed employee" placeholder stays while people still report to it), records stay. A notice appears in Settings > Firm hierarchy.

## Working hours day by day
- Settings > Working hours and days: each weekday has opens, forenoon ends, afternoon starts, closes. Saved in `firms.work_schedule`.
- Other modules (check-out time, task capacity) still read the single opening and closing time (earliest open, latest close). Moving them to the day-wise times is a next step.

## Registration number, firm logo, invitations and first-time sign-in (live)

### One-time Google setup (needed for sending invitation emails)
1. Google Cloud Console, the project used for WorkTrack: enable the **Gmail API** (APIs & Services > Library > Gmail API > Enable).
2. APIs & Services > OAuth consent screen > Data access (Scopes) > add `https://www.googleapis.com/auth/gmail.send`. Keep the app **In production** (testing mode makes refresh tokens expire after 7 days). Google may show "Google has not verified this app" to the person connecting; they can continue. WorkTrack only sends mail; it cannot read any.
3. In WorkTrack: Settings > Google Drive > connect again with the **same Google account**, and keep the "Send email on your behalf" box ticked. Invitation mails are then sent from that account, one separate message per person.
Until step 3 is done, Employee invitations shows "Email sending is not connected". New registrations ask for this permission in the Drive step.

### Registration number
Accepted: FRN = 6 digits + one of W, N, S, E, C (for example 123456W), or, for a sole proprietor, only the 6-digit ICAI membership number. Required at registration, checked again by the database. Settings can change it later.

### Firm logo
Wide (rectangular) PNG, JPEG or WebP, up to 20 MB, picked in the last registration step (after Drive is connected) or in Settings. The browser shrinks it (max 1200 x 400) and the `logo` function saves ONE file named "WorkTrack firm logo" in the firm's Drive folder. Nothing is stored in Supabase. The browser keeps a copy only to display it quickly. It replaces the old "Signed in as ..." box at the top of every menu.

### Inviting employees (no passwords)
- The hierarchy and the Users page take only the employee's email. "Send invitation mail" per person, or "Send mail to all new employees" (Settings > Employee invitations, also at the end of registration). Function `invite`.
- The mail starts "Dear <name>", says what WorkTrack is, gives the link (it always uses the address the app is open at, so it follows you to Netlify), the designation, the sign-in steps, a one-time access code, "if you are not the intended recipient please ignore", and thanks.
- First time sign-in (login page): email, Next (only if registered and invited), access code + new password (WorkTrack and firm logos shown), choose WorkTrack colour, then normal sign-in. Function `onboard`; codes are stored only as a keyed hash, last 14 days, allow 5 wrong tries.
- The colour is stored on the person's profile (`profiles.avatar_color`) and is used for the Team Chat avatar.
- "Try with sample data" was removed from the login page.
