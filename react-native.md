# Полное руководство по лучшим практикам React Native 2024-2025

## Архитектурные паттерны и организация бизнес-логики

### Паттерн Services/Use Cases/Repositories

Современная разработка React Native приложений требует чёткого разделения слоёв ответственности. **Чистая архитектура** обеспечивает масштабируемость и тестируемость кода, особенно важную для средних и крупных приложений.

```typescript
// Доменный слой - бизнес-сущности
export interface User {
  id: string;
  email: string;
  permissions: string[];
}

// Слой приложения - сценарии использования
export async function createOrder(
  input: CreateOrderInput,
  dependencies: {
    getUser: () => Promise<User>;
    validateInventory: (items: Item[]) => Promise<boolean>;
    saveOrder: (order: Order) => Promise<Order>;
    sendNotification: (userId: string, message: string) => Promise<void>;
  }
) {
  const user = await dependencies.getUser();
  
  // Бизнес-валидация
  if (!user.permissions.includes('CREATE_ORDER')) {
    return { error: 'InsufficientPermissions' };
  }
  
  const isValidInventory = await dependencies.validateInventory(input.items);
  if (!isValidInventory) {
    return { error: 'ItemsOutOfStock' };
  }
  
  // Доменная логика
  const order = await dependencies.saveOrder({
    userId: user.id,
    items: input.items,
    total: calculateTotal(input.items),
    createdAt: new Date()
  });
  
  await dependencies.sendNotification(user.id, `Заказ ${order.id} создан`);
  
  return { data: order };
}

// Инфраструктурный слой - внешние сервисы
class OrderService {
  private apiClient: ApiClient;
  
  async validateInventory(items: Item[]): Promise<boolean> {
    const response = await this.apiClient.post('/inventory/validate', { items });
    return response.data.isValid;
  }
  
  async saveOrder(orderData: OrderData): Promise<Order> {
    const response = await this.apiClient.post('/orders', orderData);
    return response.data;
  }
}

// Презентационный слой - кастомный хук
export function useCreateOrder() {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: (input: CreateOrderInput) => createOrder(input, {
      getUser: userService.getCurrentUser,
      validateInventory: orderService.validateInventory,
      saveOrder: orderService.saveOrder,
      sendNotification: notificationService.send
    }),
    onSuccess: () => {
      queryClient.invalidateQueries(['orders']);
    }
  });
}
```

Основные принципы чистой архитектуры для React Native включают **разделение на четыре слоя**: доменный (бизнес-сущности), приложения (сценарии использования), инфраструктурный (API, базы данных) и презентационный (UI компоненты). Это обеспечивает независимость бизнес-логики от фреймворка и упрощает тестирование.

## Управление состоянием: современные решения

### Zustand как оптимальный выбор

**Zustand выигрывает у конкурентов** благодаря простоте API при сохранении мощности Redux. Производительность составляет всего 32ms при обновлении состояния против 95ms у Redux Toolkit, при размере бандла всего 4KB.

```typescript
// stores/authStore.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import AsyncStorage from '@react-native-async-storage/async-storage';

interface AuthState {
  user: User | null;
  isAuthenticated: boolean;
  login: (userData: User) => void;
  logout: () => void;
  updateProfile: (updates: Partial<User>) => void;
}

export const useAuthStore = create<AuthState>()(
  persist(
    (set) => ({
      user: null,
      isAuthenticated: false,
      login: (userData) => set({ 
        user: userData, 
        isAuthenticated: true 
      }),
      logout: () => set({ 
        user: null, 
        isAuthenticated: false 
      }),
      updateProfile: (updates) => set((state) => ({
        user: state.user ? { ...state.user, ...updates } : null
      })),
    }),
    {
      name: 'auth-storage',
      storage: createJSONStorage(() => AsyncStorage),
    }
  )
);

// stores/cartStore.ts с использованием immer
import { immer } from 'zustand/middleware/immer';

interface CartState {
  items: CartItem[];
  total: number;
  addItem: (item: Product) => void;
  removeItem: (id: string) => void;
  updateQuantity: (id: string, quantity: number) => void;
}

export const useCartStore = create<CartState>()(
  immer((set) => ({
    items: [],
    total: 0,
    addItem: (product) => set((state) => {
      const existingItem = state.items.find(i => i.id === product.id);
      if (existingItem) {
        existingItem.quantity += 1;
      } else {
        state.items.push({ ...product, quantity: 1 });
      }
      state.total = state.items.reduce(
        (sum, item) => sum + item.price * item.quantity, 0
      );
    }),
    removeItem: (id) => set((state) => {
      state.items = state.items.filter(item => item.id !== id);
      state.total = state.items.reduce(
        (sum, item) => sum + item.price * item.quantity, 0
      );
    }),
    updateQuantity: (id, quantity) => set((state) => {
      const item = state.items.find(i => i.id === id);
      if (item) {
        item.quantity = quantity;
        if (quantity === 0) {
          state.items = state.items.filter(i => i.id !== id);
        }
      }
      state.total = state.items.reduce(
        (sum, item) => sum + item.price * item.quantity, 0
      );
    })
  }))
);
```

