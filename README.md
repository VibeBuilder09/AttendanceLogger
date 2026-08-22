# AttendanceLogger
# Attendance API Logger — Portable Build

A standalone Windows program that:
- Receives POST requests from attendance/biometric devices or software
- Logs every request (success/failed) to a dashboard at `http://localhost:<port>/`
- Writes successful punches into any SQL Server database, deduplicated on
  `EMP_CODE + PUNCH_DATETIME + TERMINAL_SN`

No Node.js installation is required on the target machine — everything is
bundled into one `.exe`.
## First run on the target machine

1. Double-click `AttendanceLogger.exe` once. It will exit immediately and
   create `config.json` next to itself (Delete the config.json from zip file before run the .exe).
2. Edit `config.json`:
   - `mssql` — point at that machine's SQL Server (works for a local
     instance, a different server on the network, or Azure SQL — anything
     reachable over `server:port`).
   - `server.port` / `server.receivePath` — where the app listens. Change
     these if port 3000 is already in use, or to match what the
     attendance software expects.
   - `server.apiKey` — optional. Set this to require a shared secret
     (`x-api-key` header) on incoming requests; leave blank to accept any
     request, same as the original behavior.
3. Test it by double-clicking `AttendanceLogger.exe` again — confirm the
   dashboard loads at `http://localhost:<port>/` and a test payload
   reaches SQL Server, then close that window.
4. **Now install it as an auto-start service** — see below. This is the
   recommended way to actually run it day-to-day, instead of leaving a
   console window open.
5. In your attendance device/software, set the "API URL" field to:
   ```
   http://<this-machine's-IP>:<port><receivePath>
   ```
6. Open `http://localhost:<port>/` on that machine to see the dashboard.

## Installing as an auto-start Windows Service (recommended)
This makes the program start automatically on boot, run invisibly (no
console window), and restart itself if it ever crashes. It uses
[NSSM](https://nssm.cc/) — a small, well-known tool for running any
`.exe` as a proper Windows Service, so the "no Node.js needed" promise
still holds.

That's it. The service is named `AttendanceAPILogger`. It will now:
- Start automatically every time the machine boots, before any user logs in
- Run with no visible window
- Restart itself automatically if it crashes
- Write its console output to `service-logs\service-out.log` and
  `service-logs\service-err.log` (auto-rotated at 10 MB)
  **Checking on it:**
- `sc query AttendanceAPILogger` from a command prompt, or
- Open `services.msc` and look for "Attendance API Logger"

**Changing `config.json` after the service is installed:** stop the
service first (`sc stop AttendanceAPILogger` or via `services.msc`), edit
the file, then start it again (`sc start AttendanceAPILogger`).

**Removing the service:** right-click `uninstall-service.bat` → **Run as
administrator**. This only removes the service registration — it does not
delete `AttendanceLogger.exe`, `config.json`, or `logs.json`.

## Database table — name, columns, and auto-create

The `table` section of `config.json` controls where punches get saved:

```json
"table": {
  "schema": "dbo",
  "name": "PunchLogs",
  "autoCreate": true,
  "columns": {
    "EMP_CODE": "EMP_CODE",
    "PUNCH_DATETIME": "PUNCH_DATETIME",
    "TERMINAL_SN": "TERMINAL_SN"
    /* ...and so on for every field */
  }
}
```

- **`schema` / `name`** — use any table name you want, in any database.
- **`columns`** — the left side is fixed (it's what the device sends);
  the right side is the actual column name you want in your table.
  Rename any of them freely — e.g. change `"EMP_CODE": "EMP_CODE"` to
  `"EMP_CODE": "employee_id"` and the app will create/use a column
  called `employee_id` instead. Delete a line entirely to skip saving
  that field.
- **`autoCreate`** — when `true` (the default), the app checks for the
  table the first time it connects, and if it doesn't exist, creates it
  automatically with the column names you configured. If it already
  exists, the app just uses it as-is and never touches its structure.
  Set to `false` if you'd rather create the table yourself and have the
  app fail loudly instead of auto-creating anything.

This means the same `.exe` can point at a brand-new, empty database —
first run creates the table itself, no manual `CREATE TABLE` step needed.
**Note:** deduplication (skipping a punch that's already been saved) only
works when `EMP_CODE`, `PUNCH_DATETIME`, and `TERMINAL_SN` are all mapped
to a column. If any of those three are removed from `columns`, the app
falls back to plain inserts with no dedup check.


