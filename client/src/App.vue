<template>
  <div class="app-shell" :class="{ collapsed: railMode }">
    <aside
      id="app-sidebar"
      class="sidebar"
      :class="{ open: drawerOpen }"
      :aria-label="t('nav.primary')"
    >
      <div class="sidebar-brand">
        <span v-if="railMode" class="sidebar-brand-mark" aria-hidden="true">{{ brandInitial }}</span>
        <span class="sidebar-brand-name" :class="{ 'visually-hidden': railMode }">{{ t('nav.companyName') }}</span>
        <span class="sidebar-brand-sub" :class="{ 'visually-hidden': railMode }">{{ t('nav.subtitle') }}</span>
      </div>

      <nav class="sidebar-nav">
        <router-link
          v-for="item in navItems"
          :key="item.to"
          :to="item.to"
          class="sidebar-link"
          :title="railMode ? t(item.labelKey) : null"
          :active-class="item.exact ? '' : 'active'"
          exact-active-class="active"
          @click="drawerOpen = false"
        >
          <svg class="sidebar-icon" viewBox="0 0 20 20" fill="none" stroke="currentColor"
               stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
            <path v-for="d in item.icon" :key="d" :d="d" />
          </svg>
          <span :class="{ 'visually-hidden': railMode }">{{ t(item.labelKey) }}</span>
        </router-link>
      </nav>

      <button
        v-if="!drawerMode"
        type="button"
        class="sidebar-toggle"
        aria-controls="app-sidebar"
        :aria-expanded="String(!isCollapsed)"
        :aria-label="isCollapsed ? t('nav.expand') : t('nav.collapse')"
        :title="isCollapsed ? t('nav.expand') : t('nav.collapse')"
        @click="toggleSidebar"
      >
        <svg class="sidebar-icon sidebar-toggle-icon" viewBox="0 0 20 20" fill="none" stroke="currentColor"
             stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
          <path :d="isCollapsed ? 'M7.5 4l6 6-6 6' : 'M12.5 4l-6 6 6 6'" />
        </svg>
      </button>

      <div class="sidebar-footer">
        <LanguageSwitcher :collapsed="railMode" />
        <ProfileMenu
          :collapsed="railMode"
          @show-profile-details="showProfileDetails = true"
          @show-tasks="showTasks = true"
        />
      </div>
    </aside>

    <div v-if="drawerOpen" class="scrim" @click="drawerOpen = false"></div>

    <div class="shell-main">
      <header class="mobile-bar">
        <button
          ref="hamburgerRef"
          type="button"
          class="icon-btn"
          aria-controls="app-sidebar"
          :aria-expanded="String(drawerOpen)"
          :aria-label="t('nav.openMenu')"
          @click="drawerOpen = true"
        >
          <svg viewBox="0 0 20 20" width="20" height="20" fill="none" stroke="currentColor"
               stroke-width="1.5" stroke-linecap="round" aria-hidden="true">
            <path d="M3 5h14M3 10h14M3 15h14" />
          </svg>
        </button>
        <span class="mobile-bar-title">{{ t('nav.companyName') }}</span>
      </header>

      <FilterBar />
      <main class="content">
        <router-view />
      </main>
    </div>

    <ProfileDetailsModal
      :is-open="showProfileDetails"
      @close="showProfileDetails = false"
    />

    <TasksModal
      :is-open="showTasks"
      :tasks="tasks"
      @close="showTasks = false"
      @add-task="addTask"
      @delete-task="deleteTask"
      @toggle-task="toggleTask"
    />
  </div>
</template>

<script>
import { ref, watch, nextTick, onMounted, onBeforeUnmount, computed } from 'vue'
import { api } from './api'
import { useAuth } from './composables/useAuth'
import { useI18n } from './composables/useI18n'
import FilterBar from './components/FilterBar.vue'
import ProfileMenu from './components/ProfileMenu.vue'
import ProfileDetailsModal from './components/ProfileDetailsModal.vue'
import TasksModal from './components/TasksModal.vue'
import LanguageSwitcher from './components/LanguageSwitcher.vue'

