# FlowerTrack IMS — Setup Guide

**Last updated:** 2026-11-02  
**Purpose:** Set up the FlowerTrack IMS application on a new Windows computer and continue development.

This guide is for the Laravel application. The documentation repository and application repository are separate:

- **Application source:** https://github.com/wyntheprogrammer/flowertrack-ims
- **Documentation:** https://github.com/wyntheprogrammer/flowertrack-ims-docs

> Follow the current project versions recorded in this guide and in `docs/TECHNOLOGY_STACK_AND_REFERENCES.md`. If the project files specify a different compatible version later, prefer the committed project configuration.

## 1. Required software

Install these tools on the new computer:

| Software | Project baseline / notes |
|---|---|
| Windows | Development environment used by the project |
| XAMPP | Install to `C:\xampp`; includes Apache, PHP, and MariaDB |
| PHP | 8.3.x baseline; use the XAMPP PHP executable |
| Composer | 2.x |
| Node.js | 26.x baseline |
| npm | Included with Node.js; project baseline 11.x |
| Git for Windows | 2.x |
| Visual Studio Code | Recommended editor |

The original development computer was verified with PHP 8.3.36, Composer 2.11.1, Node.js 26.13.0, npm 11.20.0, Git 2.52.0.windows.1, Laravel 13.11.0, and XAMPP MariaDB 10.11.16. These are recorded baselines, not a requirement to install identical patch versions if compatible updates are needed.

### Verify command-line tools

Open Command Prompt and run:

```cmd
php -v
composer -V
node -v
npm -v
git --version
```

If `php` is not recognized, add `C:\xampp\php` to the Windows PATH, then close and reopen Command Prompt. Verify that `where php` points to the XAMPP PHP executable:

```cmd
where php
```

### Check required PHP extensions

Run:

```cmd
php -m
```

Confirm that the Laravel-required extensions are available, including `BCMath`, `Ctype`, `cURL`, `DOM`, `Fileinfo`, `Mbstring`, `OpenSSL`, `PDO`, `pdo_mysql`, `Tokenizer`, and `XML`. The `zip` extension should also be enabled for Composer package downloads.

For XAMPP PHP, edit `C:\xampp\php\php.ini` if an extension needs enabling. Remove the leading semicolon from the relevant `extension=...` line, save, and reopen the terminal. Do not enable extensions by guessing their names; check the installed PHP version's configuration.

## 2. Clone the application repository

Choose a workspace. The original machine used `C:\xampp\htdocs\FlowerTrack`; use the same path if you want to follow the existing commands exactly.

In Command Prompt:

```cmd
cd /d C:\xampp\htdocs
git clone https://github.com/wyntheprogrammer/flowertrack-ims.git FlowerTrack
cd FlowerTrack
```

If the folder already exists, do not clone over it. Open the existing folder and check its Git status first.

Open the project in Visual Studio Code:

```cmd
code .
```

## 3. Install PHP and JavaScript dependencies

Run these commands from the application root, where `composer.json` and `package.json` are located:

```cmd
composer install
npm ci
```

- `composer install` installs the PHP packages from the committed `composer.lock`.
- `npm ci` installs the JavaScript/CSS packages from the committed `package-lock.json`. If the repository does not contain a lockfile, use `npm install` instead and commit the generated lockfile only when intentionally updating dependencies.

Do not copy the `vendor` or `node_modules` folders from another computer. Recreate them with these commands.

## 4. Create the local environment file

The `.env` file contains machine-specific settings and must not be committed.

```cmd
copy .env.example .env
php artisan key:generate
```

If a local `.env` already exists, do not overwrite it blindly. Keep the machine's local configuration and review it instead.

The application is currently in a UI-first development phase. Database configuration is only needed when database-backed features are being implemented or tested.

## 5. Database setup (when database work begins)

1. Open the XAMPP Control Panel.
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

## 6. Build frontend assets

For a production-style asset build:

```cmd
npm run build
```

For active frontend development with Vite watching for changes, open a separate terminal and run:

```cmd
npm run dev
```

Keep the Vite terminal running while developing. Stop it with `Ctrl+C` when finished.

## 7. Run the Laravel application

From the project root:

```cmd
php artisan serve
```

Open the URL shown in the terminal. The usual local URL is:

`http://127.0.0.1:8000`

If the application uses Vite during development, keep `npm run dev` running in a second terminal. If you have already run `npm run build`, the compiled assets can be served from the build output.

## 8. Daily workflow on either computer

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

## 9. Files and settings that should not be copied or committed

- Do not commit `.env` or secrets.
- Do not copy `vendor` or `node_modules`; reinstall dependencies.
- Do not overwrite local database credentials with another computer's credentials.
- Keep personal editor settings and machine-specific paths local unless the team intentionally shares them.

Git should ignore `.env`, `vendor`, and `node_modules`. Confirm with:

```cmd
git check-ignore .env vendor node_modules
```

## 10. Troubleshooting

### `php` is not recognized

Add `C:\xampp\php` to PATH, reopen Command Prompt, then run `php -v` and `where php`.

### Composer reports a missing ZIP extension

Enable the PHP `zip` extension in `C:\xampp\php\php.ini`, reopen the terminal, and verify with:

```cmd
php -m
```

### Composer or npm dependencies fail

Confirm internet access and compatible PHP/Node versions. Run commands from the application root. Use `composer install` and `npm ci` rather than copying dependency folders from another computer.

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

## Related documentation

- [Project README](../README.md)
- [AI Context](AI_CONTEXT.md)
- [Technology Stack and References](TECHNOLOGY_STACK_AND_REFERENCES.md)
- [Development Progress](DEVELOPMENT_PROGRESS.md)
- [UI/UX Guidelines](UI_UX_GUIDELINES.md)
