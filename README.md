# ChargeWay - EV Charging Station Finder & Navigation System

React (Vite) frontend + Node.js backend + SQLite database.
Drivers find nearby stations on a live map, see green / red availability, get a smart recommendation and a route.
Availability comes from drivers scanning a QR code at the charger and tapping **Start / Finish charging** (user-reported, no hardware).

## 1. Folder structure

```
chargeway/
├── package.json                  shortcuts: npm run server / client / build
├── server/                       BACKEND  (Node 22.5+, no npm packages to install)
│   ├── server.js                 HTTP server, router, serves the built React app
│   ├── db.js                     SQLite schema + demo data (node:sqlite)
│   ├── lib/
│   │   ├── auth.js               password hashing (scrypt), signed tokens
│   │   ├── charging.js           distance, charging time, availability, waiting time, recommendation
│   │   └── util.js               tiny router + input validation
│   └── routes/
│       ├── auth.js               register, login, admin login
│       ├── user.js               profile, vehicle, dashboard, history
│       ├── stations.js           nearby stations, station detail, start / finish charging
│       └── admin.js              stats, stations, slots, prices, users, activity
│
└── client/                       FRONTEND  (React 19 + Vite)
    ├── index.html
    ├── vite.config.js            dev server on :5173, proxies /api to the backend on :3000
    ├── package.json
    └── src/
        ├── main.jsx              entry: Router + Toast + Auth providers
        ├── App.jsx               every route in one place
        ├── api/
        │   ├── client.js         fetch wrapper + token storage
        │   └── endpoints.js      one function per backend endpoint (authApi, userApi, stationApi, adminApi)
        ├── context/
        │   ├── AuthContext.jsx   signed-in user, login(), logout()
        │   └── ToastContext.jsx  toast('Saved', 'good')
        ├── hooks/
        │   ├── useLoader.js      load + auto-refresh data from the API
        │   └── useInterval.js
        ├── utils/                format.js, charging.js, routing.js (OSRM), mapIcons.js, constants.js
        ├── styles/index.css      all styles (design tokens at the top)
        ├── pages/
        │   ├── Landing.jsx           5-second landing page, then redirects to /login
        │   ├── AuthPage.jsx          Sign in / Create account
        │   ├── VehiclePage.jsx       EV battery details
        │   ├── DashboardPage.jsx     battery, charging progress, profile, history
        │   ├── FindStationsPage.jsx  map + recommendations + route
        │   ├── ChargePage.jsx        opened by the QR code: pick slot, start / finish
        │   ├── NotFoundPage.jsx
        │   └── admin/                AdminLoginPage, AdminOverview, AdminStations, AdminUsers, AdminActivity
        └── components/
            ├── common/       Brand, Icons, Pill, StatCard, ProgressBar, Field, ErrorBox, Loading, Modal
            ├── layout/       Navbar, UserLayout, AdminLayout, AuthLayout, ProtectedRoute
            ├── auth/         LoginForm, RegisterForm
            ├── vehicle/      VehicleForm
            ├── battery/      BatteryGauge
            ├── dashboard/    BatteryPanel, ChargingPanel, VehicleSummary, ProfileForm, PasswordForm, HistoryTable
            ├── map/          StationMap (Leaflet), MapToolbar, RecommendationBanner, StationCard, RoutePanel
            ├── stations/     AvailabilityPill
            ├── charge/       StationHeader, SlotPicker, StartChargingCard, ActiveSessionCard, OtherSessionCard, FinishedCard
            └── admin/        StatsOverview, LiveSessions, ActivityList, AdminStationCard, SlotChip,
                              StationFormModal, LocationPicker, QrModal, UserTable
```

How the pieces connect: **page** (loads data with `useLoader`) -> **components** (draw it) -> **api/endpoints.js** (talks to the backend) -> **server/routes** -> **SQLite**.

## 2. Run it in VS Code

You need **Node.js 22.5 or newer**. Open the `chargeway` folder in VS Code, then open **two terminals**.

