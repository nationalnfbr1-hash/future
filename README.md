# Smile Care — Dental Clinic Management System

PHP + Supabase (Postgres via REST/PostgREST) admin system covering:
Dashboard, Patients, Dental Chart (32 teeth), Appointments, Treatments/Services,
Billing & Payments, and Pharmacy/Inventory.

## 1. Create the Supabase database

1. Go to your Supabase project → **SQL Editor** → New query.
2. Paste the contents of `supabase/schema.sql` and run it.
   This creates all tables (patients, dental_chart, appointments, services,
   treatments, invoices, payments, inventory_items, inventory_movements,
   staff) and enables Row Level Security with an open policy for the
   `anon` role — safe here because Supabase is only ever called from this
   PHP backend, never directly from a browser.

## 2. Configure the app

Open `config/config.php` and set:

```php
define('SUPABASE_URL', 'https://YOUR-PROJECT-REF.supabase.co');
define('SUPABASE_KEY', 'sb_publishable_mQ7iI6D0dXTgtC92ZJ3xow_Qz2iOTRr');
```

Replace `SUPABASE_URL` with your actual Project URL (Project Settings → API).
The publishable key you gave me is already wired in.

> **Security note:** the publishable/anon key is meant for browsers, but here
> it's used only from PHP running on your server, so it's never exposed to
> visitors. For a stricter setup later, switch to the **service_role** key
> (Project Settings → API) in `SUPABASE_KEY`, and tighten the RLS policies
> in `schema.sql` to deny `anon` entirely — the service role bypasses RLS.

## 3. Create your first login

From the server, run once:

```bash
php setup/create_admin.php
```

It will ask for a name, username and password, and create a `staff` row
with a properly hashed password. Then log in at `login.php`.

## 4. Point your webserver at the project root

Any PHP 8+ with `curl` extension enabled works — Apache, Nginx+PHP-FPM, or
just for local testing:

```bash
php -S localhost:8000
```

Then open `http://localhost:8000/login.php`.

## Folder guide

```
config/config.php        Supabase URL/key + app settings
includes/                Supabase REST client, header/sidebar/footer, helpers
supabase/schema.sql       Full database schema + RLS policies
setup/create_admin.php    One-time CLI script to create a login
index.php                 Dashboard
patients/                 Patient management (profile, history, x-rays/docs)
dental-chart/             Interactive 32-tooth chart per patient
appointments/             Booking, reschedule/cancel, reminders
treatments/               Service price list + treatment log
billing/                  Invoices, payments, printable receipt
pharmacy/                 Inventory + stock in/out log (only Manager can add/edit items)
staff/                    Manage Staff — create Cashier/Dentist/Manager accounts, reset passwords (Manager only)
expenses/                 Record and review clinic expenses (Manager only)
uploads/                  Patient x-rays/documents land here (created on first upload)
```

## Roles

- **Manager** (roles `manager` or `admin`): full access — can add/edit service prices, add/edit inventory items, manage staff accounts, view the full sales Report, and record Expenses.
- **Cashier** (or any other role): can take payments (Billing), search/view stock and record stock in/out, and only sees their *own* collected payments under "My Sales". They cannot change prices, add inventory items, manage staff, or see Expenses.

To add a new Cashier (or any staff) account, log in as a Manager and go to **Manage Staff** in the sidebar — no more manual SQL needed for this.

## Notes on things you'll likely want to extend

- **WhatsApp/SMS reminders**: `appointments/index.php` has a `send_reminder()`
  stub. Wire in your provider (Twilio, WhatsApp Cloud API, etc.) using the
  `WHATSAPP_API_URL` / `WHATSAPP_API_TOKEN` constants in `config/config.php`.
- **File storage**: patient x-rays/documents currently save to a local
  `uploads/` folder on the PHP server, with only the metadata (name, URL)
  stored in Supabase. For a multi-server or cloud setup, swap this for
  Supabase Storage (`/storage/v1/object/...`) using the same REST pattern
  already used in `includes/supabase_client.php`.
- **Roles**: five roles — Superior, Manager, Doctor, Nurse, Cashier. See the permission table at the bottom of Manage Staff.

## Receipt Printing (58mm, 80mm, A4)

Receipts print on any 80mm thermal printer (Xprinter, Epson TM, generic POS-80) that is installed
as a normal printer driver. No extra software is needed.

