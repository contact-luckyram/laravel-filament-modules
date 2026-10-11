# How to Set Up a Modular Structure in Laravel + Filament (nwidart/laravel-modules)

*By [Lucky Ram](https://github.com/contact-luckyram) · Laravel & Filament Developer*

As a Laravel app grows, the `app/` folder gets crowded fast. Models, controllers and Filament resources for every feature end up in the same place. **[nwidart/laravel-modules](https://github.com/nWidart/laravel-modules)** fixes this by splitting the app into self-contained **modules** such as `Blog`, `Billing` and `Users`. Each module has its own models, migrations, routes and Filament resources.

This post walks through installing it step by step and connecting it to a **Filament** admin panel.

---

## Requirements

- PHP 8.3+
- Composer
- Laravel 12 or 13
- Filament 4.x / 5.x

---

## Step 1 — Create a Laravel project

```bash
composer create-project laravel/laravel modular-app
cd modular-app
```

Set up your database in `.env`, then run:

```bash
php artisan migrate
```

---

## Step 2 — Install Filament

```bash
composer require filament/filament
php artisan filament:install --panels
php artisan make:filament-user
```

This creates `app/Providers/Filament/AdminPanelProvider.php`, and your panel is now available at `/admin`.

---

## Step 3 — Install nwidart/laravel-modules

```bash
composer require nwidart/laravel-modules
```

Publish the config file (optional, but recommended):

```bash
php artisan vendor:publish --provider="Nwidart\Modules\LaravelModulesServiceProvider"
```

---

## Step 4 — Enable module autoloading in `composer.json`

Recent versions of laravel-modules use **wikimedia/composer-merge-plugin** to autoload each module's own `composer.json`. Open the root `composer.json` and add the following.

```json
"extra": {
    "laravel": {
        "dont-discover": []
    },
    "merge-plugin": {
        "include": [
            "Modules/*/composer.json"
        ]
    }
},
"config": {
    "allow-plugins": {
        "wikimedia/composer-merge-plugin": true
    }
}
```

> If Composer asks *"Do you trust wikimedia/composer-merge-plugin…?"*, answer **yes**. If this plugin is set to `false`, your modules will **not** autoload.

Then refresh autoloading:

```bash
composer dump-autoload
```

---

## Step 5 — Create your first module

```bash
php artisan module:make Blog
```

This generates the following structure:

```
Modules/
└── Blog/
    ├── app/
    │   ├── Http/Controllers/
    │   ├── Models/
    │   └── Providers/
    ├── config/
    ├── database/
    │   ├── migrations/
    │   └── seeders/
    ├── resources/
    ├── routes/
    ├── composer.json
    └── module.json
```

Check that it is registered and enabled:

```bash
php artisan module:list
```

---

## Step 6 — Connect modules to the Filament panel

By default, Filament only discovers resources inside `app/Filament`. We need to tell the panel to also look inside each module.

Open `app/Providers/Filament/AdminPanelProvider.php` and add this loop:

```php
use Nwidart\Modules\Facades\Module;

public function panel(Panel $panel): Panel
{
    $panel = $panel
        ->default()
        ->id('admin')
        ->path('admin')
        ->login()
        ->discoverResources(in: app_path('Filament/Resources'), for: 'App\\Filament\\Resources')
        // ... the rest of the default config

    // Discover Filament resources from every enabled module
    foreach (Module::allEnabled() as $module) {
        $name = $module->getName();

        $panel->discoverResources(
            in: module_path($name, 'app/Filament/Resources'),
            for: "Modules\\{$name}\\Filament\\Resources",
        );
    }

    return $panel;
}
```

From now on, any Filament resource you place in `Modules/{Name}/app/Filament/Resources` shows up in the admin panel automatically.

---

## Step 7 — Add a Filament resource inside the module

Create a model and migration for the module:

```bash
php artisan module:make-model Post Blog -m
```

Add your columns to the generated migration in `Modules/Blog/database/migrations`, then run it:

```bash
php artisan module:migrate Blog
```

Next, create the folder `Modules/Blog/app/Filament/Resources` and add your resource there with the namespace `Modules\Blog\Filament\Resources`.

> **Tip:** If you want Artisan generators for module resources, the **[coolsam/modules](https://filamentphp.com/plugins/coolsam-modules)** plugin adds commands such as:
>
> ```bash
> composer require coolsam/modules
> php artisan module:filament:install Blog
> php artisan module:make:filament-resource
> ```

Refresh `/admin` and your **Posts** menu item appears in the sidebar.

---

## Step 8 — Enable or disable a module

One of the best parts of this setup is that a whole feature can be switched on or off with one command:

```bash
php artisan module:disable Blog   # Blog menu disappears from Filament
php artisan module:enable Blog    # and it's back
```

---

## Common issues

| Problem | Fix |
|---|---|
| `Class "Modules\Blog\..." not found` | Check the `merge-plugin` config, then run `composer dump-autoload` |
| Resource doesn't appear in the panel | Check the namespace matches the folder path, then run `php artisan optimize:clear` |
| Module not listed | Check `modules_statuses.json` in the project root |
| Plugin trust prompt declined | Set `"wikimedia/composer-merge-plugin": true` under `allow-plugins` |

---

## Wrapping up

With **nwidart/laravel-modules + Filament**, every feature lives in its own folder and plugs into the admin panel automatically. That keeps large Laravel projects clean, easy to test and easy for teams to work on in parallel.

If this helped, give the repo a ⭐ and feel free to connect:

[![GitHub](https://img.shields.io/badge/GitHub-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/contact-luckyram)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://linkedin.com/in/lucky-ram)

---

**References:** [laravel-modules docs](https://laravelmodules.com/docs/12/getting-started/upgrade) · [Packagist: nwidart/laravel-modules](https://packagist.org/packages/nwidart/laravel-modules) · [Filament plugin: coolsam/modules](https://filamentphp.com/plugins/coolsam-modules)