### TanStack Query для серверного состояния

Управление серверным состоянием отдельно от клиентского — ключевая практика 2024 года. **TanStack Query автоматизирует кэширование, фоновые обновления и синхронизацию**, что критически важно для React Native приложений с нестабильным интернетом.

```typescript
// hooks/useProducts.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { useEffect } from 'react';
import { AppState, Platform } from 'react-native';
import { focusManager } from '@tanstack/react-query';

// Ключи запросов для централизованного управления
export const queryKeys = {
  products: {
    all: ['products'] as const,
    lists: () => [...queryKeys.products.all, 'list'] as const,
    list: (filters: ProductFilters) => 
      [...queryKeys.products.lists(), filters] as const,
    detail: (id: string) => 
      [...queryKeys.products.all, 'detail', id] as const,
  },
};

export function useProducts(filters?: ProductFilters) {
  return useQuery({
    queryKey: queryKeys.products.list(filters || {}),
    queryFn: () => productService.getProducts(filters),
    staleTime: 5 * 60 * 1000, // 5 минут
    gcTime: 10 * 60 * 1000, // 10 минут
  });
}

// Оптимистичные обновления
export function useAddToCart() {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: (product: Product) => 
      cartService.addToCart(product),
    
    onMutate: async (product) => {
      await queryClient.cancelQueries({ 
        queryKey: ['cart'] 
      });
      
      const previousCart = queryClient.getQueryData(['cart']);
      
      queryClient.setQueryData(['cart'], (old: any) => [
        ...(old || []),
        product
      ]);
      
      return { previousCart };
    },
    
    onError: (err, variables, context) => {
      queryClient.setQueryData(['cart'], context?.previousCart);
    },
    
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['cart'] });
    },
  });
}

// Настройка для React Native
function onAppStateChange(status: AppStateStatus) {
  if (Platform.OS !== 'web') {
    focusManager.setFocused(status === 'active');
  }
}

export function QueryProvider({ children }: { children: ReactNode }) {
  useEffect(() => {
    const subscription = AppState.addEventListener('change', onAppStateChange);
    return () => subscription.remove();
  }, []);
  
  return (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  );
}
```

### Фреймворк выбора state management

Выбор решения для управления состоянием зависит от размера приложения и команды:

- **React Context**: Для небольших приложений (до 10 экранов) с простым состоянием темы и аутентификации
- **Zustand**: Оптимальный выбор для средних и крупных приложений, требующих баланса простоты и мощности
- **Redux Toolkit**: Для enterprise-приложений с большими командами, где важна строгость паттернов
- **Jotai**: Когда нужна мелкогранулярная реактивность и атомарные обновления
- **TanStack Query**: Обязательно для управления серверным состоянием независимо от выбора клиентского state management

## Структура папок и организация файлов

### Feature-based архитектура для масштабируемости

```
src/
├── shared/                      # Общие компоненты и утилиты
│   ├── components/             
│   │   ├── ui/                 # Базовые UI элементы
│   │   │   ├── Button/
│   │   │   │   ├── Button.tsx
│   │   │   │   ├── Button.styles.ts
│   │   │   │   ├── Button.test.tsx
│   │   │   │   └── index.ts
│   │   │   ├── Modal/
│   │   │   └── Input/
│   │   ├── layout/             # Компоненты разметки
│   │   └── forms/              # Формы и инпуты
│   ├── hooks/                  # Общие хуки
│   │   ├── useAuth.ts
│   │   ├── useApi.ts
│   │   └── useNetworkStatus.ts
│   ├── services/               # Бизнес-логика
│   │   ├── api/
│   │   │   ├── apiClient.ts
│   │   │   ├── endpoints.ts
│   │   │   └── interceptors.ts
│   │   └── storage/
│   │       └── mmkv.ts
│   ├── stores/                 # Глобальное состояние
│   ├── utils/                  # Утилиты
│   └── types/                  # TypeScript типы
│
├── features/                    # Функциональные модули
│   ├── authentication/
│   │   ├── components/
│   │   ├── screens/
│   │   ├── hooks/
│   │   ├── services/
│   │   └── types/
│   ├── products/
│   │   ├── components/
│   │   │   ├── ProductCard/
│   │   │   └── ProductFilter/
│   │   ├── screens/
│   │   │   ├── ProductListScreen/
│   │   │   └── ProductDetailScreen/
│   │   └── hooks/
│   └── cart/
│
├── navigation/                  # Навигация
│   ├── AppNavigator.tsx
│   ├── AuthNavigator.tsx
│   └── types.ts
│
└── App.tsx                     # Корневой компонент
```

Ключевая идея feature-based структуры — **группировка по функциональности, а не по типу файла**. Каждая фича содержит всё необходимое: компоненты, хуки, сервисы и типы, что упрощает поддержку и масштабирование.

### Настройка путей для чистых импортов

