# milk717-cursor-rules
“좋은 코드”에 대한 저만의 기준을 Cursor rules로 정의했습니다.

# Rules 판단 기준
각 룰을 선정한 근거와 판단 기준은 다음과 같습니다.

## react-hooks-order
컴포넌트 내부에서 훅이 일관된 순서로 위치해야 코드의 가독성과 유지보수성이 향상됩니다. 제가 생각하는 이상적인 훅 선언 순서는 다음과 같습니다.

0. Props 구조분해할당
1. Router Hooks (useParams, useRouter, useNavigate, useMatch, useLocation, useSearchParams)
2. Context Hooks (useContext, useTheme, useQueryClient 등 Provider에서 제공되는 값)
3. Global Store (Zustand, Redux 등 전역 상태 관리)
4. Server State (useQuery, useMutation, useMutationState 등 React Query)
5. Custom Hooks (라이브러리 훅, 자체 제작 훅)
   - useForm, useFormik 등 Form 관련 훅 포함
   - Hook 선언과 동시에 구조분해하는 것을 권장
6. Local State (useState, useReducer)
7. Refs (useRef)
8. 파생 상태 (useMemo, 계산된 const 값, Hook 반환값의 지연 구조분해)
9. Effects (useEffect, useLayoutEffect)
10. Event Handlers & Callbacks (useCallback, 일반 함수 형태의 이벤트 핸들러)

### 순서의 원칙

위쪽에 위치할수록 **넓은 범위에서 영향을 끼치는 값**입니다. 넓은 범위에서 영향을 끼친다는 것은 더 상위 개념의 상태 혹은 환경이라는 의미이며, 이는 컴포넌트 내부에서 많은 로직에 영향을 줍니다. 즉, **아래쪽의 훅이나 값들은 위쪽에서 선언된 것들에 종속**되는 구조입니다.

이러한 순서로 선언하면 **의존성 방향이 일관되어** 데이터 흐름을 명확하게 파악할 수 있습니다.

### 헷갈릴 수 있는 부분

#### 1. Context는 왜 전역 상태보다 위에 선언하나요?

이를 이해하려면 Context와 전역 상태의 용도부터 구분해야 합니다.

- **Context**: UI 상태(tab 선택 상태), 환경 설정(다크모드, 테마, queryClient) 등 **환경적인 값**
- **전역 상태**: 비즈니스 로직과 관련된 **로직적인 값**

Context는 컴포넌트 트리 내의 **기반이 되는 환경**을 제공합니다(테마, userId 등). 반면 전역 상태는 범위는 넓지만 비즈니스 로직에 가깝습니다.

컴포넌트 내에서 **"환경이 로직에 영향을 끼치는"** 흐름은 자연스럽지만, **"로직이 환경에 영향을 끼치는"** 흐름은 부자연스럽습니다. 만약 로직이 환경에 영향을 줘야 한다면, 그 로직은 현재 컴포넌트가 아닌 상위 레벨에서 수행되어야 합니다.

따라서 Context는 **환경적 성격**이 강하므로 전역 상태보다 먼저 선언합니다.

#### 2. 서버 상태는 왜 전역 상태보다 아래에 선언하나요?

서버 상태는 서버로부터 오는 값이므로 클라이언트 전역 상태보다 더 넓은 범위처럼 느껴질 수 있습니다. 하지만 서버 상태를 가져오는 로직은 전역 상태에 비해 **더 구체적이고 로직적**입니다.

**데이터 레이어 관점:**
- **전역 상태**: 클라이언트 사이드의 "진실의 원천" (Single Source of Truth)
- **서버 상태**: 서버와의 "동기화 레이어"

클라이언트의 상태를 먼저 확립한 후 서버와 동기화하는 흐름이 자연스럽습니다.

**일반적인 흐름:**
- ✅ 자연스러운 흐름: 전역 상태의 특정 값에 따라 서버에서 데이터를 불러옴
  ```typescript
  // 전역 상태에서 필터 조건을 가져오고
  const { filters, sortBy } = useAppStore()

  // 그 조건으로 서버 데이터를 쿼리
  const { data } = useQuery(['items', filters, sortBy],
    () => fetchItems({ filters, sortBy })
  )
  ```

- ❌ 부자연스러운 흐름: 서버에서 불러온 값에 따라 전역 상태의 값을 선택함
  ```typescript
  // ❌ 안티패턴: 컴포넌트에서 서버 상태에 따라 전역 상태 분기
  const { data: user } = useQuery(['user'])
  const adminStore = useAdminStore()
  const userStore = useUserStore()
  const store = user?.role === 'admin' ? adminStore : userStore

  // ✅ 올바른 패턴: 상위에서 분기 처리 후 하위로 전파
  function App() {
    const { data: user } = useQuery(['user'])
    return user?.role === 'admin' ? <AdminLayout /> : <UserLayout />
  }

  function AdminLayout() {
    const data = useAdminStore() // role이 이미 결정된 상태
    // ...
  }
  ```

**핵심 원칙:**
서버 상태에 따라 전역 상태를 선택해야 한다면, 그것은 **컴포넌트가 잘못된 레벨에 위치**했다는 신호입니다. 이런 분기는 라우팅이나 상위 레이아웃 레벨에서 처리되어야 하며, 각 컴포넌트는 필요한 전역 상태가 이미 결정된 상태에서 시작해야 합니다.

전역 상태는 **어떤 데이터를 가져올지 결정하는 조건**이 되는 경우가 많고, 서버 상태는 그 조건에 따라 **실제 데이터를 가져오는 실행 로직**에 가깝습니다. 따라서 서버 상태를 전역 상태보다 아래에 선언하는 것이 의존성 방향에 부합합니다. 



## component-cohesion

### 스켈레톤 컴포넌트

스켈레톤 컴포넌트는 본 컴포넌트와 **같은 파일에 위치**해야 합니다.

**적용 대상:**
- Suspense query 요청이나 Promise를 throw하는 컴포넌트
- Export되는 컴포넌트와 해당 스켈레톤이 1:1로 대응되는 경우

#### 같은 파일에 위치해야 하는 이유

**1. UI 구조의 유사성**
스켈레톤은 보통 본 컴포넌트와 동일하거나 매우 유사한 UI 구조를 가집니다. 레이아웃, 배치, 크기 등이 대부분 일치하므로, 본 컴포넌트를 수정할 때 스켈레톤도 함께 수정해야 하는 경우가 많습니다.

**2. 변경 추적의 용이성**
같은 파일에 위치하면 본 컴포넌트를 수정할 때 스켈레톤을 함께 수정해야 한다는 사실을 자연스럽게 인지할 수 있습니다. 만약 별도 파일에 존재한다면 스켈레톤 수정을 놓치기 쉽습니다.

#### 공통화하지 않는 이유

스켈레톤과 본 컴포넌트의 UI가 유사하다고 해서 레이아웃을 추상화하여 공통화하는 것은 지양해야 합니다.

**근거:**
- **스켈레톤**: 단순 Fallback UI를 위한 정적 컴포넌트
- **본 컴포넌트**: 비즈니스 요구사항을 담은 동적 컴포넌트

두 컴포넌트는 현재 유사해 보이지만 **서로 다른 목적**을 가지고 있으며, **독립적으로 변경**될 수 있습니다. Fallback UI 하나를 위해 공통 레이아웃을 추상화하면, 비즈니스 요구사항 변경 시 불필요한 복잡도가 증가합니다.

**결론:**
스켈레톤은 본 컴포넌트와 **같은 파일에 위치**하되, 레이아웃 골격은 **독립적으로 작성**하는 것이 유지보수에 유리합니다.