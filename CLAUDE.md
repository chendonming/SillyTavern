# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```sh
# Start the server
node server.js
npm run start

# Initialize/fix config.yaml (adds missing defaults)
npm run init

# Lint all JS files (ESLint 8 — single quotes, 4-space indent, semicolons required)
npm run lint
npm run lint:fix

# Tests (run from tests/ directory)
cd tests && npm ci --ignore-scripts  # install test deps first
cd tests && node --experimental-vm-modules node_modules/jest/bin/jest.js --config jest.config.json  # unit tests
cd tests && npx playwright test      # E2E tests
cd tests && npm test                 # both unit + E2E

# Debug mode
npm run debug

# Plugin management
npm run plugins:update
npm run plugins:install
```

CI runs ESLint and unit tests on every PR (Node 24, Ubuntu). PRs target `staging`, not `release`.

## Project Structure

SillyTavern is a Node.js (>=20, ESM) server serving a single-page vanilla JS + jQuery frontend. No frontend framework.

### Server (`server.js` → `src/server-main.js`)
- Express 4 app with helmet, CORS, CSRF, cookie-session middleware
- Routes in `src/endpoints/` — each file is an Express Router for a domain (characters, chats, settings, backends, etc.)
- AI backend adapters live in `src/endpoints/backends/`:
  - `chat-completions.js` — OpenAI-compatible and specialty providers (Claude, Gemini, Cohere, Mistral, DeepSeek, xAI, etc.)
  - `text-completions.js` — legacy text completion APIs (KoboldCPP, Ooba, Ollama, etc.)
  - `kobold.js` — KoboldAI API
- Secrets stored per-user in `data/` via `src/endpoints/secrets.js` (SecretManager class)
- Prompt format converters in `src/prompt-converters.js`

### Frontend (`public/`)
- Single-page app served from `public/index.html` (~737K, all UI is in here)
- Client entry point: `public/scripts/script.js`
- Core modules (all in `public/scripts/`):
  - `openai.js` — API connection configuration, model management, chat completion request building
  - `secrets.js` — client-side secret key registry (must mirror `src/endpoints/secrets.js` + `public/scripts/constants.js`)
  - `events.js` — event emitter system (extensions hook into `event_types`)
  - `extensions.js` — extension loader for UI extensions in `public/scripts/extensions/`
  - `slash-commands.js` — slash command system
  - `macros.js` — template macro system for prompt construction
  - `power-user.js` — UI customization settings
  - `tokenizers.js` — LLM tokenizer selection/detection by provider
  - `reasoning.js` — thinking/reasoning content extraction per provider
  - `textgen-models.js` — shared model list management
  - `chats.js` — chat history management
  - `characters.js` — character card management
- Templates rendered client-side with Handlebars
- CSS uses LESS (compiled via Easy LESS VS Code extension)

### Config (`config.yaml`)
- Server settings: port (8000), SSL, auth, whitelist, CORS, rate limiting, backups

## Architecture Patterns

### Event System
Extensions communicate via event emitter (`event_types`). Key lifecycle events: `APP_READY`, `SETTINGS_LOADED`, `CHAT_CHANGED`, `MESSAGE_RECEIVED`, `GENERATION_ENDED`, etc.

### Extensions
UI extensions live in `public/scripts/extensions/`. Each extension has its own directory with a JS entry point. Loaded by `extensions.js`. Documentation: https://docs.sillytavern.app/for-contributors/writing-extensions/

### Server Plugins
Server-side plugins in `plugins/` directory (separate git repos). Loaded by `src/plugin-loader.js`. Documentation: https://docs.sillytavern.app/for-contributors/server-plugins

### Adding a New Chat Completion Provider
Requires changes in **8+ locations** — the canonical pattern is:
1. `src/constants.js` — add to `CHAT_COMPLETION_SOURCES`
2. `public/scripts/openai.js` — mirror in `chat_completion_sources`
3. `public/scripts/secrets.js` — add `SECRET_KEYS`, `FRIENDLY_NAMES`, `INPUT_MAP`
4. `src/endpoints/secrets.js` — add to server `SECRET_KEYS`
5. `public/index.html` — add dropdown `<option>` and form `<div>` with `data-source`
6. `public/scripts/openai.js` — add to `settingsToUpdate`, `default_settings`, `apiSourceConfig`, `toggleChatCompletionForms()`, `saveModelList()`
7. `src/endpoints/backends/chat-completions.js` — add API constant, `/status` route handler, `/generate` route handler
8. `src/prompt-converters.js` — if the provider uses a non-OpenAI message format

### Slash Commands
Registered via `registerSlashCommand()` in `public/scripts/slash-commands.js`. Each command has name, callback, aliases, help text, and interaction mode.

### Prompt Manager
`public/scripts/PromptManager.js` handles the complex prompt construction pipeline (system prompt, WI, character cards, chat history, macros, etc.).

### Type Checking
Uses JSDoc annotations with TypeScript (`jsconfig.json` enables `checkJs`). No `.ts` files.

## Contribution Guidelines
- Target `staging` branch for PRs
- Keep PRs under ~200 lines when possible
- Run `npm run lint` before committing (ESLint enforces single quotes, 4-space indent, semicolons)
- VS Code recommended with ESLint, EditorConfig, and Easy LESS extensions
