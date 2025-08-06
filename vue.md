# Vue.js Style Guide

This style guide adapts core programming principles to Vue.js 3 with Composition API, using JavaScript + JSDoc for type safety in PhpStorm.

## Core Principles

### 1. Store Repeating Values as Constants

Use centralized configuration and environment variables appropriately.

```javascript
// Good: Centralized constants
// constants/app.js
export const APP_CONFIG = {
    API_ENDPOINTS: {
        USERS: '/api/users',
        POSTS: '/api/posts'
    },
    PAGINATION: {
        DEFAULT_PAGE_SIZE: 20,
        MAX_PAGE_SIZE: 100
    },
    THEMES: {
        LIGHT: 'light',
        DARK: 'dark'
    }
}

// Good: Environment variables with Vite
const API_BASE_URL = import.meta.env.VITE_API_BASE_URL

    // Bad: Hardcoded values in components
    <template>
    <div v-if="status === 'active'">...</div> <!-- Don't do this -->
</template>
```

### 2. Reuse Code - Don't Duplicate

Extract reusable logic into composables and components.

```javascript
// Good: Reusable composable
// composables/useFetch.js
import { ref, readonly } from 'vue'

export function useFetch(url) {
    const data = ref(null)
    const error = ref(null)
    const loading = ref(false)

    const fetchData = async () => {
        loading.value = true
        error.value = null

        try {
            const response = await fetch(url)
            data.value = await response.json()
        } catch (err) {
            error.value = err
        } finally {
            loading.value = false
        }
    }

    return {
        data: readonly(data),
        error: readonly(error),
        loading: readonly(loading),
        fetchData
    }
}

// Usage in component
<script setup>
    import { useFetch } from '@/composables/useFetch'

    const { data: users, loading, fetchData } = useFetch('/api/users')
</script>
```

### 3. Don't Write Monolithic Components

Break down complex components into smaller, focused components.

```vue
<!-- Good: Modular component structure -->
<template>
  <div class="user-profile">
    <UserAvatar :user="user" />
    <UserDetails :user="user" @edit="handleEdit" />
    <UserActions
        :can-delete="canDelete"
        @delete="handleDelete"
    />
  </div>
</template>

<script setup>
  import UserAvatar from './UserAvatar.vue'
  import UserDetails from './UserDetails.vue'
  import UserActions from './UserActions.vue'
  import { useUser } from '@/composables/useUser'
  import { usePermissions } from '@/composables/usePermissions'

  const { user } = useUser()
  const { canDelete } = usePermissions()
</script>

<!-- Bad: Monolithic component with everything inline -->
```

### 4. Don't Make Things Up

Always reference official Vue.js documentation for API methods.

```javascript
// Good: Document when unsure
const handleComplexScenario = () => {
    // TODO: Research Vue 3 Suspense API for this use case
    // I don't know the proper way to handle async component boundaries
    console.warn('Feature not implemented: need to research Suspense')
}

// Good: Verify API existence
// Check Vue docs at vuejs.org/api/
// Use Vue DevTools for runtime inspection
```

### 5. Don't Invent Non-existent Methods

Use TypeScript and proper tooling to verify Vue API methods.

```javascript
// Good: Use TypeScript for compile-time checks
<script setup lang="ts">
    import { ref, computed, watch } from 'vue'
    // TypeScript will error if method doesn't exist

    // Good: Runtime verification
    if (typeof this.$refs.myComponent?.customMethod === 'function') {
    this.$refs.myComponent.customMethod()
}
</script>

// Bad: Using non-existent lifecycle hooks
onCreated(() => {}) // This doesn't exist in Vue 3
```

## Vue-Specific Best Practices

### Component Structure