export default {
  name: 'App',
  components: {
    FilterBar,
    ProfileMenu,
    ProfileDetailsModal,
    TasksModal,
    LanguageSwitcher
  },
  setup() {
    const { currentUser } = useAuth()
    const { t } = useI18n()
    const showProfileDetails = ref(false)
    const showTasks = ref(false)
    const drawerOpen = ref(false)
    const hamburgerRef = ref(null)

    // Sidebar collapse. Three layouts:
    //  - drawerMode (<=900px): off-canvas drawer, always full width; collapsing n/a.
    //  - inRailRange (901-1200px): auto-collapsed rail. The toggle only flips a
    //    session-only override (railExpanded), which resets on re-entering the range
    //    and is never persisted.
    //  - wide (>1200px): userCollapsed, persisted in localStorage.
    const STORAGE_KEY = 'sidebar-collapsed'
    const readStoredCollapsed = () => {
      try {
        return window.localStorage.getItem(STORAGE_KEY) === 'true'
      } catch (err) {
        return false
      }
    }
    const userCollapsed = ref(readStoredCollapsed())
    const railExpanded = ref(false)
    const isNarrow = ref(false)
    const drawerMode = ref(false)
    const inRailRange = computed(() => isNarrow.value && !drawerMode.value)
    const isCollapsed = computed(() =>
      inRailRange.value ? !railExpanded.value : userCollapsed.value
    )
    // Collapsed rail actually rendered (never in drawer mode)
    const railMode = computed(() => isCollapsed.value && !drawerMode.value)
    const brandInitial = computed(() => (t('nav.companyName') || '').trim().charAt(0).toUpperCase())

    const toggleSidebar = () => {
      if (inRailRange.value) {
        railExpanded.value = !railExpanded.value
        return
      }
      userCollapsed.value = !userCollapsed.value
      try {
        window.localStorage.setItem(STORAGE_KEY, String(userCollapsed.value))
      } catch (err) {
        // Storage unavailable: keep in-memory choice only
      }
    }

    let narrowQuery = null
    let drawerQuery = null
    const onNarrowChange = (e) => {
      isNarrow.value = e.matches
      railExpanded.value = false
    }
    const onDrawerChange = (e) => {
      drawerMode.value = e.matches
      if (!e.matches) drawerOpen.value = false
    }

    // Drawer: Escape closes (listener only while open), body scroll locked while
    // open, focus returns to the hamburger on close.
    const onDrawerKeydown = (e) => {
      if (e.key === 'Escape') drawerOpen.value = false
    }
    const setDrawerSideEffects = (open) => {
      document.body.classList.toggle('drawer-open', open)
      if (open) document.addEventListener('keydown', onDrawerKeydown)
      else document.removeEventListener('keydown', onDrawerKeydown)
    }
    watch(drawerOpen, (open, wasOpen) => {
      setDrawerSideEffects(open)
      if (wasOpen && !open) {
        nextTick(() => {
          if (hamburgerRef.value) hamburgerRef.value.focus()
        })
      }
    })

    onMounted(() => {
      narrowQuery = window.matchMedia('(max-width: 1200px)')
      drawerQuery = window.matchMedia('(max-width: 900px)')
      isNarrow.value = narrowQuery.matches
      drawerMode.value = drawerQuery.matches
      narrowQuery.addEventListener('change', onNarrowChange)
      drawerQuery.addEventListener('change', onDrawerChange)
    })
    onBeforeUnmount(() => {
      setDrawerSideEffects(false)
      if (narrowQuery) narrowQuery.removeEventListener('change', onNarrowChange)
      if (drawerQuery) drawerQuery.removeEventListener('change', onDrawerChange)
    })

    // Icons: 20x20 viewBox paths, stroke only
    const navItems = [
      { to: '/', labelKey: 'nav.overview', exact: true,
        icon: ['M3 3h6v6H3z', 'M11 3h6v6h-6z', 'M3 11h6v6H3z', 'M11 11h6v6h-6z'] },
      { to: '/inventory', labelKey: 'nav.inventory',
        icon: ['M3 6.5L10 3l7 3.5v7L10 17l-7-3.5z', 'M3 6.5l7 3.5 7-3.5', 'M10 10v7'] },
      { to: '/orders', labelKey: 'nav.orders',
        icon: ['M5 3h10v14H5z', 'M8 7h4', 'M8 10h4', 'M8 13h2'] },
      { to: '/spending', labelKey: 'nav.finance',
        icon: ['M3 6h14v10H3z', 'M3 9h14', 'M6 13h3'] },
      { to: '/demand', labelKey: 'nav.demandForecast',
        icon: ['M3 16l4.5-5 3 3L17 5', 'M13 5h4v4'] },
      { to: '/reports', labelKey: 'nav.reports',
        icon: ['M4 17V9', 'M10 17V4', 'M16 17v-5'] }
    ]
    const apiTasks = ref([])

    // Merge mock tasks from currentUser with API tasks
    const tasks = computed(() => {
      return [...currentUser.value.tasks, ...apiTasks.value]
    })

    const loadTasks = async () => {
      try {
        apiTasks.value = await api.getTasks()
      } catch (err) {
        console.error('Failed to load tasks:', err)
      }
    }

    const addTask = async (taskData) => {
      try {
        const newTask = await api.createTask(taskData)
        // Add new task to the beginning of the array
        apiTasks.value.unshift(newTask)
      } catch (err) {
        console.error('Failed to add task:', err)
      }
    }

    const deleteTask = async (taskId) => {
      try {
        // Check if it's a mock task (from currentUser)
        const isMockTask = currentUser.value.tasks.some(t => t.id === taskId)

        if (isMockTask) {
          // Remove from mock tasks
          const index = currentUser.value.tasks.findIndex(t => t.id === taskId)
          if (index !== -1) {
            currentUser.value.tasks.splice(index, 1)
          }
        } else {
          // Remove from API tasks
          await api.deleteTask(taskId)
          apiTasks.value = apiTasks.value.filter(t => t.id !== taskId)
        }
      } catch (err) {
        console.error('Failed to delete task:', err)
      }
    }

    const toggleTask = async (taskId) => {
      try {
        // Check if it's a mock task (from currentUser)
        const mockTask = currentUser.value.tasks.find(t => t.id === taskId)

        if (mockTask) {
          // Toggle mock task status
          mockTask.status = mockTask.status === 'pending' ? 'completed' : 'pending'
        } else {
          // Toggle API task
          const updatedTask = await api.toggleTask(taskId)
          const index = apiTasks.value.findIndex(t => t.id === taskId)
          if (index !== -1) {
            apiTasks.value[index] = updatedTask
          }
        }
      } catch (err) {
        console.error('Failed to toggle task:', err)
      }
    }

    onMounted(loadTasks)

    return {
      t,
      drawerOpen,
      hamburgerRef,
      drawerMode,
      isCollapsed,
      railMode,
      brandInitial,
      toggleSidebar,
      navItems,
      showProfileDetails,
      showTasks,
      tasks,
      addTask,
      deleteTask,
      toggleTask
    }
  }
}
</script>

