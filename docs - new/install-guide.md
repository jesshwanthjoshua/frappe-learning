# Install Frappe v15 and ERPNext v15 (Ubuntu 24.04 on WSL2)

Written for: you, learning to install and run Frappe and ERPNext step by step.

Each phase explains what it does. Run the commands one phase at a time and check the result before moving on.

---

## Phase 0: Check your system

```bash
lsb_release -ds          # Ubuntu version (expect 24.04)
python3 --version        # Frappe v15 supports Python 3.10 to 3.12
node --version           # Frappe v15 needs Node 18 or 20
mariadb --version        # Frappe v15 supports MariaDB 10.6 or 11.x
redis-server --version   # Any recent version works
```

If a version is outside the expected range, fix it before continuing. Node 24 is too new for Frappe v15.

---

## Phase 1: System packages

Install the packages Frappe needs to build and run:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y mariadb-server mariadb-client libmariadb-dev pkg-config \
  curl xvfb libfontconfig1 pipx
```

Skip packages you already have. This machine already has `git`, `redis-server`, and pyenv-managed Python with `venv`, `pip`, and `setuptools`, so they are not in the command above. Check with `git --version` and `redis-server --version`.

What each package is for:
- `mariadb-server`, `mariadb-client`: the database. Each Frappe site stores its data here. Ubuntu 24.04 installs MariaDB 10.11, which Frappe v15 supports. Check with `mariadb --version`.
- `libmariadb-dev`, `pkg-config`: libraries needed to build the database driver.
- `xvfb`, `libfontconfig1`: needed to generate PDFs.
- `pipx`: installs the `bench` command in its own environment.

---

## Phase 2: Configure MariaDB

### 2a. Set the Frappe character settings

Frappe needs the `utf8mb4` character set and the `utf8mb4_unicode_ci` collation.

```bash
sudo nano /etc/mysql/mariadb.conf.d/99-frappe.cnf
```

Paste this:

```ini
[mysqld]
character-set-client-handshake = FALSE
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

[mysql]
default-character-set = utf8mb4
```

Save with Ctrl+O, Enter, then Ctrl+X.

### 2b. Set a root password

`bench new-site` logs in to MariaDB as `root` with a password. The default Ubuntu setup uses the Unix socket login, which won't work with Frappe.

```bash
sudo mariadb
```

At the `MariaDB [(none)]>` prompt, run:

```sql
ALTER USER 'root'@'localhost' IDENTIFIED VIA mysql_native_password USING PASSWORD('YourRootPassword');
FLUSH PRIVILEGES;
EXIT;
```

Replace `YourRootPassword` with a password you'll remember. You need it for `bench new-site`.

### 2c. Restart and check

```bash
sudo systemctl restart mariadb
mariadb -uroot -p -e "SHOW VARIABLES LIKE 'collation_server';"
```

Enter the root password. The output must show `utf8mb4_unicode_ci`. If it shows `utf8mb4_general_ci`, the restart didn't apply the config file.

---

## Phase 3: Node.js 20 with nvm

Frappe v15 needs Node 20. Use nvm so you can switch versions without affecting the rest of the system.

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install 20
nvm use 20
npm install -g yarn
node --version      # Should print v20.x
yarn --version
```

Run `nvm use 20` in each new terminal before running bench commands. Or add `nvm use 20` to your `~/.bashrc`.

---

## Phase 4: Install the bench CLI

`bench` is the command that manages Frappe. Install it with pipx:

```bash
pipx ensurepath
source ~/.bashrc
pipx install frappe-bench
bench --version
```

If `bench --version` prints a version number, it works.

---

## Phase 5: Create the bench folder

`bench init` creates the folder, a Python virtual environment, and the Frappe framework. The folder must be empty or not exist.

```bash
cd /home/jsj
bench init frappe-bench --version version-15
cd frappe-bench
```

Expect this to take several minutes. It downloads Frappe and installs its Python and JavaScript dependencies.

What it creates:
- `env/`: the Python virtual environment.
- `apps/frappe/`: the Frappe framework.
- `sites/`: one folder per site. A site is one running copy of the installed apps, with its own database, uploaded files, settings, and web address (for example `dev.localhost`). `bench new-site` creates it.
- `config/`, `logs/`: configuration and logs.

---

## Phase 6: Download ERPNext

Run this from inside `frappe-bench`:

```bash
bench get-app erpnext --branch version-15
```

This downloads ERPNext into `apps/erpnext/`. Check with `ls apps`. You should see `erpnext` and `frappe`.

---

## Phase 7: Create a site

A site is one Frappe instance with its own database. Create one called `dev.localhost`:

```bash
bench new-site dev.localhost \
  --mariadb-root-password YourRootPassword \
  --admin-password YourAdminPassword
```

- `--mariadb-root-password`: the root password from Phase 2b.
- `--admin-password`: the password for the `Administrator` login. Pick a new one.

This creates a database and the `sites/dev.localhost/` folder.

---

## Phase 8: Install ERPNext on the site

```bash
bench --site dev.localhost install-app erpnext
```

This creates ERPNext's tables and default data in the site's database. It can take a few minutes.

---

## Phase 9: Start and log in

```bash
bench start
```

Leave this terminal open. It runs the web server, background workers, and the scheduler.

Open a browser and go to:

```
http://dev.localhost:8000
```

Log in as `Administrator` with the admin password from Phase 7.

---

## Commands you've now used

| Command | What it does |
|---|---|
| `bench init` | Creates the bench folder and installs Frappe |
| `bench get-app` | Downloads an app into `apps/` |
| `bench new-site` | Creates a site with its own database |
| `bench --site <name> install-app` | Installs an app on a site |
| `bench start` | Runs the development server |
| `bench --site <name> migrate` | Applies database changes after an app update |
| `bench --site <name> console` | Opens a Python shell connected to the site |
| `bench --site <name> backup` | Backs up the site's database |

---

## Troubleshooting

- **`bench init` fails with "folder exists":** the target folder isn't empty. Move its contents out first.
- **`collation_server` shows `general_ci`:** restart MariaDB with `sudo systemctl restart mariadb`.
- **`new-site` fails with an access error:** check the root password, and confirm Phase 2b was applied.
- **Node version errors:** run `nvm use 20` and confirm with `node --version`.
- **Port 8000 in use:** another process is using it. Stop that process or change `webserver_port` in `sites/common_site_config.json`.

If a command fails, copy the full error message and ask for help with that message.
