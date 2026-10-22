<template>
  <div ref="rootRef" class="language-switcher">
    <button
      ref="triggerRef"
      type="button"
      class="language-button"
      :class="{ collapsed }"
      aria-haspopup="menu"
      :aria-expanded="String(isDropdownOpen)"
      :aria-label="collapsed ? localeName : null"
      :title="collapsed ? localeName : null"
      @click="toggleDropdown"
    >
      <svg
        width="20"
        height="20"
        viewBox="0 0 20 20"
        fill="none"
        class="globe-icon"
        aria-hidden="true"
      >
        <circle cx="10" cy="10" r="7.5" stroke="currentColor" stroke-width="1.5"/>
        <path d="M3 10H17" stroke="currentColor" stroke-width="1.5"/>
        <path d="M10 3C10 3 7.5 5.5 7.5 10C7.5 14.5 10 17 10 17" stroke="currentColor" stroke-width="1.5"/>
        <path d="M10 3C10 3 12.5 5.5 12.5 10C12.5 14.5 10 17 10 17" stroke="currentColor" stroke-width="1.5"/>
      </svg>
      <span v-if="!collapsed" class="language-label">{{ localeName }}</span>
      <svg
        v-if="!collapsed"
        class="chevron"
        :class="{ 'chevron-open': isDropdownOpen }"
        width="16"
        height="16"
        viewBox="0 0 16 16"
        fill="none"
        aria-hidden="true"
      >
        <path d="M4 6L8 10L12 6" stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
      </svg>
    </button>

    <div v-if="isDropdownOpen" class="dropdown-menu" role="menu" :class="{ 'dropdown-right': collapsed }">
      <button
        v-for="locale in availableLocales"
        :key="locale"
        type="button"
        role="menuitemradio"
        :aria-checked="String(currentLocale === locale)"
        class="dropdown-item"
        :class="{ active: currentLocale === locale }"
        @click="selectLanguage(locale)"
      >
        <span class="language-name">{{ getLanguageName(locale) }}</span>
        <svg
          v-if="currentLocale === locale"
          width="18"
          height="18"
          viewBox="0 0 18 18"
          fill="none"
          class="check-icon"
          aria-hidden="true"
        >
          <path d="M4 9L7.5 12.5L14 6" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
        </svg>
      </button>
    </div>
  </div>
</template>

<script setup>
import { ref, watch, onBeforeUnmount } from 'vue'
import { useI18n } from '../composables/useI18n'

defineProps({
  collapsed: { type: Boolean, default: false }
})

const { currentLocale, setLocale, availableLocales, localeName } = useI18n()

const isDropdownOpen = ref(false)

const languageNames = {
  en: 'English',
  ja: '日本語'
}

const getLanguageName = (locale) => {
  return languageNames[locale] || locale
}

const toggleDropdown = () => {
  isDropdownOpen.value = !isDropdownOpen.value
}

const rootRef = ref(null)
const triggerRef = ref(null)

const closeDropdown = (returnFocus = false) => {
  isDropdownOpen.value = false
  if (returnFocus && triggerRef.value) triggerRef.value.focus()
}

const onDocumentClick = (e) => {
  if (rootRef.value && !rootRef.value.contains(e.target)) closeDropdown()
}

const onDocumentKeydown = (e) => {
  if (e.key === 'Escape') closeDropdown(true)
}

watch(isDropdownOpen, (open) => {
  if (open) {
    document.addEventListener('click', onDocumentClick)
    document.addEventListener('keydown', onDocumentKeydown)
  } else {
    document.removeEventListener('click', onDocumentClick)
    document.removeEventListener('keydown', onDocumentKeydown)
  }
})

onBeforeUnmount(() => {
  document.removeEventListener('click', onDocumentClick)
  document.removeEventListener('keydown', onDocumentKeydown)
})

const selectLanguage = (locale) => {
  setLocale(locale)
  closeDropdown(true)
}
</script>

<style scoped>
.language-switcher {
  position: relative;
}

.language-button {
  width: 100%;
  display: flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.5rem 0.875rem;
  background: transparent;
  border: 1px solid var(--sidebar-border);
  border-radius: var(--radius-md);
  cursor: pointer;
  transition: background var(--transition);
  font-family: inherit;
  font-size: 0.875rem;
  color: var(--sidebar-text-active);
}

.language-button:hover {
  background: var(--sidebar-item-hover);
}

.language-button:focus-visible {
  outline: 2px solid var(--sidebar-focus);
  outline-offset: 2px;
}

.language-button.collapsed {
  justify-content: center;
  min-height: 40px;
  padding: var(--space-2);
}

.dropdown-menu.dropdown-right {
  width: auto;
  min-width: 160px;
  max-width: none;
  bottom: 0;
  left: calc(100% + var(--space-2));
}

.globe-icon {
  color: var(--sidebar-text);
  flex-shrink: 0;
}

.language-label {
  flex: 1;
  text-align: left;
  font-weight: 500;
}

.chevron {
  color: var(--sidebar-text);
  transition: transform 0.2s ease;
  flex-shrink: 0;
  transform: rotate(180deg);
}

.chevron-open {
  transform: rotate(0deg);
}

.dropdown-menu {
  position: absolute;
  bottom: calc(100% + var(--space-2));
  left: 0;
  width: 100%;
  min-width: 0;
  max-width: 100%;
  background: white;
  border: 1px solid #e2e8f0;
  border-radius: 10px;
  box-shadow: 0 10px 25px rgba(0, 0, 0, 0.1);
  z-index: var(--z-dropdown);
  overflow: hidden;
}

.dropdown-item {
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.75rem;
  padding: 0.75rem 1rem;
  background: none;
  border: none;
  text-align: left;
  cursor: pointer;
  transition: background 0.15s ease;
  font-family: inherit;
  font-size: 0.875rem;
  font-weight: 500;
  color: #334155;
}

.dropdown-menu .dropdown-item:focus-visible {
  outline: 2px solid var(--color-accent);
  outline-offset: -2px;
  background: #f8fafc;
}

.dropdown-item:hover {
  background: #f8fafc;
}

.dropdown-item.active {
  background: #eff6ff;
  color: #2563eb;
}

.language-name {
  flex: 1;
}

.check-icon {
  color: #2563eb;
  flex-shrink: 0;
}
</style>
