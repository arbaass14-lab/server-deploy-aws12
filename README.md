# WattWise

A responsive household electricity tracker with Google sign-in, private home profiles, appliance timers, usage history, deterministic bill estimates and optional AI source lookup.

The design uses navy, lime and ivory, with original appliance artwork included in `public/appliances.png`. India, Tamil Nadu and `Asia/Kolkata` are the onboarding defaults. Confirm your city, provider, tariff category and meter’s billing dates before recording usage.

## Start with the demo

Install **Node.js 22.12 or newer** and extract this ZIP. Open a terminal inside the `wattwise` folder:

```sh
npm ci
npm run setup
npm run dev
```

Open **http://localhost:3000** and choose **Explore the demo**. The demo works without a database, Google credentials or an AI key. It contains sample logs, a fictional ₹8/unit tariff, and a running timer. Demo changes persist only in this browser. Reset demo restores the samples.

This is a source project, not a hosted service. Production account data uses PostgreSQL. The demo is explicitly separate and never grants access to private APIs.

## Enable real accounts

### 1. Start PostgreSQL

Use your own PostgreSQL database and set `DATABASE_URL` in `.env`. Or, with Docker installed, start the included local development database:

```sh
docker compose up -d db
npm run db:migrate
npm run db:seed
```

The default `.env` connection matches `compose.yaml`. The seed adds 18 generic appliance templates only. It creates **no users and no real tariffs**. The catalog also works directly from its shared source module, so seeding is optional for display.

Docker's named volume preserves the database between restarts. `docker compose down` stops the local database; deleting its volume would delete the data. Use a separate database and backups for production.

### 2. Configure Google OAuth