<style>
:root {
  /* Color: neutral slate plus one accent */
  --color-bg: #f8fafc;
  --color-surface: #ffffff;
  --color-border: #e2e8f0;
  --color-border-subtle: #f1f5f9;
  --color-text: #0f172a;
  --color-text-secondary: #475569;
  --color-text-muted: #64748b;
  --color-accent: #2563eb;
  --color-accent-soft: #eff6ff;

  /* Status */
  --color-success: #059669;
  --color-warning: #d97706;
  --color-danger: #dc2626;
  --color-info: #2563eb;

  /* Sidebar */
  --sidebar-width: 248px;
  --sidebar-width-collapsed: 72px;
  --sidebar-bg: #0f172a;
  --sidebar-text: #cbd5e1;
  --sidebar-text-active: #ffffff;
  --sidebar-item-hover: rgba(255, 255, 255, 0.06);
  --sidebar-item-active: rgba(255, 255, 255, 0.12);
  --sidebar-border: rgba(255, 255, 255, 0.08);
  --sidebar-focus: #93c5fd;

  /* Spacing: 4px scale. Use only these. */
  --space-1: 0.25rem;
  --space-2: 0.5rem;
  --space-3: 0.75rem;
  --space-4: 1rem;
  --space-5: 1.25rem;
  --space-6: 1.5rem;
  --space-8: 2rem;
  --space-10: 2.5rem;

  /* Radius */
  --radius-sm: 6px;
  --radius-md: 8px;
  --radius-lg: 12px;
  --radius-pill: 999px;

  /* Elevation */
  --shadow-sm: 0 1px 2px rgba(15, 23, 42, 0.05);
  --shadow-md: 0 4px 12px rgba(15, 23, 42, 0.08);
  --shadow-lg: 0 12px 32px rgba(15, 23, 42, 0.16);

  /* Type */
  --font-sans: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  --text-xs: 0.75rem;
  --text-sm: 0.875rem;
  --text-base: 1rem;
  --text-lg: 1.125rem;
  --text-xl: 1.5rem;
  --weight-regular: 400;
  --weight-medium: 500;
  --weight-semibold: 600;

  /* Layout */
  --mobile-bar-height: 61px;
  --content-max-width: 1400px;
  --content-padding: var(--space-8);

  /* Stacking: modals must sit above the sidebar and its drawer scrim */
  --z-sticky: 50;
  --z-sidebar: 100;
  --z-scrim: 150;
  --z-dropdown: 200;
  --z-modal: 300;

  --transition: 150ms ease;
}

