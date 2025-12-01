# Daily Weather Alerts with n8n

This project is a weather alert system built on n8n:

- Users subscribe through a form by entering their email and a list of cities.
- A scheduled workflow runs every morning, loads active subscriptions from Supabase, calls OpenWeatherMap for each city, classifies the weather into alert types, logs the run to Supabase, and sends one summary email per city via Gmail.

Workflows:

1. Weather Alert Subscription – collects user details and stores them in Supabase.
2. Daily Weather – cron-style job that sends daily alerts and notes them.

---

## Features

### Core

- Daily scheduled alerts at 8:00 AM (Cron node).
- Uses OpenWeatherMap API for every city.
- Writes each run to a weather_logs table in Supabase, including the raw API JSON.
- Sends emails via Gmail with subject:  
  `Daily Weather for <CITY> – <YYYY-MM-DD>`
- Email body includes:
  - Temperature
  - Feels-like temperature
  - Condition / description
  - Humidity
  - Wind speed
  - Local sunset time (formatted string)
  - Alert type: `PRECIPITATION`, `HEAT`, `FROST`, or `NONE`

### Bonus

- Multiple cities per user: we record one subscription per `(email, city)`.
- Cities are configurable via the external form; no cities are hard-coded in the workflows.
- Additional metrics: feels-like temperature and a sunset time.
- Retry logic on key HTTP nodes (OpenWeatherMap and Supabase insert) at 3 times max every 5 seconds.

---

## 1. API Setup & Key Configuration

### OpenWeatherMap

1. Create an OpenWeatherMap account and generate an API key.
2. Open the Daily Weather workflow in n8n.
3. In the OpenWeatherMap Request node, set:
   - Query parameter `appid` to your API key.
   - Query parameter `units` to `imperial` (for °F).

### Supabase

1. Create a Supabase project.
2. Note your project URL (e.g. `https://<project-ref>.supabase.co`) and service role key.
3. In each Supabase HTTP Request node (both workflows), set:
   - URL to the appropriate REST endpoint:
     - `https://<project-ref>.supabase.co/rest/v1/weather_subscriptions`
     - `https://<project-ref>.supabase.co/rest/v1/weather_logs`
   - Headers:
     - `apikey`: service role key
     - `Authorization`: `Bearer <service_role_key>`
     - `Content-Type`: `application/json`
     - For inserts: `Prefer`: `return=representation`

---

## 2. Supabase Details

weather_subscriptions

| Column       | Type        | Notes                                      |
|--------------|------------|--------------------------------------------|
| id           | uuid       | Primary key, default `gen_random_uuid()`   |
| email        | text       | Subscriber email, not null                 |
| city         | text       | City name, not null                        |
| active       | boolean    | Default `true`                             |
| created_at   | timestamptz| Default `now()`                             |

weather_logs

| Column           | Type        | Notes                                      |
|------------------|------------|--------------------------------------------|
| id               | uuid       | Primary key, default `gen_random_uuid()`   |
| run_at           | timestamptz| Default `now()`                            |
| city             | text       | City name                                  |
| temperature      | float8     | Current temperature (°F)                   |
| temperature_unit | text       | e.g. `"F"`                                 |
| feels_like       | float8     | Feels-like temperature (°F)                |
| condition        | text       | Weather description                        |
| humidity         | int        | Humidity (%)                               |
| wind_speed       | float8     | Wind speed (mph)                           |
| sunset_time      | text       | Local sunset time (formatted)             |
| alert_type       | text       | `precipitation`, `heat`, `frost`, `none`  |
| raw_response     | jsonb      | Full OpenWeatherMap JSON payload          |

---

## 3. Email Configuration

Both workflows share the same Gmail OAuth2 credential.

### Gmail credential

In n8n, go to New Gmail OAuth2 and follow to connect your Gmail / Google Workspace account.

Gmail usage:

- `Send Confirmation Email` (subscription workflow) sends a simple “you’re subscribed” message for each `(email, city)`.
- `Send Daily Weather Email` (daily workflow) sends the daily summary:
  - `To` is set from the subscription item’s `email`.
  - `Subject` uses `Daily Weather for <CITY> – <DATE>`.
  - `Message` uses the `summary` field built earlier in the workflow.

---

## 4. How to Import and Run the Workflows

### Import

1. In n8n, click `Workflows → Import from File`.
2. Import `Weather Alert Subscription.json`.
3. Import `Daily Weather.json`.
4. In each workflow, update:
   - Supabase URLs and headers.
   - OpenWeatherMap API key.
   - Gmail credential reference.

### Configure the schedule

In the `Daily Weather` workflow:

- The `Schedule Trigger` node is set to run once a day at 8:00 AM. Adjust the hour/minute if needed.
- Turn the workflow to Active so the schedule actually fires.

### Test the subscription flow

1. Open the `Weather Alert Subscription` workflow.
2. Click the form trigger node (`Weather Subscription Form`) and open its Test/Production URL in a browser.
3. Submit the form with:
   - Email (your address)
   - Cities (e.g. `Atlanta, Chicago, New York`)
4. Check:
   - Supabase `weather_subscriptions` has one row per city.
   - Your inbox has one confirmation email per city.

### Test the daily weather flow

1. Open the `Daily Weather` workflow.
2. Click `Execute Workflow` to run it once manually.
3. Verify:
   - `Load Active Subscriptions` returns one item per `(email, city)`.
   - `OpenWeatherMap Request` succeeds for each item (retries if needed).
   - `Set Weather Fields` has the normalized metrics and formatted sunset time.
   - The routing nodes set the correct `alert_type` per city.
   - `Insert Weather Log (Supabase)` writes rows into `weather_logs`.
   - `Send Daily Weather Email` sends you one email per subscribed city.

Once manual tests look good, leaving both workflows Active will keep the system running automatically.