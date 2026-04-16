# Chapter 1: Getting Started with Angular

Every framework has a first impression -- the handful of minutes between installing a CLI tool and seeing something render in a browser. Angular's first impression has changed dramatically over the past decade. What was once a ceremony of modules, decorators, and zone.js polyfills has become a streamlined, signal-first development experience. Angular v21 generates zoneless, standalone applications with Vitest as the default test runner. If you have tried Angular before and bounced off the boilerplate, it is worth a fresh look.

This chapter covers the practical setup: installing the tools, generating the FinancialApp project that we will build on throughout the book, and understanding the files the CLI creates. By the end, you will have a running application, a configured linter, and a production build -- the foundation for everything that follows.

## Tooling

### Development Environment

Angular development requires a code editor and a terminal. Visual Studio Code (VS Code) is the most popular choice in the Angular ecosystem, and it is what we will use throughout this book. The Angular Language Service extension provides autocompletion, type checking, and inline error reporting inside templates. Install it from the VS Code extensions marketplace.

Beyond the editor, you will want:

- A modern terminal (the integrated terminal in VS Code works fine)
- Git for version control
- A recent browser with developer tools (Chrome or Firefox)

The Angular CLI handles compilation, bundling, and serving. You do not need to install Webpack, esbuild, or any other bundler directly -- the CLI manages those tools under the hood using its Vite-based builder.

### Node.js

Angular requires Node.js version 20 or later. Angular v21 officially supports Node 20 (LTS) and Node 22. You can verify your installed version:

```bash
node --version
# v22.14.0 (or any 20.x / 22.x release)
```

If you manage multiple projects with different Node requirements, use a version manager like `nvm` (macOS/Linux) or `nvm-windows`:

```bash
nvm install 22
nvm use 22
```

The companion codebase (`financial-app/`) includes an `.nvmrc` file that pins the expected Node version. Running `nvm use` inside that directory will switch to the correct version automatically.

### Angular CLI

The Angular CLI is a command-line tool that generates projects, scaffolds components, runs development servers, and builds production bundles. Install it globally:

```bash
npm install -g @angular/cli
```

Verify the installation:

```bash
ng version
```

You should see output listing Angular CLI v21 along with the versions of Node and the package manager it detected. If you prefer not to install globally, you can use `npx` to run CLI commands without a global install:

```bash
npx @angular/cli new financial-app
```

Throughout this book we will use the shorter `ng` form, assuming a global install.

### Example Project Repository

Every chapter builds on the same project: a personal finance management application called FinancialApp. The companion codebase lives in the `financial-app/` directory alongside these chapter files. You can follow along by generating a fresh project (as we will do in this chapter) or by cloning the companion repository and checking out the branch for Chapter 1.

The domain covers accounts, transactions, portfolios, and clients -- rich enough to demonstrate real architectural patterns without drowning in business logic. We will introduce the domain model gradually as each chapter adds new features.

## Getting Started with the Angular CLI

### Generating a New Project

Create the FinancialApp project with the `ng new` command:

```bash
ng new financial-app --style=scss --ssr=false
```

The CLI prompts you to confirm a few options. Here is what it creates:

- `--style=scss` selects SCSS for stylesheets. You can also use `css`, `less`, or `sass`.
- `--ssr=false` disables server-side rendering. We will cover SSR and hydration in [Chapter 17](ch17-defer-ssr-hydration.md).

As of Angular v21, the generated project is **standalone by default** (no `NgModule` anywhere), **zoneless by default** (no `zone.js` polyfill), and uses **Vitest** as its test runner. These were opt-in features in earlier versions. Now they are the starting point.

The CLI also initializes a Git repository and runs `npm install` for you. When it finishes, you have a fully functional project ready to serve.

### Starting an Angular Application

Navigate into the project directory and start the development server:

```bash
cd financial-app
ng serve
```

The CLI compiles the application and starts a local server, typically at `http://localhost:4200`. Open that URL in your browser and you will see Angular's default welcome page. The development server watches for file changes and reloads the browser automatically.

You can also specify a port if 4200 is already in use:

```bash
ng serve --port 4300
```

The `ng serve` command uses Vite under the hood, which provides fast cold starts and near-instant hot module replacement (HMR). Changes to component templates and styles reflect in the browser almost immediately.

### Project Structure of CLI Projects

After generation, the project directory looks like this:

```
financial-app/
├── src/
│   ├── app/
│   │   ├── app.component.ts
│   │   ├── app.component.html
│   │   ├── app.component.scss
│   │   ├── app.component.spec.ts
│   │   ├── app.config.ts
│   │   └── app.routes.ts
│   ├── index.html
│   ├── main.ts
│   └── styles.scss
├── public/
│   └── favicon.ico
├── angular.json
├── package.json
├── tsconfig.json
├── tsconfig.app.json
└── tsconfig.spec.json
```

A few things to note about the structure:

- **No `app.module.ts`**: Angular v21 projects do not use `NgModule` for bootstrapping. The application is configured through `app.config.ts` instead.
- **No `zone.js`**: the polyfills array in `angular.json` is empty. Change detection is driven by signals.
- **`public/` instead of `assets/`**: static assets now live in a `public/` directory at the project root, following web-platform conventions.
- **`app.routes.ts`**: routing configuration lives in its own file, ready to be passed to `provideRouter()`.

### Inspecting the Generated Source Code

Let us walk through the key files that the CLI generates.

#### App Component with Signals and Data Bindings

The root component in `app.component.ts` is a standalone component -- no module declaration required:

```typescript
// src/app/app.component.ts
import { Component } from '@angular/core';
import { RouterOutlet } from '@angular/router';

@Component({
  selector: 'app-root',
  imports: [RouterOutlet],
  templateUrl: './app.component.html',
  styleUrl: './app.component.scss',
})
export class AppComponent {
  title = 'financial-app';
}
```

Several things distinguish this from older Angular code:

- **No `standalone: true`**: as of Angular v19, all components are standalone by default. The property only appears if you explicitly set it to `false` (which you should not do in new code).
- **`imports` instead of a module**: the component declares its own template dependencies directly. Here it imports `RouterOutlet` so the template can use `<router-outlet>`.
- **No `NgModule` wrapper**: the component is self-contained. There is no `declarations` array, no `AppModule` class.

The template in `app.component.html` uses property binding to display the title:

```html
<!-- src/app/app.component.html -->
<h1>Welcome to {{ title }}!</h1>
<router-outlet />
```

The `{{ title }}` interpolation binds to the `title` property on the component class. We will replace this default template with our own layout soon. In [Chapter 2](ch02-signal-components.md), we will convert simple properties like `title` into signals and explore input/output bindings in depth.

#### Bootstrapping the App Component

The entry point of the application is `main.ts`, which calls `bootstrapApplication()`:

```typescript
// src/main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { appConfig } from './app/app.config';
import { AppComponent } from './app/app.component';

bootstrapApplication(AppComponent, appConfig);
```

This is the modern bootstrap pattern. Instead of passing an `NgModule` to `platformBrowserDynamic().bootstrapModule()`, you pass a component and a configuration object. The configuration lives in `app.config.ts`:

```typescript
// src/app/app.config.ts
import { ApplicationConfig, provideZonelessChangeDetection } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideHttpClient } from '@angular/common/http';
import { provideAnimationsAsync } from '@angular/platform-browser/animations/async';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideZonelessChangeDetection(),
    provideRouter(routes),
    provideHttpClient(),
    provideAnimationsAsync(),
  ],
};
```

Each `provide*()` function registers a set of related services:

- **`provideZonelessChangeDetection()`** enables signal-based change detection without zone.js. Angular schedules change detection when signals update, when events fire, or when async operations complete.
- **`provideRouter(routes)`** configures the router with the route definitions from `app.routes.ts`. We will build on these routes extensively in [Chapter 4](ch04-router.md).
- **`provideHttpClient()`** registers the `HttpClient` service for making HTTP requests. Without this call, injecting `HttpClient` throws a runtime error.
- **`provideAnimationsAsync()`** registers the animation engine lazily. This is preferred over `provideAnimations()` because it defers loading of the animation code until an animation actually triggers, reducing the initial bundle size.

The route configuration starts empty, ready for you to add routes:

```typescript
// src/app/app.routes.ts
import { Routes } from '@angular/router';

export const routes: Routes = [];
```

#### Connecting the Root Component to the Start Page

The `index.html` file is a standard HTML page with one Angular-specific element:

```html
<!-- src/index.html -->
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <title>FinancialApp</title>
    <base href="/" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <link rel="icon" type="image/x-icon" href="favicon.ico" />
  </head>
  <body>
    <app-root></app-root>
  </body>
</html>
```

The `<app-root>` element is the mount point. When `bootstrapApplication()` runs, it finds this element (matching the `selector: 'app-root'` in `AppComponent`) and renders the component tree inside it. The `<base href="/">` tag tells the Angular router to use the root URL as its base path.