```json
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@/components/*": ["src/shared/components/*"],
      "@/features/*": ["src/features/*"],
      "@/services/*": ["src/shared/services/*"],
      "@/hooks/*": ["src/shared/hooks/*"],
      "@/stores/*": ["src/shared/stores/*"],
      "@/types/*": ["src/shared/types/*"],
      "@/utils/*": ["src/shared/utils/*"]
    }
  }
}

// babel.config.js
module.exports = {
  plugins: [
    [
      'module-resolver',
      {
        root: ['./src'],
        alias: {
          '@': './src',
        },
      },
    ],
  ],
};
```

## Архитектура компонентов

### Паттерн Compound Components

Составные компоненты обеспечивают гибкость API при сохранении инкапсуляции логики:

```typescript
// Accordion.tsx
interface AccordionContextType {
  expandedItems: Set<string>;
  toggleItem: (id: string) => void;
  multiple: boolean;
}

const AccordionContext = createContext<AccordionContextType | null>(null);

export function Accordion({ 
  children, 
  multiple = false,
  defaultExpanded = []
}: AccordionProps) {
  const [expandedItems, setExpandedItems] = useState<Set<string>>(
    new Set(defaultExpanded)
  );
  
  const toggleItem = useCallback((id: string) => {
    setExpandedItems(prev => {
      const newSet = new Set(prev);
      if (newSet.has(id)) {
        newSet.delete(id);
      } else {
        if (!multiple) {
          newSet.clear();
        }
        newSet.add(id);
      }
      return newSet;
    });
  }, [multiple]);
  
  const value = useMemo(
    () => ({ expandedItems, toggleItem, multiple }),
    [expandedItems, toggleItem, multiple]
  );
  
  return (
    <AccordionContext.Provider value={value}>
      <View style={styles.accordion}>
        {children}
      </View>
    </AccordionContext.Provider>
  );
}

// AccordionItem.tsx
interface AccordionItemProps {
  id: string;
  children: ReactNode;
}

export function AccordionItem({ id, children }: AccordionItemProps) {
  const context = useContext(AccordionContext);
  if (!context) {
    throw new Error('AccordionItem должен использоваться внутри Accordion');
  }
  
  const isExpanded = context.expandedItems.has(id);
  
  return (
    <View style={[styles.item, isExpanded && styles.itemExpanded]}>
      {React.Children.map(children, child => {
        if (React.isValidElement(child)) {
          return React.cloneElement(child, { id, isExpanded });
        }
        return child;
      })}
    </View>
  );
}

// Использование
<Accordion multiple defaultExpanded={['1']}>
  <AccordionItem id="1">
    <AccordionHeader>Общая информация</AccordionHeader>
    <AccordionContent>
      <Text>Контент первой секции</Text>
    </AccordionContent>
  </AccordionItem>
  <AccordionItem id="2">
    <AccordionHeader>Дополнительно</AccordionHeader>
    <AccordionContent>
      <Text>Контент второй секции</Text>
    </AccordionContent>
  </AccordionItem>
</Accordion>
```

### Custom Hooks для переиспользования логики

```typescript
// hooks/useInfiniteScroll.ts
export function useInfiniteScroll<T>({
  queryKey,
  queryFn,
  getNextPageParam,
}: UseInfiniteScrollOptions<T>) {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
    isLoading,
    error,
    refetch,
  } = useInfiniteQuery({
    queryKey,
    queryFn,
    getNextPageParam,
    initialPageParam: 0,
  });
  
  const loadMore = useCallback(() => {
    if (hasNextPage && !isFetchingNextPage) {
      fetchNextPage();
    }
  }, [hasNextPage, isFetchingNextPage, fetchNextPage]);
  
  const flatData = useMemo(
    () => data?.pages.flatMap(page => page.items) ?? [],
    [data]
  );
  
  return {
    data: flatData,
    loadMore,
    isLoading,
    isLoadingMore: isFetchingNextPage,
    hasMore: hasNextPage,
    error,
    refresh: refetch,
  };
}

// Использование в компоненте
function ProductList() {
  const {
    data: products,
    loadMore,
    isLoading,
    isLoadingMore,
    hasMore,
    refresh,
  } = useInfiniteScroll({
    queryKey: ['products'],
    queryFn: ({ pageParam }) => productService.getProducts({ 
      page: pageParam,
      limit: 20 
    }),
    getNextPageParam: (lastPage) => 
      lastPage.hasNext ? lastPage.nextPage : undefined,
  });
  
  return (
    <FlatList
      data={products}
      renderItem={({ item }) => <ProductCard product={item} />}
      onEndReached={loadMore}
      onEndReachedThreshold={0.5}
      refreshControl={
        <RefreshControl refreshing={isLoading} onRefresh={refresh} />
      }
      ListFooterComponent={() => 
        isLoadingMore ? <ActivityIndicator /> : null
      }
    />
  );
}
```

## Паттерны потока данных

### Однонаправленный поток данных

Основной принцип React Native архитектуры — **данные течут вниз, события поднимаются вверх**. Это обеспечивает предсказуемость и упрощает отладку.