@media (prefers-reduced-motion: reduce) {
  :root { --transition: 0ms; }
}

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body.drawer-open {
  overflow: hidden;
}

body {
  font-family: var(--font-sans);
  background: var(--color-bg);
  color: var(--color-text);
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

/* Shell */
.app-shell {
  display: grid;
  grid-template-columns: var(--sidebar-width) minmax(0, 1fr);
  transition: grid-template-columns var(--transition);
  min-height: 100vh;
  min-height: 100dvh;
  background: var(--color-bg);
  font-family: var(--font-sans);
  color: var(--color-text);
}

.app-shell.collapsed {
  grid-template-columns: var(--sidebar-width-collapsed) minmax(0, 1fr);
}

.sidebar {
  position: sticky;
  top: 0;
  height: 100vh;
  height: 100dvh;
  display: flex;
  flex-direction: column;
  padding: var(--space-4);
  transition: padding var(--transition);
  background: var(--sidebar-bg);
  color: var(--sidebar-text);
  z-index: var(--z-sidebar);
}

.sidebar-brand {
  display: flex;
  flex-direction: column;
  gap: var(--space-1);
  padding: var(--space-2) var(--space-3) var(--space-5);
}

.sidebar-brand-name {
  font-size: var(--text-lg);
  font-weight: var(--weight-semibold);
  color: var(--sidebar-text-active);
  letter-spacing: -0.025em;
}

.sidebar-brand-sub {
  font-size: var(--text-xs);
  color: var(--sidebar-text);
}

.sidebar-nav {
  display: flex;
  flex-direction: column;
  gap: var(--space-1);
  flex: 1;
  overflow-y: auto;
}

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

.sidebar-link:hover {
  background: var(--sidebar-item-hover);
  color: var(--sidebar-text-active);
}

.sidebar-link.active {
  background: var(--sidebar-item-active);
  color: var(--sidebar-text-active);
}

.icon-btn:focus-visible {
  outline: 2px solid var(--color-accent);
  outline-offset: 2px;
}

.sidebar a:focus-visible,
.sidebar button:focus-visible {
  outline: 2px solid var(--sidebar-focus);
  outline-offset: 2px;
}

.sidebar-icon {
  width: 20px;
  height: 20px;
  flex: none;
}

.visually-hidden {
  position: absolute;
  width: 1px;
  height: 1px;
  margin: -1px;
  padding: 0;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}

.sidebar-brand-mark {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 40px;
  height: 40px;
  border-radius: var(--radius-md);
  background: var(--sidebar-item-active);
  color: var(--sidebar-text-active);
  font-size: var(--text-lg);
  font-weight: var(--weight-semibold);
}

.sidebar-toggle {
  display: flex;
  align-items: center;
  justify-content: flex-end;
  min-height: 40px;
  margin-bottom: var(--space-2);
  padding: var(--space-2) var(--space-3);
  background: none;
  border: none;
  border-radius: var(--radius-md);
  color: var(--sidebar-text);
  cursor: pointer;
  transition: background var(--transition), color var(--transition);
}

.sidebar-toggle:hover {
  background: var(--sidebar-item-hover);
  color: var(--sidebar-text-active);
}

.app-shell.collapsed .sidebar {
  padding-left: var(--space-3);
  padding-right: var(--space-3);
}

.app-shell.collapsed .sidebar-brand {
  align-items: center;
  padding-left: 0;
  padding-right: 0;
}

.app-shell.collapsed .sidebar-link {
  justify-content: center;
  min-height: 40px;
  padding-left: 0;
  padding-right: 0;
}

.app-shell.collapsed .sidebar-toggle {
  justify-content: center;
  padding-left: 0;
  padding-right: 0;
}

.sidebar-footer {
  display: flex;
  flex-direction: column;
  gap: var(--space-2);
  padding-top: var(--space-4);
  border-top: 1px solid var(--sidebar-border);
}

.shell-main {
  min-width: 0;
  display: flex;
  flex-direction: column;
}

.content {
  flex: 1;
  width: 100%;
  max-width: var(--content-max-width);
  padding: var(--content-padding);
}

.mobile-bar,
.scrim {
  display: none;
}

.icon-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 36px;
  height: 36px;
  background: none;
  border: 1px solid var(--color-border);
  border-radius: var(--radius-md);
  color: var(--color-text);
  cursor: pointer;
}

