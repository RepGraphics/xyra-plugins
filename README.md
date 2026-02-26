# Plugin System

XyraPanel has an experimental plugin system.

## What A Plugin Can Do

- Load server-side logic at startup (`entry.server`).
- Register a Nuxt module at startup (`entry.module`).
- Register runtime hooks (`panel:ready`, `request:before`, `request:after`).
- Extend the frontend using a Nuxt layer (`entry.nuxtLayer`).
- Add admin navigation items (`contributions.adminNavigation`).
- Add client dashboard navigation items (`contributions.dashboardNavigation`).
- Add server sidebar navigation items (`contributions.serverNavigation`).
- Inject components into named UI slot outlets (`contributions.uiSlots`).
- Use per-plugin render scope settings in Admin (`global` or selected eggs).

## Quick Start

### Install via CLI (recommended)

Use the built-in installer instead of manually copying files:

```bash
# list installed plugins
xyra plugins list

# install from local folder
xyra plugins install ./my-plugin

# install from git
xyra plugins install https://github.com/acme/xyra-plugin-example

# if plugin.json is nested in the repo
xyra plugins install https://github.com/acme/plugin-monorepo --manifest packages/my-plugin

# remove plugin
xyra plugins remove my-plugin
```

Notes:

- The installer expects a `plugin.json` manifest in the source root (or the folder passed via `--manifest`).
- Use `--force` to overwrite an existing plugin directory.
- Restart the panel process after install/remove so Nuxt layer changes are picked up.

### Install from Admin UI

Open `/admin/plugins` and use **Install Plugin**:

- **From server path**: install from an existing directory on the panel host (or from a `plugin.json` path).
- **From archive upload**: upload `.zip`, `.tar`, `.tar.gz`, or `.tgz` and install from the extracted contents.
- **manifestPath** (optional): use this when `plugin.json` is nested inside a monorepo/archive.
- **force overwrite**: replace an existing plugin that has the same `id`.

Notes:

- Uploaded archives are extracted to a temporary directory first, then copied into `extensions/<pluginId>/` (or `XYRA_PLUGIN_INSTALL_DIR` if configured).
- `.zip` archives are unzipped; `.tar`, `.tar.gz`, and `.tgz` archives are untarred.
- Source and archive installs reject symlinks/special file entries and unsafe traversal paths.
- If a plugin includes `entry.nuxtLayer` or `entry.module`, the installer now attempts to apply changes automatically:
  - Dev mode: triggers a Nuxt dev reload.
  - Production: can schedule a process restart when a process manager is detected or when `XYRA_PLUGIN_AUTO_RESTART=true`.
  - If auto-apply is disabled, restart manually.
- Each plugin now has a default **global** render scope in `/admin/plugins`.
  - Switch to **Specific eggs** to limit where server-side plugin UI renders.
  - This scope controls server plugin contributions (for example server sidebar items).

### 1. Create a plugin folder

```text
extensions/
  acme-tools/
    plugin.json
    dist/
      server.mjs
    modules/
      xyra.ts
    ui/
      app/
        components/
        pages/
```

### 2. Add `plugin.json`

```json
{
  "id": "acme-tools",
  "name": "Acme Tools",
  "version": "1.0.0",
  "description": "Adds internal tooling",
  "enabled": true,
  "entry": {
    "server": "./dist/server.mjs",
    "module": "./modules/xyra.ts",
    "nuxtLayer": "./ui"
  },
  "contributions": {
    "adminNavigation": [
      {
        "id": "acme-tools-dashboard",
        "label": "Acme Tools",
        "to": "/admin/acme-tools",
        "icon": "i-lucide-puzzle",
        "order": 450,
        "permission": "admin.settings.read"
      }
    ],
    "dashboardNavigation": [
      {
        "id": "acme-tools-client",
        "label": "Acme Tools",
        "to": "/acme-tools",
        "icon": "i-lucide-puzzle",
        "order": 320
      }
    ],
    "serverNavigation": [
      {
        "id": "acme-tools-server",
        "label": "Acme Tools",
        "to": "acme-tools",
        "icon": "i-lucide-puzzle",
        "order": 450
      }
    ],
    "uiSlots": [
      {
        "slot": "admin.layout.after-content",
        "component": "AcmeAdminBanner",
        "order": 100
      }
    ]
  }
}
```

### 3. Add a server entry (optional)

`dist/server.mjs`:

```js
export default {
  async setup(ctx) {
    ctx.log.info('acme-tools loaded', { version: ctx.plugin.version });
  },
  hooks: {
    'panel:ready': async (payload, ctx) => {
      ctx.log.info('panel ready', payload);
    },
    'request:after': async (payload, ctx) => {
      if (payload.statusCode >= 500) {
        ctx.log.warn('request failed', { statusCode: payload.statusCode });
      }
    },
  },
};
```

### 4. Add frontend files (optional)

If using `entry.nuxtLayer`, place components/pages in the layer:

```text
extensions/acme-tools/ui/app/components/AcmeAdminBanner.vue
extensions/acme-tools/ui/app/pages/admin/acme-tools.vue
```

### 5. Add a Nuxt module entry (optional)

`modules/xyra.ts`:

```ts
import { defineNuxtModule, addPlugin, createResolver } from '@nuxt/kit';

export default defineNuxtModule({
  meta: {
    name: 'acme-tools',
  },
  setup() {
    const { resolve } = createResolver(import.meta.url);
    addPlugin(resolve('../runtime/plugins/acme-tools'));
  },
});
```

Nuxt module entries are loaded into the panel `modules` config automatically when `entry.module` is set.

### 6. Restart and verify

- Restart `pnpm dev` after creating/changing plugin manifests or layer paths.
- Check runtime state at `/admin/plugins`.

## Manifest Reference

`plugin.json` fields:

| Field                           | Type      | Required | Notes                                              |
| ------------------------------- | --------- | -------- | -------------------------------------------------- |
| `id`                            | `string`  | yes      | Must match `^[a-z0-9][a-z0-9._-]*$` and be unique. |
| `name`                          | `string`  | yes      | Display name in admin/runtime summaries.           |
| `version`                       | `string`  | yes      | Free-form version string.                          |
| `description`                   | `string`  | no       | Optional metadata.                                 |
| `author`                        | `string`  | no       | Optional metadata.                                 |
| `website`                       | `string`  | no       | Optional metadata.                                 |
| `enabled`                       | `boolean` | no       | Defaults to enabled when omitted.                  |
| `entry.server`                  | `string`  | no       | Relative file path inside plugin dir.              |
| `entry.module`                  | `string`  | no       | Relative Nuxt module file path inside plugin dir.  |
| `entry.nuxtLayer`               | `string`  | no       | Relative directory path inside plugin dir.         |
| `contributions.adminNavigation` | `array`   | no       | Extra admin nav entries.                           |
| `contributions.dashboardNavigation` | `array`   | no       | Extra client dashboard sidebar entries.            |
| `contributions.serverNavigation` | `array`   | no       | Extra server sidebar entries.                      |
| `contributions.uiSlots`         | `array`   | no       | UI slot injections.                                |

Validation and safety rules:

- `entry.server`, `entry.module`, and `entry.nuxtLayer` must be relative paths.
- Paths cannot escape the plugin directory.
- Missing paths are reported as discovery errors.
- `entry.module` must resolve to a file.
- `entry.nuxtLayer` must resolve to a directory.

## Runtime API

A server plugin can export:

- A default function `(ctx) => {}`.
- A default object with `setup` and `hooks`.

Context shape:

```ts
{
  plugin,      // resolved manifest + paths
  nitroApp,    // Nitro app instance (opaque)
  log,         // info/warn/error/debug prefixed logger
  emitHook,    // emit a plugin hook manually
}
```

Hook handler shape:

```ts
async function hook(payload, ctx) {}
```

Built-in emitted hooks:

- `panel:ready`
  - payload: `{ loaded: number, failed: number }`
- `request:before`
  - payload: `{ event }`
- `request:after`
  - payload: `{ event, statusCode: number }`

## Frontend Contributions

### Admin navigation

Each contribution entry supports:

- `id`, `label`, `to`, `icon`, `order`
- `permission`: string or string array (array means any-of)
- `children`: nested items

Notes:

- Runtime prefixes ids to avoid collisions: `plugin:<pluginId>:<id>`.
- Admin users bypass permission filtering.

### Server navigation

`contributions.serverNavigation` uses the same item shape as admin navigation (`id`, `label`, `to`, `icon`, `order`, optional `permission` and `children`).

Path handling in the server sidebar:

- Absolute paths are used as-is (example: `/server/{id}/players`).
- Relative paths are resolved under the current server (example: `players` -> `/server/<id>/players`).
- Dynamic server id placeholders are supported: `{id}`, `[id]`, and `:id`.

### Dashboard navigation

`contributions.dashboardNavigation` uses the same item shape as admin navigation (`id`, `label`, `to`, `icon`, `order`, optional `permission` and `children`).

Path handling in the dashboard sidebar:

- Absolute paths are supported directly (example: `/acme-tools`).
- Relative paths are normalized to root paths (example: `acme-tools` -> `/acme-tools`).