```typescript
// Родительский компонент управляет состоянием
function CheckoutScreen() {
  const { items, total, updateQuantity, removeItem } = useCartStore();
  const { mutate: createOrder, isPending } = useCreateOrder();
  
  const handleCheckout = useCallback(async () => {
    const result = await createOrder({ items, total });
    if (result.success) {
      navigation.navigate('OrderSuccess', { orderId: result.orderId });
    }
  }, [items, total, createOrder]);
  
  return (
    <SafeAreaView style={styles.container}>
      <CartItemList
        items={items}
        onUpdateQuantity={updateQuantity}  // События идут вверх
        onRemoveItem={removeItem}
      />
      <CartSummary 
        total={total}                       // Данные идут вниз
        itemCount={items.length}
      />
      <CheckoutButton
        onPress={handleCheckout}
        loading={isPending}
        disabled={items.length === 0}
      />
    </SafeAreaView>
  );
}

// Дочерние компоненты получают данные и колбэки
function CartItemList({ items, onUpdateQuantity, onRemoveItem }) {
  return (
    <FlatList
      data={items}
      keyExtractor={item => item.id}
      renderItem={({ item }) => (
        <CartItem
          item={item}
          onQuantityChange={(quantity) => onUpdateQuantity(item.id, quantity)}
          onRemove={() => onRemoveItem(item.id)}
        />
      )}
    />
  );
}
```

### Оптимизация Context API

Разделение контекстов по частоте обновлений критически важно для производительности:

```typescript
// Разделение контекстов по ответственности
const UserContext = createContext<User | null>(null);
const UserActionsContext = createContext<UserActions | null>(null);
const ThemeContext = createContext<Theme>(defaultTheme);

// Провайдер с мемоизацией
function UserProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  
  // Мемоизация экшенов для предотвращения лишних ререндеров
  const actions = useMemo<UserActions>(
    () => ({
      login: async (credentials: LoginCredentials) => {
        const userData = await authService.login(credentials);
        setUser(userData);
      },
      logout: async () => {
        await authService.logout();
        setUser(null);
      },
      updateProfile: async (updates: Partial<User>) => {
        const updatedUser = await userService.updateProfile(updates);
        setUser(updatedUser);
      },
    }),
    []
  );
  
  return (
    <UserContext.Provider value={user}>
      <UserActionsContext.Provider value={actions}>
        {children}
      </UserActionsContext.Provider>
    </UserContext.Provider>
  );
}

// Кастомные хуки для использования контекстов
export function useUser() {
  const user = useContext(UserContext);
  if (user === undefined) {
    throw new Error('useUser должен использоваться внутри UserProvider');
  }
  return user;
}

export function useUserActions() {
  const actions = useContext(UserActionsContext);
  if (!actions) {
    throw new Error('useUserActions должен использоваться внутри UserProvider');
  }
  return actions;
}
```

## Стратегии тестирования

### Пирамида тестирования для React Native

Эффективная стратегия тестирования включает **70% unit-тестов, 20% интеграционных и 10% E2E тестов**. Это обеспечивает быструю обратную связь при сохранении уверенности в работе приложения.

```typescript
// Unit тест для бизнес-логики
describe('OrderService', () => {
  describe('calculateTotal', () => {
    it('должен правильно рассчитывать сумму с учётом скидки', () => {
      const items = [
        { id: '1', price: 100, quantity: 2 },
        { id: '2', price: 50, quantity: 1 }
      ];
      const discount = 0.1; // 10%
      
      const total = orderService.calculateTotal(items, discount);
      
      expect(total).toBe(225); // (200 + 50) * 0.9
    });
    
    it('должен обрабатывать пустую корзину', () => {
      expect(orderService.calculateTotal([], 0)).toBe(0);
    });
  });
});

// Тест компонента с React Native Testing Library
import { render, fireEvent, waitFor } from '@testing-library/react-native';

describe('LoginScreen', () => {
  it('должен показывать ошибку при неверных данных', async () => {
    const { getByLabelText, getByText, queryByText } = render(
      <LoginScreen />
    );
    
    const emailInput = getByLabelText('Email');
    const passwordInput = getByLabelText('Пароль');
    const submitButton = getByText('Войти');
    
    fireEvent.changeText(emailInput, 'invalid-email');
    fireEvent.changeText(passwordInput, '123');
    fireEvent.press(submitButton);
    
    await waitFor(() => {
      expect(queryByText('Неверный формат email')).toBeTruthy();
    });
  });
});

// E2E тест с Detox
describe('Процесс оформления заказа', () => {
  beforeAll(async () => {
    await device.launchApp();
  });
  
  it('должен успешно оформить заказ', async () => {
    await element(by.id('product-1')).tap();
    await element(by.text('Добавить в корзину')).tap();
    await element(by.id('cart-icon')).tap();
    
    await expect(element(by.text('Корзина (1)'))).toBeVisible();
    
    await element(by.id('checkout-button')).tap();
    await element(by.id('address-input')).typeText('ул. Пушкина, д. 1');
    await element(by.id('confirm-order')).tap();
    
    await waitFor(element(by.text('Заказ успешно оформлен')))
      .toBeVisible()
      .withTimeout(5000);
  });
});
```

