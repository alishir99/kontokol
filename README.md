# Kontokoll

One file, `index.html`. No build step, no dependencies, no server required.

## Deploy it so anyone can use it

The app is a static page. Every visitor's data lives in their own browser, so
one deployment serves any number of people and you never hold their data.

Pick one, all free:

- **Netlify Drop**: go to app.netlify.com/drop and drag the `spending-dashboard`
  folder onto the page. You get a public URL in about ten seconds. Add your own
  domain later if you want.
- **Cloudflare Pages**: create a project, upload the folder, done.
- **GitHub Pages**: push the folder to a repo, then Settings, Pages, deploy from
  branch. The URL is `https://<user>.github.io/<repo>/`.

That is the whole deployment. Nothing below is required for other people to use
the app.

## What the profile gives a user

Signing in with Google is not just a login. It is what makes the history follow
the person instead of the browser, and the history is what the pattern analysis
runs on.

- **Profile card**: who is signed in, how many months are tracked, how many
  transactions, the average month, and the average share put aside.
- **Recurring costs**: merchants that come back month after month, with what
  each one costs per year at the current rate. This is where a 123 kr habit
  turns out to be 8 856 kr a year.
- **Your own average**: this month per category against the mean of the finished
  months, so a category creeping upwards is visible before the money is gone.
- **Which day the money goes**: average spending per weekday across the whole
  history.
- **Budgets from your usual months**: one button that sets every category budget
  to the median of what the person actually spent in the finished months. Median,
  not average, so one disastrous month does not raise the target.

Everything above also works signed out, on whatever is stored in that browser.
Login makes it portable and durable.

## Optional: Google sign-in and sync across devices

Without this, the app forgets nothing but remembers only per browser. Import on
your laptop and your phone knows nothing about it. Turning this on gives each
person a Google login and keeps their own transactions, budgets and category
overrides in sync between their devices. It also means their financial data now
sits in your database, which is a real responsibility. Read the last section
before you switch it on for other people.

### 1. Create a Supabase project

supabase.com, new project, free tier. Note the project URL and the `anon` public
key from Project Settings, API.

### 2. Create the table

Run this in the Supabase SQL editor:

```sql
create table snapshots (
  user_id    uuid primary key references auth.users on delete cascade,
  data       jsonb not null,
  updated_at timestamptz not null default now()
);

alter table snapshots enable row level security;

create policy "own row read"   on snapshots for select using (auth.uid() = user_id);
create policy "own row write"  on snapshots for insert with check (auth.uid() = user_id);
create policy "own row update" on snapshots for update using (auth.uid() = user_id);
```

Row level security is what makes this safe. Every request carries the signed-in
user's token, and the database refuses to return anyone else's row. The `anon`
key in the HTML is meant to be public, it grants nothing on its own.

### 3. Turn on Google

In Supabase: Authentication, Providers, Google, enable.

It asks for a Google client ID and secret. Get those from Google Cloud Console:
create a project, APIs and Services, Credentials, Create OAuth client ID, type
Web application. Set the authorised redirect URI to the one Supabase shows you,
which looks like `https://<project-ref>.supabase.co/auth/v1/callback`.

Then in Supabase, Authentication, URL Configuration, add your deployed site URL
to the redirect allow list. Add `http://localhost:8765` too if you test locally.

### 4. Paste two values into the app

Near the top of the script block in `index.html`:

```js
var CLOUD = {url: "https://<project-ref>.supabase.co", anonKey: "<anon key>"};
```

Leave them empty and the app stays fully local, which is how it ships. Fill them
in and a "Sign in with Google" button appears in the header.

Redeploy. That is it.

### How the sync behaves

- Signing in pulls the stored copy and merges it with whatever is already in this
  browser. Transactions merge by id, so importing the same CSV twice on two
  devices is harmless.
- Any change pushes back about one and a half seconds later.
- Budgets and category overrides resolve toward the device you are currently on.
  Fine for one person with a laptop and a phone. If two people ever edit the same
  account at once, that rule needs replacing with per-field timestamps.
- Signing out leaves the local copy alone. Clear removes it.

## Before you invite other people

The moment you store their statements, you are handling other people's financial
data, and under GDPR you are the controller. At minimum: a privacy policy saying
what you store and for how long, a way for someone to delete their account and
data, and a real password on the Supabase project. Keep the local-only version
if you do not want any of that. It is a legitimate product on its own, and it is
why the app ships with sync switched off.
