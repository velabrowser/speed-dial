# Vela speed dial

Home screen curation for the [Vela browser](https://github.com/fatkulnurk/com.ngambeg.vela).

Vela ships a bundled copy of every file in this repository, and also fetches it over the network so
the home screen can be updated without shipping an app release. This repository is the network copy.

The speed dial is **curated per language**: an Indonesian user gets Indonesian news and shopping
sites, an Arabic user gets Arabic ones. There is no single global list.

## How the app consumes these files

| Step | Behaviour |
|---|---|
| Requested URL | `https://raw.githubusercontent.com/velabrowser/speed-dial/main/home_shortcuts_<tag>.json` |
| `<tag>` | The active app language, as an Android resource qualifier |
| Refresh | On app start and on a language change, at most once per 12 hours per language |
| Cache | `files/home/home_shortcuts_<tag>.json` inside the app's private storage |
| Precedence | downloaded cache → bundled asset for that language → bundled English asset → built-in fallback |

Three consequences for anyone editing these files:

1. **Editing a file here is enough for online users.** The app prefers a downloaded copy over the
   one bundled in the APK, so a change lands on the next refresh. There is no release needed.
2. **It is not instant.** A device that refreshed less than 12 hours ago keeps its cached copy until
   the window passes.
3. **A broken file is ignored, not fatal.** The app validates before caching, so a malformed payload
   leaves the previous copy in place rather than blanking the home screen. The failure is silent —
   check the app's log or re-read the file if a change "did not apply".

## Files

| File | Language |
|---|---|
| `home_shortcuts_en.json` | English (base, and the fallback for every unsupported language) |
| `home_shortcuts_in.json` | Indonesian |
| `home_shortcuts_zh.json` | Chinese |
| `home_shortcuts_es.json` | Spanish |
| `home_shortcuts_fr.json` | French |
| `home_shortcuts_de.json` | German |
| `home_shortcuts_pt.json` | Portuguese |
| `home_shortcuts_ar.json` | Arabic |
| `home_shortcuts_ja.json` | Japanese |
| `home_shortcuts_ru.json` | Russian |
| `home_shortcuts_hi.json` | Hindi |
| `home_shortcuts_vi.json` | Vietnamese |

The tag is the Android resource qualifier, so Indonesian is `in`, not `id`. The app accepts `id`
from the device and maps it to `in`, but the file name must use `in`.

`home_shortcuts.json` is **legacy** and frozen. It is the single global list that shipped before the
per-language split, kept only so that app builds older than that change still have something to
fetch. Nothing in the current app reads it. Do not edit it.

## Format

```json
{
  "schemaVersion": 2,
  "locale": "in",
  "speedDial": [
    { "name": "Google", "url": "https://www.google.com" },
    { "name": "YouTube", "url": "https://www.youtube.com" },
    { "name": "WhatsApp", "url": "https://web.whatsapp.com" },
    { "name": "Tokopedia", "url": "https://www.tokopedia.com" }
  ],
  "recommended": { "name": "Kompas", "url": "https://www.kompas.com" },
  "categories": [
    {
      "id": "ai",
      "name": "AI",
      "items": [
        { "name": "ChatGPT", "url": "https://chatgpt.com" }
      ]
    }
  ]
}
```

| Field | Required | Notes |
|---|---|---|
| `schemaVersion` | yes | `2`. Bump only with a coordinated app change. |
| `locale` | yes | Must equal the `<tag>` in the file name. The app rejects a mismatch. |
| `speedDial` | yes | Exactly 4 entries. Rendered as the Top Sites row. |
| `recommended` | recommended | One entry. Must not duplicate a top site. |
| `categories[].id` | yes | Stable identifier, same set in every file. Not rendered. |
| `categories[].name` | yes | Rendered. Translated per language. |
| `categories[].items` | yes | At least 8 entries. |

### Enforced at runtime by the app

- Every `url` must start with `https://`. Plain `http://`, `javascript:` and relative URLs are
  dropped silently.
- Every `name` must be non-empty.
- A payload whose `locale` does not match the requested file is rejected outright, so a copy-paste
  mistake cannot serve the Arabic list to Indonesian users.
- A payload with **no** `locale` field is accepted, which is what keeps the legacy flat format
  loadable.

Silently means exactly that: a dropped entry produces a shorter grid, not an error.

### Enforced by the app's test suite

These are checked by `HomeShortcutsAssetTest` in the app repository, which parses the bundled copies
of these files. They will not catch a mistake made only here, but they will catch it the moment the
same file is bundled into a release.

- Exactly 4 `speedDial` entries.
- Exactly the category ids `ai`, `news`, `shopping`, `finance`, `learn`, in that order.
- At least 8 items per category.
- `recommended` present and not duplicating a top site.
- A non-English file whose top sites are identical to the English ones.

### Editorial guidance

- **`speedDial` has exactly four slots** because the home grid has four columns. Choose the four
  sites a first-time user in that market is most likely to want, in that order.
- **Category order matters.** A phone shows the first **four** categories on the home screen; the
  rest sit behind "See all". Put the categories that market cares about first.
- **`id` must stay identical across files.** Only `name` is translated. Keeping the ids aligned is
  what lets the app's test suite detect a curation that was never actually localised.
- **At least 8 sites per category**, so two preview rows are full on a wide screen.
- **`recommended` must not repeat a top site**, otherwise the same destination appears twice on one
  screen.

## Adding a language

A new language needs changes in two places, and it only works if both land.

1. In this repository, add `home_shortcuts_<tag>.json`.
2. In the [app repository](https://github.com/fatkulnurk/com.ngambeg.vela), add the locale: the
   `AppLanguage` enum entry, the `values-<tag>/` string files, the `locales_config.xml` entry, and a
   bundled copy of the same JSON at `app/src/main/assets/home/home_shortcuts_<tag>.json`.

The app's `HomeShortcutsAssetTest` fails if a supported language has no bundled curation file, so
the two sides cannot drift apart unnoticed.

## Editing guidelines

- Keep brand names in their usual local form (`Tokopedia`, `VnExpress`, `Yahoo! JAPAN`); do not
  translate them.
- Translate category `name` naturally, not literally.
- Prefer canonical root domains over deep links, which rot faster.
- Avoid sites that require a login to show anything useful, and sites that are region-locked in the
  market the file targets.
- Verify a URL resolves before committing it. A typo that still parses as `https://` will pass every
  check and simply produce a broken tile.

## Licensing and content

The files contain only site names and URLs. All listed names and marks belong to their respective
owners, and inclusion is not an endorsement.