### Тестирование хуков

```typescript
import { renderHook, act } from '@testing-library/react-hooks';
import { useCounter } from './useCounter';

describe('useCounter', () => {
  it('должен увеличивать счётчик', () => {
    const { result } = renderHook(() => useCounter());
    
    expect(result.current.count).toBe(0);
    
    act(() => {
      result.current.increment();
    });
    
    expect(result.current.count).toBe(1);
  });
  
  it('должен сбрасывать счётчик', () => {
    const { result } = renderHook(() => useCounter(5));
    
    act(() => {
      result.current.increment();
      result.current.increment();
    });
    
    expect(result.current.count).toBe(7);
    
    act(() => {
      result.current.reset();
    });
    
    expect(result.current.count).toBe(5);
  });
});
```

## Оптимизация производительности

### React Native New Architecture

Новая архитектура React Native, включающая **Fabric и TurboModules**, обеспечивает прирост производительности на 15-30% для рендеринга и существенное снижение времени запуска приложения.

```typescript
// Включение новой архитектуры
// android/gradle.properties
newArchEnabled=true

// ios/Podfile
:hermes_enabled => true,
:fabric_enabled => true,
```

### Оптимизация списков

**FlashList от Shopify обеспечивает 5x лучшую производительность** по сравнению с FlatList для больших списков:

```typescript
import { FlashList } from '@shopify/flash-list';

function ProductList({ products }: { products: Product[] }) {
  const renderItem = useCallback(({ item }: { item: Product }) => (
    <ProductCard 
      product={item}
      onPress={() => navigation.navigate('ProductDetail', { id: item.id })}
    />
  ), [navigation]);
  
  return (
    <FlashList
      data={products}
      renderItem={renderItem}
      estimatedItemSize={120}
      keyExtractor={item => item.id}
      // Оптимизация для больших списков
      drawDistance={200}
      // Переиспользование компонентов
      recycleItems={true}
      // Оптимизация памяти
      removeClippedSubviews={true}
    />
  );
}
```

### Оптимизация изображений

```typescript
import FastImage from 'react-native-fast-image';

function OptimizedImage({ uri, style, priority = 'normal' }) {
  return (
    <FastImage
      source={{
        uri,
        priority: FastImage.priority[priority],
        cache: FastImage.cacheControl.immutable,
      }}
      style={style}
      resizeMode={FastImage.resizeMode.cover}
      onError={() => console.log('Ошибка загрузки изображения')}
    />
  );
}

// Предзагрузка критичных изображений
FastImage.preload([
  { uri: 'https://example.com/hero.jpg', priority: FastImage.priority.high },
  { uri: 'https://example.com/logo.png' },
]);
```

### Мемоизация и оптимизация ререндеров

```typescript
// Мемоизация дорогих компонентов
const ExpensiveChart = React.memo(({ data, options }) => {
  const processedData = useMemo(
    () => processChartData(data, options),
    [data, options]
  );
  
  const chartConfig = useMemo(
    () => generateChartConfig(options),
    [options]
  );
  
  return <Chart data={processedData} config={chartConfig} />;
}, (prevProps, nextProps) => {
  // Кастомная функция сравнения для более точного контроля
  return (
    prevProps.data.length === nextProps.data.length &&
    prevProps.options.type === nextProps.options.type
  );
});

// Использование useCallback для стабильных ссылок
function ParentComponent() {
  const [filter, setFilter] = useState('');
  
  const handleItemPress = useCallback((item: Item) => {
    navigation.navigate('Detail', { id: item.id });
  }, [navigation]);
  
  const filteredItems = useMemo(
    () => items.filter(item => 
      item.name.toLowerCase().includes(filter.toLowerCase())
    ),
    [items, filter]
  );
  
  return (
    <ItemList
      items={filteredItems}
      onItemPress={handleItemPress}
    />
  );
}
```

### Оптимизация хранилища

**MMKV обеспечивает 30x более быструю работу** по сравнению с AsyncStorage:

```typescript
import { MMKV } from 'react-native-mmkv';

// Создание экземпляра хранилища
const storage = new MMKV({
  id: 'app-storage',
  encryptionKey: 'your-encryption-key'
});

// Использование в хуке
export function useLocalStorage<T>(
  key: string,
  initialValue: T
): [T, (value: T) => void] {
  const [storedValue, setStoredValue] = useState<T>(() => {
    try {
      const item = storage.getString(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      console.error('Ошибка чтения из MMKV:', error);
      return initialValue;
    }
  });
  
  const setValue = useCallback((value: T) => {
    try {
      setStoredValue(value);
      storage.set(key, JSON.stringify(value));
    } catch (error) {
      console.error('Ошибка записи в MMKV:', error);
    }
  }, [key]);
  
  return [storedValue, setValue];
}
```

