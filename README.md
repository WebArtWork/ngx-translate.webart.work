# @wawjs/ngx-translate

Angular language and runtime translation package from Web Art Work.

`ngx-translate` is SSR-safe and focused on two features:

- `LanguageService` plus `provideLanguage(...)` for active language state, defaults, registry management, and optional persistence
- `TranslateService` plus `provideTranslate(...)` and `TranslateDirective` for signal-based runtime translations

## License

[MIT](LICENSE)

## Installation

```bash
npm i --save @wawjs/ngx-translate
```

## Usage

```ts
import { provideTranslate } from '@wawjs/ngx-translate';

export const appConfig = {
	providers: [
		provideTranslate({
			defaultLanguage: 'en',
			language: 'en',
			languages: ['en', 'de', 'fr'],
			folder: '/i18n/',
			folders: ['/i18n/articles/', '/i18n/common/'],
		}),
	],
};
```

## Language-Only Bootstrap

```ts
import { provideLanguage } from '@wawjs/ngx-translate';

export const appConfig = {
	providers: [
		provideLanguage({
			defaultLanguage: 'en',
			languages: ['en', 'de', 'fr'],
		}),
	],
};
```

## Auto-Detecting the Initial Language

Set `detectLanguage` to pick the initial language from device/browser signals
when there is no explicit `language` and no persisted language yet. A detected
code is only used if it matches one of the configured `languages`.

```ts
provideTranslate({
	defaultLanguage: 'en',
	languages: ['en', 'de', 'fr', 'es', 'pt'],
	detectLanguage: true, // uses default order: ['browser', 'timezone']
});
```

`true` runs the built-in detectors in order — `'browser'` (`navigator.languages`
/ `navigator.language`) then `'timezone'` (`Intl.DateTimeFormat` resolved
timezone, mapped through a coarse built-in table) — and stops at the first
match found among the configured languages. Pass an explicit array to control
order, drop a detector, or add a custom one:

```ts
import { detectTimezoneLanguage, provideTranslate } from '@wawjs/ngx-translate';

provideTranslate({
	defaultLanguage: 'en',
	languages: ['en', 'de', 'fr'],
	detectLanguage: [
		'browser',
		() => detectTimezoneLanguage({ 'Europe/Zurich': 'de', 'Europe/Paris': 'fr' }),
	],
});
```

Detection only runs in the browser (SSR-safe no-op on the server) and never
overrides an explicit `language` or a previously persisted selection.

## Available Features

| Name                                                                                        | Description                                                             |
| ------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| `LanguageService`                                                                           | Active language state, registry management, validation, and persistence |
| `TranslateService`                                                                          | Signal-based runtime translation loading and updates                    |
| `TranslateDirective`                                                                        | Directive that translates explicit text, inline text, and attributes    |
| `provideLanguage`, `provideTranslate`                                                       | Environment providers for bootstrap                                     |
| `Language`, `LanguageInput`, `ProvideLanguageConfig`, `ProvideTranslateConfig`, `Translate` | Public types                                                            |

## Language Service

`LanguageService` manages the active language, default language, available languages, and optional persistence.

### Methods

- `init(config?)`
- `language(): string`
- `defaultLanguage(): string`
- `allLanguages(): Language[]`
- `languages(): Language[]`
- `getLanguage(code: string)`
- `hasLanguage(code: string): boolean`
- `setLanguages(languages, syncCurrentLanguage?)`
- `setLanguage(code: string): Promise<boolean>`

## Translate Service

`TranslateService` manages runtime translations and updates translation signals reactively.

### Features

- inline translations through config
- file loading from `/i18n/{language}.json`
- optional multi-folder loading such as `/i18n/common/{language}.json` and `/i18n/articles/{language}.json`
- object-map and compact array translation payloads
- signal-based translation updates
- variable interpolation with `{{name}}` placeholders
- source-text fallback when no translation exists
- optional persisted language selection

### Methods

- `init(config?)`
- `language()`
- `defaultLanguage()`
- `setLanguage(language: string)`
- `loadTranslations(language: string)`
- `loadExtraTranslations(paths: string[], options?)`
- `loadExtraTranslation(path: string, options?)`
- `translate(text: string)`
- `interpolate(text: string, vars?)`
- `setMany(translations: Translate[])`
- `setOne(translation: Translate)`
- `get()`

