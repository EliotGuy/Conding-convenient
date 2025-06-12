# React Development Conventions

You are a senior React engineer with expertise in modern React development patterns. You provide nuanced, accurate, and factual guidance while demonstrating excellent reasoning skills.

When writing React code, you MUST follow these principles.

- Keep components focused on a single responsibility
- Use meaningful names for components, props, and functions
- Follow TypeScript best practices with proper type definitions
- Write clean, maintainable code with consistent patterns
- Consider performance implications of your implementation choices
- Follow the user's requirements carefully & to the letter

## Quick Reference

### 🚨 Critical Conventions
- **Project Structure**: Organize by features with clear separation of concerns
- **File Naming**: Use PascalCase for components, camelCase for utilities
- **Component Organization**: Each feature has dedicated .types.ts, .api.ts, .page.tsx, .form.tsx, .table.tsx files
- **Form Handling**: Use TanStack Form with Zod validation
- **Data Fetching**: Use TanStack Query for all server state
- **Routing**: Use TanStack Router with file-based routing
- **Type Safety**: Define explicit types for all API payloads and component props

### Common Patterns
- **API Functions**: `fetch{Entity}`, `create{Entity}`, `update{Entity}`, `delete{Entity}`
- **Query Hooks**: `use{Action}{Entity}` (e.g., `useCreateProject`)
- **Component Props**: `{Component}Props` (e.g., `ProjectsTableProps`)
- **API Payloads & Responses**:
    - List Request: `{Entity}ListGetPayload` (e.g., `ProjectListGetPayload`)
    - List Response: `{Entity}ListGetResponse` (e.g., `ProjectListGetResponse`)
    - Detail Request: `{Entity}DetailGetPayload` (e.g., `ProjectDetailGetPayload`)
    - Detail Response: `{Entity}DetailGetResponse` (e.g., `ProjectDetailGetResponse`)
    - Create Request: `{Entity}CreatePayload` (e.g., `ProjectCreatePayload`)
    - Update Request: `{Entity}UpdatePayload` (e.g., `ProjectUpdatePayload`)
- **Query Keys**: `{ENTITY}_QUERY_KEY` (e.g., `PROJECTS_QUERY_KEY`)
- **Callbacks**: `onSubmit`, `onView`, `onEdit`, `onDelete`, `onCreate`, `onUpdate`

## Priority Levels

Conventions in this document are marked with priority levels:

- 🚨 **Critical**: Must be followed without exception.
- ⚠️ **Important**: Should be followed in most cases.
- 👍 **Recommended**: Best practices that improve code quality.

## Tech Stack 🚨

Our React applications use the following core technologies:

- **Build Tool**: Vite
- **HTTP Client**: Axios
- **Styling**: TailwindCSS
- **Routing**: TanStack Router
- **Data Fetching**: TanStack Query
- **Form Handling**: TanStack Form
- **Form Validation**: Zod

## Project Structure 🚨

Organize code by features with a clear separation of concerns. Feature-specific modules (containing components like forms, tables, details, API logic, types, etc.) reside within a dedicated directory under `src/features/`. Page components, which are used for routing, are located in a top-level `src/Pages/` directory. Shared UI and common components that are not feature-specific remain in `src/components/`.

Refer to the "Project Structure: Feature-Based Organization" section in `react/examples.md` for a visual representation.

**Rationale**: Feature-based organization improves maintainability by grouping all aspects of a feature (UI, logic, types) together. Placing these feature modules in a dedicated `src/features/` directory provides a clear distinction from shared components and page components. Separating page components into `src/Pages/` enhances clarity for routing.

## Feature Organization ⚠️

Each feature should be organized with these standard components:

- **{Feature}.api.ts**: Functions to call APIs using TanStack Query
- **{Feature}.types.ts**: Types for all components, including request payloads and API responses
- **{Feature}.form.tsx**: Form component for adding/editing resources via API
- **{Feature}.table.tsx**: Container component that displays resources in a table with pagination and CRUD actions
- **{Feature}.detail.tsx**: Component to display detailed information about a single resource
- **index.ts**: Optional barrel file re-exporting key modules from the feature.

Page components (e.g., `{Feature}.page.tsx`) are located in a separate top-level `Pages` directory. See the project structure diagram in `react/examples.md`.

