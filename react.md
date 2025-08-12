# React JS Best Practices - Спецификация для Claude Code

## 📋 Оглавление

1. [Структура проекта](#структура-проекта)
2. [Компоненты](#компоненты)
3. [Хуки (Hooks)](#хуки-hooks)
4. [Управление состоянием](#управление-состоянием)
5. [Производительность](#производительность)
6. [Стилизация](#стилизация)
7. [Тестирование](#тестирование)
8. [Обработка ошибок](#обработка-ошибок)
9. [Безопасность](#безопасность)
10. [Доступность (a11y)](#доступность-a11y)
11. [Код-стайл и форматирование](#код-стайл-и-форматирование)

## Структура проекта

### Рекомендуемая структура папок

```
src/
├── components/          # Переиспользуемые UI компоненты
│   ├── common/          # Общие компоненты (Button, Input, etc.)
│   ├── layout/          # Компоненты разметки (Header, Footer, etc.)
│   └── features/        # Компоненты функциональности
├── pages/               # Компоненты страниц
├── hooks/               # Кастомные хуки
├── services/            # API и внешние сервисы
├── store/               # Управление состоянием (Redux, Zustand, etc.)
├── utils/               # Утилиты и хелперы
├── constants/           # Константы приложения
└── assets/              # Статические ресурсы
```

### Организация компонентов

```
components/
└── Button/
    ├── index.js         # Экспорт
    ├── Button.jsx       # Компонент
    ├── Button.test.jsx  # Тесты
    └── Button.module.css # Стили
```

## Компоненты

### Используйте функциональные компоненты

```jsx
// ✅ Хорошо
const UserProfile = ({ user }) => {
  return <div>{user.name}</div>;
};

// ❌ Избегайте классовых компонентов
class UserProfile extends Component {
  render() {
    return <div>{this.props.user.name}</div>;
  }
}
```

### Именование компонентов

- Используйте PascalCase для компонентов
- Используйте описательные имена
- Префиксы для специальных типов компонентов (Layout*, Page*, etc.)

```jsx
// ✅ Хорошо
const UserProfileCard = () => {};
const LayoutHeader = () => {};
const PageHome = () => {};

// ❌ Плохо
const user_card = () => {};
const Component1 = () => {};
```

### Деструктуризация пропсов

```jsx
// ✅ Хорошо
const UserCard = ({ name, email, avatar }) => {
  return (
    <div>
      <img src={avatar} alt={name} />
      <h3>{name}</h3>
      <p>{email}</p>
    </div>
  );
};

// ❌ Избегайте
const UserCard = (props) => {
  return (
    <div>
      <h3>{props.name}</h3>
    </div>
  );
};
```

### Значения по умолчанию для пропсов

```jsx
// ✅ Хорошо - деструктуризация со значениями по умолчанию
const Button = ({ 
  variant = 'primary', 
  size = 'medium',
  disabled = false,
  onClick,
  children 
}) => {
  return (
    <button
      className={`btn btn-${variant} btn-${size}`}
      disabled={disabled}
      onClick={onClick}
    >
      {children}
    </button>
  );
};

// ✅ Альтернатива - defaultProps
Button.defaultProps = {
  variant: 'primary',
  size: 'medium',
  disabled: false
};
```

### Условный рендеринг

```jsx
// ✅ Хорошо - для простых условий
{isLoggedIn && <UserMenu />}

// ✅ Хорошо - для if/else
{isLoggedIn ? <UserMenu /> : <LoginButton />}

// ✅ Хорошо - для множественных условий
const renderContent = () => {
  if (isLoading) return <Spinner />;
  if (error) return <Error message={error} />;
  if (!data) return <Empty />;
  return <DataList data={data} />;
};
```

## Хуки (Hooks)

### Правила хуков

1. Вызывайте хуки только на верхнем уровне
2. Вызывайте хуки только из React функций

```jsx
// ✅ Хорошо
const Component = () => {
  const [count, setCount] = useState(0);
  useEffect(() => {
    // эффект
  }, []);
  
  return <div>{count}</div>;
};

// ❌ Плохо
const Component = () => {
  if (condition) {
    const [count, setCount] = useState(0); // Нельзя в условии!
  }
};
```

### Кастомные хуки

```jsx
// ✅ Хорошо - переиспользуемая логика
const useDebounce = (value, delay) => {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const handler = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => clearTimeout(handler);
  }, [value, delay]);

  return debouncedValue;
};

// Использование
const SearchInput = () => {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 500);
  
  useEffect(() => {
    if (debouncedQuery) {
      // Выполнить поиск
    }
  }, [debouncedQuery]);
};
```

### useEffect - правильное использование

```jsx
// ✅ Хорошо - четкие зависимости
useEffect(() => {
  const fetchData = async () => {
    const result = await api.getUser(userId);
    setUser(result);
  };
  
  fetchData();
}, [userId]); // Зависимость от userId

// ✅ Хорошо - cleanup функция
useEffect(() => {
  const timer = setTimeout(() => {
    console.log('Timer fired');
  }, 1000);
  
  return () => clearTimeout(timer); // Cleanup
}, []);
```

## Управление состоянием

### Локальное состояние vs Глобальное

```jsx
// ✅ Локальное состояние для UI
const ToggleButton = () => {
  const [isOpen, setIsOpen] = useState(false);
  return <button onClick={() => setIsOpen(!isOpen)}>{isOpen ? 'Close' : 'Open'}</button>;
};

// ✅ Глобальное состояние для данных приложения
// Используйте Context API, Redux, Zustand, MobX для:
// - Данных пользователя
// - Настроек приложения
// - Кэша данных
```

### Context API - правильное использование

```jsx
// ✅ Хорошо - создание контекста
const ThemeContext = createContext();
const AuthContext = createContext();

// Provider компонент
const ThemeProvider = ({ children }) => {
  const [theme, setTheme] = useState('light');
  
  const toggleTheme = () => {
    setTheme(prev => prev === 'light' ? 'dark' : 'light');
  };
  
  return (
    <ThemeContext.Provider value={{ theme, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  );
};

// Custom hook для безопасного использования
const useTheme = () => {
  const context = useContext(ThemeContext);
  if (!context) {
    throw new Error('useTheme must be used within ThemeProvider');
  }
  return context;
};
```

## Производительность

### React.memo для оптимизации

```jsx
// ✅ Хорошо - мемоизация дорогих компонентов
const ExpensiveComponent = React.memo(({ data }) => {
  return <ComplexVisualization data={data} />;
}, (prevProps, nextProps) => {
  // Кастомная функция сравнения
  return prevProps.data.id === nextProps.data.id;
});
```

### useMemo и useCallback

```jsx
// ✅ useMemo для дорогих вычислений
const Component = ({ items }) => {
  const expensiveValue = useMemo(() => {
    return items.reduce((sum, item) => sum + item.value, 0);
  }, [items]);

  // ✅ useCallback для стабильных ссылок
  const handleClick = useCallback((id) => {
    dispatch({ type: 'SELECT_ITEM', payload: id });
  }, [dispatch]);
  
  return <ItemList onItemClick={handleClick} />;
};
```

### Ленивая загрузка

```jsx
// ✅ Хорошо - code splitting
const LazyComponent = lazy(() => import('./HeavyComponent'));

const App = () => {
  return (
    <Suspense fallback={<Loading />}>
      <LazyComponent />
    </Suspense>
  );
};
```

### Виртуализация списков

```jsx
// ✅ Хорошо - для больших списков используйте react-window
import { FixedSizeList } from 'react-window';

const BigList = ({ items }) => {
  const Row = ({ index, style }) => (
    <div style={style}>
      {items[index].name}
    </div>
  );
  
  return (
    <FixedSizeList
      height={600}
      itemCount={items.length}
      itemSize={35}
      width='100%'
    >
      {Row}
    </FixedSizeList>
  );
};
```

## Стилизация

### CSS Modules

```jsx
// ✅ Хорошо - изолированные стили
import styles from './Button.module.css';

const Button = ({ variant, children }) => {
  return (
    <button className={`${styles.button} ${styles[variant]}`}>
      {children}
    </button>
  );
};
```

### CSS-in-JS (styled-components, emotion)

```jsx
// ✅ Хорошо - динамические стили
import styled from 'styled-components';

const StyledButton = styled.button`
  padding: 10px 20px;
  border: none;
  border-radius: 4px;
  background-color: ${props => 
    props.variant === 'primary' ? '#007bff' : '#6c757d'
  };
  
  &:hover {
    opacity: 0.8;
  }
`;

const Button = ({ variant = 'primary', children }) => {
  return <StyledButton variant={variant}>{children}</StyledButton>;
};
```

### Классы - условное применение

```jsx
// ✅ Хорошо - используйте библиотеку classnames
import classNames from 'classnames';

const Button = ({ variant, size, disabled, children }) => {
  const buttonClass = classNames(
    'btn',
    `btn-${variant}`,
    `btn-${size}`,
    {
      'btn-disabled': disabled,
      'btn-loading': isLoading
    }
  );
  
  return <button className={buttonClass}>{children}</button>;
};
```

## Тестирование

### Компонентные тесты

```jsx
// ✅ Хорошо - тестирование поведения
import { render, screen, fireEvent } from '@testing-library/react';

describe('Button', () => {
  it('should call onClick when clicked', () => {
    const handleClick = jest.fn();
    render(<Button onClick={handleClick}>Click me</Button>);
    
    fireEvent.click(screen.getByText('Click me'));
    expect(handleClick).toHaveBeenCalledTimes(1);
  });
  
  it('should be disabled when disabled prop is true', () => {
    render(<Button disabled>Click me</Button>);
    
    const button = screen.getByText('Click me');
    expect(button).toBeDisabled();
  });
});
```

### Тестирование хуков

```jsx
// ✅ Хорошо - изолированное тестирование хуков
import { renderHook, act } from '@testing-library/react-hooks';

test('useCounter', () => {
  const { result } = renderHook(() => useCounter());
  
  expect(result.current.count).toBe(0);
  
  act(() => {
    result.current.increment();
  });
  
  expect(result.current.count).toBe(1);
});
```

### Тестирование асинхронного кода

```jsx
// ✅ Хорошо - waitFor для асинхронных операций
import { render, screen, waitFor } from '@testing-library/react';

test('loads and displays user data', async () => {
  render(<UserProfile userId="123" />);
  
  expect(screen.getByText('Loading...')).toBeInTheDocument();
  
  await waitFor(() => {
    expect(screen.getByText('John Doe')).toBeInTheDocument();
  });
});
```

## Обработка ошибок

### Error Boundaries

```jsx
// ✅ Хорошо - граница ошибок
class ErrorBoundary extends Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false, error: null };
  }
  
  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }
  
  componentDidCatch(error, errorInfo) {
    console.error('Error caught by boundary:', error, errorInfo);
    // Отправка в сервис логирования
  }
  
  render() {
    if (this.state.hasError) {
      return (
        <div className="error-fallback">
          <h2>Что-то пошло не так</h2>
          <button onClick={() => this.setState({ hasError: false })}>
            Попробовать снова
          </button>
        </div>
      );
    }
    
    return this.props.children;
  }
}

// Использование
const App = () => {
  return (
    <ErrorBoundary>
      <AppContent />
    </ErrorBoundary>
  );
};
```

### Обработка асинхронных ошибок

```jsx
// ✅ Хорошо - правильная обработка
const useAsyncData = (asyncFunction) => {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    const fetchData = async () => {
      try {
        setLoading(true);
        setError(null);
        const result = await asyncFunction();
        setData(result);
      } catch (err) {
        setError(err.message || 'Произошла ошибка');
      } finally {
        setLoading(false);
      }
    };

    fetchData();
  }, []);

  return { data, loading, error };
};

// Использование
const UserList = () => {
  const { data: users, loading, error } = useAsyncData(() => api.getUsers());
  
  if (loading) return <Spinner />;
  if (error) return <ErrorMessage message={error} />;
  if (!users?.length) return <EmptyState />;
  
  return <UserGrid users={users} />;
};
```

## Безопасность

### Предотвращение XSS

```jsx
// ✅ Хорошо - безопасный рендеринг
const SafeComponent = ({ userInput }) => {
  return <div>{userInput}</div>; // React автоматически экранирует
};

// ⚠️ Осторожно с dangerouslySetInnerHTML
const DangerousComponent = ({ htmlContent }) => {
  // Используйте только с проверенным контентом
  // Рекомендуется использовать библиотеку DOMPurify
  return <div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(htmlContent) }} />;
};
```

### Валидация пропсов

```jsx
// ✅ Хорошо - PropTypes для валидации
import PropTypes from 'prop-types';

const UserCard = ({ user, onEdit, isAdmin }) => {
  return (
    <div className="user-card">
      <h3>{user.name}</h3>
      <p>{user.email}</p>
      {isAdmin && <button onClick={() => onEdit(user.id)}>Edit</button>}
    </div>
  );
};

UserCard.propTypes = {
  user: PropTypes.shape({
    id: PropTypes.string.isRequired,
    name: PropTypes.string.isRequired,
    email: PropTypes.string.isRequired,
  }).isRequired,
  onEdit: PropTypes.func,
  isAdmin: PropTypes.bool
};

UserCard.defaultProps = {
  isAdmin: false
};
```

## Доступность (a11y)

### Семантическая разметка

```jsx
// ✅ Хорошо
const Navigation = () => {
  return (
    <nav aria-label="Main navigation">
      <ul>
        <li><a href="/home">Home</a></li>
        <li><a href="/about">About</a></li>
      </ul>
    </nav>
  );
};

// ✅ Хорошо - ARIA атрибуты
const Modal = ({ isOpen, onClose, title, children }) => {
  return (
    <div
      role="dialog"
      aria-modal="true"
      aria-labelledby="modal-title"
      aria-hidden={!isOpen}
    >
      <h2 id="modal-title">{title}</h2>
      <button onClick={onClose} aria-label="Close modal">×</button>
      {children}
    </div>
  );
};
```

### Управление фокусом

```jsx
// ✅ Хорошо - управление фокусом
const SearchForm = () => {
  const inputRef = useRef(null);
  
  useEffect(() => {
    inputRef.current?.focus();
  }, []);
  
  return (
    <form role="search">
      <label htmlFor="search-input">Search</label>
      <input
        ref={inputRef}
        id="search-input"
        type="search"
        aria-label="Search products"
      />
    </form>
  );
};
```

### Интерактивные элементы

```jsx
// ✅ Хорошо - доступные интерактивные элементы
const AccessibleButton = ({ onClick, children, ariaLabel }) => {
  return (
    <button
      onClick={onClick}
      aria-label={ariaLabel}
      onKeyDown={(e) => {
        if (e.key === 'Enter' || e.key === ' ') {
          e.preventDefault();
          onClick();
        }
      }}
    >
      {children}
    </button>
  );
};

// ✅ Хорошо - skip links
const Layout = () => {
  return (
    <>
      <a href="#main-content" className="skip-link">
        Skip to main content
      </a>
      <Header />
      <main id="main-content">
        {/* Основной контент */}
      </main>
    </>
  );
};
```

## Код-стайл и форматирование

### ESLint конфигурация

```json
{
  "extends": [
    "eslint:recommended",
    "plugin:react/recommended",
    "plugin:react-hooks/recommended",
    "plugin:jsx-a11y/recommended"
  ],
  "parserOptions": {
    "ecmaVersion": 2021,
    "sourceType": "module",
    "ecmaFeatures": {
      "jsx": true
    }
  },
  "env": {
    "browser": true,
    "es2021": true,
    "node": true
  },
  "rules": {
    "react/prop-types": "warn",
    "react/react-in-jsx-scope": "off",
    "no-unused-vars": ["error", { "argsIgnorePattern": "^_" }],
    "no-console": "warn"
  }
}
```

### Prettier конфигурация

```json
{
  "semi": true,
  "trailingComma": "es5",
  "singleQuote": true,
  "printWidth": 100,
  "tabWidth": 2,
  "useTabs": false,
  "arrowParens": "always",
  "endOfLine": "lf"
}
```

### Импорты - порядок и группировка

```jsx
// ✅ Хорошо - организованные импорты
// 1. React
import React, { useState, useEffect } from 'react';

// 2. Библиотеки
import { useQuery } from '@tanstack/react-query';
import { format } from 'date-fns';

// 3. Компоненты
import Button from '@/components/common/Button';
import UserCard from '@/components/features/UserCard';

// 4. Хуки
import useAuth from '@/hooks/useAuth';
import useDebounce from '@/hooks/useDebounce';

// 5. Утилиты и константы
import { formatCurrency } from '@/utils/format';
import { API_ENDPOINTS } from '@/constants';

// 6. Стили
import styles from './Component.module.css';
```

## Дополнительные рекомендации

### Избегайте преждевременной оптимизации

- Сначала пишите чистый, читаемый код
- Оптимизируйте только после профилирования
- Используйте React DevTools Profiler

### Документация компонентов

```jsx
/**
 * Кнопка с различными вариантами стилизации
 * 
 * @component
 * @example
 * <Button variant="primary" size="large" onClick={handleClick}>
 *   Click me
 * </Button>
 */
const Button = ({ variant, size, onClick, children }) => {
  // Реализация компонента
};

Button.propTypes = {
  /** Вариант стилизации кнопки */
  variant: PropTypes.oneOf(['primary', 'secondary', 'danger']),
  /** Размер кнопки */
  size: PropTypes.oneOf(['small', 'medium', 'large']),
  /** Обработчик клика */
  onClick: PropTypes.func,
  /** Содержимое кнопки */
  children: PropTypes.node.isRequired
};
```

### Обработка форм

```jsx
// ✅ Хорошо - контролируемые компоненты
const ContactForm = () => {
  const [formData, setFormData] = useState({
    name: '',
    email: '',
    message: ''
  });
  const [errors, setErrors] = useState({});

  const handleChange = (e) => {
    const { name, value } = e.target;
    setFormData(prev => ({
      ...prev,
      [name]: value
    }));
    // Очистка ошибки при изменении
    if (errors[name]) {
      setErrors(prev => ({ ...prev, [name]: '' }));
    }
  };

  const validate = () => {
    const newErrors = {};
    if (!formData.name) newErrors.name = 'Имя обязательно';
    if (!formData.email) newErrors.email = 'Email обязателен';
    if (!formData.message) newErrors.message = 'Сообщение обязательно';
    return newErrors;
  };

  const handleSubmit = (e) => {
    e.preventDefault();
    const newErrors = validate();
    if (Object.keys(newErrors).length > 0) {
      setErrors(newErrors);
      return;
    }
    // Отправка формы
    console.log('Form submitted:', formData);
  };

  return (
    <form onSubmit={handleSubmit}>
      <div>
        <label htmlFor="name">Имя</label>
        <input
          id="name"
          name="name"
          value={formData.name}
          onChange={handleChange}
          aria-invalid={!!errors.name}
          aria-describedby={errors.name ? 'name-error' : undefined}
        />
        {errors.name && <span id="name-error">{errors.name}</span>}
      </div>
      {/* Остальные поля формы */}
      <button type="submit">Отправить</button>
    </form>
  );
};
```

### Версионирование и зависимости

- Фиксируйте версии в package.json
- Регулярно обновляйте зависимости
- Используйте npm audit для проверки безопасности

## Заключение

Эти best practices помогут создавать масштабируемые, поддерживаемые и производительные React приложения. Помните, что это рекомендации, а не жесткие правила - адаптируйте их под нужды вашего проекта и команды.