.mobile-bar-title {
  font-size: var(--text-base);
  font-weight: var(--weight-semibold);
  color: var(--color-text);
}

@media (max-width: 900px) {
  .app-shell,
  .app-shell.collapsed {
    grid-template-columns: minmax(0, 1fr);
  }

  .sidebar {
    position: fixed;
    inset: 0 auto 0 0;
    width: var(--sidebar-width);
    transform: translateX(-100%);
    visibility: hidden;
    transition: transform var(--transition), visibility var(--transition);
    z-index: calc(var(--z-scrim) + 1);
  }

  .sidebar.open {
    transform: translateX(0);
    visibility: visible;
  }

  .scrim {
    display: block;
    position: fixed;
    inset: 0;
    background: rgba(15, 23, 42, 0.5);
    z-index: var(--z-scrim);
  }

  .mobile-bar {
    display: flex;
    align-items: center;
    gap: var(--space-3);
    padding: var(--space-3) var(--space-4);
    background: var(--color-surface);
    border-bottom: 1px solid var(--color-border);
    position: sticky;
    top: 0;
    height: var(--mobile-bar-height);
    box-sizing: border-box;
    z-index: calc(var(--z-sticky) + 1);
  }

  .content {
    padding: var(--space-4);
  }
}

/* Shared classes */
.page-header {
  margin-bottom: var(--space-6);
}

