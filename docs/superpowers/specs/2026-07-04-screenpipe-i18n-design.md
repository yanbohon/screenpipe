# Screenpipe Desktop i18n Design

## Goal

Add a maintainable first-pass internationalization layer to the Screenpipe desktop app so users can switch the UI between English and Simplified Chinese. The first release should translate the primary app shell, settings navigation, and the most frequently used settings controls without attempting to localize every string in the codebase.

## Context

The desktop UI currently has no i18n dependency or locale provider. User-facing strings are spread across more than 500 TypeScript and TSX files under `apps/screenpipe-app-tauri/app`, `apps/screenpipe-app-tauri/components`, and `apps/screenpipe-app-tauri/lib`.

There is already a `languages` setting and `lib/language.ts`, but that setting controls transcription language hints. It must remain separate from UI language selection. Reusing it for interface language would mix speech recognition configuration with app display preferences and would create confusing behavior.

The app is a Tauri desktop app using Next.js static output. Locale routing, middleware-based negotiation, or server-side translation loading would add avoidable build and runtime risk. A client-side provider with static dictionaries is enough for the first pass.

## Scope

The first implementation covers:

- A persistent UI language preference with two supported locales: `en` and `zh-CN`.
- A lightweight translation provider available to all React app pages.
- Static English and Simplified Chinese message dictionaries.
- A language selector in Settings, preferably in the Display section because it is a UI presentation preference.
- Translated primary Home navigation labels:
  - Home / chat affordances that are visible in the shell.
  - Pipes.
  - Timeline.
  - Meetings.
  - Brain.
  - Connections.
  - Settings.
  - Help.
- Translated Settings navigation:
  - Group labels: App, Data & Privacy, Account.
  - Section labels: Display, General, AI models, Recording, Shortcuts, Notifications, Usage, Privacy, Storage, Speakers, Team, Account, Get free month.
  - Back to app.
  - Settings search placeholder and basic empty/loading states.
- Translated high-frequency settings text in Display and General:
  - Section descriptions.
  - Language selector label and option labels.
  - Theme / display density / timeline visibility controls where present.
  - Auto-start, auto-update, check for updates, enhanced AI, onboarding reset, and related button/toast copy in General.
- English fallback for any missing key.

## Non-Goals

The first implementation does not translate:

- User data, captured screen/audio content, OCR output, memories, notes, chat messages, or model output.
- Pipe names, pipe descriptions, generated pipe content, or content coming from remote registries.
- Prompt templates, skill markdown, API payloads, or strings that are sent to models as behavior instructions.
- Raw backend errors. These may be wrapped later, but the first pass should not hide diagnostic detail.
- Every settings subsection. Deep settings pages can migrate incrementally after the i18n layer exists.
- Locale-specific date, time, number, or plural formatting beyond simple string interpolation.
- Route-based locale URLs such as `/zh-CN/settings`.

## Recommended Approach

Use a small in-repo i18n layer instead of adding `next-intl`, `react-i18next`, or route middleware.

Reasons:

- No new dependency or lockfile churn for the first pass.
- No change to Next.js routing or Tauri static export behavior.
- Lower merge conflict risk when syncing future upstream Screenpipe changes.
- Translation calls can be introduced incrementally in the files already being touched.
- The first-pass locale set is small and does not need async namespace loading.

## Architecture

Create a focused i18n module under `apps/screenpipe-app-tauri/lib/i18n/`.

Expected files:

- `types.ts`
  - Defines `Locale = "en" | "zh-CN"`.
  - Defines `DEFAULT_LOCALE = "en"`.
  - Defines `SUPPORTED_LOCALES`.
  - Exposes locale validation helpers.
- `messages/en.ts`
  - English source messages.
- `messages/zh-CN.ts`
  - Simplified Chinese translations.
- `messages/index.ts`
  - Exports the locale-to-message map.
  - Exports the inferred `MessageKey` type from English messages.
- `format.ts`
  - Looks up a message by key.
  - Falls back to English when the active locale is missing the key.
  - Supports simple interpolation with named values, for example `{version}`.
- `provider.tsx`
  - Defines `I18nProvider`.
  - Defines `useI18n()`.
  - Reads and writes `settings.uiLanguage` through `useSettings()`.
  - Sets `document.documentElement.lang` to the active locale.

The provider should be mounted in `app/providers.tsx` inside `SettingsProvider` so it can read and persist the preference:

