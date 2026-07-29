<template>
  <aside class="sidebar" :class="{ collapsed }">
    <nav class="sidebar-nav">
      <router-link
        v-for="item in navItems"
        :key="item.path"
        :to="item.path"
        class="nav-item"
        :class="{ active: $route.path === item.path }"
      >
        <svg
          class="nav-icon"
          :viewBox="item.icon.viewBox"
          :aria-label="item.label"
        >
          <path :d="item.icon.d" />
        </svg>
        <span class="nav-label">{{ t(`nav.${item.labelKey}`) }}</span>
      </router-link>
    </nav>

    <button class="sidebar-toggle" @click="$emit('toggle')" :title="collapsed ? 'Expand sidebar' : 'Collapse sidebar'">
      <svg class="toggle-icon" viewBox="0 0 24 24" :aria-label="collapsed ? 'Expand' : 'Collapse'">
        <path d="M9 6l6 6-6 6" />
      </svg>
    </button>
  </aside>
</template>

<script>
import { useI18n } from '../composables/useI18n'

export default {
  name: 'AppSidebar',
  props: {
    collapsed: {
      type: Boolean,
      default: false
    }
  },
  emits: ['toggle'],
  setup() {
    const { t } = useI18n()

    const navItems = [
      {
        path: '/',
        labelKey: 'overview',
        label: 'Overview',
        icon: {
          viewBox: '0 0 24 24',
          d: 'M3 3h8v8H3V3zm10 0h8v8h-8V3zM3 13h8v8H3v-8zm10 0h8v8h-8v-8z'
        }
      },
      {
        path: '/inventory',
        labelKey: 'inventory',
        label: 'Inventory',
        icon: {
          viewBox: '0 0 24 24',
          d: 'M4 3h6v6H4V3zm10 0h6v6h-6V3zM4 13h6v6H4v-6zm10 0h6v6h-6v-6z'
        }
      },
      {
        path: '/orders',
        labelKey: 'orders',
        label: 'Orders',
        icon: {
          viewBox: '0 0 24 24',
          d: 'M3 3h18v2H3V3zm0 4h18v11H3V7zm2 2v7h14V9H5z'
        }
      },
      {
        path: '/spending',
        labelKey: 'finance',
        label: 'Finance',
        icon: {
          viewBox: '0 0 24 24',
          d: 'M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm0 18c-4.42 0-8-3.58-8-8s3.58-8 8-8 8 3.58 8 8-3.58 8-8 8zm3.5-9c.83 0 1.5-.67 1.5-1.5S16.33 8 15.5 8 14 8.67 14 9.5s.67 1.5 1.5 1.5zm-7 0c.83 0 1.5-.67 1.5-1.5S9.33 8 8.5 8 7 8.67 7 9.5 7.67 11 8.5 11zm3.5 6.5c2.33 0 4.31-1.46 5.11-3.5H6.89c.8 2.04 2.78 3.5 5.11 3.5z'
        }
      },
      {
        path: '/demand',
        labelKey: 'demandForecast',
        label: 'Demand Forecast',
        icon: {
          viewBox: '0 0 24 24',
          d: 'M5 9.2h3V19H5zM10.6 5h2.8v14h-2.8zm5.6 8H19v6h-2.8z'
        }
      },
      {
        path: '/reports',
        labelKey: 'reports',
        label: 'Reports',
        icon: {
          viewBox: '0 0 24 24',
          d: 'M19 3H5c-1.1 0-2 .9-2 2v14c0 1.1.9 2 2 2h14c1.1 0 2-.9 2-2V5c0-1.1-.9-2-2-2zm0 16H5V5h14v14zm-5.04-6.71l-2.75 3.54-2.96-3.83c-.3-.4-.94-.4-1.23 0-.29.39-.29 1.02 0 1.41l3.54 4.58c.29.39.92.39 1.22 0l3.85-4.95c.29-.39.29-1.02 0-1.41-.29-.4-.93-.4-1.23 0z'
        }
      },
      {
        path: '/restocking',
        labelKey: 'restocking',
        label: 'Restocking',
        icon: {
          viewBox: '0 0 24 24',
          d: 'M7.25 13.27a.75.75 0 0 0-1.06 0l-.97.97V9.5a.75.75 0 0 0-1.5 0v4.74l-.97-.97a.75.75 0 1 0-1.06 1.06l2.5 2.5a.75.75 0 0 0 1.06 0l2.5-2.5a.75.75 0 0 0 0-1.06z'
        }
      }
    ]

    return {
      t,
      navItems
    }
  }
}
</script>

<style scoped>
.sidebar {
  width: 220px;
  height: calc(100vh - 70px);
  background: #0f172a;
  border-right: 1px solid #1e293b;
  position: sticky;
  top: 70px;
  align-self: flex-start;
  display: flex;
  flex-direction: column;
  transition: width 0.25s ease;
  overflow: hidden;
}

.sidebar.collapsed {
  width: 60px;
}

.sidebar-nav {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  padding: 1rem 0.5rem;
  overflow-y: auto;
  overflow-x: hidden;
}

.nav-item {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.75rem;
  color: #94a3b8;
  text-decoration: none;
  border-radius: 6px;
  transition: all 0.2s ease;
  position: relative;
  border-right: 3px solid transparent;
  white-space: nowrap;
}

.nav-item:hover {
  background: rgba(255, 255, 255, 0.08);
  color: #cbd5e1;
}

.nav-item.active {
  background: #eff6ff;
  color: #2563eb;
  border-right-color: #2563eb;
}

.nav-icon {
  width: 24px;
  height: 24px;
  flex-shrink: 0;
  stroke: currentColor;
  stroke-width: 1.5;
  fill: none;
}

.sidebar.collapsed .nav-icon {
  stroke-width: 2;
}

.nav-label {
  font-size: 0.875rem;
  font-weight: 500;
  opacity: 1;
  transition: opacity 0.25s ease, width 0.25s ease;
  width: auto;
  overflow: hidden;
}

.sidebar.collapsed .nav-label {
  opacity: 0;
  width: 0;
}

.sidebar-toggle {
  padding: 0.75rem;
  margin: 0.5rem;
  background: rgba(255, 255, 255, 0.05);
  border: 1px solid rgba(255, 255, 255, 0.1);
  border-radius: 6px;
  color: #94a3b8;
  cursor: pointer;
  transition: all 0.2s ease;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.sidebar-toggle:hover {
  background: rgba(255, 255, 255, 0.1);
  color: #cbd5e1;
}

.toggle-icon {
  width: 20px;
  height: 20px;
  stroke: currentColor;
  stroke-width: 2;
  fill: none;
  transition: transform 0.25s ease;
}

.sidebar.collapsed .toggle-icon {
  transform: rotate(180deg);
}

/* Scrollbar styling */
.sidebar-nav::-webkit-scrollbar {
  width: 6px;
}

.sidebar-nav::-webkit-scrollbar-track {
  background: transparent;
}

.sidebar-nav::-webkit-scrollbar-thumb {
  background: rgba(255, 255, 255, 0.2);
  border-radius: 3px;
}

.sidebar-nav::-webkit-scrollbar-thumb:hover {
  background: rgba(255, 255, 255, 0.3);
}
</style>