### UI slots

Each contribution entry supports:

- `slot` (string, required)
- `component` (string, required)
- `order` (number, optional, default `500`)
- `permission` (string or string array)
- `props` (object)

Current slot outlets:

- `app.wrapper.before`
- `app.wrapper.after`
- `admin.wrapper.before`
- `admin.wrapper.after`
- `admin.layout.before-navbar`
- `admin.layout.after-navbar`
- `admin.layout.before-content`
- `admin.layout.after-content`
- `admin.dashboard.before-content`
- `admin.dashboard.after-content`
- `client.wrapper.before`
- `client.wrapper.after`
- `client.layout.before-navbar`
- `client.layout.after-navbar`
- `client.layout.before-content`
- `client.layout.after-content`
- `client.dashboard.before-content`
- `client.dashboard.after-content`
- `server.wrapper.before`
- `server.wrapper.after`
- `server.layout.before-navbar`
- `server.layout.after-navbar`
- `server.layout.before-content`
- `server.layout.after-content`
- `server.console.power-buttons.before`
- `server.console.power-buttons.after`
- `server.console.between-terminal-and-stats`
- `server.console.after-stats`
- `server.console.stats-card.before`
- `server.console.stats-card.after`
- `server.activity.table.before`
- `server.activity.table.after`
- `server.files.create-buttons.before`
- `server.files.create-buttons.after`
- `server.startup.command.before`
- `server.settings.top`
- `server.settings.bottom`

The `component` must be resolvable by the Nuxt app (typically via the plugin Nuxt layer).

Wrapper slots are intended for global concerns. A plugin component rendered in wrapper slots can call `useHead()` to register layout-level CSS/JS and metadata.

## Environment

Plugin discovery uses:

- `XYRA_PLUGIN_DIRS` (comma-separated plugin roots; each entry can be a parent directory of plugins or a direct plugin folder path, absolute or relative to panel root)
- fallback: `XYRA_PLUGINS_DIR`
- `XYRA_PLUGIN_INSTALL_DIR` (optional installer destination root; defaults to `extensions`)

Default:

```dotenv
XYRA_PLUGIN_DIRS="extensions"
```

Optional restart automation env:

```dotenv
XYRA_PLUGIN_AUTO_RESTART="false" # set true in production when supervised (pm2/systemd/docker restart policy)
XYRA_PLUGIN_RESTART_DELAY_MS="1500"
```

## Inspection APIs

- `GET /api/admin/plugins` (admin-only runtime summary; requires `admin.settings.read` for API keys)
- `POST /api/admin/plugins/install` (admin-only install endpoint; JSON path mode or multipart archive mode; requires `admin.settings.write` for API keys)
- `GET /api/admin/plugins/scopes` (admin-only plugin scope + available eggs; requires `admin.settings.read` for API keys)
- `PATCH /api/admin/plugins/:id/scope` (admin-only; set plugin scope to `global` or `eggs`; requires `admin.settings.write` for API keys)
- `PATCH /api/admin/plugins/:id/state` (admin-only; enable/disable a plugin; requires `admin.settings.write` for API keys)
- `DELETE /api/admin/plugins/:id` (admin-only; uninstall plugin files and remove scope settings; requires `admin.settings.write` for API keys)
- `GET /api/plugins/contributions` (authenticated, permission-filtered contributions; pass `serverId` to apply egg scope filtering)

## Troubleshooting

### Plugin does not appear in `/admin/plugins`

- Confirm manifest location is either `extensions/<id>/plugin.json` or `<custom-path>/plugin.json`.
- Confirm `XYRA_PLUGIN_DIRS` includes either the plugin parent directory or the direct plugin folder path.
- Check manifest JSON syntax.

### Plugin discovered but not loaded

- Check `entry.server` exists and is valid ESM.
- Ensure export is either a function or an object with `setup`/`hooks`.
- Read plugin errors in `/admin/plugins` and server logs.

### Slot component does not render

- Confirm `entry.nuxtLayer` points to a valid Nuxt layer directory.
- Confirm component name matches what Nuxt resolves globally.
- Restart dev server after manifest/layer changes.

### Module entry does not execute

- Confirm `entry.module` points to a file (example: `./modules/xyra.ts`).
- Confirm module default export uses `defineNuxtModule(...)`.
- Restart dev server after adding/changing `entry.module`.

### Navigation item not visible

- Check `permission` value matches actual session permissions.
- Check user role and ACL context.

## Security Notes

Plugins run trusted code on your server process. There is no sandbox or signature verification yet. Only install plugins you trust.