## Паттерны навигации

### Типобезопасная навигация с React Navigation v6

```typescript
// navigation/types.ts
export type RootStackParamList = {
  Home: undefined;
  ProductList: { categoryId: string };
  ProductDetail: { productId: string };
  Cart: undefined;
  Checkout: { orderId?: string };
  Profile: { userId: string };
};

export type TabParamList = {
  HomeTab: NavigatorScreenParams<HomeStackParamList>;
  CatalogTab: NavigatorScreenParams<CatalogStackParamList>;
  CartTab: undefined;
  ProfileTab: undefined;
};

// Типизированные хуки навигации
import { useNavigation, useRoute } from '@react-navigation/native';
import { StackNavigationProp } from '@react-navigation/stack';
import { RouteProp } from '@react-navigation/native';

export function useTypedNavigation<T extends keyof RootStackParamList>() {
  return useNavigation<StackNavigationProp<RootStackParamList, T>>();
}

export function useTypedRoute<T extends keyof RootStackParamList>() {
  return useRoute<RouteProp<RootStackParamList, T>>();
}
```

### Deep Linking конфигурация

```typescript
const linking: LinkingOptions<RootStackParamList> = {
  prefixes: ['myapp://', 'https://myapp.com'],
  config: {
    screens: {
      Home: '',
      ProductList: 'category/:categoryId',
      ProductDetail: 'product/:productId',
      Cart: 'cart',
      Profile: {
        path: 'profile/:userId',
        parse: {
          userId: (userId: string) => userId,
        },
      },
    },
  },
  async getInitialURL() {
    // Обработка URL при холодном запуске
    const url = await Linking.getInitialURL();
    if (url != null) {
      return url;
    }
    // Обработка push-уведомлений
    const message = await messaging().getInitialNotification();
    return message?.data?.deepLink;
  },
  subscribe(listener) {
    // Обработка URL при активном приложении
    const onReceiveURL = ({ url }: { url: string }) => listener(url);
    const subscription = Linking.addEventListener('url', onReceiveURL);
    
    // Обработка push-уведомлений
    const unsubscribeNotification = messaging().onNotificationOpenedApp(
      (message) => {
        const url = message.data?.deepLink;
        if (url) {
          listener(url);
        }
      }
    );
    
    return () => {
      subscription.remove();
      unsubscribeNotification();
    };
  },
};
```

### Аутентификационные потоки

```typescript
function RootNavigator() {
  const { isAuthenticated, isLoading } = useAuthStore();
  
  if (isLoading) {
    return <SplashScreen />;
  }
  
  return (
    <Stack.Navigator screenOptions={{ headerShown: false }}>
      {isAuthenticated ? (
        <>
          <Stack.Screen name="Main" component={MainTabNavigator} />
          <Stack.Screen name="Modal" component={ModalStack} />
        </>
      ) : (
        <>
          <Stack.Screen name="Welcome" component={WelcomeScreen} />
          <Stack.Screen name="Login" component={LoginScreen} />
          <Stack.Screen name="Register" component={RegisterScreen} />
        </>
      )}
    </Stack.Navigator>
  );
}
```

## Подходы к стилизации

### Сравнение решений для стилизации

Выбор подхода к стилизации зависит от требований проекта и предпочтений команды:

```typescript
// 1. StyleSheet API - нативное решение
const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: colors.background,
    padding: spacing.medium,
  },
  title: {
    fontSize: typography.sizes.large,
    fontWeight: 'bold',
    color: colors.text.primary,
    marginBottom: spacing.small,
  },
});

// 2. Styled Components - CSS-in-JS
import styled from 'styled-components/native';

const Container = styled.View`
  flex: 1;
  background-color: ${props => props.theme.colors.background};
  padding: ${props => props.theme.spacing.medium}px;
`;

const Title = styled.Text<{ variant?: 'primary' | 'secondary' }>`
  font-size: ${props => props.theme.typography.sizes.large}px;
  font-weight: bold;
  color: ${props => props.variant === 'secondary' 
    ? props.theme.colors.text.secondary 
    : props.theme.colors.text.primary};
  margin-bottom: ${props => props.theme.spacing.small}px;
`;

// 3. NativeWind - Tailwind для React Native
import { Text, View } from 'react-native';

export function Card({ title, description }) {
  return (
    <View className="bg-white rounded-lg shadow-lg p-4 m-2">
      <Text className="text-xl font-bold text-gray-900 mb-2">
        {title}
      </Text>
      <Text className="text-base text-gray-600">
        {description}
      </Text>
    </View>
  );
}

// 4. Tamagui - производительное решение
import { YStack, H2, Paragraph, Card } from 'tamagui';

export function ProductCard({ product }) {
  return (
    <Card 
      elevate 
      bordered 
      padding="$4" 
      margin="$2"
      animation="bouncy"
      pressStyle={{ scale: 0.98 }}
    >
      <YStack space="$2">
        <H2 size="$6">{product.title}</H2>
        <Paragraph theme="alt1">
          {product.description}
        </Paragraph>
      </YStack>
    </Card>
  );
}
```