**Rationale**: This separation creates a predictable file structure that clearly defines responsibilities, making it easier for developers to locate specific functionality and maintain separation of concerns. Page components are grouped separately for clarity in routing.

## Naming Conventions ⚠️

**Rationale**: Consistent naming patterns make code more predictable, improve searchability, and communicate the purpose and relationships between components without requiring extensive documentation.

### Files 🚨

- **Component Files**: PascalCase (`Projects.tsx`)
- **Utility Files**: camelCase (`api.ts`)
- **Feature Files**: Prefixed with feature name in PascalCase (`Projects.types.ts`)
- **Page Components**: Suffixed with `.page.tsx` (`Projects.page.tsx`)
- **Form Components**: Suffixed with `.form.tsx` (`Projects.form.tsx`)
- **Table Components**: Suffixed with `.table.tsx` (`Projects.table.tsx`)
- **Detail Components**: Suffixed with `.detail.tsx` (`Projects.detail.tsx`)
- **Empty State Components**: Suffixed with `.empty.tsx` (`Projects.empty.tsx`)

### Types/Interfaces ⚠️

- Use PascalCase for type definitions
- API payload and response types follow these patterns:
    - List Request: `{Entity}ListGetPayload` (e.g., `ProjectListGetPayload`)
    - List Response: `{Entity}ListGetResponse` (e.g., `ProjectListGetResponse`)
    - Detail Request: `{Entity}DetailGetPayload` (e.g., `ProjectDetailGetPayload`)
    - Detail Response: `{Entity}DetailGetResponse` (e.g., `ProjectDetailGetResponse`)
    - Create Request: `{Entity}CreatePayload` (e.g., `ProjectCreatePayload`)
    - Update Request: `{Entity}UpdatePayload` (e.g., `ProjectUpdatePayload`)
    - (Note: For Create/Update operations, the response is often the entity itself, e.g., `Project` or `EntityDetailGetResponse`)
- Component props follow the pattern: `{Component}Props` (e.g., `ProjectsTableProps`)
- Component type definitions follow the pattern: `{Component}Component` (e.g., `ProjectsTableComponent`)

**Example:**
```typescript
// Good
type Project = { // This can also serve as ProjectDetailGetResponse
  uuid: string;
  url: string;
  type: "WORDPRESS" | "SHOPIFY" | "CUSTOM";
};

interface ProjectListGetPayload extends PaginationReq<Project> {} // Assuming PaginationReq is defined
type ProjectListGetResponse = PaginatedRes<Project>; // Assuming PaginatedRes is defined

interface ProjectDetailGetPayload {
  uuid: string;
}
// type ProjectDetailGetResponse = Project; // Often the entity itself

interface ProjectCreatePayload {
  url: string;
  type: "WORDPRESS" | "SHOPIFY" | "CUSTOM";
}

interface ProjectsTableProps {
  data: Project[];
  onView?: (project: Project) => void;
}

// Bad
interface props {  // lowercase, generic name
  projectData: any[];  // using any, generic name
  handleView: Function;  // using Function type
}
```

### Functions ⚠️

- Use camelCase for function names
- API functions follow patterns:
  - `fetch{Entity}` or `fetch{Entity}List` for GET operations
  - `create{Entity}`, `update{Entity}`, `delete{Entity}` for mutations
  - `use{Action}{Entity}` for React Query hooks
- Utility functions should be descriptive of their purpose (e.g., `getDefaultSearchValues`)

### Constants 👍

- Use UPPER_SNAKE_CASE for constants (e.g., `API_URL`, `PROJECTS_ENDPOINT`)
- Query and mutation keys follow the pattern: `{ENTITY}_{ACTION}_KEY` (e.g., `PROJECTS_QUERY_KEY`, `PROJECTS_MUTATION_KEY`)

## API Conventions ⚠️

### Endpoint Structure 👍

- Collection endpoints: `/[resource]/` (e.g., `/projects/`)
- Resource endpoints: `/[resource]/:uuid/` (e.g., `/projects/:uuid/`)

### Request/Response Types 🚨

- Use consistent typing for API responses using Axios
- Implement reusable pagination types for list endpoints
- Follow consistent naming patterns for request and response types

### TanStack Query Integration ⚠️

