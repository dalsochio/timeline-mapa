# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Vue 3 + Ionic mobile application that creates a timeline map using Mapbox GL JS. The app tracks user location in real-time and stores the location history locally using Ionic Storage.

## Key Architecture

- **Framework**: Vue 3 with Composition API (`<script setup>`)
- **Mobile Framework**: Ionic Vue 8
- **Build Tool**: Vite
- **Map Library**: Mapbox GL JS
- **Storage**: Ionic Storage for local data persistence
- **Mobile**: Capacitor for native mobile features

## Development Commands

```bash
# Development server
npm run dev

# Build for production
npm run build

# Type checking
vue-tsc

# Linting
npm run lint

# Unit tests
npm run test:unit

# E2E tests
npm run test:e2e

# Preview production build
npm run preview
```

## Project Structure

- `src/App.vue` - Root component with ion-router-outlet
- `src/main.ts` - App initialization with Ionic Vue setup
- `src/router/index.ts` - Vue Router configuration with tab-based navigation
- `src/views/` - Page components (Tab1Page contains the main map functionality)
- `src/components/` - Reusable components
- `src/theme/` - CSS variables and theming

## Key Features

### Map Implementation (Tab1Page.vue)
- Uses Mapbox GL JS with personal access token
- Geolocation tracking every 2 seconds
- Stores location timeline in Ionic Storage
- Renders user location with marker
- Includes route layer for potential path visualization

### Data Storage
- Uses Ionic Storage for local persistence
- Stores location coordinates as JSON array in 'timeline' key
- Automatic initialization of empty timeline on first run

## Development Notes

- **Mapbox Token**: Currently hardcoded in Tab1Page.vue (consider moving to environment variables)
- **Location Permissions**: App requests geolocation permissions on load
- **Tab Navigation**: Uses Ionic's tab-based navigation pattern
- **TypeScript**: Full TypeScript support with proper configuration
- **Testing**: Configured for both unit tests (Vitest) and E2E tests (Cypress)

## Build & Deploy

The app is configured for both web and mobile deployment:
- Web build outputs to `dist/`
- Capacitor configuration in `capacitor.config.ts`
- Mobile builds use Capacitor CLI commands

## Testing

- Unit tests use Vitest with jsdom environment
- E2E tests use Cypress
- Test files located in `tests/` directory