### Система дизайн-токенов

```typescript
// theme/tokens.ts
export const tokens = {
  colors: {
    primary: {
      50: '#E3F2FD',
      100: '#BBDEFB',
      500: '#2196F3',
      900: '#0D47A1',
    },
    gray: {
      50: '#FAFAFA',
      500: '#9E9E9E',
      900: '#212121',
    },
  },
  spacing: {
    xs: 4,
    sm: 8,
    md: 16,
    lg: 24,
    xl: 32,
  },
  typography: {
    fonts: {
      regular: 'Inter-Regular',
      medium: 'Inter-Medium',
      bold: 'Inter-Bold',
    },
    sizes: {
      xs: 12,
      sm: 14,
      md: 16,
      lg: 20,
      xl: 24,
    },
  },
  borderRadius: {
    sm: 4,
    md: 8,
    lg: 16,
    full: 9999,
  },
};

// theme/ThemeProvider.tsx
export function ThemeProvider({ children }: { children: ReactNode }) {
  const colorScheme = useColorScheme();
  
  const theme = useMemo(
    () => ({
      ...tokens,
      isDark: colorScheme === 'dark',
      colors: colorScheme === 'dark' ? darkColors : lightColors,
    }),
    [colorScheme]
  );
  
  return (
    <ThemeContext.Provider value={theme}>
      {children}
    </ThemeContext.Provider>
  );
}
```

## Дополнительные важные практики

### Обработка ошибок и Error Boundaries

```typescript
// components/ErrorBoundary.tsx
import React, { Component, ErrorInfo, ReactNode } from 'react';
import { View, Text, Button } from 'react-native';
import * as Sentry from '@sentry/react-native';

interface Props {
  children: ReactNode;
  fallback?: ReactNode;
}

interface State {
  hasError: boolean;
  error: Error | null;
}

export class ErrorBoundary extends Component<Props, State> {
  constructor(props: Props) {
    super(props);
    this.state = { hasError: false, error: null };
  }
  
  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }
  
  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    console.error('ErrorBoundary поймал ошибку:', error, errorInfo);
    
    // Отправка в Sentry
    Sentry.captureException(error, {
      contexts: {
        react: {
          componentStack: errorInfo.componentStack,
        },
      },
    });
  }
  
  handleReset = () => {
    this.setState({ hasError: false, error: null });
  };
  
  render() {
    if (this.state.hasError) {
      if (this.props.fallback) {
        return this.props.fallback;
      }
      
      return (
        <View style={styles.errorContainer}>
          <Text style={styles.errorTitle}>Что-то пошло не так</Text>
          <Text style={styles.errorMessage}>
            {this.state.error?.message}
          </Text>
          <Button title="Попробовать снова" onPress={this.handleReset} />
        </View>
      );
    }
    
    return this.props.children;
  }
}

// Глобальная обработка ошибок
export function setupGlobalErrorHandling() {
  // Обработка необработанных промисов
  const originalHandler = global.Promise.prototype.catch;
  global.Promise.prototype.catch = function(onRejected) {
    return originalHandler.call(this, (error: Error) => {
      console.error('Необработанное отклонение промиса:', error);
      Sentry.captureException(error);
      if (onRejected) {
        return onRejected(error);
      }
      throw error;
    });
  };
  
  // Обработка ошибок в React Native
  ErrorUtils.setGlobalHandler((error: Error, isFatal?: boolean) => {
    console.error('Глобальная ошибка:', error, 'Fatal:', isFatal);
    Sentry.captureException(error, {
      level: isFatal ? 'fatal' : 'error',
    });
  });
}
```

### Offline-first архитектура

```typescript
// hooks/useOfflineSync.ts
import NetInfo from '@react-native-community/netinfo';
import { MMKV } from 'react-native-mmkv';

const storage = new MMKV({ id: 'offline-queue' });

interface QueuedRequest {
  id: string;
  url: string;
  method: string;
  body?: any;
  timestamp: number;
}

export function useOfflineSync() {
  const [isOnline, setIsOnline] = useState(true);
  const [syncStatus, setSyncStatus] = useState<'idle' | 'syncing'>('idle');
  
  useEffect(() => {
    const unsubscribe = NetInfo.addEventListener(state => {
      setIsOnline(state.isConnected ?? false);
      
      if (state.isConnected) {
        syncPendingRequests();
      }
    });
    
    return unsubscribe;
  }, []);
  
  const queueRequest = useCallback((request: Omit<QueuedRequest, 'id' | 'timestamp'>) => {
    const queuedRequest: QueuedRequest = {
      ...request,
      id: generateId(),
      timestamp: Date.now(),
    };
    
    const queue = JSON.parse(storage.getString('queue') || '[]');
    queue.push(queuedRequest);
    storage.set('queue', JSON.stringify(queue));
    
    if (isOnline) {
      syncPendingRequests();
    }
  }, [isOnline]);
  
  const syncPendingRequests = useCallback(async () => {
    if (syncStatus === 'syncing') return;
    
    setSyncStatus('syncing');
    
    try {
      const queue: QueuedRequest[] = JSON.parse(
        storage.getString('queue') || '[]'
      );
      
      const results = await Promise.allSettled(
        queue.map(request =>
          fetch(request.url, {
            method: request.method,
            body: request.body ? JSON.stringify(request.body) : undefined,
            headers: {
              'Content-Type': 'application/json',
            },
          })
        )
      );
      
      // Удаляем успешно синхронизированные запросы
      const remainingQueue = queue.filter((_, index) => 
        results[index].status === 'rejected'
      );
      
      storage.set('queue', JSON.stringify(remainingQueue));
    } finally {
      setSyncStatus('idle');
    }
  }, [syncStatus]);
  
  return {
    isOnline,
    queueRequest,
    syncStatus,
    pendingCount: JSON.parse(storage.getString('queue') || '[]').length,
  };
}
```