- Query options functions: `{entity}{Action}QueryOptions`
- Mutation hooks: `use{Action}{Entity}`
- Implement proper error handling and loading states
- Use `useSuspenseQuery` for queries that should suspend the component

**Example:**
```typescript
// Query options function
const projectListQueryOptions = (payload: ProjectListGetPayload) => {
  return queryOptions({
    queryKey: [PROJECTS_QUERY_KEY, payload],
    queryFn: () => fetchProjectList(payload),
  });
};

// Mutation hook
const useCreateProject = () => {
  return useMutation({
    mutationKey: [PROJECTS_MUTATION_KEY],
    mutationFn: (payload: ProjectCreatePayload) => createProject(payload),
  });
};

// Usage in component
const projectsQuery = useSuspenseQuery(
  projectListQueryOptions(projectListPayload)
);
```

## Component Conventions 🚨

### Page Components ⚠️

- Handle routing, data fetching, and state management for a specific route
- Can incorporate components from other features when appropriate
- Follow consistent layout patterns with headers, action buttons, and content areas
- Organize code in consistent sections:
  1. Routing setup (TanStack Router APIs)
  2. State management (modal states, selected items)
  3. Data fetching (queries with proper options)
  4. Mutations (create, update, delete operations)
  5. Callback functions (grouped by purpose)
  6. JSX rendering

### Table Components ⚠️

- Act as container components with both presentation and logic
- Include pagination controls for navigating large datasets
- Provide UI elements for CRUD operations (create, view, edit, delete)
- Handle row-level actions through dropdown menus or action buttons
- Use entity UUID as the key for list items
- Format data appropriately for display
- Include proper accessibility attributes

### Form Components 🚨

- Use TanStack Form for form state management
- Use Zod for form validation
- Implement generic form props with type parameters for flexibility
- Support both creation and update operations
- Handle form submission and validation errors

**Rationale**: TanStack Form with Zod validation provides type-safe form handling with robust validation, reducing runtime errors and improving developer experience through better TypeScript integration.

**Example:**
```typescript
// Form component with generic type parameter
const ProjectsForm = <T extends ProjectCreatePayload | ProjectUpdatePayload>({
  formId,
  onSubmit,
  defaultValues,
}: ProjectsFormProps<T>) => {
  const form = useAppForm({
    defaultValues: {
      url: defaultValues?.url ?? "",
      type: defaultValues?.type ?? "WORDPRESS",
    },
    onSubmit: async ({ value }) => {
      onSubmit?.(value as T);
    },
  });

  return (
    <form id={formId} onSubmit={form.handleSubmit}>
      <form.Field
        name="url"
        validators={{
          onChange: z.string().url("Invalid URL"),
        }}
      >
        {(field) => (
          <Input
            value={field.state.value}
            onChange={(e) => field.handleChange(e.target.value)}
            invalid={field.state.meta.errors.length > 0}
          />
        )}
      </form.Field>
    </form>
  );
};
```

### Detail Components ⚠️

- Display comprehensive information about a single resource
- Typically rendered at resource-specific routes (e.g., "/projects/:uuid")
- May include related data from other entities
- Provide actions relevant to the specific resource (edit, delete, etc.)

### Empty State Components 👍

- Create dedicated components for empty states
- Provide clear actions for users to take
- Use consistent styling and messaging

## Callback Patterns ⚠️

- Use consistent callback naming:
  - `onSubmit` for form submissions
  - `onView`, `onEdit`, `onDelete` for table row actions
  - `onCreate`, `onUpdate`, `onDelete` for mutation actions
  - `onClose`, `onConfirm` for modal actions

## State Management 🚨

### Server State 🚨

- Use TanStack Query for all server state management
- Define query keys in feature-specific constants files
- Implement proper error handling and loading states

**Rationale**: TanStack Query handles complex server state challenges like caching, synchronization, and background updates, significantly reducing boilerplate code and providing a consistent pattern for data fetching.

### Local State ⚠️

- Use React's built-in state management (useState, useReducer) for simple component state
- Consider context API for shared state within a feature
- Use state for UI elements like modals

## Modal Pattern ⚠️

- Use a reusable Modal component for all dialogs
- Pass form IDs to connect forms with modals
- Implement consistent props for modals

## Common Mistakes to Avoid ⚠️

