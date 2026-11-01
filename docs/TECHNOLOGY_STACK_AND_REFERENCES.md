# Technology Stack and Reference Repositories

**Last updated:** 2026-11-01

This document records the requested repositories as learning/reference resources. Listing a repository does **not** mean its code or package is automatically part of FlowerTrack IMS.

## Classification

- **Core direction:** planned technology for the application.
- **Reference:** source code, documentation, examples, or learning material.
- **Optional candidate:** may be evaluated if a real requirement needs it.
- **Alternative:** not planned for simultaneous use with the selected core technology.

## Core technology direction

| Technology | Planned use | Status |
|---|---|---|
| Laravel | Main PHP web framework | Core direction; Laravel 13.11.0 starter application installed locally |
| PHP | Server-side language | Core; PHP 8.3.36 installed through XAMPP |
| MySQL | Relational database | Core direction; XAMPP bundles MariaDB 10.11.16 for local development |
| HTML | Page structure | Core |
| JavaScript | Browser interactions | Core |
| Bootstrap | Main UI component framework | Core preference |
| Flexbox | Layout and alignment | Core styling technique |
| Custom CSS | Colors, typography, sizing, spacing, visual refinement | Core styling technique |
| Python | Potential spoilage prediction work | To decide |

## Local development setup

The initial Windows development environment is installed and the Laravel starter page has been verified locally.

| Tool | Current setup |
|---|---|
| XAMPP | Installed in `C:\xampp`; local dashboard/phpMyAdmin accessible |
| Apache | XAMPP Apache available; ports 80 and 443 had to be freed from an existing Apache process |
| Database server | XAMPP MariaDB 10.11.16; port 3306 initially conflicted with an existing MySQL process |
| PHP | 8.3.36 at `C:\xampp\php\php.exe` |
| Composer | 2.11.1 |
| PHP ZIP | Enabled in XAMPP's `php.ini` |
| Laravel | 13.11.0 installed in `C:\xampp\htdocs\FlowerTrack` |
| Visual Studio Code | Installed; FlowerTrack project opened with `code .` |
| Local Laravel URL | `http://127.0.0.1:8000` (verified with `php artisan serve`) |
| Node.js / npm | Not yet verified |

The XAMPP package version itself has not yet been recorded. The bundled database reports MariaDB 10.11.16; document this accurately as MariaDB rather than assuming it is Oracle MySQL.

## 1. Laravel