```vue
<!-- Recommended component order -->
<template>
  <!-- Template content -->
</template>

<script setup lang="ts">
  // 1. Imports
  import { ref, computed, onMounted } from 'vue'
  import type { User } from '@/types'

  // 2. Props & Emits
  interface Props {
    userId: number
    showDetails?: boolean
  }

  const props = withDefaults(defineProps<Props>(), {
    showDetails: true
  })

  const emit = defineEmits<{
    update: [user: User]
    delete: [id: number]
  }>()

  // 3. Reactive data
  const user = ref<User | null>(null)
  const loading = ref(false)

  // 4. Computed properties
  const fullName = computed(() =>
      user.value ? `${user.value.firstName} ${user.value.lastName}` : ''
  )

  // 5. Methods
  const updateUser = async () => {
    // Implementation
  }

  // 6. Lifecycle hooks
  onMounted(() => {
    fetchUser()
  })
</script>

<style scoped>
  /* Component styles */
</style>
```

### Naming Conventions

- **Components**: PascalCase (`UserProfile.vue`)
- **Props**: camelCase in JS, kebab-case in templates
- **Events**: kebab-case (`@user-updated`)
- **Slots**: kebab-case (`<template #header-content>`)
- **Composables**: Start with "use" (`useUser`, `useFetch`)

### Props Validation

```javascript
// Good: Comprehensive prop validation
const props = defineProps({
    status: {
        type: String,
        required: true,
        validator: (value) => {
            return ['active', 'inactive', 'pending'].includes(value)
        }
    },
    items: {
        type: Array,
        default: () => []
    },
    user: {
        type: Object as PropType<User>,
        required: true
    }
})
```

#### Rules for When Not to Use Composables

- Tightly coupled with DOM elements — e.g., iframe, canvas, video
- Requires template refs — elements must be accessible from the template
- Event-driven logic — relies on raw DOM event handlers
- Component-specific logic — only used in a single component
- Complex initialization sequence — depends on DOM and logic order


### State Management with Pinia

```javascript
// stores/user.js
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

export const useUserStore = defineStore('user', () => {
    // State
    const currentUser = ref(null)
    const users = ref([])

    // Getters
    const isAuthenticated = computed(() => !!currentUser.value)

    // Actions
    const login = async (credentials) => {
        try {
            const response = await api.login(credentials)
            currentUser.value = response.data
        } catch (error) {
            throw error
        }
    }

    return {
        // State (readonly)
        currentUser: readonly(currentUser),
        users: readonly(users),
        // Getters
        isAuthenticated,
        // Actions
        login
    }
})
```

### Performance Optimization

```vue
<template>
  <!-- Use v-show for frequently toggled elements -->
  <div v-show="isVisible">...</div>

  <!-- Use v-if for conditionally rendered elements -->
  <ExpensiveComponent v-if="shouldRender" />

  <!-- Use key for list items -->
  <li v-for="item in items" :key="item.id">
    {{ item.name }}
  </li>

  <!-- Use v-memo for expensive renders -->
  <div v-for="item in list" :key="item.id" v-memo="[item.id, item.updated]">
    <ExpensiveItem :item="item" />
  </div>
</template>

<script setup>
  // Use shallowRef for large datasets
  import { shallowRef } from 'vue'

  const largeList = shallowRef([])

  // Use computed for derived state
  const filteredList = computed(() =>
      largeList.value.filter(item => item.active)
  )
</script>
```

### Testing

```javascript
// Component test with Vitest
import { mount } from '@vue/test-utils'
import { describe, it, expect } from 'vitest'
import UserProfile from '@/components/UserProfile.vue'

describe('UserProfile', () => {
    it('renders user name', () => {
        const wrapper = mount(UserProfile, {
            props: {
                user: { name: 'John Doe' }
            }
        })

        expect(wrapper.text()).toContain('John Doe')
    })

    it('emits update event', async () => {
        const wrapper = mount(UserProfile)

        await wrapper.find('button').trigger('click')

        expect(wrapper.emitted('update')).toBeTruthy()
    })
})
```

## Tools and Verification

- **Vue DevTools**: Runtime component inspection
- **TypeScript**: Compile-time type checking
- **Volar**: Vue language server for IDEs
- **ESLint + Prettier**: Code quality and formatting
- **Vitest**: Unit testing framework

## References

- [Vue.js Official Style Guide](https://vuejs.org/style-guide/)
- [Vue.js API Reference](https://vuejs.org/api/)
- [Vue.js Composition API](https://vuejs.org/guide/extras/composition-api-faq.html)
- [Pinia Documentation](https://pinia.vuejs.org/)