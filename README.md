# Dependency Injection Error

The Filament Navigation Items are cached by octane when they should not be. See the [.mp4](filament-dependecy-injection.mp4) file in this project for a demo (using laravel sail in this project with two users created by `sail artisan make:filament-user`)

This issue occurs because laravel octane caches the singleton provided in filament/src/FilamentServiceProvider.php line 72-73

```php

namespace Filament;

...

class FilamentServiceProvider extends PackageServiceProvider
{
    ...

    public function packageRegistered(): void
    {
        $this->app->singleton('filament', function (): FilamentManager {
            return app(FilamentManager::class);
        });

        ...
    }

   ...
}

```

The filament/src/FilamentManager.php FilamentManager class registers all the navigation items on 

```php
namespace Filament;

...

class FilamentManager
{
   ...
   lines 82 - 103 

   public function mountNavigation(): void
    {
        foreach ($this->getPages() as $page) {
            $page::registerNavigationItems();
        }

        foreach ($this->getResources() as $resource) {
            $resource::registerNavigationItems();
        }

        $this->isNavigationMounted = true;
    }

    public function registerNavigationGroups(array $groups): void
    {
        $this->navigationGroups = array_merge($this->navigationGroups, $groups);
    }

    public function registerNavigationItems(array $items): void
    {
        $this->navigationItems = array_merge($this->navigationItems, $items);
    }

    ...
}


```

Since the new FilamentManager object is cached by octane as a singleton these values will never change until the octane worker reloads

However the pages and resources are set up to conditionaly register navigation items

```php
namespace Filament\Pages;

...

class Page extends Component implements Forms\Contracts\HasForms
{
    ...

    public static function registerNavigationItems(): void
    {
        if (! static::shouldRegisterNavigation()) {
            return;
        }

        Filament::registerNavigationItems(static::getNavigationItems());
    }
    ...
}

namespace Filament\Resources;

...

class Resource
{
    ...
    lines 53 - 64

    public static function registerNavigationItems(): void
    {
        if (! static::shouldRegisterNavigation()) {
            return;
        }

        if (! static::canViewAny()) {
            return;
        }

        Filament::registerNavigationItems(static::getNavigationItems());
    }

    ...
}

```

this means that whatever navigation items the first user that an Octane worker sees will stay registered forever on that octane worker


## Solutions

1. stop binding FilamentManager as a singleton in the service provider (not the best solution since a new object will be created every time the facade is used)
2. Always register every navigation item and then move the conditional rendering code into the appropriate blade files (the better solution but requires more work)