| Repository | Description | Classification |
|---|---|---|
| [laravel/laravel](https://github.com/laravel/laravel) | Laravel application starter project | Reference |
| [laravel/framework](https://github.com/laravel/framework) | Laravel framework source | Reference |
| [filamentphp/filament](https://github.com/filamentphp/filament) | Admin panels, forms, and tables | Optional candidate; evaluate against custom Bootstrap UI |
| [spatie/laravel-permission](https://github.com/spatie/laravel-permission) | Roles and permissions for Laravel | Optional candidate |
| [Maatwebsite/Laravel-Excel](https://github.com/SpartnerNL/Laravel-Excel) | Excel import and export | Optional candidate |

## 2. PHP

| Repository | Description | Classification |
|---|---|---|
| [php/php-src](https://github.com/php/php-src) | PHP language source code | Reference |
| [PHPMailer/PHPMailer](https://github.com/PHPMailer/PHPMailer) | Email sending library | Optional candidate |
| [CodeIgniter4/CodeIgniter4](https://github.com/CodeIgniter4/CodeIgniter4) | Alternative PHP framework | Alternative; not planned alongside Laravel |
| [ziadoz/awesome-php](https://github.com/ziadoz/awesome-php) | Curated PHP libraries and resources | Reference |
| [thedevdojo/wave](https://github.com/thedevdojo/wave) | Laravel-based SaaS starter | Reference/alternative starter |

## 3. JavaScript

| Repository | Description | Classification |
|---|---|---|
| [trekhleb/javascript-algorithms](https://github.com/trekhleb/javascript-algorithms) | Algorithms and data structures learning reference | Reference |
| [chartjs/Chart.js](https://github.com/chartjs/Chart.js) | Charts and graphs | Optional candidate |
| [jquery/jquery](https://github.com/jquery/jquery) | DOM manipulation library | Optional; use only if needed |
| [axios/axios](https://github.com/axios/axios) | HTTP client for requests | Optional candidate |
| [SortableJS/Sortable](https://github.com/SortableJS/Sortable) | Drag-and-drop sorting | Optional candidate |

## 4. Bootstrap and admin UI references

| Repository | Description | Classification |
|---|---|---|
| [twbs/bootstrap](https://github.com/twbs/bootstrap) | Official Bootstrap framework | Core reference |
| [StartBootstrap/startbootstrap-sb-admin-2](https://github.com/StartBootstrap/startbootstrap-sb-admin-2) | Admin dashboard template | Reference |
| [ColorlibHQ/AdminLTE](https://github.com/ColorlibHQ/AdminLTE) | Bootstrap admin dashboard template | Reference |
| [coreui/coreui-free-bootstrap-admin-template](https://github.com/coreui/coreui-free-bootstrap-admin-template) | Bootstrap admin template | Reference |
| [StartBootstrap/startbootstrap-sb-admin](https://github.com/StartBootstrap/startbootstrap-sb-admin) | Admin dashboard UI | Reference |

Templates are references only. The project owner prefers a deliberate, custom-designed UI and has not approved adopting any template wholesale.

## 5. CSS

| Repository | Description | Classification |
|---|---|---|
| [animate-css/animate.css](https://github.com/animate-css/animate.css) | CSS animations | Optional |
| [necolas/normalize.css](https://github.com/necolas/normalize.css) | CSS normalization | Reference/optional |
| [tailwindlabs/tailwindcss](https://github.com/tailwindlabs/tailwindcss) | Utility-first CSS framework | Alternative; Bootstrap remains the chosen UI preference |
| [saadeghi/daisyui](https://github.com/saadeghi/daisyui) | Tailwind component library | Alternative; depends on Tailwind |
| [nostalgic-css/NES.css](https://github.com/nostalgic-css/NES.css) | Retro-style CSS library | Reference only |

## 6. HTML and learning resources

| Repository | Description | Classification |
|---|---|---|
| [whatwg/html](https://github.com/whatwg/html) | HTML standard | Reference |
| [mdn/learning-area](https://github.com/mdn/learning-area) | HTML, CSS, and JavaScript learning examples | Learning reference |
| [30-seconds/30-seconds-of-code](https://github.com/30-seconds/30-seconds-of-code) | Short code examples | Learning reference |
| [FrontendMentor](https://github.com/FrontendMentor) | Frontend practice resources | Learning reference; verify specific challenge repository |

## 7. Python

| Repository | Description | Classification |
|---|---|---|
| [python/cpython](https://github.com/python/cpython) | Python language source code | Reference |
| [pallets/flask](https://github.com/pallets/flask) | Lightweight Python web framework | Optional candidate |
| [fastapi/fastapi](https://github.com/fastapi/fastapi) | API framework | Optional candidate |
| [scikit-learn/scikit-learn](https://github.com/scikit-learn/scikit-learn) | Machine learning tools | Optional candidate for spoilage prediction |
| [Asabeneh/30-Days-Of-Python](https://github.com/Asabeneh/30-Days-Of-Python) | Python learning course | Learning reference |

## 8. MySQL and database references

| Repository | Description | Classification |
|---|---|---|
| [mysql/mysql-server](https://github.com/mysql/mysql-server) | MySQL server source | Reference |
| [datacharmer/test_db](https://github.com/datacharmer/test_db) | Sample employee database | Learning reference |
| [kanboard/kanboard](https://github.com/kanboard/kanboard) | PHP application with database examples | Reference |
| [phpmyadmin/phpmyadmin](https://github.com/phpmyadmin/phpmyadmin) | MySQL/MariaDB administration tool | Development tool/reference |
| [laravel/laravel](https://github.com/laravel/laravel) | Laravel application structure, including migrations | Reference |

## Usage rule

Before adding a package or template:
1. Identify the requirement it solves.
2. Check compatibility with the chosen Laravel/PHP versions.
3. Consider whether Bootstrap and native Laravel features already solve it.
4. Explain trade-offs to the project owner.
5. Add it to the approved stack only after a decision.
