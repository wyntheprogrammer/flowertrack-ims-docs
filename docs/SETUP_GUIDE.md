# FlowerTrack IMS — Windows Setup Guide

**Last updated:** 2026-11-17  
**Purpose:** Prepare a new Windows computer for FlowerTrack IMS development, including official installer links, installation steps, PHP configuration, and verification.

This guide is for the Laravel application. The documentation repository and application repository are separate:

- **Application source:** https://github.com/wyntheprogrammer/flowertrack-ims
- **Documentation:** https://github.com/wyntheprogrammer/flowertrack-ims-docs

> **Version rule:** Match the project's committed `composer.lock` and `package-lock.json`. The versions below reflect the verified FlowerTrack development environment as of this guide's update. Patch releases may be newer; use a compatible stable release and do not upgrade project dependencies casually.

## New-device setup: read these steps too

Run each command separately, press Enter, and wait for it to finish. Do not paste several commands as one continuous line.

- Before `npm ci` in PowerShell, follow [PowerShell npm setup](#powershell-npm-setup).
- After installing PHP/frontend dependencies, complete [Python setup for every device](#python-setup-for-every-device).
- To restore accounts and records, follow [Import database data](#import-database-data). Git pull and migrations do not transfer database records.
- The application now has database-backed modules; database and Python setup are needed to use those modules and Spoilage prediction.

## 1. Compatibility and installer checklist

| Tool | FlowerTrack baseline | Compatibility / selection |
|---|---|---|
| Windows | Windows 10/11 64-bit | Use x64 installers on standard Intel/AMD PCs; choose ARM64 builds only for Windows ARM devices. |
| XAMPP | PHP 8.3.36; MariaDB 10.11.16 | Install a Windows XAMPP build that bundles PHP **8.3 or newer**. The Apache Friends download page currently lists PHP 8.3.36. |
| PHP | 8.3.x | Laravel 13 requires PHP **8.3 or newer**. Use the PHP bundled with XAMPP for both CLI and Apache to avoid mismatched PHP versions. |
| Composer | 2.11.1; Composer 2.x | Use the current Composer 2 stable Windows installer. Composer must use the XAMPP PHP executable. |
| Node.js | 26.13.0 | Use Node.js **22.12 or newer** for the project's Vite 8 toolchain. The project was verified with Node 26.13.0. Prefer a supported LTS release that satisfies this minimum. |
| npm | 11.20.0 | Installed together with Node.js. Use the version included with the selected Node installer unless the project specifically requires otherwise. |
| Git for Windows | 2.52.0.windows.1 | Current maintained Git for Windows x64 installer is suitable. |
| Visual Studio Code | Current stable | User Installer (x64) is recommended for a single Windows account. |

### Official download pages

Download installers only from these official project/vendor pages:

1. **XAMPP for Windows:** https://www.apachefriends.org/download.html  
   Apache Friends download page: https://www.apachefriends.org/
2. **Composer Windows installer:** https://getcomposer.org/Composer-Setup.exe  
   Composer installation instructions: https://getcomposer.org/doc/00-intro.md#installation-windows
3. **Node.js (choose LTS):** https://nodejs.org/en/download  
   npm installation guidance: https://docs.npmjs.com/downloading-and-installing-node-js-and-npm/
4. **Git for Windows:** https://git-scm.com/install/windows
5. **Visual Studio Code for Windows:** https://code.visualstudio.com/Download  
   Windows installation instructions: https://code.visualstudio.com/docs/setup/windows

**Important:** Do not download separate PHP, MariaDB/MySQL, or npm installers when using XAMPP and the Node.js installer described here. XAMPP includes PHP and MariaDB; Node.js includes npm.

## 2. Install XAMPP (Apache, PHP, MariaDB)

1. Open the official [Apache Friends download page](https://www.apachefriends.org/download.html).
2. Under Windows, select a XAMPP installer containing PHP **8.3 or newer**. The official page currently lists a Windows build with PHP 8.3.36. Do not select an older PHP 8.2/8.1 build for this Laravel 13 project.
3. Run the downloaded `.exe`. Approve the Windows permission prompt if it appears.
4. In the component selection screen, make sure these are selected:
   - Apache
   - MySQL (XAMPP's bundled database server is MariaDB)
   - PHP
   - phpMyAdmin
5. Install to **`C:\xampp`**. Avoid `C:\Program Files` to reduce Windows permission/UAC issues with local development files.
6. Finish the installer and open **XAMPP Control Panel**.
7. Start Apache when you need the local web server. Start MySQL when database access is needed.
8. Open `http://localhost/` to verify the XAMPP dashboard. When MySQL is running, phpMyAdmin is normally available at `http://localhost/phpmyadmin`.

### Configure the PHP command-line PATH

The terminal must use the PHP bundled with XAMPP.

1. Press Start and search for **Edit the system environment variables**.
2. Open **Environment Variables**.
3. Under the user Path (or System Path if appropriate), choose **Edit** → **New**.
4. Add:

   ```text
   C:\xampp\php
   ```

5. Move this entry above other PHP entries if another PHP installation exists. Do not delete another entry unless you know it is unused.
6. Save all dialogs, close every open Command Prompt/PowerShell window, and open a new one.

Run:

```cmd
where php
php -v
php --ini
```

Expected: `where php` should list `C:\xampp\php\php.exe` first, and `php --ini` should show the loaded configuration file under `C:\xampp\php\php.ini`.

If another PHP path appears first, correct the Path order and reopen the terminal. Do not continue until CLI PHP is the intended XAMPP PHP version.

### Enable and verify required PHP extensions

1. Open `C:\xampp\php\php.ini` in a text editor (VS Code or Notepad).
2. Search for each extension directive. If it is commented out with a leading semicolon, remove only that semicolon. Typical directives are:

   ```ini
   extension=curl
   extension=fileinfo
   extension=mbstring
   extension=mysqli
   extension=openssl
   extension=pdo_mysql
   extension=zip
   ```

   Some PHP builds may use a slightly different directive format. Follow the directives already present in the installed `php.ini`; do not add duplicate entries.
3. Save the file. Restart Apache from XAMPP Control Panel if you changed extensions used by Apache. Close and reopen the terminal for CLI checks.
4. Verify the loaded configuration and modules:

   ```cmd
   php --ini
   php -m
   php -r "echo extension_loaded('pdo_mysql') ? 'pdo_mysql OK' : 'pdo_mysql MISSING';"
   php -r "echo extension_loaded('zip') ? 'zip OK' : 'zip MISSING';"
   ```

For Laravel 13, confirm the PHP modules required by the framework are available: **Ctype, cURL, DOM, Fileinfo, Filter, Hash, Mbstring, OpenSSL, PCRE, PDO, Session, Tokenizer, and XML**. For FlowerTrack's MariaDB connection, also enable **pdo_mysql**. The **zip** extension is recommended for Composer archive downloads and avoids the ZIP-extension warning seen during the original setup.

If an expected extension is missing, check the `extension_dir` setting and the matching DLL in `C:\xampp\php\ext`. Use DLLs supplied with that exact PHP build; do not copy extension DLLs from a different PHP version.

## 3. Install Composer

1. Download and run the official [Composer-Setup.exe](https://getcomposer.org/Composer-Setup.exe).
2. When the installer asks for the PHP executable, browse to:

   ```text
   C:\xampp\php\php.exe
   ```

3. If the installer offers to add Composer to PATH, keep that option enabled.
4. Complete the wizard. Close and reopen Command Prompt.
5. Verify:

   ```cmd
   composer -V
   where composer
   php -v
   ```

The Composer version should report Composer 2.x, and the PHP version shown by `php -v` must still be the XAMPP PHP version. If Composer reports a different PHP executable, rerun the installer and select `C:\xampp\php\php.exe`.

## 4. Install Node.js and npm

1. Open the official [Node.js download page](https://nodejs.org/en/download).
2. Download the Windows x64 installer for a supported **LTS** release. Ensure it is Node.js **22.12 or newer** to satisfy Vite 8. The original FlowerTrack environment used Node.js 26.13.0 and npm 11.20.0.
3. Run the `.msi` installer. Keep the options to install Node.js runtime, npm package manager, and add Node to PATH enabled. The optional native build tools are not required for the current project setup unless a dependency later requests them.
4. Finish the wizard, close and reopen Command Prompt.
5. Verify:

   ```cmd
   node -v
   npm -v
   where node
   where npm
   ```

The commands must resolve to the newly installed Node.js/npm locations. Avoid installing a separate npm package globally just to match the recorded patch number; the project's lockfile controls its local dependencies.

## 5. Install Git for Windows

1. Download the official [Git for Windows installer](https://git-scm.com/install/windows).
2. Choose the x64 standalone installer for a standard Intel/AMD Windows computer.
3. Run the installer. The defaults are generally suitable. At the command-line PATH screen, choose the option that allows Git from the command line and third-party software (commonly **Git from the command line and also from 3rd-party software**).
4. For the default editor, select Visual Studio Code if it is already installed; otherwise keep the default and change it later if desired.
5. For line endings, the usual Windows default **Checkout Windows-style, commit Unix-style line endings** is suitable.
6. Complete installation and reopen Command Prompt.
7. Verify:

   ```cmd
   git --version
   where git
   ```

Optional one-time identity setup (use your own GitHub identity):

```cmd
git config --global user.name "Your GitHub Name"
git config --global user.email "your-verified-github-email@example.com"
git config --global --list
```

Do not copy someone else's email or credentials. Git identity is for commit attribution; GitHub authentication is handled separately when you push.

## 6. Install Visual Studio Code

1. Open the official [Visual Studio Code download page](https://code.visualstudio.com/Download).
2. Download **Windows User Installer x64** for a standard Intel/AMD PC. User Setup is recommended for one account and normally does not require administrator permissions. Use System Installer only if VS Code must be installed for all users.
3. Run the installer and accept the license.
4. Recommended setup options:
   - Add to PATH
   - Add **Open with Code** to file context menu (optional)
   - Add **Open with Code** to directory context menu (optional)
   - Register Code as an editor for supported file types (optional)
5. Finish installation. Close and reopen Command Prompt.
6. Verify:

   ```cmd
   code --version
   ```

If `code` is not recognized, reopen the terminal first. If it remains unavailable, rerun the VS Code installer and ensure the PATH option is enabled.

### Recommended VS Code extensions

These are editor extensions, not required runtime packages. Install them from the VS Code Extensions view (`Ctrl+Shift+X`) as needed:

- **PHP Intelephense** — PHP language support and IntelliSense.
- **Laravel** (official Laravel extension) — Laravel/Blade development support.
- **Blade Formatter** — Blade template formatting.
- **EditorConfig for VS Code** — consistent editor settings when the repository contains an `.editorconfig`.

Do not install extensions from unknown publishers without reviewing their publisher and permissions.

## 7. Final environment verification

Close and reopen Command Prompt, then run the complete check:

```cmd
where php
php -v
php --ini
php -m
composer -V
where composer
node -v
npm -v
where node
where npm
git --version
where git
code --version
```

Before cloning the application, confirm:

- PHP is 8.3 or newer and points to `C:\xampp\php\php.exe`.
- Composer is 2.x and uses the XAMPP PHP executable.
- The Laravel-required PHP extensions listed above are enabled, plus `pdo_mysql` and `zip`.
- Node.js satisfies Vite 8's minimum (22.12 or newer).
- npm, Git, and VS Code commands work from a newly opened terminal.

## 8. Clone the application repository

The original machine used `C:\xampp\htdocs\FlowerTrack`. Use the same path to follow the commands exactly.

```cmd
cd /d C:\xampp\htdocs
git clone https://github.com/wyntheprogrammer/flowertrack-ims.git FlowerTrack
cd FlowerTrack
code .
```

If the `FlowerTrack` folder already exists, do not clone over it. Open that folder and check `git status` first.

### PowerShell npm setup

If using PowerShell (including VS Code's PowerShell terminal), allow local scripts for your Windows account before running `npm ci`:

```powershell
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
```

If prompted, enter `Y` and press Enter. This persists for the current Windows user and normally does not require Administrator access. RemoteSigned permits local scripts while requiring downloaded scripts to be signed. It can resolve the `npm.ps1 cannot be loaded because running scripts is disabled` error.

Verify the effective setting:

```powershell
Get-ExecutionPolicy
```

An organization Group Policy may override this setting. You can instead run `npm.cmd ci` and `npm.cmd run build`, or use Command Prompt; those commands do not require changing PowerShell policy. The execution-policy command is PowerShell-only, not a Command Prompt command.

Reference: [Microsoft Set-ExecutionPolicy](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.security/set-executionpolicy).

The message “packages are looking for funding” is informational; it is not an installation failure.

## 9. Install project dependencies

Run these commands from the application root, where `composer.json`, `composer.lock`, `package.json`, and `package-lock.json` are located:

```cmd
composer install
npm ci
```

- `composer install` installs PHP dependencies from the committed `composer.lock`.
- `npm ci` installs frontend dependencies from the committed `package-lock.json`.
- Do not copy `vendor` or `node_modules` from another computer.
- Do not run `composer update` or `npm update` as part of normal setup; those commands can change locked dependency versions.

If the repository genuinely has no `package-lock.json`, use `npm install` instead. The FlowerTrack repository currently uses a lockfile, so `npm ci` is the expected command.

## 10. Create the local environment file

The `.env` file contains machine-specific settings and secrets. It must not be committed.

```cmd
copy .env.example .env
php artisan key:generate
```

If a local `.env` already exists, do not overwrite it blindly. Keep the local configuration and review it instead.

The application includes implemented database-backed modules. Configure MariaDB before opening them. Python is also required for generating Spoilage predictions; follow the dedicated setup below.

## 11. Database setup

1. Open XAMPP Control Panel.
2. Start **MySQL** (the bundled server is MariaDB) when database access is needed.
3. Open phpMyAdmin, usually at `http://localhost/phpmyadmin`.
4. Create a local database for FlowerTrack, for example `flowertrack_ims`.
5. Update the database settings in the local `.env` file. Example for a default local XAMPP setup:

```dotenv
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=flowertrack_ims
DB_USERNAME=root
DB_PASSWORD=
```

Use the actual username, password, host, and port configured on that computer. Do not assume the example credentials are correct for every installation.

For a new empty database, run the committed migrations. If restoring a full SQL backup or presentation snapshot, import it first using the instructions below, then run only pending migrations:

```cmd
php artisan migrate
```

Do not run migrations on a shared or production database without confirming the target environment and migration impact.

### Import database data

Migrations create/update table structure. They do not copy users, stock, orders or daily history from another device.

#### Transfer your current records from another device

1. On the old device, open phpMyAdmin, select `flowertrack_ims`, then choose **Export → Quick → SQL → Export**.
2. Transfer that SQL file privately to the new device.
3. On the new device, start Apache and MySQL in XAMPP. Open `http://localhost/phpmyadmin`.
4. Select `flowertrack_ims` (create it if absent), choose **Import**, select the SQL file, and click **Import/Go**.
5. Prefer an empty target for a full backup. If the target already has records, export a backup first. A dump containing DROP statements replaces those tables; one without them may report existing-table conflicts. Do not blindly import a full dump over data you want to retain.
6. Confirm local `.env` points to `flowertrack_ims`, then run each command separately:

```cmd
php artisan config:clear
php artisan migrate
php artisan migrate:status
php artisan view:clear
php artisan storage:link
```

Keep the restored `migrations` table: it tells Laravel which schema changes already ran. Never use `migrate:fresh` to restore records; it drops tables.

Copy uploaded files separately: SQL contains image/proof paths, not the actual files. Merge the old device's `storage/app/public` and `storage/app/private` contents into the new project's matching folders. Keep the new device's DB settings local; do not commit `.env` or private SQL exports.

#### Use the supplied presentation records instead

Download [FlowerTrack_Presentation_Database.zip](https://chatgpt.com/library?file_id=libfile_9226141c730481919411b81df2222dc5&account_id=personal), extract it, and import `flowertrack_ims_presentation.sql` through phpMyAdmin using the steps above.

**This supplied SQL explicitly selects flowertrack_ims and drops/recreates its listed tables, replacing their records, accounts and sessions. Back up existing data before importing.** Reimporting restores the same sample records and replaces changes made since the previous import.

Merge the extracted `storage` folder into `C:\\xampp\\htdocs\\FlowerTrack`; it includes the private synthetic delivery proof. Run the post-import commands above and log in again.

Local presentation login: `admin@demo.example.test` / `DemoFlower!2026`. These are public test credentials for synthetic data, not production credentials. Historical records use October–November 2026 dates, and the supplied prediction snapshots cover 15–21 November 2026. Use matching report filters; current predictions can regenerate from available history.

### Python setup for every device

The project's `.venv` is excluded from Git. Create it separately on each computer; do not copy the old device's venv. Install Python from [python.org Windows downloads](https://www.python.org/downloads/windows/) if `py --version` is not recognized. The new device's Python **3.14.2** passed the forecast self-test on 2026-11-17; the earlier working device used 3.14.10. Install the committed requirements rather than manually selecting library versions.

Open the terminal in `C:\\xampp\\htdocs\\FlowerTrack`. **Run the following blocks one at a time**, waiting for each to finish.

1. Check Python:

```cmd
py --version
```

2. Create the environment:

```cmd
py -m venv .venv
```

3. Update pip inside it:

```cmd
.\\.venv\\Scripts\\python.exe -m pip install --upgrade pip
```

4. Install the project's pinned libraries:

```cmd
.\\.venv\\Scripts\\python.exe -m pip install -r requirements.txt
```

5. Test the forecasting engine:

```cmd
.\\.venv\\Scripts\\python.exe scripts\\spoilage\\predict.py --self-test
```

Expected:

```json
{"status": "ok", "message": "Forecast self-test passed."}
```

6. Test batch forecasting and the Laravel bridge:

```cmd
.\\.venv\\Scripts\\python.exe scripts\\spoilage\\predict_many.py --self-test
php artisan config:clear
php artisan tinker --execute="dump(app(\\App\\Services\\SpoilagePythonService::class)->health());"
```

Health should return status `ok` and installed Python/library versions. If a command fails, resolve it before continuing.

No activation or Activate.ps1 is needed: these commands directly use the environment's executable. The PowerShell npm policy change above is separate from this Python setup. Laravel defaults to `.venv/Scripts/python.exe` on Windows. If `SPOILAGE_PYTHON_BINARY` is set in `.env`, make sure it points to this device's executable, then clear config cache.

If you see “unrecognized arguments” after `py -m venv`, check whether multiple commands were accidentally joined together. If “The system cannot find the path specified” follows, successfully create `.venv` before running its Python executable.

After the engine/bridge checks pass, open Spoilage Prediction and wait for the automatic refresh to finish. A self-test verifies the engine; the browser check also verifies Laravel integration and available database history.

## 12. Build and run FlowerTrack IMS

Build frontend assets:

```cmd
npm run build
```

For active frontend development, open a second terminal in the project folder and run:

```cmd
npm run dev
```

Keep the Vite terminal running while developing. Stop it with `Ctrl+C` when finished.

In another terminal, start Laravel:

```cmd
php artisan serve
```

Open the URL shown in the terminal. The usual local URL is:

`http://127.0.0.1:8000`

When using Vite's development server, keep `npm run dev` running in the second terminal.

## 13. Daily workflow on either computer

Before editing:

```cmd
cd /d C:\xampp\htdocs\FlowerTrack
git status
git pull origin main
```

Only pull when the working tree is clean or after safely committing/stashing local work. If Git reports conflicts, stop and resolve them before continuing.

After completing an approved module, follow the project workflow:

1. Review and finalize the module.
2. Update the relevant Markdown documentation in the documentation repository.
3. Run the appropriate checks/build.
4. Review `git status` and stage only the intended application files.
5. Commit and push to `origin main`.
6. Confirm the push succeeded and the working tree is clean.
7. Proceed to the next module only after verification.

The documentation repository and application repository must each be cloned and updated separately if you plan to edit both from the new computer.

## 14. Files and settings that should not be copied or committed

- Do not commit `.env` or secrets.
- Do not copy `vendor` or `node_modules`; reinstall dependencies.
- Do not overwrite local database credentials with another computer's credentials.
- Keep personal editor settings and machine-specific paths local unless the team intentionally shares them.

Git should ignore `.env`, `vendor`, and `node_modules`. Confirm from the application root:

```cmd
git check-ignore .env vendor node_modules
```

## 15. Troubleshooting

### `php` is not recognized or the wrong PHP version runs

Check `where php`. Add `C:\xampp\php` to PATH and move it above other PHP installations. Close and reopen the terminal, then rerun `php -v` and `php --ini`.

### Composer reports a missing ZIP extension

Enable `extension=zip` in `C:\xampp\php\php.ini`, save, reopen the terminal, and verify with `php -m`. Confirm `php --ini` points to the same file you edited.

### Composer uses a different PHP executable

Rerun Composer-Setup.exe and select `C:\xampp\php\php.exe`. Then verify `where php`, `php -v`, and `composer -V`.

### Vite reports an unsupported Node.js version

Run `node -v`. Install Node.js 22.12 or newer, reopen the terminal, and verify `where node` to make sure an older Node installation is not first on PATH.

### Composer or npm dependencies fail

Confirm internet access and compatible PHP/Node versions. Run commands from the application root. Use `composer install` and `npm ci` rather than copying dependency folders from another computer. Read the first reported missing extension or engine requirement before changing versions.

### Vite assets are missing or stale

Run:

```cmd
npm ci
npm run build
```

For active development, use `npm run dev` and keep that terminal open.

### Database connection fails

Confirm that XAMPP's MariaDB service is running, then verify `DB_HOST`, `DB_PORT`, `DB_DATABASE`, `DB_USERNAME`, and `DB_PASSWORD` in the local `.env`. Check for another MySQL/MariaDB service already using port 3306.

### Apache or database port is already in use

Another local service may already occupy ports 80/443 or 3306. Identify the process before stopping anything. Do not terminate an unknown service; choose a free port or configure the local environment deliberately.

### XAMPP shows `MySQL shutdown unexpectedly` when multiple XAMPP instances are open

In one FlowerTrack development setup, this error was resolved because **two XAMPP instances/control panels were running at the same time**. Closing the duplicate instance allowed MySQL to start normally.

1. Close the XAMPP Control Panel windows you do not need.
2. Open **Task Manager** (`Ctrl+Shift+Esc`).
3. In **Processes** (or **Details**), look for duplicate XAMPP Control Panel processes, commonly named `xampp-control.exe`.
4. Confirm which entry belongs to the extra XAMPP Control Panel, then select **End task** for that duplicate only.
5. Keep one XAMPP Control Panel open. If needed, right-click it and choose **Run as administrator**, then try starting MySQL again.
6. If MySQL still shuts down unexpectedly, check the XAMPP MySQL log and verify whether another MySQL/MariaDB service is using port 3306. Do not end an unknown `mysqld.exe` process, delete database files, or replace anything inside `C:\\xampp\\mysql\\data` as a first fix.

**Important:** This fix applies when duplicate XAMPP Control Panel instances are actually present. If there is only one instance, investigate the MySQL error log and port usage instead; the same message can have other causes.

---

## Official technical references

- [Laravel 13 installation](https://laravel.com/docs/13.x/installation)
- [Laravel 13 server requirements](https://laravel.com/docs/13.x/deployment#server-requirements)
- [Vite 8 Node.js compatibility](https://vite.dev/guide/)
- [PHP for Windows downloads](https://windows.php.net/download/)
- [Composer Windows installation](https://getcomposer.org/doc/00-intro.md#installation-windows)
- [Node.js and npm installation](https://docs.npmjs.com/downloading-and-installing-node-js-and-npm/)
- [Git for Windows installation](https://git-scm.com/install/windows)
- [VS Code Windows installation](https://code.visualstudio.com/docs/setup/windows)

## Related project documentation

- [Project README](../README.md)
- [AI Context](AI_CONTEXT.md)
- [Technology Stack and References](TECHNOLOGY_STACK_AND_REFERENCES.md)
- [Development Progress](DEVELOPMENT_PROGRESS.md)
- [UI/UX Guidelines](UI_UX_GUIDELINES.md)


## Reports PDF and Excel export dependencies

From Windows CMD in the application directory, install the Reports export packages before building the frontend:

```bat
cd /d C:\xampp\htdocs\FlowerTrack
npm install jspdf jspdf-autotable exceljs
npm run build
```

| Package | Purpose |
|---|---|
| `jspdf` | Generate PDF reports in the browser |
| `jspdf-autotable` | Render paginated report tables inside the PDF |
| `exceljs` | Generate actual Excel `.xlsx` workbooks |

Commit both `package.json` and `package-lock.json` with the Reports implementation. On a fresh checkout where these dependencies are already committed, use `npm ci` followed by `npm run build`; there is no need to reinstall them individually.

If Vite reports a failed/unresolved import for one of these packages, run the install command above from the application root and rebuild. Do not mark the package external merely to suppress the error; the report code needs the dependency bundled for browser exports. Package versions should follow the application's committed lockfile; none are asserted by this guide.


## Spoilage Python setup and device transfer — 2026-11-15

The Spoilage engine checkpoint is application commit `53251a5`. Python runs as a local process invoked by Laravel; no separate Python web server is required.

### Verified environment

| Component | Owner-verified version |
|---|---|
| Python | 3.14.10, Windows x64 |
| NumPy | 2.5.4 |
| pandas | 2.3.3 |
| statsmodels | 0.15.0 |

These are the versions reported on the owner's working device, not a requirement to upgrade every computer to a newer release. Install compatible Python separately on each computer; venv creates an isolated environment using an already installed interpreter. A venv and its libraries are not transferred by Git.

### First, prepare the dependency manifest on the working device

The owner confirmed requirements.txt and Python ignore rules pushed in commit `f213583`. New devices install from that committed manifest. The freeze command below is for intentionally recording a later dependency change on the working device; it is not required again merely to pull the project. In PowerShell, from `C:\\xampp\\htdocs\\FlowerTrack`:

```powershell
.\.venv\Scripts\python.exe -m pip freeze | Set-Content -Encoding utf8 requirements.txt
```

This captures installed dependencies and their versions, including transitive dependencies. UTF-8 avoids Windows PowerShell's default UTF-16 redirection encoding.

Add these entries to the application's existing `.gitignore` without replacing its other entries:

```gitignore
/.venv/
__pycache__/
*.py[cod]
```

These Python ignore entries were saved in `f213583`. When intentionally changing the manifest or ignore rules later, review and save only those files:

```powershell
git add requirements.txt .gitignore
git diff --cached --stat
git commit -m "Record Spoilage Python dependencies and ignore local environment"
git push origin main
```

The manifest/ignore commit is already confirmed. The commands above are retained for later dependency changes; routine setup on another device uses the next section.

### Set up another device

1. Install XAMPP/PHP, Composer, Git, Node/npm and compatible Python as described in this guide. Start MySQL before database-dependent Laravel commands.
2. Clone the application if it is absent, or pull the existing clean checkout. Preserve any wanted local changes before pulling.
3. Install dependencies and configure the local environment:

```powershell
cd C:\xampp\htdocs
git clone https://github.com/wyntheprogrammer/flowertrack-ims.git FlowerTrack
cd FlowerTrack
composer install
npm.cmd ci
Copy-Item .env.example .env
php artisan key:generate
```

Clone/copy/key generation above are for a **new local checkout only**. On an existing checkout use `git pull origin main`, `composer install`, and `npm.cmd ci`; preserve its existing .env and APP_KEY. Configure DB_DATABASE=flowertrack_ims and the device's actual DB host/port/user/password. Create the database in phpMyAdmin if absent. Project .env files, uploaded images/private delivery proofs and database contents are not restored by git pull.

4. Create the Python environment and install the committed requirements:

```powershell
py -3.14 --version
py -3.14 -m venv .venv
.\.venv\Scripts\python.exe -m pip install --upgrade pip
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
```

Use `py --list` if the requested interpreter cannot be found. Do not run the requirements install before the manifest is committed and pulled. If a package version cannot install on a different Python/platform, match the working Python environment rather than silently changing the pins. No Activate.ps1 or execution-policy change is needed because commands call the venv executable directly.

5. Verify the Python engine and Laravel bridge:

```powershell
.\.venv\Scripts\python.exe scripts\spoilage\predict.py --self-test
php artisan config:clear
php artisan tinker --execute="dump(app(\App\Services\SpoilagePythonService::class)->health());"
```

Expected: self-test returns status ok; health returns Python/NumPy/pandas/statsmodels versions. `config/spoilage.php` defaults to the project's .venv/Scripts/python.exe on Windows and .venv/bin/python on other systems. Set SPOILAGE_PYTHON_BINARY only if using a different absolute executable path, then clear config cache. Do not copy a different device's absolute path.

6. Prepare database and frontend:

```powershell
php artisan migrate
php artisan storage:link
npm.cmd run build
php artisan serve
```

Migrate executes pending files in database/migrations and records them in the migrations table. It creates/updates schema; it does not copy accounts, stock or historical observations from the other device. Do not use migrate:fresh on a database whose contents you want to keep. For the supplied complete SQL snapshot, follow the import steps below instead of resetting it with migrate:fresh.

### Reusable presentation SQL in the same database

Owner explicitly chose **flowertrack_ims**. Download/extract the supplied FlowerTrack_Presentation_Database.zip, back up current records, then import flowertrack_ims_presentation.sql through phpMyAdmin. Its explicit USE selects flowertrack_ims, and DROP/CREATE statements replace the listed existing tables and records. Repeating the import restores the exact same sample data and clears old sessions.

Copy the extracted storage folder into the project, merging folders. The private synthetic proof resides at storage/app/private/delivery-proofs/presentation-demo-proof.png under Laravel's default local disk root. Keep .env and APP_KEY; run config:clear and view:clear and log in again. Sample Admin: admin@demo.example.test / DemoFlower!2026. These are public synthetic local-test credentials.

The snapshot includes October–November 2026 sales/history and prediction snapshots dated 15–21 November 2026. Date-based reports must include those dates. It is not a rolling dataset and does not establish model accuracy. The earlier separate-demo-database seeder/reset package is an alternative artifact, **not required for this same-database SQL import**.

### Runtime boundaries and troubleshooting

- `npm.ps1 cannot be loaded`: follow [PowerShell npm setup](#powershell-npm-setup), or use `npm.cmd` in PowerShell.
- DB connection refused: check XAMPP MySQL and .env host/port, then config:clear. Clearing a database-backed cache needs MySQL running.
- No account after migrate: migrations do not create a login account. Restore the SQL/backup or follow the existing Admin-account setup; never assume credentials are transferred through Git.
- Missing Python executable/import: create .venv on this device and install requirements using that exact executable.
- Failed health/self-test: resolve it before relying on forecasts. InsufficientData is a legitimate lack-of-history result, not Low risk.
- Git does not store the local database or .venv. Store code/config/migrations/requirements in Git and keep private database backups separately.
- Checkpoint `46e4145` now includes the Spoilage page API and automatic browser-triggered refresh. The initial engine-only scope of `53251a5` is historical. A completed-day observation collector and daily scheduler are still separate work; do not fabricate missing history.



## Automatic Spoilage page refresh — 2026-11-16

Application checkpoint: `46e4145`. Dependency manifest/ignore checkpoint: `f213583`. Pull both through the latest main branch before setting up another device.

- Run `.\\.venv\\Scripts\\python.exe -m pip install -r requirements.txt` after creating that device's venv.
- Run `.\\.venv\\Scripts\\python.exe scripts\\spoilage\\predict_many.py --self-test`.
- Run `php artisan test --filter=SpoilageAutoRefreshTest` and `npm.cmd run build`, then clear compiled views if necessary.
- Open the Spoilage page and use Ctrl + F5 after a new frontend build. Saved results load first; stale/missing inputs trigger one automatic POST and one Python process for the changed batch set. Unchanged results do not run Python.
- No manual Update All click, extra Python web server, queue worker, new migration or database reset is needed for this checkpoint.
- Retain the SystemRoot/WINDIR environment forwarding in SpoilagePythonService on Windows. The enhanced health check tests socket/asyncio and the ARIMA import, not only package version imports.
- The request uses the shared Laravel cache lock; the configured cache backend and existing cache tables must work. Overlapping tabs wait for the current computation. Python timeout defaults to 120 seconds; PHP refresh execution is allowed up to 180 seconds and the lock expires after 240 seconds if a process dies.
- Same-day saved-date comparisons use whereDate for MySQL/SQLite compatibility. Do not revert the service to the earlier plain string date comparison.
- Owner verification: full suite 107 passed / 1,439 assertions; dedicated suite 9 passed / 62 assertions. Build passed with existing image/chunk warnings. Final browser acceptance of this automatic flow has not been explicitly supplied.
- This refresh requires an open page request and existing valid completed observations. It does not collect those observations automatically or run while the page is closed.