**Rationale**: Understanding what not to do is as important as knowing best practices. These anti-patterns can lead to maintenance issues, performance problems, and bugs.

### Component Anti-Patterns ⚠️

- **Overly Complex Components**: Don't create components that handle too many responsibilities. Split them into smaller, focused components.
  ```typescript
  // Bad: Component doing too much
  const ProjectPage = () => {
    // Data fetching, state management, form handling, and UI rendering all in one component
    // ...100+ lines of code
  };
  
  // Good: Split into smaller components with clear responsibilities
  const ProjectPage = () => {
    // Only handles page-level concerns
    return (
      <>
        <ProjectHeader />
        <ProjectForm />
        <ProjectTable />
      </>
    );
  };
  ```

- **Inconsistent Naming**: Don't mix naming conventions within the same codebase.
  ```typescript
  // Bad: Inconsistent naming
  const get_projects = () => {}; // snake_case
  const FetchUsers = () => {};   // PascalCase for non-component
  ```

- **Prop Drilling**: Avoid passing props through multiple levels of components.
  ```typescript
  // Bad: Excessive prop drilling
  const App = () => {
    const [user, setUser] = useState(null);
    return <Main user={user} setUser={setUser} />;
  };
  
  const Main = ({ user, setUser }) => {
    return <Profile user={user} setUser={setUser} />;
  };
  
  // Good: Use context for deeply shared state
  const UserContext = createContext();
  
  const App = () => {
    const [user, setUser] = useState(null);
    return (
      <UserContext.Provider value={{ user, setUser }}>
        <Main />
      </UserContext.Provider>
    );
  };
  ```

### TypeScript Anti-Patterns 🚨

- **Using `any`**: Avoid using `any` type as it defeats the purpose of TypeScript.
  ```typescript
  // Bad
  const handleData = (data: any) => {};
  
  // Good
  interface User {
    id: string;
    name: string;
  }
  const handleData = (data: User) => {};
  ```

- **Type Assertions Without Validation**: Don't use type assertions (`as`) without runtime validation.
  ```typescript
  // Bad: Unsafe type assertion
  const userData = JSON.parse(response) as User;
  
  // Good: Validate with Zod before assertion
  const userSchema = z.object({
    id: z.string(),
    name: z.string(),
  });
  const parsed = userSchema.parse(JSON.parse(response));
  const userData: User = parsed;
  ```

### API and Data Fetching Anti-Patterns ⚠️

- **Inline API Calls**: Don't make API calls directly in components.
  ```typescript
  // Bad: API call directly in component
  const ProjectList = () => {
    useEffect(() => {
      axios.get('/api/projects').then(res => setProjects(res.data));
    }, []);
  };
  
  // Good: Use dedicated API functions and TanStack Query
  const ProjectList = () => {
    const projectsQuery = useSuspenseQuery(projectListQueryOptions(payload));
  };
  ```

- **Missing Error Handling**: Always handle potential errors in API calls.
  ```typescript
  // Bad: No error handling
  const fetchProjects = async () => {
    const res = await api.get('/projects/');
    return res.data;
  };
  
  // Good: Proper error handling
  const fetchProjects = async () => {
    try {
      const res = await api.get('/projects/');
      return res.data;
    } catch (error) {
      console.error('Failed to fetch projects:', error);
      throw new Error('Failed to fetch projects');
    }
  };
  ```

### Form Handling Anti-Patterns 🚨

- **Uncontrolled Form Inputs**: Don't mix controlled and uncontrolled inputs.
  ```typescript
  // Bad: Mixing controlled and uncontrolled inputs
  const Form = () => {
    const [name, setName] = useState('');
    return (
      <form>
        <input value={name} onChange={(e) => setName(e.target.value)} />
        <input defaultValue="" /> {/* Uncontrolled */}
      </form>
    );
  };
  
  // Good: Consistent controlled inputs with TanStack Form
  const Form = () => {
    const form = useAppForm({
      defaultValues: { name: '', email: '' },
      onSubmit: ({ value }) => {},
    });
    
    return (
      <form onSubmit={form.handleSubmit}>
        <form.Field name="name">
          {(field) => (
            <Input
              value={field.state.value}
              onChange={(e) => field.handleChange(e.target.value)}
            />
          )}
        </form.Field>
      </form>
    );
  };
  ```

