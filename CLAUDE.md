# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview
Weeklog is a minimalist weekly journaling app for founders built with React, Firebase, and TanStack Router. It features typeform-style prompts, real-time AI feedback, searchable timeline, and optional public sharing. The project uses a Firebase Functions backend with Python and a React SPA frontend.

## Development Commands

### Frontend Development
- `npm run dev` or `npm run start` - Start development server on port 3000
- `npm run build` - Build for production with Vite and TypeScript compilation
- `npm run serve` - Preview production build
- `npm run test` - Run Vitest tests
- `npm run lint` - Run ESLint
- `npm run format` - Run Prettier 
- `npm run check` - Format with Prettier and fix ESLint errors

### Firebase Development
- Functions are written in Python 3.13 located in `functions/`
- Use Firebase emulator suite for local development
- Configuration in `firebase.json` includes Firestore, Functions, Hosting, and Auth emulators

### Shadcn Components
Add new UI components using: `pnpx shadcn@latest add button`

## Architecture

### Frontend Stack
- **Framework**: React 19 + Vite
- **Routing**: TanStack Router with code-based routing (see `src/main.tsx`)
- **Styling**: Tailwind CSS v4 with Shadcn UI components
- **State Management**: TanStack Query for server state, TanStack Form for forms
- **Testing**: Vitest with jsdom environment
- **Build**: TypeScript with strict mode

### Backend Stack  
- **Functions**: Cloud Functions for Firebase (Python 3.13, HTTP 2nd gen)
- **Database**: Firestore with security rules
- **Authentication**: Firebase Auth
- **Hosting**: Firebase Hosting with SPA rewrites

### Key Directories
- `src/components/` - Reusable React components
- `src/routes/` - Route components (demo files can be deleted)
- `src/components/ui/` - Shadcn UI components
- `src/hooks/` - Custom React hooks
- `src/lib/` - Utility functions
- `functions/` - Firebase Functions (Python)

### Routing Architecture
Uses TanStack Router with code-based routing defined in `src/main.tsx`. Routes are created with `createRoute()` and added to the route tree. The root route includes a Header component and TanStack Router devtools.

### API Architecture
Backend follows the design in `design_document.md`:
- Functions serve HTTP endpoints under `/api/**`
- LLM integrations with BYOK (Bring Your Own Key) support
- Server-side encryption with Cloud KMS
- Streaming responses via SSE for real-time feedback

## Environment & Configuration
- TypeScript path aliases: `@/*` maps to `./src/*`
- Environment variables managed via T3Env in `src/env.ts`
- Firestore region: `asia-southeast1`
- ESLint uses TanStack's configuration
- Prettier for code formatting

## Testing
- Uses Vitest with jsdom for React component testing
- Test files: `**/*.test.tsx`
- Global test setup includes React Testing Library

## Demo Files
Files prefixed with `demo.*` are example implementations and can be safely deleted.