```tsx
<SettingsProvider>
  <I18nProvider>
    <AuthGuard>
      ...
    </AuthGuard>
  </I18nProvider>
</SettingsProvider>
```

Components use:

```tsx
const { t, locale, setLocale } = useI18n();
```

Message usage should prefer stable semantic keys:

```tsx
t("settings.general.autoStart.title")
t("settings.general.autoStart.description")
```

Dynamic text should use named interpolation:

```tsx
t("settings.general.updateReady.description", { version: pending.version })
```

## Persistence

Add `uiLanguage?: Locale` to the `Settings` type in `lib/hooks/use-settings.tsx`.

Default behavior:

- New installs default to English.
- Existing installs with no `uiLanguage` keep English.
- Invalid persisted values fall back to English without throwing.
- Changing the language writes to the existing settings store via `updateSettings({ uiLanguage: nextLocale })`.

Do not reuse the existing `languages` array. That array remains the transcription language configuration.

## UI Design

Add a compact language selector in Settings -> Display. The control should use the existing settings UI style and primitives. A `Select` is preferred because this is a small option set:

- Label: `Language`
- Description: `Choose the app interface language.`
- Options:
  - `English`
  - `简体中文`

When Chinese is active, the label and description become:

- `语言`
- `选择应用界面语言。`

The language change should apply immediately without requiring restart.

## Settings Search Compatibility

The existing settings search has a text-based field lookup path. Its own comment notes that localization can break field scrolling if rendered headings no longer match English search index labels.

For the first pass:

- Translate settings section navigation and top-level search UI.
- Keep internal search field IDs stable.
- When translating field headings that are searchable, add or use a stable `anchor` field in `SettingsField`.
- Update `scrollToSettingsField` to prefer `anchor` when present and fall back to the current text lookup.
- Search result display can use translated labels when a field has an i18n key, but English keywords should remain available so both English and Chinese queries can find common settings.

This avoids coupling search behavior to the currently displayed language.

## Error Handling

The i18n layer should not throw during rendering because a missing translation key should not break the desktop app.

Rules:

- Missing active-locale key falls back to English.
- Missing English key returns the key string itself and logs a development warning.
- Missing interpolation value leaves the placeholder visible in development and replaces it with an empty string in production.
- Invalid locale values normalize to `en`.

## Testing Strategy

Use test-first implementation for the i18n utilities and focused component behavior.

Unit tests:

- `formatMessage` returns English text for English locale.
- `formatMessage` returns Chinese text for Chinese locale.
- Missing Chinese key falls back to English.
- Missing key returns the key.
- Interpolation replaces named placeholders.
- Invalid locale normalizes to English.

Component tests:

- The i18n provider reads `settings.uiLanguage`.
- Calling `setLocale("zh-CN")` persists `uiLanguage`.
- A translated Home navigation label changes when locale changes.
- The Settings language selector shows the current language and persists changes.

Verification commands:

```bash
cd apps/screenpipe-app-tauri
bunx vitest run --config vitest.config.ts lib/i18n components/settings app/settings
bunx tsc --noEmit
```

The full test suite is large and has known exclusions. Focused tests plus typecheck are the required first-pass gate.

## Rollout Plan

1. Add i18n utility tests first.
2. Implement the static dictionaries and formatter.
3. Add `uiLanguage` to settings and wire `I18nProvider`.
4. Add the Settings -> Display language selector.
5. Migrate Home shell navigation labels.
6. Migrate Settings navigation and search shell labels.
7. Migrate high-frequency Display and General settings text.
8. Run focused tests and typecheck.
9. Commit the i18n framework and first translated surface as one feature commit.

## Maintenance Notes

When upstream Screenpipe changes add new visible text, the fork can continue to render English until those strings are migrated. New translations should be added close to the component work rather than via broad automatic string replacement.

Avoid translating identifiers, enum values, API request fields, model IDs, provider names, telemetry event names, localStorage keys, or test IDs.

The first pass intentionally keeps English as the source locale. This makes future upstream merges easier because untranslated upstream copy remains valid.

## Open Follow-Ups

After the first pass is merged and verified, later work can expand translation coverage to:

- Account and subscription screens.
- Recording settings.
- AI model configuration.
- Privacy and storage sections.
- Meeting notes UI.
- Pipe store UI.
- Chat empty states and suggestion chips.
