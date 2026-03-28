# Work Horses -- "work horses"

Sprint Worker (`/loop 30m`) + PR Watchdog (`/loop 10m`).

The execution engine: Worker opens PRs, Watchdog fixes review comments and auto-merges after both gates pass.
