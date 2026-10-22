<template>
  <div ref="rootRef" class="profile-menu">
    <button
      ref="triggerRef"
      type="button"
      class="profile-button"
      :class="{ collapsed }"
      aria-haspopup="menu"
      :aria-expanded="String(isDropdownOpen)"
      :aria-label="collapsed ? currentUser.name : null"
      :title="collapsed ? currentUser.name : null"
      @click="toggleDropdown"
    >
      <div class="avatar">
        {{ getInitials(currentUser.name) }}
      </div>
      <span v-if="!collapsed" class="profile-name">{{ currentUser.name }}</span>
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
      <div class="dropdown-header">
        <div class="avatar-large">
          {{ getInitials(currentUser.name) }}
        </div>
        <div class="user-info">
          <div class="user-name">{{ currentUser.name }}</div>
          <div class="user-email">{{ currentUser.email }}</div>
        </div>
      </div>

      <div class="dropdown-divider"></div>

      <button
        type="button"
        role="menuitem"
        class="dropdown-item"
        @click="showProfileDetails"
      >
        <svg width="18" height="18" viewBox="0 0 18 18" fill="none" aria-hidden="true">
          <path d="M9 9C10.6569 9 12 7.65685 12 6C12 4.34315 10.6569 3 9 3C7.34315 3 6 4.34315 6 6C6 7.65685 7.34315 9 9 9Z" stroke="currentColor" stroke-width="1.5"/>
          <path d="M15 15C15 12.7909 12.3137 11 9 11C5.68629 11 3 12.7909 3 15" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/>
        </svg>
        {{ t('profile.profileDetails') }}
      </button>

      <button
        type="button"
        role="menuitem"
        class="dropdown-item"
        @click="showTasks"
      >
        <svg width="18" height="18" viewBox="0 0 18 18" fill="none" aria-hidden="true">
          <path d="M15 3H3C2.44772 3 2 3.44772 2 4V14C2 14.5523 2.44772 15 3 15H15C15.5523 15 16 14.5523 16 14V4C16 3.44772 15.5523 3 15 3Z" stroke="currentColor" stroke-width="1.5"/>
          <path d="M6 7L8 9L12 5" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
        </svg>
        {{ t('profile.myTasks') }}
        <span v-if="pendingTaskCount > 0" class="task-badge">{{ pendingTaskCount }}</span>
      </button>

      <div class="dropdown-divider"></div>

      <button
        type="button"
        role="menuitem"
        class="dropdown-item logout"
        @click="handleLogout"
      >
        <svg width="18" height="18" viewBox="0 0 18 18" fill="none" aria-hidden="true">
          <path d="M7 15H4C3.44772 15 3 14.5523 3 14V4C3 3.44772 3.44772 3 4 3H7" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/>
          <path d="M11 12L15 9M15 9L11 6M15 9H7" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
        </svg>
        {{ t('profile.logout') }}
      </button>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch, onBeforeUnmount } from 'vue'
import { useAuth } from '../composables/useAuth'
import { useI18n } from '../composables/useI18n'

defineProps({
  collapsed: { type: Boolean, default: false }
})

const { currentUser, logout, getInitials } = useAuth()
const { t } = useI18n()

const isDropdownOpen = ref(false)
const emit = defineEmits(['show-profile-details', 'show-tasks'])

const pendingTaskCount = computed(() => {
  return currentUser.value.tasks.filter(task => task.status === 'pending').length
})

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

const showProfileDetails = () => {
  closeDropdown()
  emit('show-profile-details')
}

const showTasks = () => {
  closeDropdown()
  emit('show-tasks')
}

const handleLogout = () => {
  closeDropdown()
  logout()
}
</script>

<style scoped>
.profile-menu {
  position: relative;
}

.profile-button {
  width: 100%;
  display: flex;
  align-items: center;
  gap: 0.625rem;
  padding: 0.5rem 0.875rem;
  background: transparent;
  border: 1px solid var(--sidebar-border);
  border-radius: var(--radius-md);
  cursor: pointer;
  transition: background var(--transition);
  font-family: inherit;
}

.profile-button:hover {
  background: var(--sidebar-item-hover);
}

.profile-button:focus-visible {
  outline: 2px solid var(--sidebar-focus);
  outline-offset: 2px;
}

.avatar {
  width: 32px;
  height: 32px;
  border-radius: 50%;
  background: linear-gradient(135deg, #2563eb 0%, #1e40af 100%);
  color: white;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 600;
  font-size: 0.75rem;
  letter-spacing: 0.025em;
}

.profile-name {
  font-size: 0.875rem;
  font-weight: 500;
  color: var(--sidebar-text-active);
  flex: 1;
  text-align: left;
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.chevron {
  color: var(--sidebar-text);
  transition: transform 0.2s ease;
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

.profile-button.collapsed {
  justify-content: center;
  min-height: 40px;
  padding: var(--space-2);
}

.dropdown-menu.dropdown-right {
  width: auto;
  min-width: 280px;
  max-width: none;
  bottom: 0;
  left: calc(100% + var(--space-2));
}

.dropdown-header {
  padding: 1rem;
  display: flex;
  gap: 0.875rem;
  align-items: center;
  background: #f8fafc;
}

.avatar-large {
  width: 48px;
  height: 48px;
  border-radius: 50%;
  background: linear-gradient(135deg, #2563eb 0%, #1e40af 100%);
  color: white;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 700;
  font-size: 1rem;
  letter-spacing: 0.025em;
  flex-shrink: 0;
}

.user-info {
  flex: 1;
  min-width: 0;
}

.user-name {
  overflow-wrap: anywhere;
  font-weight: 600;
  color: #0f172a;
  font-size: 0.938rem;
  margin-bottom: 0.25rem;
}

.user-email {
  font-size: 0.813rem;
  color: #64748b;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.dropdown-divider {
  height: 1px;
  background: #e2e8f0;
  margin: 0.5rem 0;
}

.dropdown-item {
  width: 100%;
  display: flex;
  align-items: center;
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

.dropdown-item svg {
  color: #64748b;
  flex-shrink: 0;
}

.dropdown-item.logout {
  color: #dc2626;
}

.dropdown-item.logout svg {
  color: #dc2626;
}

.dropdown-item.logout:hover {
  background: #fef2f2;
}

.task-badge {
  margin-left: auto;
  background: #2563eb;
  color: white;
  font-size: 0.75rem;
  font-weight: 600;
  padding: 0.125rem 0.5rem;
  border-radius: 12px;
  min-width: 20px;
  text-align: center;
}
</style>
