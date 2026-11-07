# Frappe / ERPNext install notes (this machine)

Written for: you, as a reference when reinstalling Frappe v15 and ERPNext v15 on Ubuntu 24.04 under WSL2.

Raw conversation transcripts are in `transcripts/`. They are the full history of the session and can be searched with `grep`.

## Current state (after the reset)

- `/home/jsj/frappe-bench` is an empty folder. It is ready for `bench init`.
- MariaDB is the Ubuntu 24.04 package, version 10.11.14. It is running.
- Redis is running on port 6379.
- Node 20.20.2 is available through nvm. Use it for Frappe v15. Node 24 is too new for v15.
- Python 3.10.18 is the pyenv default and is supported by Frappe v15.
- Leftovers still to clean up: `frappe-bench` installed in pyenv 3.13.5 and 3.14.5, and in pipx.

## Install steps, in order

1. **Check prerequisites.** Confirm Python 3.10 to 3.12, Node 18 or 20, MariaDB 10.6 or 11.x, and Redis.
2. **Set the MariaDB charset.** `/etc/mysql/mariadb.conf.d/99-frappe.cnf` must set `utf8mb4` and `utf8mb4_unicode_ci`. Restart MariaDB after changing it: `sudo systemctl restart mariadb`.
3. **Set root to password login.** `bench new-site` needs the MariaDB root user to log in with a password, not the Unix socket.
4. **Install the bench CLI.** `pipx install frappe-bench`.
5. **Create the bench.** From `/home/jsj`, run `bench init frappe-bench --version version-15`. The target folder must not exist or must be empty. Use Node 20 on the PATH.
6. **Download ERPNext.** From inside the bench, run `bench get-app erpnext --branch version-15`.
7. **Create a site.** Run `bench new-site dev.localhost --mariadb-root-password <pw> --admin-password <pw>`.
8. **Install ERPNext on the site.** Run `bench --site dev.localhost install-app erpnext`.
9. **Start the bench.** Run `bench start`, then open http://dev.localhost:8000 and log in as Administrator.

## Problems we hit and how they were fixed

- **MariaDB 13.0.2 from a third-party mirror.** Frappe v15 documents support for 10.6 and 11.x only. We removed it and installed 10.11 from Ubuntu.
- **Collation not applied after the config change.** The running server kept `utf8mb4_general_ci`. A restart applies `99-frappe.cnf`, which overrides `50-server.cnf`.
- **Node 24 with Frappe v15.** Use nvm to switch to Node 20 before running `bench init`.
- **`bench get-app` was interrupted.** Delete the partial folder and start over. Do not resume it.

## Reference

- Frappe installation guide: https://docs.frappe.io/framework/user/en/installation
- Frappe bench CLI reference: https://github.com/frappe/bench
