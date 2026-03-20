# Architecture: runtime

## Purpose

The Symfony Runtime component — decouples application bootstrapping from the entry-point script. Allows the same application to run under traditional PHP-FPM, CLI, ReactPHP, FrankenPHP workers, or other environments by swapping a runtime implementation.

## Directory Structure

```
RuntimeInterface.php                    - Contract: getResolver(callable $app): ResolverInterface
ResolverInterface.php                   - Contract: resolve(): RunnerInterface
RunnerInterface.php                     - Contract: run(): int (exit code)
SymfonyRuntime.php                      - Default implementation: handles HttpKernel and Console apps
GenericRuntime.php                      - Minimal runtime for non-Symfony apps
Runner/
  ClosureRunner.php                     - Wraps a closure as a runner
  Symfony/
    HttpKernelRunner.php                - Runs an HttpKernelInterface application (web)
    ConsoleApplicationRunner.php        - Runs a Symfony Console Application
    ResponseRunner.php                  - Runs an app that returns a Response directly
  FrankenPhpWorkerRunner.php            - FrankenPHP worker mode runner
Resolver/
  ClosureResolver.php                   - Resolves a closure app to the right runner
  DebugClosureResolver.php             - Debug-mode resolver with error handling
Internal/
  Console/                              - Type-specific runtime wrappers for CLI objects
  HttpFoundation/                       - Request/Response runtime wrappers
  HttpKernel/                           - HttpKernelInterface runtime wrapper
  SymfonyErrorHandler.php              - Registers the Symfony error handler
  BasicErrorHandler.php                - Minimal error handler
  MissingDotenv.php                    - Friendly error when dotenv is absent
  ComposerPlugin.php                   - Composer plugin: configures the runtime via composer.json extra
```

## Key Design Decisions

- **Three-interface chain**: `RuntimeInterface` → `ResolverInterface` → `RunnerInterface` cleanly separates bootstrap, argument resolution, and execution — each step is independently substitutable.
- **Zero framework coupling in RunnerInterface**: `run()` returns an `int` exit code. The runner decides how to call the application — web, CLI, or event loop — without the application knowing which environment it's in.
- **Composer plugin**: Detects the application's runtime class from `composer.json` extra and configures the entry-point script automatically.
- **FrankenPHP worker mode**: `FrankenPhpWorkerRunner` implements a keep-alive worker loop for FrankenPHP's built-in PHP worker model.

## Extension Points

- Implement `RuntimeInterface` to support custom SAPIs (Swoole, RoadRunner, OpenSwoole, etc.).
- Implement `RunnerInterface` to add custom lifecycle hooks around `$kernel->handle($request)`.

## Dependency Flow

```
Entry point (public/index.php)
  └─> SymfonyRuntime::getResolver(fn() => new Kernel())
        └─> ClosureResolver::resolve()  — inspects callable signature
              └─> HttpKernelRunner (web) or ConsoleApplicationRunner (CLI)
                    └─> run() — handle request, send response, terminate
```