.page-header h2 {
  font-size: var(--text-xl);
  font-weight: var(--weight-semibold);
  color: var(--color-text);
  margin-bottom: var(--space-1);
  letter-spacing: -0.025em;
}

.page-header p {
  color: var(--color-text-muted);
  font-size: var(--text-sm);
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: var(--space-5);
  margin-bottom: var(--space-6);
}

.stat-card {
  background: var(--color-surface);
  padding: var(--space-5);
  border-radius: var(--radius-lg);
  border: 1px solid var(--color-border);
  box-shadow: var(--shadow-sm);
  transition: border-color var(--transition), box-shadow var(--transition);
}

.stat-card:hover {
  border-color: #cbd5e1;
  box-shadow: var(--shadow-md);
}

.stat-label {
  color: var(--color-text-muted);
  font-size: var(--text-sm);
  font-weight: var(--weight-semibold);
  text-transform: uppercase;
  letter-spacing: 0.5px;
  margin-bottom: var(--space-2);
}

.stat-value {
  font-size: 2.25rem;
  font-weight: var(--weight-semibold);
  color: var(--color-text);
  letter-spacing: -0.025em;
}

.stat-card.warning .stat-value {
  color: var(--color-warning);
}

.stat-card.success .stat-value {
  color: var(--color-success);
}

.stat-card.danger .stat-value {
  color: var(--color-danger);
}

.stat-card.info .stat-value {
  color: var(--color-info);
}

.card {
  background: var(--color-surface);
  border-radius: var(--radius-lg);
  padding: var(--space-5);
  border: 1px solid var(--color-border);
  box-shadow: var(--shadow-sm);
  margin-bottom: var(--space-5);
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: var(--space-4);
  padding-bottom: var(--space-3);
  border-bottom: 1px solid var(--color-border);
}

.card-title {
  font-size: var(--text-lg);
  font-weight: var(--weight-semibold);
  color: var(--color-text);
  letter-spacing: -0.025em;
}

.table-container {
  overflow-x: auto;
}

table {
  width: 100%;
  border-collapse: collapse;
}

thead {
  background: var(--color-bg);
  border-top: 1px solid var(--color-border);
  border-bottom: 1px solid var(--color-border);
}

th {
  text-align: left;
  padding: var(--space-2) var(--space-3);
  font-weight: var(--weight-semibold);
  color: var(--color-text-secondary);
  font-size: var(--text-xs);
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

td {
  padding: var(--space-2) var(--space-3);
  border-top: 1px solid var(--color-border-subtle);
  color: var(--color-text-secondary);
  font-size: var(--text-sm);
}

tbody tr {
  transition: background-color var(--transition);
}

tbody tr:hover {
  background: var(--color-bg);
}

.badge {
  display: inline-block;
  padding: var(--space-1) var(--space-3);
  border-radius: var(--radius-sm);
  font-size: var(--text-xs);
  font-weight: var(--weight-semibold);
  text-transform: uppercase;
  letter-spacing: 0.025em;
}

.badge.success,
.badge.increasing {
  background: #d1fae5;
  color: #065f46;
}

.badge.warning,
.badge.medium {
  background: #fed7aa;
  color: #92400e;
}

.badge.danger,
.badge.decreasing,
.badge.high {
  background: #fecaca;
  color: #991b1b;
}

.badge.info,
.badge.low {
  background: #dbeafe;
  color: #1e40af;
}

.badge.stable {
  background: #e0e7ff;
  color: #3730a3;
}

.loading {
  text-align: center;
  padding: var(--space-10);
  color: var(--color-text-muted);
  font-size: var(--text-sm);
}

.error {
  background: #fef2f2;
  border: 1px solid #fecaca;
  color: #991b1b;
  padding: var(--space-4);
  border-radius: var(--radius-md);
  margin: var(--space-4) 0;
  font-size: var(--text-sm);
}
</style>