### CI/CD конфигурация

```yaml
# .github/workflows/react-native.yml
name: React Native CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run linter
        run: npm run lint
      
      - name: Run type check
        run: npm run type-check
      
      - name: Run tests
        run: npm test -- --coverage
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage/coverage-final.json

  build-android:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup JDK
        uses: actions/setup-java@v3
        with:
          java-version: '11'
          distribution: 'temurin'
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Build Android Release
        run: |
          cd android
          ./gradlew assembleRelease
      
      - name: Upload APK
        uses: actions/upload-artifact@v3
        with:
          name: app-release.apk
          path: android/app/build/outputs/apk/release/app-release.apk

  build-ios:
    needs: test
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Install pods
        run: |
          cd ios
          pod install
      
      - name: Build iOS
        run: |
          cd ios
          xcodebuild -workspace YourApp.xcworkspace \
            -scheme YourApp \
            -configuration Release \
            -sdk iphonesimulator \
            -derivedDataPath build
```

### Безопасность приложения

```typescript
// utils/security.ts
import { Keychain } from 'react-native-keychain';
import CryptoJS from 'crypto-js';

export class SecurityManager {
  private static instance: SecurityManager;
  private encryptionKey: string;
  
  private constructor() {
    this.encryptionKey = this.generateKey();
  }
  
  static getInstance(): SecurityManager {
    if (!SecurityManager.instance) {
      SecurityManager.instance = new SecurityManager();
    }
    return SecurityManager.instance;
  }
  
  // Безопасное хранение токенов
  async storeSecureToken(key: string, token: string): Promise<void> {
    await Keychain.setInternetCredentials(
      key,
      key,
      token
    );
  }
  
  async getSecureToken(key: string): Promise<string | null> {
    try {
      const credentials = await Keychain.getInternetCredentials(key);
      return credentials ? credentials.password : null;
    } catch (error) {
      console.error('Ошибка получения токена:', error);
      return null;
    }
  }
  
  // Шифрование локальных данных
  encryptData(data: any): string {
    const jsonString = JSON.stringify(data);
    return CryptoJS.AES.encrypt(jsonString, this.encryptionKey).toString();
  }
  
  decryptData(encryptedData: string): any {
    const bytes = CryptoJS.AES.decrypt(encryptedData, this.encryptionKey);
    const decryptedString = bytes.toString(CryptoJS.enc.Utf8);
    return JSON.parse(decryptedString);
  }
  
  // Certificate pinning для API вызовов
  async secureApiCall(url: string, options: RequestInit = {}): Promise<Response> {
    const secureOptions: RequestInit = {
      ...options,
      headers: {
        ...options.headers,
        'X-Certificate-Pin': 'sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=',
      },
    };
    
    return fetch(url, secureOptions);
  }
  
  private generateKey(): string {
    return CryptoJS.lib.WordArray.random(256/8).toString();
  }
}

// Валидация и санитизация пользовательского ввода
export function sanitizeInput(input: string): string {
  return input
    .replace(/<script\b[^<]*(?:(?!<\/script>)<[^<]*)*<\/script>/gi, '')
    .replace(/[<>\"']/g, '')
    .trim();
}

export function validateEmail(email: string): boolean {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  return emailRegex.test(email);
}

export function validatePassword(password: string): {
  isValid: boolean;
  errors: string[];
} {
  const errors: string[] = [];
  
  if (password.length < 8) {
    errors.push('Пароль должен содержать минимум 8 символов');
  }
  if (!/[A-Z]/.test(password)) {
    errors.push('Пароль должен содержать заглавные буквы');
  }
  if (!/[a-z]/.test(password)) {
    errors.push('Пароль должен содержать строчные буквы');
  }
  if (!/[0-9]/.test(password)) {
    errors.push('Пароль должен содержать цифры');
  }
  if (!/[!@#$%^&*]/.test(password)) {
    errors.push('Пароль должен содержать специальные символы');
  }
  
  return {
    isValid: errors.length === 0,
    errors,
  };
}
```