In [Google Cloud Console](https://console.cloud.google.com/apis/credentials), configure your OAuth consent screen and create an OAuth client of type **Web application**. Add your Google account as a test user while the consent screen is in testing mode.

Set the local authorized JavaScript origin to:

```text
http://localhost:3000
```

Set the authorized redirect URI to exactly:

```text
http://localhost:3000/api/auth/callback/google
```

Copy the client ID and client secret into `.env`:

```dotenv
GOOGLE_CLIENT_ID="your-client-id.apps.googleusercontent.com"
GOOGLE_CLIENT_SECRET="your-client-secret"
NEXTAUTH_URL="http://localhost:3000"
```

`npm run setup` generates a random `NEXTAUTH_SECRET` if it is missing. It preserves existing settings and does not print the secret. Restart the app after changing environment variables.

**Continue with Google** becomes enabled when the required credentials, database URL and session secret are present. The first successful verified Google login creates a database user. Returning logins find that user by Google’s stable account ID. Users then create or choose a home profile.

For deployment, use an HTTPS domain in `NEXTAUTH_URL` and add its exact `/api/auth/callback/google` URI to Google. See the [NextAuth Google provider documentation](https://next-auth.js.org/providers/google).

### 3. Create a home and confirm billing details

Enter your home name, city, provider, tariff category, time zone, cycle length and anchor date. The anchor is a known **start date of a provider billing period**, not necessarily the payment due date. Both monthly and every-two-month cycles are supported. End-of-month anchors clamp correctly in shorter months.

TNPDCL is a suggested provider name, not a determination of your account’s applicable tariff. Confirm it against your bill. Once usage or tariffs exist, the profile’s region, provider, category, time zone and billing cycle are preserved to avoid rewriting historical calculations. Create another profile for a different meter or changed billing rules. Home names and city labels remain editable.

Add appliance power from the input rating label or a suitable average measurement. Catalog wattages are illustrative estimates. Use a duty cycle for a cycling appliance, or enter measured interval kWh. Do not enter an AC’s cooling capacity as electrical input power.

### 4. Add the applicable tariff

Open **Bill calculator → Add tariff version**. Enter rates from your provider or regulator, including effective date, source URL, review date and applicable charges. Confirm eligibility for subsidies yourself.

**No current Tamil Nadu tariff is bundled or claimed to be verified.** If lookup cannot establish reliable rules, manual entry remains available. The app never silently substitutes demo prices into a real account.

## Optional AI and location lookup

Set `OPENAI_API_KEY` on the server to enable **AI insights**, exact-model appliance research and official tariff searches. `OPENAI_MODEL` defaults to `gpt-4.1`; use an available model that supports the Responses API and web search. API usage is billed to the configured API account.

Appliance lookup prefers manufacturer specifications and [BEE appliance-label information](https://beeindia.gov.in/show_content.php?lang=1&level=2&lid=397&ls_id=249). Tamil Nadu tariff lookup restricts web search to the configured provider/regulator domains. Answers include citation links when returned. Review them before manually entering values; lookup never installs a tariff or changes appliance power automatically.

Usage explanations send only appliance names, aggregate runtime, consumption and estimated energy cost for the selected date range. No Google name, email or home address is sent in that request. The model is told not to invent numbers. All bill arithmetic and the savings slider use deterministic TypeScript code, independent of AI. Limit: 20 lookup requests per account per hour.

Set `GEOCODING_CONTACT` to your application contact email or website to enable **Use my location**. Browser permission is requested first. Rounded coordinates are sent to the app’s reverse-geocoding service and then OpenStreetMap Nominatim; they are not stored in the application database. Provider and tariff still need confirmation. Nominatim’s public service is appropriate for small deployments with its usage policy; at scale use your own compliant geocoder and shared rate limiter. Manual location entry always works.

Configuration is documented in `.env.example`. Keep secrets on the server and keep `.env` out of source control.

## What is included

| Area               | Behavior                                                                                                                                                    |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Login and profiles | Real Google OAuth, first-login user creation, server authorization, private Google accounts, multiple meter/home profiles                                   |
| Navigation         | Hamburger sidebar with account photo/name/email, active home, location, switching, settings and logout                                                      |
| Catalog            | 18 appliance types, category/name search, custom appliances, responsive cards and mobile search toggle                                                      |
| My appliances      | Brand/model search, quantity, room, watts, duty cycle, source/basis, label notes, editing, archive/restore                                                  |
| Usage              | Date + duration or start/end entry, Start/Stop, database timestamps, retry protection, overlap checks, editing/deletion and measured kWh                    |
| Dashboard          | Selected-range consumption/cost/runtime/top appliance, today/month totals, daily/monthly charts, previous-period comparison, recent activity and projection |
| Bills              | Household slabs, all-units tariffs, conditional bands, energy surcharge, fixed charges, energy credits/subsidies, taxes and transparent breakdown           |
| History            | Date filters, CSV export, immutable saved calendar or billing-period reports with tariff snapshots                                                          |
| Insights           | Largest consumer, practical guidance, deterministic savings scenario and optional AI explanation/web lookup                                                 |

The catalog includes fans, lights, ACs, refrigerators, televisions, washing machines, water heaters, induction cooktops, microwaves, kettles, mixers, irons, pumps, computers, laptops, routers, vacuum cleaners and chargers.

## Calculation behavior

```text
kWh = watts × operating hours × quantity × duty cycle ÷ 1,000
1 electricity unit = 1 kWh
```

One 100 W appliance running for 10 hours at full duty cycle uses an estimated 1 kWh. If your input wattage is already an average, use 100% duty cycle to avoid applying the average twice.

Intervals use UTC timestamps and are clipped to the home’s local date boundaries, including across midnight. For measured kWh, partial intervals allocate the measured total proportionally by duration because this app has no finer-grained meter samples. Completed logs store duration, calculated energy, quantity and power assumptions. Later appliance edits do not rewrite those assumptions.

Tariff slabs apply **once to total tracked household kWh in each provider billing cycle**. Appliance/entry costs are proportional shares of the household energy charge, including energy-related tax. Fixed charges appear separately. Calendar reports allocate fixed charges by their share of cycle time; the bill screen shows the full cycle’s fixed charge.

The tariff editor supports progressive and all-units rates, optional conditional consumption bands, a flat fixed charge per cycle, per-kWh surcharge, fixed energy credit and percentage tax on energy or subtotal. Its advanced JSON editor supports cases where total consumption changes the applicable slab schedule. See `docs/TARIFFS.md` for the schema.

A tariff change within a cycle makes the estimate unavailable until provider-specific transition logic is supplied. The app does not guess prorating, slab scaling, demand charges, time-of-use pricing, arrears or other unsupported rules. Historical tariff versions are retained. A newer version with the same effective date corrects live reports; already-saved reports retain their captured values.

Runtime is **user-recorded**. This app does not detect appliances running, control devices, or include a smart-plug integration. Consumption is estimated unless the user supplies measured kWh. The projected bill extrapolates current-cycle logged consumption over elapsed calendar time after at least three recorded days; it is labelled incomplete and excludes untracked loads. Missing logs can make projections too low.

## Build and tests

```sh
npm test
npm run typecheck
npm run build
npm run start
```

Unit tests require no database. The integration test is skipped by default. To run it, configure a **dedicated test database**, migrate it, and run with `WATTWISE_INTEGRATION=1`. For example on macOS/Linux:

```sh
# .env.test contains DATABASE_URL for your dedicated test database
node --env-file=.env.test node_modules/prisma/build/index.js migrate deploy
WATTWISE_INTEGRATION=1 node --env-file=.env.test --import tsx --test tests/database.test.ts
```

On PowerShell set `$env:WATTWISE_INTEGRATION = "1"`, then run the same Node command without the leading assignment. Fixtures use unique IDs and clean up their own records. See `docs/VALIDATION.md` for the checks completed for this deliverable and their limits.

For a server deployment, install dependencies with `npm ci`, run `npm run db:migrate` against the production database, build, and run `npm run start`. `Dockerfile` is an optional Node container recipe; pass environment variables at runtime and run migrations once as a deployment step. It does not provision OAuth or a database. Production Docker deployment has not been exercised in this environment.

## Project map

| Path                               | Purpose                                                           |
| ---------------------------------- | ----------------------------------------------------------------- |
| `app/`                             | Next.js pages, API routes, global responsive styles               |
| `components/`                      | Login, forms, charts and application screens                      |
| `lib/energy.ts`                    | Pure consumption, billing, date, allocation and savings functions |
| `lib/service.ts`                   | Authorized database operations and snapshot creation              |
| `lib/auth.ts`                      | Google OAuth and JWT sessions                                     |
| `lib/lookup.ts`, `lib/location.ts` | Optional server-side external services                            |
| `prisma/`                          | PostgreSQL schema, checked-in migration and catalog seed          |
| `tests/`                           | Unit and database integration tests                               |
| `public/`                          | Original appliance illustration and favicon                       |

Framework reference: [Next.js installation and runtime requirements](https://nextjs.org/docs/app/getting-started/installation). AI integration reference: [OpenAI web search](https://developers.openai.com/api/docs/guides/tools-web-search). Geocoding reference: [Nominatim reverse API](https://nominatim.org/release-docs/latest/api/Reverse/).

## Troubleshooting

- **Google button disabled:** check the required `.env` values and restart the server. The demo remains available.
- **OAuth redirect mismatch:** the redirect URI must match the scheme, host, port and callback path exactly. Check the consent screen’s test users.
- **Workspace cannot load:** verify PostgreSQL is reachable and run the migration. Google first-login user creation also needs the database.
- **Costs show —:** add a tariff effective on/before the cycle start. A mid-cycle tariff change also disables estimates intentionally.
- **Timer still running after reopening:** this is expected. Its start timestamp is stored. Stop it or correct the entry.
- **Usage overlaps:** edit the existing interval, choose another time, or use separate appliance records for units with different operating schedules.
- **Archive blocked:** stop the appliance’s active timer first. To edit an archived appliance’s historical usage, restore the appliance temporarily.
- **AI/location unavailable:** verify the optional configuration or use manual entry. Tracking and calculations do not depend on these services.
