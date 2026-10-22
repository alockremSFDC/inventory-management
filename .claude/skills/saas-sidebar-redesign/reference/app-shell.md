# App shell reference

Copy-and-adapt snippets for the sidebar layout. Adjust route list, labels, and icons to the app. Tokens come from `tokens.css`.

## Template

```vue
<template>
  <div class="app-shell">
    <aside class="sidebar" :class="{ open: drawerOpen }" aria-label="Primary">
      <div class="sidebar-brand">{{ t('nav.companyName') }}</div>

      <nav class="sidebar-nav">
        <router-link
          v-for="item in navItems"
          :key="item.to"
          :to="item.to"
          class="sidebar-link"
          active-class="active"
          exact-active-class="active"
          @click="drawerOpen = false"
        >
          <svg class="sidebar-icon" viewBox="0 0 20 20" fill="none" stroke="currentColor"
               stroke-width="1.5" aria-hidden="true">
            <path :d="item.icon" />
          </svg>
          <span>{{ t(item.labelKey) }}</span>
        </router-link>
      </nav>

      <div class="sidebar-footer">
        <!-- language switcher and profile menu go here; their dropdowns must open upward -->
      </div>
    </aside>

    <div v-if="drawerOpen" class="scrim" @click="drawerOpen = false"></div>

    <div class="shell-main">
      <header class="mobile-bar">
        <button class="icon-btn" :aria-label="t('nav.openMenu')" @click="drawerOpen = true">
          <svg viewBox="0 0 20 20" width="20" height="20" fill="none" stroke="currentColor" stroke-width="1.5">
            <path d="M3 5h14M3 10h14M3 15h14" />
          </svg>
        </button>
      </header>

      <!-- page-level filters/toolbars render here, inside the content column -->
      <main class="content"><router-view /></main>
    </div>
  </div>
</template>
```

Notes:
- Drive the nav from an array (`navItems` with `to`, `labelKey`, `icon` path) so labels go through i18n and keys are unique.
- Use `active-class` and let vue-router set `aria-current="page"` on the exact match. Do not compare `$route.path` by hand; it breaks nested routes.
- `/` needs `exact-active-class` only, or it matches every route. Use `:to="{ path: '/' }"` with `exact-active-class="active"` and leave `active-class` off for that item.
- Keep icons as inline SVG with `currentColor` unless the app already has an icon library.

## CSS (global, non-scoped)

```css
.app-shell {
  display: grid;
  grid-template-columns: var(--sidebar-width) minmax(0, 1fr);
  min-height: 100vh;
  background: var(--color-bg);
  font-family: var(--font-sans);
  color: var(--color-text);
}

.sidebar {
  position: sticky;
  top: 0;
  height: 100vh;
  display: flex;
  flex-direction: column;
  padding: var(--space-4);
  background: var(--sidebar-bg);
  color: var(--sidebar-text);
  z-index: var(--z-sidebar);
}
.sidebar-brand {
  padding: var(--space-2) var(--space-3) var(--space-5);
  font-size: var(--text-lg);
  font-weight: var(--weight-semibold);
  color: var(--sidebar-text-active);
}
.sidebar-nav { display: flex; flex-direction: column; gap: var(--space-1); flex: 1; overflow-y: auto; }

.sidebar-link {
  display: flex;
  align-items: center;
  gap: var(--space-3);
  padding: var(--space-2) var(--space-3);
  border-radius: var(--radius-md);
  font-size: var(--text-sm);
  font-weight: var(--weight-medium);
  color: var(--sidebar-text);
  text-decoration: none;
  transition: background var(--transition), color var(--transition);
}
.sidebar-link:hover { background: var(--sidebar-item-hover); }
.sidebar-link.active { background: var(--sidebar-item-active); color: var(--sidebar-text-active); }
.sidebar-link:focus-visible { outline: 2px solid var(--color-accent); outline-offset: 2px; }
.sidebar-icon { width: 20px; height: 20px; flex: none; }

.sidebar-footer {
  display: flex;
  flex-direction: column;
  gap: var(--space-2);
  padding-top: var(--space-4);
  border-top: 1px solid var(--sidebar-border);
}

.shell-main { min-width: 0; display: flex; flex-direction: column; }
.content {
  width: 100%;
  max-width: var(--content-max-width);
  padding: var(--content-padding);
}
.mobile-bar, .scrim { display: none; }

/* Drawer under 900px */
@media (max-width: 900px) {
  .app-shell { grid-template-columns: minmax(0, 1fr); }
  .sidebar {
    position: fixed;
    inset: 0 auto 0 0;
    width: var(--sidebar-width);
    transform: translateX(-100%);
    transition: transform var(--transition);
  }
  .sidebar.open { transform: translateX(0); }
  .scrim {
    display: block;
    position: fixed;
    inset: 0;
    background: rgba(15, 23, 42, 0.5);
    z-index: var(--z-scrim);
  }
  .sidebar { z-index: calc(var(--z-scrim) + 1); }
  .mobile-bar {
    display: flex;
    align-items: center;
    padding: var(--space-3) var(--space-4);
    background: var(--color-surface);
    border-bottom: 1px solid var(--color-border);
    position: sticky;
    top: 0;
    z-index: var(--z-sticky);
  }
  .content { padding: var(--space-4); }
}
```

## Shared component classes

Many apps style `.card`, `.badge`, tables, and `.page-header` globally and the views depend on them. Keep the class names; change only the values to tokens:

```css
.card {
  background: var(--color-surface);
  border: 1px solid var(--color-border);
  border-radius: var(--radius-lg);
  box-shadow: var(--shadow-sm);
  padding: var(--space-5);
}
.page-header { margin-bottom: var(--space-6); }
.page-header h1 { font-size: var(--text-xl); font-weight: var(--weight-semibold); }
.badge {
  display: inline-flex;
  padding: var(--space-1) var(--space-2);
  border-radius: var(--radius-pill);
  font-size: var(--text-xs);
  font-weight: var(--weight-medium);
}
```

## Knock-on fixes

| Old assumption | Fix |
| --- | --- |
| `position: sticky; top: <header height>` on a filter bar | Change to `top: 0` (the header is gone), or move the filters into the page header |
| Dropdown opens with `top: calc(100% + .5rem)` | In the sidebar footer use `bottom: calc(100% + var(--space-2)); left: 0` so it opens upward |
| Modal `z-index` below 100 | Use `var(--z-modal)` so it covers the sidebar |
| Container `max-width` plus `margin: 0 auto` in several places | Keep one: `.content` |
| Nav label hard-coded in English | Add a `nav.*` key to every locale file |
