---
title: '실무에서 바로 쓰는 React Custom Hooks 7가지 패턴'
description: 'React 개발 효율을 극대화하는 Custom Hooks 패턴을 소개합니다. 실무에서 자주 사용하는 7가지 훅을 직접 만들어보며 재사용 가능한 로직 분리 방법을 익혀보세요.'
pubDate: '2025-01-30'
heroImage: 'https://images.unsplash.com/photo-1633356122544-f134324a6cee?w=1200&q=80'
---

프론트엔드 개발을 하다 보면 비슷한 로직을 여러 컴포넌트에서 반복해서 작성하는 경우가 정말 많습니다. API 호출, 폼 상태 관리, 로컬 스토리지 연동 같은 코드를 매번 복사 붙여넣기 하고 있진 않으신가요? 이런 반복적인 로직을 깔끔하게 분리할 수 있는 방법이 바로 Custom Hooks입니다.

오늘은 제가 실무에서 직접 사용하면서 검증된 7가지 Custom Hooks 패턴을 공유하려고 해요. 단순히 코드만 보여드리는 게 아니라, 왜 이렇게 만들었는지, 어떤 상황에서 쓰면 좋은지까지 자세하게 설명해 드릴게요.

## Custom Hooks가 뭔가요?

본격적으로 패턴을 살펴보기 전에, Custom Hooks에 대해 간단히 짚고 넘어갈게요. Custom Hooks는 React의 기본 Hooks(useState, useEffect 등)를 조합해서 만든 재사용 가능한 함수예요. 이름은 반드시 `use`로 시작해야 하고, 내부에서 다른 Hooks를 자유롭게 사용할 수 있습니다.

```javascript
// Custom Hook의 기본 구조
function useMyCustomHook() {
  const [state, setState] = useState(initialValue);

  useEffect(() => {
    // 사이드 이펙트 로직
  }, []);

  return { state, setState };
}
```

Custom Hooks를 잘 활용하면 컴포넌트의 로직과 UI를 깔끔하게 분리할 수 있어요. 테스트하기도 훨씬 쉬워지고, 같은 로직을 여러 컴포넌트에서 재사용할 수 있게 됩니다.

