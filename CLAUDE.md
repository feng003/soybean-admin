# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

SoybeanAdmin is a Vue3 admin template built with Vite7, TypeScript, NaiveUI, and UnoCSS. It uses a pnpm monorepo architecture with workspace packages.

## Development Commands

### Core Scripts
- `pnpm dev` - Start development server (test mode)
- `pnpm dev:prod` - Start development server (production mode)
- `pnpm build` - Build for production
- `pnpm build:test` - Build for test environment
- `pnpm typecheck` - Run TypeScript type checking
- `pnpm lint` - Run ESLint with auto-fix
- `pnpm preview` - Preview production build

### Additional Commands
- `pnpm gen-route` - Generate routes using elegant-router
- `pnpm commit` - Interactive commit with conventional format
- `pnpm commit:zh` - Interactive commit in Chinese
- `pnpm cleanup` - Clean up project files
- `pnpm release` - Release new version

## Environment Requirements

- Node.js: >=20.19.0
- pnpm: >=10.5.0
- Always use pnpm for package management (never npm or yarn)

## Architecture Overview

### Monorepo Structure
- `packages/` - Workspace packages:
  - `@sa/axios` - HTTP client utilities
  - `@sa/hooks` - Reusable Vue composition hooks
  - `@sa/utils` - Common utility functions
  - `@sa/color` - Color system and palettes
  - `@sa/materials` - UI components and layouts
  - `@sa/scripts` - Development scripts and tools
  - `@sa/uno-preset` - UnoCSS preset configuration

### Source Code Structure
- `src/router/elegant/` - Auto-generated routing system using elegant-router
- `src/store/modules/` - Pinia stores (auth, route, tab, theme, app)
- `src/service/api/` - API service layer with axios
- `src/layouts/` - Layout components (base, blank) with modules
- `src/views/` - Page components organized by feature
- `src/components/` - Reusable components (common, custom, advanced)
- `src/hooks/` - Vue composition API hooks
- `src/utils/` - Utility functions and helpers

### Key Features
- **Automatic Route Generation**: Uses elegant-router to auto-generate routes from file structure
- **Multi-Layout System**: Base layout with header/sider/content, blank layout for auth pages
- **Theme System**: Dynamic theming with UnoCSS integration
- **Authentication**: JWT-based auth with role-based permissions
- **Internationalization**: Built-in i18n support with vue-i18n
- **State Management**: Pinia stores for app state, auth, routing, and tabs

### Configuration Files
- `vite.config.ts` - Vite configuration with plugins and proxy setup
- `uno.config.ts` - UnoCSS configuration and presets
- `eslint.config.js` - ESLint configuration using @soybeanjs/eslint-config
- `tsconfig.json` - TypeScript configuration

### Git Hooks
Pre-commit hook runs: `pnpm typecheck && pnpm lint && git diff --exit-code`

## Development Notes

- Development server runs on port 9527
- Preview server runs on port 9725
- Uses TypeScript strict mode
- Components should follow PascalCase naming
- Route generation is automatic - modify file structure to add routes
- Always run typecheck and lint before committing changes