Terminal 1 - backend (nothing to install):
```
cd server
npm start
```
Terminal 2 - frontend:
```
cd client
npm install
npm run dev
```
Open **http://localhost:5173**.

| Role   | Login |
|--------|-------|
| Driver | `demo@chargeway.com` / `Demo@1234`, or register a new account |
| Admin  | `/admin/login` with `admin@chargeway.com` / `Admin@123` (change before deploying) |

The first backend start creates `server/data/chargeway.db` with 8 demo stations around Pune.
To put them in your city (before the first run, or after deleting `server/data`):
`SEED_LAT=19.0760 SEED_LNG=72.8777 npm start` (Windows PowerShell: `$env:SEED_LAT="19.0760"; $env:SEED_LNG="72.8777"; npm start`).

### Using the React template you already downloaded
Easiest: use this `client` folder as your project. If you want to keep your own template instead:
1. Copy this `client/src` over your template's `src` (delete the template's `App.css`, `assets` etc.).
2. Replace your `index.html` and `vite.config.js` with the ones here (the proxy to `/api` is required).
3. In your template run: `npm install react-router-dom leaflet qrcode.react`

### One server for everything (production style)
```
cd client && npm run build      # creates client/dist
cd ../server && npm start       # open http://localhost:3000 (serves the React app and the API)
```

## 3. Routes

| URL | Page | Who |
|-----|------|-----|
| `/` | Landing (5 s, then `/login`) | everyone |
| `/login`, `/register` | Sign in / create account | everyone |
| `/vehicle` | Battery details | driver |
| `/dashboard` | Dashboard + profile | driver |
| `/stations` | Map, recommendation, route | driver |
| `/charge/:stationId` | QR page: start / finish charging | driver |
| `/admin/login` | Admin sign in | everyone |
| `/admin`, `/admin/stations`, `/admin/users`, `/admin/activity` | Admin area | admin |

`ProtectedRoute` sends signed-out visitors to the right login page. A driver who scans a QR code while signed out is
brought back to that charger after signing in.

## 4. How the QR availability works

1. Admin: **Stations > QR code** (the address inside the code is editable) and print it for the charger.
2. Driver scans it, picks a free slot and taps **Start charging**. The slot becomes *occupied*; when every slot is taken the station turns red.
3. Driver taps **Finish charging**. The slot is free again and the energy and cost are saved.

It is **user-reported availability, not independently verified**, so: a slot is auto-released 30 minutes after the expected
finish if the driver forgets, the admin can override any slot, and the charge page says availability is driver-reported.
For phones to open the QR code, run the app on an address they can reach (your computer's network address on the same Wi-Fi, or a deployed site), not `localhost`.

## 5. Smart recommendation (for your viva)

1. Keep stations that are open (active, with a working slot) and within the car's range (10% reserve).
2. If any have a free slot, only those compete; otherwise the best busy one wins and its waiting time is shown.
3. Score (lower is better) = `0.45 x distance + 0.25 x waiting time + 0.30 x price`, each scaled 0 to 1.
4. If the nearest station is full, the banner says so and names the better alternative.

Charging time = `capacity x (target% - current%) / min(car kW, charger kW) / 0.92`. Distance uses the Haversine formula.

## 6. Database

`users` (role user / admin, status active / blocked) - `vehicles` (one per user) - `stations` - `slots` (available / occupied / offline) -
`sessions` (one row per charging session) - `activity` (audit log). Deleting a station removes its slots and sessions (`ON DELETE CASCADE`).

## 7. Notes

- The map tiles (OpenStreetMap), routes (public OSRM server) and fonts load from the internet, so be online. If routing is unreachable the app draws a straight line with estimated distance and time.
- Browsers give location access only on `https` or `localhost`; otherwise the map starts in the demo area and you can drag the blue dot.
- Security: scrypt-hashed passwords, signed tokens re-checked against the database on every request (blocked users lose access at once), role checks on every admin route, server-side validation, rate-limited login.
