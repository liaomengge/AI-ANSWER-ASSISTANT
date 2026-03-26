# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development and verification

This repository is a Chrome Extension (Manifest V3). There is no `package.json`, build pipeline, test suite, or lint script in the repo.

Common verification workflow:

- Load the repo root as an unpacked extension in Chrome via `chrome://extensions/`
- Use the extension on a target questionnaire page and verify scan/start/stop flows from the side panel UI
- After changing built-in templates, reload the unpacked extension so updated JSON and content scripts are picked up
- Verify template matching in the page context with `window.siteMatcher.matchTemplate(window.location.href)`
- Verify template initialization in the page context with `window.templateManager.init()` if needed

## Architecture overview

The extension is built around a template-first scanning flow with AI fallback.

### Runtime structure

- `manifest.json` defines a Manifest V3 extension with:
  - `background.js` as the service worker
  - `popup.html` as the side panel entry
  - content scripts injected on all pages, including `modules/site-matcher.js`, `modules/template-manager.js`, `modules/scanner-enhanced.js`, and `content.js`
  - `templates/*.json` exposed as web-accessible resources for built-in site templates

### Main execution flow

1. The user opens the side panel (`popup.html` + `popup.js`) and triggers scan/start/stop actions.
2. `popup.js` ensures content scripts are present in the active tab and sends commands to `content.js`.
3. `content.js` initializes template support, tries `window.siteMatcher.matchTemplate(window.location.href)`, and prefers template-based scanning.
4. If a built-in or custom template matches, `modules/scanner-enhanced.js` parses questions directly from DOM selectors.
5. If template scanning is unavailable or fails, `content.js` falls back to AI-assisted page analysis.
6. For answering, `content.js` requests one question at a time from the background worker, then applies answers by clicking/filling DOM elements.

### Key modules

- `background.js`
  - Handles runtime messages such as AI calls, HTML analysis, and stats tracking
  - Normalizes OpenAI-compatible chat-completions requests
  - Contains outbound stats reporting logic

- `popup.js`
  - Controls the side panel UI
  - Manages model configuration and active model selection
  - Starts scan/answer flows and can programmatically inject content scripts

- `content.js`
  - Orchestrates scanning and answering on the active page
  - Uses template matching first, then AI fallback
  - Applies answers directly to page inputs

- `modules/site-matcher.js`
  - Registers templates and matches the current URL using `urlPatterns` / `urlRegex`

- `modules/template-manager.js`
  - Loads built-in templates from `templates/*.json`
  - Persists custom templates and template stats in `chrome.storage.sync`
  - Registers templates with `window.siteMatcher`

- `modules/scanner-enhanced.js`
  - Parses question blocks using template-provided selectors
  - Generates selectors used later for answer application

- `template-manager-ui.js`
  - Renders built-in/custom template information in the template management UI

## Template system

Built-in templates live in `templates/*.json` and follow the same schema:

- site metadata: `siteId`, `siteName`, `version`, `description`
- URL matching: `urlPatterns`, `urlRegex`
- DOM parsing rules under `selectors`
- scanner behavior flags under `features`
- implementation notes in `notes`

When adding a new built-in template:

1. Add the JSON file under `templates/`
2. Register it in `modules/template-manager.js`
3. Confirm its host/path patterns match the real site
4. Manually verify at least scan behavior on a real page after reloading the extension

## Practical notes

- README describes the extension as an AI-powered questionnaire/exam answering assistant; treat behavioral changes carefully because content scripts can automatically interact with third-party pages.
- The repo currently documents built-in support for 问卷星, 腾讯问卷, and 问卷网.
- Since there is no local test harness in the repo, manual browser verification is the primary validation path.