- **Missing Form Validation**: Always validate form inputs.
  ```typescript
  // Bad: No validation
  const form = useAppForm({
    defaultValues: { email: '' },
    onSubmit: ({ value }) => {},
  });
  
  // Good: With validation
  const form = useAppForm({
    defaultValues: { email: '' },
    onSubmit: ({ value }) => {},
  });
  
  <form.Field
    name="email"
    validators={{
      onChange: z.string().email("Invalid email address"),
    }}
  >
    {/* ... */}
  </form.Field>
  ```

### Performance Anti-Patterns ⚠️

- **Unnecessary Re-renders**: Avoid creating new functions or objects in render.
  ```typescript
  // Bad: Creating new function on every render
  const Component = ({ data }) => {
    return (
      <Button onClick={() => handleClick(data)}>Click</Button>
    );
  };
  
  // Good: Memoized callback
  const Component = ({ data }) => {
    const handleButtonClick = useCallback(() => {
      handleClick(data);
    }, [data]);
    
    return (
      <Button onClick={handleButtonClick}>Click</Button>
    );
  };
  ```

- **Large Component Trees**: Don't render large lists without virtualization.
  ```typescript
  // Bad: Rendering all items at once
  const List = ({ items }) => {
    return (
      <div>
        {items.map(item => <Item key={item.id} {...item} />)}
      </div>
    );
  };
  
  // Good: Using virtualization for large lists
  import { useVirtualizer } from '@tanstack/react-virtual';
  
  const List = ({ items }) => {
    const virtualizer = useVirtualizer({
      count: items.length,
      getScrollElement: () => parentRef.current,
      estimateSize: () => 50,
    });
    
    return (
      <div ref={parentRef}>
        <div
          style={{
            height: `${virtualizer.getTotalSize()}px`,
            position: 'relative',
          }}
        >
          {virtualizer.getVirtualItems().map(virtualRow => (
            <div
              key={items[virtualRow.index].id}
              style={{
                position: 'absolute',
                top: 0,
                left: 0,
                width: '100%',
                height: `${virtualRow.size}px`,
                transform: `translateY(${virtualRow.start}px)`,
              }}
            >
              <Item {...items[virtualRow.index]} />
            </div>
          ))}
        </div>
      </div>
    );
  };
  ```

## Routing Conventions 🚨

### TanStack Router Integration 🚨

- Use file-based routing with TanStack Router's `createFileRoute` function
- Place route files in a dedicated routes directory structure that mirrors the URL structure
- Export a `Route` constant from each route file

**Rationale**: File-based routing creates a clear mapping between the URL structure and the codebase, making it easier to understand and maintain routing logic while enabling type-safe route handling.

### Route Configuration ⚠️

#### Basic Routes 👍

For simple routes without data dependencies.

#### Data-Dependent Routes 🚨

For routes that require data loading:
- Use `loader` functions to prefetch data before rendering components
- Use `validateSearch` to normalize and validate search parameters
- Use `loaderDeps` to specify dependencies for the loader function

### Search Parameters ⚠️

- Use the `getDefaultSearchValues` utility to normalize pagination parameters
- Apply consistent defaults for pagination, sorting, and filtering
- Type search parameters properly using generics

### Data Loading 🚨

- Use TanStack Query's `prefetchQuery` for data that can be loaded in parallel
- Use `ensureQueryData` for critical data that must be loaded before rendering
- Structure API payloads consistently based on route parameters and search values

**Example:**
```typescript
// In a route loader
export const Route = createFileRoute("/_app/p/$puuid/")({
  loaderDeps: ({ search }) => search,
  loader: async ({ context: { queryClient }, params, deps }) => {
    const payload = {
      uuid: params.puuid,
      limit: deps.limit,
      offset: deps.offset,
      sort_by: deps.sort_by,
      sort_direction: deps.sort_direction,
    };
    
    // Prefetch data in parallel
    await Promise.all([
      queryClient.prefetchQuery(projectQueryOptions({ uuid: params.puuid })),
      queryClient.prefetchQuery(imagesQueryOptions(payload))
    ]);
  },
  component: ProjectDetail,
});
```

### Route Parameters ⚠️

- Use descriptive names for route parameters (e.g., `$puuid` for project UUID)
- Access route parameters via the `params` object in loaders and components
- Use route parameters to construct API request payloads