Example:

```ts
import { inject } from '@angular/core';
import { TranslateService } from '@wawjs/ngx-translate';

const _translateService = inject(TranslateService);

const title = _translateService.translate('Create project');
const message = _translateService.interpolate('Phrase has counter {{count}}', {
	count: 5,
});

void _translateService.setLanguage('de');
```

## Translation Payloads

Object JSON keeps the source text and translated text together.

```json
{
	"hello": "привіт",
	"now": "зараз"
}
```

Array JSON is also supported when the default language file is the root source
array. For example, with `defaultLanguage: 'en'`:

`/i18n/en.json`

```json
["hello", "now"]
```

`/i18n/ua.json`

```json
["привіт", "зараз"]
```

Then this template:

```html
<span translate>hello</span>
```

renders as `привіт` when the active language is `ua`.

Array payloads are index-based. Keep the default language array and translated
language arrays in the same order; missing translated items fall back to the
source text and length mismatches are reported with a console warning.

## Route/Page Extra JSON Bundles

You can merge additional JSON translation files for a specific page without replacing the base language file.

```ts
import { ActivatedRoute } from '@angular/router';
import { inject } from '@angular/core';
import { TranslateService } from '@wawjs/ngx-translate';

const _route = inject(ActivatedRoute);
const _translateService = inject(TranslateService);

const slug = _route.snapshot.paramMap.get('slug') || '';

await _translateService.loadExtraTranslations(['/i18n/articles/', `/i18n/article/${slug}`]);

await _translateService.loadExtraTranslation('/i18n/articles/');
```

Notes:

- For folder-style paths, `/{language}.json` is appended automatically.
- `{language}` and `:language` placeholders are replaced automatically.
- Existing translations are kept by default and then overridden by later files.
- Array extra payloads are paired with the matching default-language array from the same URL pattern.
- Extra translation URLs are cached per language and are not fetched again on repeat calls.
- Pass `replace: true` to replace cached translations for that language with only the loaded files.
- Pass `forceReload: true` to refetch URLs that were already cached.

## Translate Directive

```html
<h1 translate>Create project</h1>
<button translate>Save</button>
<h2 translate="phrase" [vars]="{ count: 5 }"></h2>
<span
	[translate]="{
		title: 'This is hello world title',
		content: 'hello world',
		ariaLabel: 'hello world label',
	}"
></span>
```

If the active translation contains placeholders such as
`Phrase has counter {{count}}`, bind `[vars]="{ count: 5 }"` to render
`Phrase has counter 5`. The same variables are applied to translated host text
and translated attributes. Missing variables keep their original `{{name}}`
placeholder so incomplete translation data stays visible.

Use `content` for the host text. Other object keys are translated and written as
attributes; camelCase keys such as `ariaLabel` are written as dash-case
attributes such as `aria-label`.

## Notes

- Language persistence uses guarded browser storage access and is skipped during SSR.
- Translation payloads can come from inline config, `folder`, `folders`, and route/page extra JSON files.
- File payloads can be object maps or compact string arrays paired by index with the default language array.
- Missing translations safely fall back to the source text.

## AI Coding Agents

This package includes [AI.md](AI.md) with copyable instructions for Codex, Claude Code, Cursor, and other coding agents.

Copy this into the consuming project's `AGENTS.md`, `CLAUDE.md`, or equivalent file:

```md
- This Angular project uses `@wawjs/ngx-translate` for runtime language state and signal-based translations.
- Import public APIs from `@wawjs/ngx-translate`.
- Prefer bootstrapping with `provideTranslate({...})` when translations are needed, or `provideLanguage({...})` for language-only state.
- Register app translations with `provideTranslate(...)` and use `TranslateService` or `TranslateDirective` instead of creating a parallel translation bootstrap path.
- Translation JSON can use object maps or compact string arrays; array payloads must keep the same order as the default language array.
- Prefer `LanguageService` for active language state, validation, defaults, and persistence before adding app-specific language utilities.
- Keep SSR-safe behavior intact. Do not add unguarded direct access to browser storage for language persistence when the package already handles it.
- Use `detectLanguage: true` (or an explicit `LanguageDetector[]`) instead of hand-rolling `navigator.language`/timezone detection; it only applies when there is no explicit `language` and no persisted language.
```