- **Invoice / payment / refund receipt:** open an invoice (Billing) -> `🖨 Print 80mm`, or the `🖨 80mm`
  button next to a single payment in Payment history.
- **POS sale receipt:** after Complete Sale -> `🖨 Print 80mm`.
- **Settings -> 80mm Thermal Receipt:** clinic address, phone, footer message, and auto-print on/off.

One-time browser print setup (Chrome / Edge):
1. Choose the 80mm printer as the destination.
2. Paper size: 80mm (or the driver's "80mm x Receipt" size). Margins: None. Headers and footers: off.
3. For one-click printing, start Chrome with `--kiosk-printing` (prints straight to the default printer).

Files: `billing/thermal.php`, `pos/thermal.php`, `includes/thermal_layout.php`, `assets/css/thermal.css`.
Run `add_thermal_receipt_settings.sql` once in Supabase for the new settings columns.
Printer model (58mm / 80mm / A4) is chosen in Settings -> Receipt Printer, and can be switched per print from the dropdown on the receipt page.

## Settings hub

Settings is a card hub (Manager only). Clinic profile, general, appearance, clinical, appointments, braces, billing,
receipt printer, invoice templates, security and notifications are stored in `clinic_settings` / `app_settings`
(run `add_settings_hub.sql` once). Treatment Categories, User & Permissions and Backup open their own pages.
Notification channels other than "System alert" are saved but need a messaging gateway to actually send.

## Reminders (SMS / WhatsApp / Email)

1. Run `add_notifications.sql` in Supabase (patient email + message log).
2. Put your gateway credentials in `config/config.php` (Twilio for SMS + WhatsApp, `MAIL_FROM` / `SMTP_*` for email).
3. Settings -> Notification Settings: choose the channel, days before, and edit the messages; use "Send a test message".
4. "Send due reminders now" sends today's batch; schedule `cron/send_reminders.php` daily to automate it.
Without a gateway, the 💬 WhatsApp / ✉ SMS buttons on Appointments and Braces open WhatsApp / the phone SMS app prefilled.

## Daily Activities Report

`Daily Report` in the sidebar (all staff). Counts patients attended, appointments scheduled, follow-ups contacted,
missed appointments and treatments assisted from the system for any date; each staff member can correct the
figures, add notes and save their own report. Managers also see every staff report for the day. Run
`add_daily_reports.sql` once. Appointments have a new **Missed** action; the WhatsApp/SMS buttons log each contact.

## Roles

Superior (top; only role that can create/edit/delete Managers and Superiors), Manager, Doctor, Nurse, Cashier.
Run `update_roles.sql` once: it maps old roles (dentist, receptionist, ...) to the five roles and promotes your main account to Superior.
The last active Superior can never be disabled, demoted or deleted.

## Staff registration & salary (payroll)

- **Manage Staff**: register (photo, department, employment status, date joined, optional basic salary), search/filter,
  `Edit`, profile page (`staff/view.php`), role change, disable, reset password, delete. Everyone has *My Profile*.
- **Payroll** (Manager / Superior): set each person's salary (basic, allowances, standard deduction, paid-by), create
  the month's payslips for everyone, adjust a draft (bonus, overtime, deductions), mark it paid. Paying records a
  "Salaries" expense and repays any salary advance. Staff see their own paid payslips under *My Payslips*.
- **Salary advances**: record an advance and how much to deduct per month; it is taken off payslips automatically.
- Managers can only handle Doctor / Nurse / Cashier pay; only a Superior sees Manager and Superior pay.
- Run `add_payroll.sql` once.

## Staff Registration page

`Staff Registration` (Manager / Superior): pick a role card to see what it can and cannot do; Doctor adds Specialty +
License no. + expiry (required: specialty, license no.), Nurse adds License no. + expiry. Includes confirm-password,
username/email validation, optional salary, and sequential Staff IDs (EMP-00001). Run `add_staff_role_fields.sql` once.

## Sidebar menu

Dashboard · **Clinical** (Patients, Appointments, Dental Chart, Treatments, Braces Follow-up, + Prescriptions, Lab Orders, Vital Signs)
· **Finance** (POS, Cashier Shift, Billing & Payments, Sales Report / My Sales, Expenses)
· **Management** (Staff Category, Staff Registration, Staff Salary, Staff Reports, Inventory, Settings)
· **My Work** (My Attendance, Daily Report, My Payslips, My Profile) · Logout.
Sections and items appear only for the roles allowed to use them.
