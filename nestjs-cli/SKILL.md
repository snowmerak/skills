---
name: nestjs-cli
description: NestJS CLI commands and generators for scaffolding projects, creating modules, controllers, services, and managing the development workflow. Use when setting up NestJS projects, generating code, or using CLI commands.
license: MIT
metadata:
  author: snowmerak
  version: "1.0"
  framework: nestjs
  category: cli
---

# NestJS CLI Skills

This skill covers the NestJS CLI tool for scaffolding, generating code, and managing NestJS projects.

## Installation

```bash
npm install -g @nestjs/cli
```

## Basic Commands

### Project Creation

```bash
# Create a new NestJS project
nest new project-name

# Create with stricter TypeScript settings
nest new project-name --strict

# Create with yarn
nest new project-name --package-manager yarn

# Create with pnpm
nest new project-name --package-manager pnpm
```

### Code Generation

The NestJS CLI provides powerful generators to scaffold code:

```bash
# Generate a resource (controller + service + module)
nest g resource [name] [--dry-run] [--spec] [--flat] [--no-spec] [--crud]

# Generate a controller
nest g controller [name] [--dry-run] [--spec] [--flat]

# Generate a service
nest g service [name] [--dry-run] [--spec] [--flat]

# Generate a module
nest g module [name] [--dry-run] [--flat]

# Generate a provider
nest g provider [name] [--dry-run] [--spec] [--flat]

# Generate a class
nest g class [name] [--dry-run] [--spec] [--flat]

# Generate a gateway (WebSockets)
nest g gateway [name] [--dry-run] [--spec] [--flat]

# Generate a filter
nest g filter [name] [--dry-run] [--spec]

# Generate a pipe
nest g pipe [name] [--dry-run] [--spec] [--flat]

# Generate an interceptor
nest g interceptor [name] [--dry-run] [--spec] [--flat]

# Generate a guard
nest g guard [name] [--dry-run] [--spec] [--flat]

# Generate a middleware
nest g middleware [name] [--dry-run] [--spec] [--flat]

# Generate a resolver (GraphQL)
nest g resolver [name] [--dry-run] [--spec] [--flat]

# Generate a subscription (GraphQL)
nest g subscription [name] [--dry-run] [--spec] [--flat]
```

### Generator Options

- `--dry-run` - Show what would be generated without creating files
- `--spec` - Generate spec files (default: true)
- `--no-spec` - Don't generate spec files
- `--flat` - Generate files in the same directory (don't create subdirectory)
- `--crud` - Generate CRUD endpoints (only with `nest g resource`)

### CRUD Generator

The `nest g resource` command generates a full CRUD resource:

```bash
nest g resource pets

# Prompts:
# ? What transport layer do you use? [Enter] REST API
# ? Do you want to generate CRUD entry points? (y) Yes
# ? What protocol does this package use? (graphql)
# ? Where should the DTOs be imported from? (@nestjs/mapped-types)
# ? Which database platform do you use? (typeorm)
# ? Schematic options (comma separated): (flat)
```

This generates:
- Controller with CRUD endpoints
- Service with CRUD methods
- Module
- DTOs with validation
- Entity (if using TypeORM)
- Tests

## Project Commands

### Development

```bash
# Start development server with watch mode
npm run start

# Start development server with watch mode (shorthand)
npm run start:dev

# Start production mode
npm run start:prod

# Start with SWC compiler (20x faster builds)
npm run start -- -b swc
```

### Building

```bash
# Build the project
npm run build

# Build with SWC compiler
npm run build -- -b swc
```

### Code Quality

```bash
# Lint and autofix with ESLint
npm run lint

# Format with Prettier
npm run format
```

### Testing

```bash
# Run unit tests
npm run test

# Run unit tests in watch mode
npm run test:watch

# Run unit tests in coverage mode
npm run test:cov

# Run e2e tests
npm run test:e2e

# Run all tests
npm run test:e2e
```

### Additional Commands

```bash
# Display help
nest --help

# Show version
nest -v

# Create an application
nest new

# Create a library
nest generate library [name]

# Build a library
nest build [library-name]

# Publish a library
nest publish [library-name]
```

## tsconfig.json Configuration

The NestJS CLI generates a `tsconfig.json` with useful settings:

```json
{
  "compilerOptions": {
    "module": "commonjs",
    "declaration": true,
    "removeComments": true,
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true,
    "allowSyntheticDefaultImports": true,
    "target": "ES2021",
    "sourceMap": true,
    "outDir": "./dist",
    "baseUrl": "./",
    "incremental": true,
    "skipLibCheck": true,
    "strictNullChecks": false,
    "noImplicitAny": false,
    "strictBindCallApply": false,
    "forceConsistentCasingInFileNames": false,
    "noFallthroughCasesInSwitch": false
  }
}
```

## nest-cli.json Configuration

Customize the CLI behavior with `nest-cli.json`:

```json
{
  "$schema": "https://json.schemastore.org/nest-cli",
  "collection": "@nestjs/schematics",
  "sourceRoot": "src",
  "compilerOptions": {
    "deleteOutDir": true,
    "webpack": true,
    "tsConfigPath": "tsconfig.build.json",
    "plugins": [
      {
        "name": "@nestjs/swagger",
        "options": {
          "classValidatorShim": true,
          "introspectComments": true
        }
      }
    ]
  }
}
```

## Best Practices

1. **Use generators** - Leverage `nest g` commands for consistent code structure
2. **Feature modules** - Group related code into feature modules
3. **CRUD generator** - Use `nest g resource` for quick CRUD API scaffolding
4. **Linting** - Run `npm run lint` before committing
5. **Testing** - Always generate tests with `--spec` flag
6. **SWC** - Use SWC for faster builds in development

## References

- [NestJS CLI Documentation](https://docs.nestjs.com/cli/overview)
- [NestJS CRUD Generator](https://docs.nestjs.com/recipes/crud-generator)
- [NestJS Schematics](https://github.com/nestjs/schematics)