### Installing Additional Packages

As the FinancialApp grows, you will need packages beyond what the CLI provides. Install them with your package manager:

```bash
npm install @angular/material @angular/cdk
```

For development-only tools like linters or code formatters:

```bash
npm install --save-dev eslint angular-eslint
```

A word of caution: the Angular ecosystem moves in lockstep. All `@angular/*` packages in a project should share the same major version. Mixing Angular v21 core packages with v19 Material will produce obscure build errors. The CLI's `ng update` command helps keep everything aligned:

```bash
ng update @angular/core @angular/cli
```

### Adding Components and Styles

Generate a new component with the CLI:

```bash
ng generate component accounts/account-list
```

This creates four files inside `src/app/accounts/account-list/`:

```
src/app/accounts/account-list/
├── account-list.component.ts
├── account-list.component.html
├── account-list.component.scss
└── account-list.component.spec.ts
```

The generated component is standalone with a blank template and an empty test file using Vitest. You can also use shorthand:

```bash
ng g c accounts/account-detail
```

For global styles that apply across the entire application -- CSS resets, font imports, theme variables -- use `src/styles.scss`. Component-specific styles go in each component's `.scss` file, where Angular's view encapsulation keeps them scoped.

### Configuring the Angular CLI

The `angular.json` file is the central configuration for the CLI. It defines build targets, file paths, and builder options. The most commonly adjusted settings are:

```json
{
  "projects": {
    "financial-app": {
      "architect": {
        "build": {
          "options": {
            "outputPath": "dist/financial-app",
            "styles": ["src/styles.scss"],
            "scripts": []
          }
        },
        "serve": {
          "options": {
            "port": 4200
          }
        }
      }
    }
  }
}
```

You can add entries to the `styles` array to include third-party CSS (for example, an Angular Material prebuilt theme), or adjust the `outputPath` to match your deployment pipeline. The file supports per-configuration overrides -- `development` and `production` configurations are created by default, each with appropriate optimization settings.

### Initializing the Linter

Angular v21 no longer ships a linter out of the box, but the angular-eslint project fills the gap. Set it up with a single schematic:

```bash
ng add angular-eslint
```

This installs the necessary packages and creates an `.eslintrc.json` (or `eslint.config.js`) at the project root with Angular-specific rules. Run the linter with:

```bash
ng lint
```

The default rules catch common issues: unused imports, missing accessibility attributes in templates, and incorrect lifecycle hook usage. You can customize the rules as your team's conventions develop. Linting is especially valuable in larger projects where consistency across dozens of components matters.

### Building the Application

When you are ready to deploy, create a production build:

```bash
ng build
```

By default this runs with the `production` configuration, which enables:

- Ahead-of-time (AOT) compilation
- Tree-shaking of unused code
- Minification and bundling
- Source map generation (disabled in production by default)

The output lands in `dist/financial-app/`. Inside you will find an `index.html`, a handful of JavaScript bundles, and your static assets from the `public/` directory. The total bundle size for a fresh project is remarkably small -- under 100 KB for the initial load, thanks to zoneless change detection eliminating the zone.js polyfill entirely.

To inspect the bundle sizes:

```bash
ng build --stats-json
npx webpack-bundle-analyzer dist/financial-app/stats.json
```

Keeping an eye on bundle sizes from the start prevents surprises when the application grows. We will revisit build optimization techniques in [Chapter 17](ch17-defer-ssr-hydration.md) when we cover `@defer` blocks and lazy loading.

## Summary

This chapter set up the development environment and created the FinancialApp project that will serve as our running example. We covered the core tooling -- Node.js, the Angular CLI, and VS Code -- and walked through the files the CLI generates. The key points to carry forward:

- Angular v21 projects are **standalone and zoneless by default**. There are no `NgModule` classes and no zone.js polyfill in new projects.
- The application is bootstrapped with `bootstrapApplication()` and an `ApplicationConfig` object, not a module.
- Provider functions like `provideRouter()`, `provideHttpClient()`, and `provideAnimationsAsync()` configure framework services declaratively in `app.config.ts`.
- **Vitest** is the default test runner, replacing Karma and Jasmine.
- The CLI handles project generation, component scaffolding, linting, and production builds.

In [Chapter 2](ch02-signal-components.md), we will dive into Angular's signal-based component model -- `input()`, `output()`, `model()`, and the new control flow syntax -- and start building the first real features of the FinancialApp.