![코딩하는 개발자의 모습](https://images.unsplash.com/photo-1461749280684-dccba630e2f6?w=1200&q=80)

## 1. useLocalStorage - 로컬 스토리지 상태 관리

첫 번째로 소개할 훅은 `useLocalStorage`예요. 웹 애플리케이션에서 사용자 설정이나 임시 데이터를 저장할 때 로컬 스토리지를 정말 많이 쓰는데요, 매번 JSON.parse, JSON.stringify 하는 게 은근 귀찮잖아요.

```typescript
import { useState, useEffect } from 'react';

function useLocalStorage<T>(key: string, initialValue: T) {
  // 초기값 설정 - lazy initialization 사용
  const [storedValue, setStoredValue] = useState<T>(() => {
    if (typeof window === 'undefined') {
      return initialValue;
    }

    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      console.warn(`Error reading localStorage key "${key}":`, error);
      return initialValue;
    }
  });

  // 값이 변경될 때마다 로컬 스토리지에 저장
  useEffect(() => {
    if (typeof window === 'undefined') {
      return;
    }

    try {
      window.localStorage.setItem(key, JSON.stringify(storedValue));
    } catch (error) {
      console.warn(`Error setting localStorage key "${key}":`, error);
    }
  }, [key, storedValue]);

  return [storedValue, setStoredValue] as const;
}
```

사용 방법은 useState랑 거의 똑같아요. 첫 번째 인자로 키 이름을, 두 번째 인자로 초기값을 넣어주면 됩니다.

```typescript
function SettingsPage() {
  const [theme, setTheme] = useLocalStorage('theme', 'light');
  const [fontSize, setFontSize] = useLocalStorage('fontSize', 16);

  return (
    <div>
      <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
        테마 변경: {theme}
      </button>
      <input
        type="range"
        value={fontSize}
        onChange={(e) => setFontSize(Number(e.target.value))}
      />
    </div>
  );
}
```

이 훅의 장점은 컴포넌트가 언마운트되었다가 다시 마운트되어도 값이 유지된다는 거예요. 새로고침해도 데이터가 날아가지 않으니까 사용자 경험이 훨씬 좋아집니다.

## 2. useFetch - 데이터 페칭의 정석

API 호출은 거의 모든 웹 앱에서 필수적으로 필요한 기능이에요. 로딩 상태 관리, 에러 처리, 데이터 캐싱까지 고려하면 코드가 금방 복잡해지는데, 이걸 깔끔하게 정리한 게 `useFetch` 훅입니다.

```typescript
import { useState, useEffect, useCallback } from 'react';

interface UseFetchResult<T> {
  data: T | null;
  loading: boolean;
  error: Error | null;
  refetch: () => void;
}

function useFetch<T>(url: string, options?: RequestInit): UseFetchResult<T> {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  const fetchData = useCallback(async () => {
    setLoading(true);
    setError(null);

    try {
      const response = await fetch(url, options);

      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }

      const result = await response.json();
      setData(result);
    } catch (err) {
      setError(err instanceof Error ? err : new Error('알 수 없는 에러가 발생했습니다'));
    } finally {
      setLoading(false);
    }
  }, [url, options]);

  useEffect(() => {
    fetchData();
  }, [fetchData]);

  return { data, loading, error, refetch: fetchData };
}
```

실제로 사용할 때는 이렇게 씁니다.

```typescript
function UserProfile({ userId }: { userId: string }) {
  const { data: user, loading, error, refetch } = useFetch<User>(
    `/api/users/${userId}`
  );

  if (loading) return <Skeleton />;
  if (error) return <ErrorMessage message={error.message} onRetry={refetch} />;
  if (!user) return null;

  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  );
}
```

로딩, 에러, 데이터 상태를 한 번에 관리할 수 있어서 컴포넌트 코드가 훨씬 깔끔해져요. `refetch` 함수도 제공하니까 "다시 시도" 버튼 같은 것도 쉽게 구현할 수 있습니다.

![노트북으로 코딩하는 모습](https://images.unsplash.com/photo-1498050108023-c5249f4df085?w=1200&q=80)

## 3. useDebounce - 성능 최적화의 핵심

검색창 자동완성이나 실시간 필터링 기능을 만들 때, 사용자가 타이핑할 때마다 API를 호출하면 서버에 엄청난 부하가 걸려요. 이럴 때 사용하는 게 디바운스(Debounce)인데요, 연속된 호출 중 마지막 호출만 실행하도록 해주는 기법이에요.

```typescript
import { useState, useEffect } from 'react';

function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value);

  useEffect(() => {
    const timer = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => {
      clearTimeout(timer);
    };
  }, [value, delay]);

  return debouncedValue;
}
```

검색 기능에 적용하면 이렇게 됩니다.

```typescript
function SearchComponent() {
  const [searchTerm, setSearchTerm] = useState('');
  const debouncedSearchTerm = useDebounce(searchTerm, 300);

  const { data: results } = useFetch<SearchResult[]>(
    `/api/search?q=${debouncedSearchTerm}`,
    { skip: !debouncedSearchTerm }
  );

  return (
    <div>
      <input
        type="text"
        value={searchTerm}
        onChange={(e) => setSearchTerm(e.target.value)}
        placeholder="검색어를 입력하세요"
      />
      <SearchResults results={results} />
    </div>
  );
}
```

사용자가 빠르게 타이핑해도 300ms 동안 추가 입력이 없을 때만 API를 호출해요. 서버 부하도 줄이고, 불필요한 네트워크 요청도 막을 수 있습니다.

## 4. useOnClickOutside - 모달/드롭다운 필수 훅

드롭다운 메뉴나 모달을 만들 때 바깥 영역을 클릭하면 닫히게 하는 기능, 자주 구현하시죠? 매번 이벤트 리스너를 붙였다 떼었다 하는 게 번거로운데, 이걸 훅으로 만들어두면 정말 편해요.

```typescript
import { useEffect, RefObject } from 'react';

function useOnClickOutside<T extends HTMLElement>(
  ref: RefObject<T>,
  handler: (event: MouseEvent | TouchEvent) => void
) {
  useEffect(() => {
    const listener = (event: MouseEvent | TouchEvent) => {
      const el = ref?.current;

      // ref 안쪽을 클릭한 경우 무시
      if (!el || el.contains(event.target as Node)) {
        return;
      }

      handler(event);
    };

    document.addEventListener('mousedown', listener);
    document.addEventListener('touchstart', listener);

    return () => {
      document.removeEventListener('mousedown', listener);
      document.removeEventListener('touchstart', listener);
    };
  }, [ref, handler]);
}
```

드롭다운에 적용하면 이렇게 씁니다.

```typescript
function Dropdown() {
  const [isOpen, setIsOpen] = useState(false);
  const dropdownRef = useRef<HTMLDivElement>(null);

  useOnClickOutside(dropdownRef, () => setIsOpen(false));

  return (
    <div ref={dropdownRef}>
      <button onClick={() => setIsOpen(!isOpen)}>
        메뉴 열기
      </button>
      {isOpen && (
        <ul className="dropdown-menu">
          <li>옵션 1</li>
          <li>옵션 2</li>
          <li>옵션 3</li>
        </ul>
      )}
    </div>
  );
}
```

터치 이벤트도 함께 처리해서 모바일에서도 잘 동작해요. 모달, 툴팁, 드롭다운 등 다양한 UI 컴포넌트에 활용할 수 있습니다.

![깔끔한 작업 환경](https://images.unsplash.com/photo-1517694712202-14dd9538aa97?w=1200&q=80)

## 5. useMediaQuery - 반응형 로직 처리

CSS 미디어 쿼리는 스타일에만 적용할 수 있잖아요. 그런데 가끔 JavaScript 로직에서도 화면 크기에 따라 다르게 동작해야 할 때가 있어요. 예를 들어 모바일에서는 햄버거 메뉴를, 데스크톱에서는 일반 메뉴를 보여주는 경우처럼요.

```typescript
import { useState, useEffect } from 'react';

function useMediaQuery(query: string): boolean {
  const [matches, setMatches] = useState(false);

  useEffect(() => {
    const media = window.matchMedia(query);

    // 초기값 설정
    setMatches(media.matches);

    // 변경 감지 리스너
    const listener = (event: MediaQueryListEvent) => {
      setMatches(event.matches);
    };

    media.addEventListener('change', listener);

    return () => {
      media.removeEventListener('change', listener);
    };
  }, [query]);

  return matches;
}
```

실제 사용 예시를 볼게요.

```typescript
function Navigation() {
  const isMobile = useMediaQuery('(max-width: 768px)');
  const prefersDarkMode = useMediaQuery('(prefers-color-scheme: dark)');

  return (
    <nav>
      {isMobile ? (
        <MobileMenu />
      ) : (
        <DesktopMenu />
      )}
    </nav>
  );
}
```

화면 크기가 변경될 때 실시간으로 반응하니까, 사용자가 브라우저 창 크기를 조절해도 즉시 적절한 UI로 전환됩니다. 다크모드 감지 같은 것도 이 훅으로 처리할 수 있어요.

## 6. useForm - 폼 상태 관리 간소화

폼 처리는 React에서 가장 귀찮은 작업 중 하나예요. 각 입력 필드마다 state를 만들고, onChange 핸들러를 연결하고, 유효성 검사까지 하려면 코드가 순식간에 복잡해져요. 이걸 하나의 훅으로 깔끔하게 정리해봤습니다.

```typescript
import { useState, useCallback, ChangeEvent, FormEvent } from 'react';

interface UseFormOptions<T> {
  initialValues: T;
  onSubmit: (values: T) => void | Promise<void>;
  validate?: (values: T) => Partial<Record<keyof T, string>>;
}

interface UseFormReturn<T> {
  values: T;
  errors: Partial<Record<keyof T, string>>;
  isSubmitting: boolean;
  handleChange: (e: ChangeEvent<HTMLInputElement | HTMLTextAreaElement | HTMLSelectElement>) => void;
  handleSubmit: (e: FormEvent) => void;
  setFieldValue: (name: keyof T, value: any) => void;
  reset: () => void;
}

function useForm<T extends Record<string, any>>({
  initialValues,
  onSubmit,
  validate
}: UseFormOptions<T>): UseFormReturn<T> {
  const [values, setValues] = useState<T>(initialValues);
  const [errors, setErrors] = useState<Partial<Record<keyof T, string>>>({});
  const [isSubmitting, setIsSubmitting] = useState(false);

  const handleChange = useCallback((
    e: ChangeEvent<HTMLInputElement | HTMLTextAreaElement | HTMLSelectElement>
  ) => {
    const { name, value, type } = e.target;
    const newValue = type === 'checkbox'
      ? (e.target as HTMLInputElement).checked
      : value;

    setValues(prev => ({ ...prev, [name]: newValue }));

    // 입력 시 해당 필드 에러 초기화
    if (errors[name as keyof T]) {
      setErrors(prev => ({ ...prev, [name]: undefined }));
    }
  }, [errors]);

  const setFieldValue = useCallback((name: keyof T, value: any) => {
    setValues(prev => ({ ...prev, [name]: value }));
  }, []);

  const handleSubmit = useCallback(async (e: FormEvent) => {
    e.preventDefault();

    // 유효성 검사
    if (validate) {
      const validationErrors = validate(values);
      if (Object.keys(validationErrors).length > 0) {
        setErrors(validationErrors);
        return;
      }
    }

    setIsSubmitting(true);

    try {
      await onSubmit(values);
    } finally {
      setIsSubmitting(false);
    }
  }, [values, validate, onSubmit]);

  const reset = useCallback(() => {
    setValues(initialValues);
    setErrors({});
  }, [initialValues]);

  return {
    values,
    errors,
    isSubmitting,
    handleChange,
    handleSubmit,
    setFieldValue,
    reset
  };
}
```

회원가입 폼에 적용하면 이렇게 됩니다.

```typescript
function SignUpForm() {
  const { values, errors, isSubmitting, handleChange, handleSubmit } = useForm({
    initialValues: {
      email: '',
      password: '',
      confirmPassword: '',
      agreeToTerms: false
    },
    validate: (values) => {
      const errors: Record<string, string> = {};

      if (!values.email) {
        errors.email = '이메일을 입력해주세요';
      } else if (!/\S+@\S+\.\S+/.test(values.email)) {
        errors.email = '올바른 이메일 형식이 아닙니다';
      }

      if (!values.password) {
        errors.password = '비밀번호를 입력해주세요';
      } else if (values.password.length < 8) {
        errors.password = '비밀번호는 8자 이상이어야 합니다';
      }

      if (values.password !== values.confirmPassword) {
        errors.confirmPassword = '비밀번호가 일치하지 않습니다';
      }

      if (!values.agreeToTerms) {
        errors.agreeToTerms = '약관에 동의해주세요';
      }

      return errors;
    },
    onSubmit: async (values) => {
      const response = await fetch('/api/signup', {
        method: 'POST',
        body: JSON.stringify(values)
      });
      // 처리 로직...
    }
  });

  return (
    <form onSubmit={handleSubmit}>
      <div>
        <input
          name="email"
          type="email"
          value={values.email}
          onChange={handleChange}
          placeholder="이메일"
        />
        {errors.email && <span className="error">{errors.email}</span>}
      </div>

      <div>
        <input
          name="password"
          type="password"
          value={values.password}
          onChange={handleChange}
          placeholder="비밀번호"
        />
        {errors.password && <span className="error">{errors.password}</span>}
      </div>

      <div>
        <input
          name="confirmPassword"
          type="password"
          value={values.confirmPassword}
          onChange={handleChange}
          placeholder="비밀번호 확인"
        />
        {errors.confirmPassword && <span className="error">{errors.confirmPassword}</span>}
      </div>

      <div>
        <label>
          <input
            name="agreeToTerms"
            type="checkbox"
            checked={values.agreeToTerms}
            onChange={handleChange}
          />
          약관에 동의합니다
        </label>
        {errors.agreeToTerms && <span className="error">{errors.agreeToTerms}</span>}
      </div>

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? '처리중...' : '가입하기'}
      </button>
    </form>
  );
}
```

폼 관련 로직이 전부 훅 안에 캡슐화되어 있어서 컴포넌트가 훨씬 깔끔해졌죠? 유효성 검사 로직도 한눈에 파악할 수 있고, 에러 메시지 표시도 일관되게 처리할 수 있습니다.

![카페에서 개발하는 모습](https://images.unsplash.com/photo-1522202176988-66273c2fd55f?w=1200&q=80)

## 7. useIntersectionObserver - 무한 스크롤과 지연 로딩

마지막으로 소개할 훅은 `useIntersectionObserver`예요. 무한 스크롤, 이미지 지연 로딩, 스크롤 애니메이션 같은 기능을 구현할 때 필수적인 Intersection Observer API를 쉽게 사용할 수 있게 해주는 훅입니다.

```typescript
import { useEffect, useRef, useState, RefObject } from 'react';

interface UseIntersectionObserverOptions {
  root?: Element | null;
  rootMargin?: string;
  threshold?: number | number[];
  freezeOnceVisible?: boolean;
}

function useIntersectionObserver<T extends HTMLElement>(
  options: UseIntersectionObserverOptions = {}
): [RefObject<T>, boolean] {
  const {
    root = null,
    rootMargin = '0px',
    threshold = 0,
    freezeOnceVisible = false
  } = options;

  const elementRef = useRef<T>(null);
  const [isVisible, setIsVisible] = useState(false);

  useEffect(() => {
    const element = elementRef.current;
    if (!element) return;

    // 이미 보이고 freeze 옵션이 켜져있으면 더 이상 관찰하지 않음
    if (freezeOnceVisible && isVisible) return;

    const observer = new IntersectionObserver(
      ([entry]) => {
        setIsVisible(entry.isIntersecting);
      },
      { root, rootMargin, threshold }
    );

    observer.observe(element);

    return () => {
      observer.disconnect();
    };
  }, [root, rootMargin, threshold, freezeOnceVisible, isVisible]);

  return [elementRef, isVisible];
}
```

무한 스크롤에 적용하면 이렇게 됩니다.

```typescript
function InfinitePostList() {
  const [posts, setPosts] = useState<Post[]>([]);
  const [page, setPage] = useState(1);
  const [hasMore, setHasMore] = useState(true);

  const [loadMoreRef, isLoadMoreVisible] = useIntersectionObserver<HTMLDivElement>({
    rootMargin: '100px', // 100px 전에 미리 로드 시작
  });

  useEffect(() => {
    if (isLoadMoreVisible && hasMore) {
      loadMorePosts();
    }
  }, [isLoadMoreVisible, hasMore]);

  const loadMorePosts = async () => {
    const newPosts = await fetchPosts(page);

    if (newPosts.length === 0) {
      setHasMore(false);
      return;
    }

    setPosts(prev => [...prev, ...newPosts]);
    setPage(prev => prev + 1);
  };

  return (
    <div>
      {posts.map(post => (
        <PostCard key={post.id} post={post} />
      ))}

      {hasMore && (
        <div ref={loadMoreRef} className="loading-indicator">
          로딩 중...
        </div>
      )}
    </div>
  );
}
```

이미지 지연 로딩에도 활용할 수 있어요.

```typescript
function LazyImage({ src, alt }: { src: string; alt: string }) {
  const [imageRef, isVisible] = useIntersectionObserver<HTMLImageElement>({
    rootMargin: '50px',
    freezeOnceVisible: true // 한 번 보이면 계속 로드 상태 유지
  });

  return (
    <img
      ref={imageRef}
      src={isVisible ? src : '/placeholder.jpg'}
      alt={alt}
      loading="lazy"
    />
  );
}
```

스크롤해서 이미지가 뷰포트에 들어올 때만 실제 이미지를 로드하니까 초기 페이지 로딩 속도가 크게 개선됩니다.

## Custom Hooks 작성 시 주의할 점

지금까지 7가지 유용한 Custom Hooks를 살펴봤는데요, 마지막으로 훅을 작성할 때 주의해야 할 점들을 정리해 드릴게요.

### 1. 이름은 반드시 use로 시작

React가 Hook 규칙을 검사할 수 있도록 항상 `use`로 시작하는 이름을 사용하세요. `useLocalStorage`, `useFetch` 처럼요.

### 2. 조건문 안에서 호출하지 않기

훅은 항상 컴포넌트의 최상위 레벨에서 호출해야 해요. if문이나 반복문 안에서 호출하면 안 됩니다.

```typescript
// 잘못된 사용
if (someCondition) {
  const [value, setValue] = useLocalStorage('key', ''); // 에러!
}

// 올바른 사용
const [value, setValue] = useLocalStorage('key', '');
if (someCondition) {
  // value 사용
}
```

### 3. 의존성 배열 관리에 신경 쓰기

useEffect나 useCallback의 의존성 배열을 정확하게 관리하세요. ESLint의 `exhaustive-deps` 규칙을 활성화해두면 빠뜨린 의존성을 알려줍니다.

### 4. 불필요한 리렌더링 방지

반환하는 객체나 함수가 매번 새로 생성되지 않도록 useCallback, useMemo를 적절히 활용하세요.

```typescript
// 매번 새 객체가 생성됨
return { data, loading, error };

// useMemo로 최적화
return useMemo(() => ({ data, loading, error }), [data, loading, error]);
```

### 5. 타입 안정성 확보

TypeScript를 사용한다면 제네릭을 활용해서 타입 안정성을 확보하세요. 훅을 사용하는 쪽에서 자동완성과 타입 체크의 혜택을 받을 수 있습니다.

![팀이 함께 개발하는 모습](https://images.unsplash.com/photo-1531482615713-2afd69097998?w=1200&q=80)

## 마치며

오늘 소개한 7가지 Custom Hooks는 제가 실제 프로젝트에서 정말 자주 사용하는 패턴들이에요. 처음에는 직접 만드는 게 번거롭게 느껴질 수 있지만, 한 번 만들어두면 프로젝트 전체에서 재사용할 수 있어서 개발 효율이 크게 올라갑니다.

물론 모든 상황에 Custom Hook이 정답은 아니에요. 단순한 로직이라면 그냥 컴포넌트 안에 두는 게 나을 수도 있고, 복잡한 상태 관리가 필요하다면 Redux나 Zustand 같은 전용 라이브러리를 쓰는 게 좋을 수도 있습니다.

중요한 건 "이 로직이 다른 곳에서도 쓰일까?"를 항상 생각해보는 거예요. 그 대답이 "예"라면 Custom Hook으로 분리하는 걸 고려해보세요.

여러분만의 Custom Hooks 패턴이 있다면 댓글로 공유해주세요. 다른 개발자들에게도 분명 도움이 될 거예요. 다음에는 더 유용한 React 팁으로 찾아뵐게요!

---

**관련 글 추천**
- React 성능 최적화 완벽 가이드
- TypeScript와 React 함께 사용하기
- 2025년 프론트엔드 개발 트렌드
