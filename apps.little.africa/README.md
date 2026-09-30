# Afropiano Edition 1 Chuka: deploy on Render + Supabase

The site is plain HTML, CSS and JS (folder `public/`). One small Node server (`server.js`) shows the pages and runs the payment code (`api/`). Supabase is only the database. Nothing else to build.

## Deploy (about 10 minutes)

**1. Supabase (database)**
1. Create a project at supabase.com.
2. Click **Connect** at the top, choose **Session pooler**, copy the connection string, put your database password in it. (Do not use "Direct connection": Render cannot reach it.)
3. You do NOT need to run any SQL. The server creates the tables (`afr_*`) by itself and retries on every request if the database was down. The old `orders_v2`, `bank_credits`, `tickets` tables are no longer used and can be dropped.

**2. GitHub**
Put this whole folder in a new GitHub repository.

**3. Render**
1. Render > **New > Blueprint** > pick the repository. Render reads `render.yaml`.
2. It asks for `DATABASE_URL`: paste the Supabase string. `TICKET_SECRET`, `ADMIN_KEY` and `IPN_SECRET` are generated for you.
3. When it is live, open the service > **Environment**, copy `ADMIN_KEY`. That is your password for `https://YOUR-SITE.onrender.com/admin`.

(Without Blueprint: New > Web Service, Build `npm install`, Start `node server.js`, and add the four variables from `.env.example`.)

**4. Check it works**
- `https://YOUR-SITE.onrender.com/healthz?db=1` must show `{"ok":true,"db":true}`. If `db` is false it prints the reason (usually the wrong connection string).
- Open Render > Logs. You should see `Database ready.` or `Database tables created.`

## Before the event
- Free Render sleeps after 15 minutes without visitors and takes about a minute to wake. Switch to the **Starter** plan (about $7/month) before tickets go on sale, or the first buyer may wait.
- Test one real KES 10 payment with your own phone (change the price temporarily in `api/_lib/catalog.js`), then approve it in /admin and confirm the ticket shows.
- Fill `CONTACT_WHATSAPP` in `public/assets/js/config.js`.

## How payment works (new flow)
1. Customer picks a package, types one name per person, taps **Book your ticket**.
2. The **payment page opens instantly**. Nothing waits for the server: the order reference (`AFR-XXXXXX`) is made in the browser. It shows Paybill **247247**, Account **1500184456952**, the amount and the reference, each with a Copy button.
3. Customer pays by M-Pesa, pastes the M-Pesa code and taps **I've paid**.
4. The server saves the order and checks the money. If money with that code (and at least that amount) is already in, the **ticket is issued immediately**. If not, the page says **Awaiting verification** and checks again every few seconds until confirmed.
5. A ticket is never issued from the customer's code alone. It needs real money in the `afr_credits` table. Each M-Pesa code works once.

**How real payments get into `afr_credits`** (use one or more):
- **Automatic, SMS forwarder (easiest):** install an SMS-forwarding app on the phone that receives the Equity/M-Pesa payment SMS. Point it to `POST https://YOUR-SITE/api/ipn` with header `x-ipn-secret: <IPN_SECRET>` and JSON body `{"message":"<the SMS text>"}`. The code and amount are read from the text.
- **Automatic, bank feed:** if Equity gives you payment notifications, send them to the same URL: `{"txn_code":"...","amount":800,"account":"1500184456952"}`.
- **Manual:** `/admin` > "Add received payments": paste statement lines or SMS messages. Matching orders are approved instantly. You can also approve or reject any order by hand.

## Prices and offers
Edit `api/_lib/catalog.js` (real values) and keep `public/assets/js/config.js` the same (display only). The offer end can also be set with the `OFFER_ENDS` variable in Render.

## Gate
`/admin` > Gate scanner: paste what the QR scanner reads. It says VALID, CHECKED IN, ALREADY USED or INVALID.
