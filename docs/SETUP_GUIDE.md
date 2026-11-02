# FlowerTrack IMS — Windows Setup Guide

**Last updated:** 2026-11-02  
**Purpose:** Prepare a new Windows computer for FlowerTrack IMS development, including official installer links, installation steps, PHP configuration, and verification.

This guide is for the Laravel application. The documentation repository and application repository are separate:

- **Application source:** https://github.com/wyntheprogrammer/flowertrack-ims
- **Documentation:** https://github.com/wyntheprogrammer/flowertrack-ims-docs

> **Version rule:** Match the project's committed `composer.lock` and `package-lock.json`. The versions below reflect the verified FlowerTrack development environment as of this guide's update. Patch releases may be newer; use a compatible stable release and do not upgrade project dependencies casually.

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

The application is currently in a UI-first development phase. Database configuration is only needed when database-backed features are being implemented or tested.

## 11. Database setup (when database work begins)

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

When migrations are ready and the schema is confirmed, run:

```cmd
php artisan migrate
```

Do not run migrations on a shared or production database without confirming the target environment and migration impact.

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
