# React Code Examples

Use these examples as reference for implementing React conventions.

## API Setup: Centralized HTTP Client Configuration

This section demonstrates how to set up a global Axios instance for API calls. The centralized setup ensures consistent headers, base URL, and error handling across all API requests.

```typescript
// api.ts (global API setup)
import axios from "axios";
import { API_URL } from "@/constants.ts";

const api = axios.create({
  baseURL: API_URL,
  withCredentials: true,
});

export { api };
```

## Global Types: Type-Safe Pagination Patterns

These type definitions provide a consistent interface for paginated API requests and responses. They are used throughout the application to ensure type safety when working with API data.

```typescript
// types.ts (global types)
type PaginatedRes<T> = {
  items: T[];
  limit: number;
  offset: number;
  total: number;
};

interface PaginationReq<T> {
  // Pagination
  limit: number;
  offset: number;

  // Sorting
  sort_by: keyof T;
  sort_direction: "asc" | "desc";

  // Filtering
  filter: string;

  // Date range
  start_date?: string;
  end_date?: string;
}

export type { PaginationReq, PaginatedRes };
```

## Global Utils: Search Parameter Normalization

Utility functions provide reusable logic across the application. The `getDefaultSearchValues` function normalizes search parameters for consistent pagination, sorting, and filtering behavior.

```typescript
// utils.ts (global utils)
import { PaginationReq } from "./types.ts";

const getDefaultSearchValues = <T>(search?: Partial<PaginationReq<T>>) => ({
  offset: Number(search?.offset ?? 0),
  limit: Number(search?.limit ?? 10),
  filter: String(search?.filter ?? ""),
  sort_by: (search?.sort_by ?? "name") as keyof T,
  sort_direction: (search?.sort_direction ?? "asc") as "asc" | "desc",
});

export { getDefaultSearchValues };
```

## Global Constants

```typescript
// constants.ts (global constants)
// const API_URL = import.meta.env.VITE_API_URL
const API_URL = "/api";

export { API_URL };
```

## Feature Types: Comprehensive Type System for a Feature

Type definitions for a feature should be comprehensive, covering entity types, API payloads, and component props. This example shows the complete type system for a "Projects" feature, following the naming conventions outlined in vite.md.

```typescript
// projects/Projects.types.ts
import type { ReactElement } from "react";
import { PaginationReq } from "@/types.ts";

type Project = {
  uuid: string;
  url: string;
  type: "WORDPRESS" | "SHOPIFY" | "CUSTOM";
};

// GET /projects/:uuid
interface ProjectDetailGetPayload { // Renamed from ProjectGetPayload
  uuid: string;
}
type ProjectDetailGetResponse = Project; // Response for getting a single project

// GET /projects/
interface ProjectListGetPayload extends PaginationReq<Project> {}
type ProjectListGetResponse = PaginatedRes<Project>; // Response for getting a list of projects

// POST /projects/
interface ProjectCreatePayload extends Omit<Project, "uuid"> {
  url: string;
}

// PUT /projects/:uuid
type ProjectUpdatePayload = Partial<ProjectCreatePayload> &
  Pick<Project, "uuid">;

// Table types
interface ProjectsTableProps {
  data: Project[];
  onView?: (project: Project) => void;
  onEdit?: (project: Project) => void;
  onDelete?: (project: Project) => void;
}

type ProjectsTableComponent = (
  props: ProjectsTableProps
) => ReactElement | null;

// Form types
interface ProjectsFormProps<
  T extends ProjectCreatePayload | ProjectUpdatePayload,
> {
  formId?: string;
  onSubmit?: (data: T) => void;
  defaultValues?: T;
  isLoading?: boolean;
  hideActions?: boolean;
}

export type {
  Project,
  ProjectCreatePayload,
  ProjectUpdatePayload,
  ProjectDetailGetPayload, // Renamed
  ProjectDetailGetResponse, // Added
  ProjectListGetPayload,
  ProjectListGetResponse, // Added
  ProjectsTableComponent,
  ProjectsTableProps,
  ProjectsFormProps,
};
```

## Feature Constants

```typescript
// projects/Projects.constants.ts
const PROJECTS_ENDPOINT = "/projects/";

const PROJECTS_QUERY_KEY = "QUERY_PROJECTS";
const PROJECTS_MUTATION_KEY = "MUTATE_PROJECTS";

export { PROJECTS_ENDPOINT, PROJECTS_QUERY_KEY, PROJECTS_MUTATION_KEY };
```

## Feature API Integration: CRUD Operations with TanStack Query

The API integration layer follows a consistent pattern for each CRUD operation:
1. A function to make the API call (`fetchProject`, `createProject`, etc.)
2. A query options function for GET operations (`projectGetQueryOptions`)
3. A mutation hook for POST/PUT/DELETE operations (`useCreateProject`)

This separation allows for reuse across components and provides a clean interface for data fetching.

```typescript
// projects/Projects.api.ts
import { AxiosResponse } from "axios";
import {
  PROJECTS_ENDPOINT,
  PROJECTS_MUTATION_KEY,
  PROJECTS_QUERY_KEY,
} from "@/features/projects/Projects.constants.ts";
import {
  Project,
  ProjectCreatePayload,
  ProjectDetailGetPayload, // Renamed
  ProjectDetailGetResponse, // Added
  ProjectListGetPayload,
  ProjectListGetResponse, // Added
} from "@/features/projects/Projects.types.ts";
import { PaginatedRes } from "@/types.ts"; // PaginatedRes is used by ProjectListGetResponse
import { api } from "@/api.ts";
import { queryOptions, useMutation } from "@tanstack/react-query";

/**
 * Create a Project
 */

type CreateRes = AxiosResponse<Project>;
const createProject = async (payload: ProjectCreatePayload) => {
  const res: CreateRes = await api.post(PROJECTS_ENDPOINT, payload);
  return res.data;
};

const useCreateProject = () => {
  return useMutation({
    mutationKey: [PROJECTS_MUTATION_KEY],
    mutationFn: (payload: ProjectCreatePayload) => createProject(payload),
  });
};

/**
 * Get a Project
 * GET /projects/:uuid/
 */

type GetRes = AxiosResponse<ProjectDetailGetResponse>; // Updated response type

const fetchProject = async (payload: ProjectDetailGetPayload): Promise<ProjectDetailGetResponse> => { // Updated payload and return type
  const res: GetRes = await api.get(PROJECTS_ENDPOINT + payload.uuid + "/");
  return res.data;
};

const projectGetQueryOptions = (payload: ProjectDetailGetPayload) => { // Updated payload type
  return queryOptions({
    queryKey: [PROJECTS_QUERY_KEY, payload],
    queryFn: () => fetchProject(payload),
  });
};

/**
 * Get a list of Projects
 * GET /projects/
 */

type ListRes = AxiosResponse<ProjectListGetResponse>; // Updated response type

const fetchProjectList = async (payload: ProjectListGetPayload): Promise<ProjectListGetResponse> => { // Updated return type
  const res: ListRes = await api.get(PROJECTS_ENDPOINT, {
    params: payload,
  });
  return res.data;
};

const projectListQueryOptions = (payload: ProjectListGetPayload) => { // queryFn will return ProjectListGetResponse
  return queryOptions({
    queryKey: [PROJECTS_QUERY_KEY, payload],
    queryFn: () => fetchProjectList(payload),
  });
};

/**
 * Delete a Project
 * DELETE /projects/:uuid/
 */

const deleteProject = async (uuid: string) => {
  const res = await api.delete(PROJECTS_ENDPOINT + uuid + "/");
  return res.data;
};

const useDeleteProject = () => {
  return useMutation({
    mutationKey: [PROJECTS_MUTATION_KEY],
    mutationFn: (uuid: string) => deleteProject(uuid),
  });
};

export {
  projectGetQueryOptions,
  fetchProject,
  projectListQueryOptions,
  fetchProjectList,
  useDeleteProject,
  deleteProject,
  useCreateProject,
  createProject,
};
```

## Form Component: Type-Safe Forms with TanStack Form and Zod

Form components use TanStack Form with Zod validation to create type-safe, validated forms. This example demonstrates:
1. Generic type parameters to support both creation and update operations
2. Form field validation with Zod schemas
3. Proper error handling and display
4. Conditional rendering based on form state

```typescript
// projects/Projects.form.tsx
import type { FormEvent } from "react";
import { z } from "zod";
import { createFormHook, createFormHookContexts } from "@tanstack/react-form";
import { Input } from "@/components/ui/input.tsx";
import { Field, Label, ErrorMessage } from "@/components/ui/fieldset.tsx";
import { Button } from "@/components/ui/button.tsx";
import type {
  ProjectCreatePayload,
  ProjectUpdatePayload,
  ProjectsFormProps,
} from "@/features/projects/Projects.types.ts";

const { fieldContext, formContext } = createFormHookContexts();

const { useAppForm } = createFormHook({
  fieldComponents: {
    Input,
  },
  formComponents: {
    Button,
  },
  fieldContext,
  formContext,
});

const ProjectsForm = <T extends ProjectCreatePayload | ProjectUpdatePayload>({
  formId,
  onSubmit,
  isLoading = false,
  defaultValues,
  hideActions = false,
}: ProjectsFormProps<T>) => {
  // Type guard function to check if we have a ProjectUpdatePayload
  const isUpdatePayload = (
    value: ProjectCreatePayload | ProjectUpdatePayload | undefined
  ): value is ProjectUpdatePayload => {
    return value !== undefined && "uuid" in value;
  };

  const form = useAppForm({
    defaultValues: {
      url: defaultValues?.url ?? "",
      type:
        (defaultValues?.type as "WORDPRESS" | "SHOPIFY" | "CUSTOM") ??
        "WORDPRESS",
    },
    onSubmit: async ({ value }) => {
      // Use the type guard to safely access uuid
      const payload = isUpdatePayload(defaultValues)
        ? ({ ...value, uuid: defaultValues.uuid } as T)
        : (value as T);

      onSubmit?.(payload);
    },
  });

  const handleSubmit = (event: FormEvent<HTMLFormElement>) => {
    event.preventDefault();
    event.stopPropagation();
    form.handleSubmit();
  };

  return (
    <form
      id={formId}
      action="#"
      method="POST"
      className="grid w-full max-w-sm grid-cols-1 gap-8"
      onSubmit={handleSubmit}
    >
      <Field>
        <form.Field
          name="url"
          validators={{
            onChange: z.string().url("Invalid URL"),
            onBlur: z.string().url("Invalid URL"),
          }}
        >
          {(field) => {
            return (
              <>
                <Label htmlFor={field.name}>URL</Label>
                <Input
                  id={field.name}
                  name={field.name}
                  value={field.state.value}
                  type="url"
                  onChange={(e) => field.handleChange(e.target.value)}
                  invalid={field.state.meta.errors.length > 0}
                />
                {field.state.meta.errors.length > 0 && (
                  <ErrorMessage>
                    {field.state.meta.errors?.[0]?.message}
                  </ErrorMessage>
                )}
              </>
            );
          }}
        </form.Field>
      </Field>
      {!hideActions && (
        <form.Subscribe
          selector={(state) => [state.canSubmit, state.isSubmitting]}
          children={([_, isSubmitting]) => (
            <Button type="submit">
              {isSubmitting || isLoading ? "..." : "Submit"}
            </Button>
          )}
        />
      )}
    </form>
  );
};

export { ProjectsForm };
```

## Table Component: Data Display with Row-Level Actions

Table components display collections of entities with actions for CRUD operations. This example shows:
1. Rendering a list of items with proper key usage
2. Handling row-level actions through a dropdown menu
3. Formatting data for display
4. Implementing event handlers for table interactions

```typescript
// projects/Projects.table.tsx
import type { MouseEvent } from "react";
import { EllipsisHorizontalIcon } from "@heroicons/react/16/solid";
import { Badge } from "@/components/ui/badge";
import {
  Dropdown,
  DropdownButton,
  DropdownItem,
  DropdownMenu,
} from "@/components/ui/dropdown";
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from "@/components/ui/table";

import type {
  Project,
  ProjectsTableComponent,
} from "@/features/projects/Projects.types.ts";
import { Text } from "@/components/ui/text.tsx";

const ProjectsTable: ProjectsTableComponent = ({ data, onView, onDelete }) => {
  const handleOnDelete = (project: Project) => {
    onDelete?.(project);
  };

  return (
    <div className="mt-6 ring-1 ring-zinc-950/10 dark:ring-white/10 px-3 sm:mx-0 sm:rounded-lg">
      <Table className="[--gutter:--spacing(6)] sm:[--gutter:--spacing(8)]">
        <TableHead>
          <TableRow>
            <TableHeader>URL</TableHeader>
            <TableHeader>Status</TableHeader>
            <TableHeader className="relative w-0">
              <span className="sr-only">Actions</span>
            </TableHeader>
          </TableRow>
        </TableHead>
        <TableBody>
          {data.map((project) => (
            <TableRow key={project.uuid}>
              <TableCell>
                <div className="flex items-center gap-4">
                  <div>
                    <div className="font-medium">
                      {project.url.replace("https://", "")}
                      <span className="inline-flex ml-2">
                        <Text>({project.uuid})</Text>
                      </span>
                    </div>
                  </div>
                </div>
              </TableCell>
              <TableCell>
                <Badge color="lime">Online</Badge>
              </TableCell>
              <TableCell>
                <div className="-mx-3 -my-1.5 sm:-mx-2.5">
                  <Dropdown>
                    <DropdownButton plain aria-label="More options">
                      <EllipsisHorizontalIcon />
                    </DropdownButton>
                    <DropdownMenu anchor="bottom end">
                      <DropdownItem
                        href="#"
                        onClick={(event: MouseEvent<HTMLButtonElement>) => {
                          event.preventDefault();
                          event.stopPropagation();
                          onView?.(project);
                        }}
                      >
                        View
                      </DropdownItem>
                      <DropdownItem
                        onClick={() => {
                          handleOnDelete(project);
                        }}
                      >
                        Delete
                      </DropdownItem>
                    </DropdownMenu>
                  </Dropdown>
                </div>
              </TableCell>
            </TableRow>
          ))}
        </TableBody>
      </Table>
    </div>
  );
};

export { ProjectsTable };
```

## Page Component: Orchestrating Data and UI at the Route Level

Page components are the top-level components that handle routing, data fetching, and state management for a specific route. This example demonstrates:
1. Using TanStack Router for navigation and search parameters
2. Managing modal state for create and delete operations
3. Implementing data fetching with TanStack Query
4. Handling mutations with proper loading, success, and error states
5. Organizing callback functions by purpose

Let's break down the implementation into logical sections:

### 1. Component Setup and Imports

```typescript
// Pages/Projects.page.tsx
import { useState } from "react";
import { useSuspenseQuery } from "@tanstack/react-query";
import { getRouteApi } from "@tanstack/react-router";
import { PlusIcon } from "@heroicons/react/20/solid";
import {
  Project,
  ProjectCreatePayload,
  ProjectListGetPayload,
} from "@/features/projects/Projects.types.ts";
import {
  projectListQueryOptions,
  useDeleteProject,
  useCreateProject,
} from "@/features/projects/Projects.api.ts";
import { Heading } from "@/components/ui/heading.tsx";
import { Button } from "@/components/ui/button.tsx";
import { ProjectsEmpty } from "@/features/projects/Projects.empty.tsx";
import { ProjectsTable } from "@/features/projects/Projects.table.tsx";
import { Modal } from "@/components/common/modal/Modal.tsx";
import { ProjectsForm } from "@/features/projects/Projects.form.tsx";
import type {
  ImageListGetPayload,
  Image,
} from "@/features/images/Images.types.ts"; // Assuming 'images' is a feature module
import { getDefaultSearchValues } from "@/utils.ts";
import { Breadcrumbs } from "@/components/ui/breadcrumbs.tsx";

const routeApi = getRouteApi("/_app/p/");
```

### 2. Component State and Routing

```typescript
const ProjectsPage = () => {
  // Component state for modals and selected item
  const [isDeleteModalOpen, setIsDeleteModalOpen] = useState(false);
  const [isCreateModalOpen, setIsCreateModalOpen] = useState(false);
  const [selectedProject, setSelectedProject] = useState<Project | null>(null);
  
  // TanStack Router hooks for navigation and search params
  const search = routeApi.useSearch();
  const navigate = routeApi.useNavigate();

  // Default search parameters for navigation
  const imageListGetPayload: Omit<ImageListGetPayload, "project_uuid"> = {
    ...getDefaultSearchValues(),
    sort_by: "url" as keyof Image,
  };

  // Construct API payload from search parameters
  const projectListPayload: ProjectListGetPayload = {
    ...search,
  };
```

### 3. Data Fetching with TanStack Query

```typescript
  // Fetch projects list with TanStack Query
  const projectsQuery = useSuspenseQuery(
    projectListQueryOptions(projectListPayload)
  );
```

### 4. Create Project Mutation

```typescript
  /**
   * Create a Project
   * POST /projects/
   */

  // Mutation hook
  const createProject = useCreateProject();

  // Form callback
  const onCreate = (payload: ProjectCreatePayload) => {
    // Consider adding alternative user feedback if toast is removed (e.g., inline messages, console logs)
    createProject.mutate(payload, {
      onSuccess: () => {
        projectsQuery.refetch();
        setIsCreateModalOpen(false);
        // Example: console.log("Project created successfully");
      },
      onError: () => {
        // Example: console.error("Unable to create project");
      },
    });
  };
```

### 5. Delete Project Mutation

```typescript
  /**
   * Delete a Project
   * DELETE /projects/:uuid/
   */

  // Mutation hook
  const deleteProject = useDeleteProject();

  // Table callback
  const onTableItemDelete = (project: Project) => {
    setSelectedProject(project);
    setIsDeleteModalOpen(true);
  };

  // Form callback
  const onDelete = () => {
    if (selectedProject) {
      // Consider adding alternative user feedback if toast is removed
      deleteProject.mutate(selectedProject.uuid, {
        onSuccess: () => {
          projectsQuery.refetch();
          setIsDeleteModalOpen(false);
          setSelectedProject(null);
          // Example: console.log("Project deleted successfully");
        },
        onError: () => {
          // Example: console.error("Unable to delete project");
        },
      });
    }
  };
```

### 6. Table and Modal Callbacks

```typescript
  /**
   * Table callbacks
   */

  const onTableItemView = async (project: Project) => {
    await navigate({
      to: "/p/$puuid",
      params: { puuid: project.uuid },
      search: {
        ...imageListGetPayload,
      },
    });
  };

  /**
   * Modal callbacks
   */

  const onClose = () => {
    setIsDeleteModalOpen(false);
    setSelectedProject(null);
  };

  // Breadcrumb configuration
  let breadcrumbPages = [
    {
      name: "Projects",
      href: "/p",
      current: true,
    },
  ];
```

### 7. Component Rendering

```typescript
  return (
    <>
      {/* Delete Confirmation Modal */}
      {isDeleteModalOpen && (
        <Modal
          title={"Are you sure you want to delete this project?"}
          description={
            "All the images associated with the project will be deleted. It won't affect the website itself."
          }
          isOpen={isDeleteModalOpen}
          setIsOpen={setIsDeleteModalOpen}
          onConfirm={onDelete}
          onClose={onClose}
        />
      )}

      {/* Create Project Modal */}
      {isCreateModalOpen && (
        <Modal
          formId="create-project-form"
          title={"Create a new project"}
          description={
            "Please enter the URL of the project you want to create."
          }
          isOpen={isCreateModalOpen}
          setIsOpen={setIsCreateModalOpen}
          onClose={() => setIsCreateModalOpen(false)}
          isLoading={createProject.isPending}
        >
          <ProjectsForm
            hideActions
            formId="create-project-form"
            onSubmit={(values) => {
              // @ts-ignore
              onCreate(values);
            }}
            isLoading={false}
          />
        </Modal>
      )}

      {/* Page Header with Breadcrumbs */}
      <div className="flex py-4">
        <Breadcrumbs pages={breadcrumbPages} />
      </div>
      <div className="flex w-full flex-wrap items-end justify-between gap-4 border-b border-zinc-950/10 pb-6 dark:border-white/10">
        <Heading>Projects</Heading>
        <div className="flex gap-4">
          <Button onClick={() => setIsCreateModalOpen(true)}>
            <PlusIcon aria-hidden="true" className="-ml-0.5 mr-1.5 size-5" />
            New Project
          </Button>
        </div>
      </div>

      {/* Conditional Rendering Based on Data */}
      {projectsQuery.data.total === 0 ? (
        <ProjectsEmpty onCreate={() => setIsCreateModalOpen(true)} />
      ) : (
        <div>
          <ProjectsTable
            data={projectsQuery.data.items}
            onView={onTableItemView}
            onDelete={onTableItemDelete}
          />
        </div>
      )}
    </>
  );
};

export { ProjectsPage };
```

### Key Patterns to Note

1. **State Organization**: Modal states and selected items are grouped together at the top
2. **Data Flow**: The component follows a clear data flow from API to UI
3. **Callback Grouping**: Related callbacks are grouped together with clear comments
4. **Conditional Rendering**: The component conditionally renders different UI based on data state
5. **Modal Pattern**: Modals are conditionally rendered with appropriate props
6. **User Feedback**: Ensure mutations provide user feedback (e.g., inline messages, console logs) for loading, success, and error states.

## Modal Component: Reusable Dialog with Form Integration

The Modal component is a reusable dialog that supports forms, confirmations, and information display. This example shows:
1. Using Headless UI for accessible modal implementation
2. Supporting form submission through formId prop
3. Handling loading states during async operations
4. Managing modal visibility with proper transitions

```typescript
// components/common/modal/Modal.tsx
import { Fragment, ReactNode } from "react";
import { Dialog, Transition } from "@headlessui/react";
import { Button } from "@/components/ui/button.tsx";

interface ModalProps {
  title: string;
  description?: string;
  isOpen: boolean;
  setIsOpen: (isOpen: boolean) => void;
  onClose?: () => void;
  onConfirm?: () => void;
  children?: ReactNode;
  formId?: string;
  isLoading?: boolean;
}

const Modal = ({
  title,
  description,
  isOpen,
  setIsOpen,
  onClose,
  onConfirm,
  children,
  formId,
  isLoading = false,
}: ModalProps) => {
  const handleClose = () => {
    setIsOpen(false);
    onClose?.();
  };

  return (
    <Transition.Root show={isOpen} as={Fragment}>
      <Dialog as="div" className="relative z-10" onClose={handleClose}>
        <Transition.Child
          as={Fragment}
          enter="ease-out duration-300"
          enterFrom="opacity-0"
          enterTo="opacity-100"
          leave="ease-in duration-200"
          leaveFrom="opacity-100"
          leaveTo="opacity-0"
        >
          <div className="fixed inset-0 bg-gray-500 bg-opacity-75 transition-opacity" />
        </Transition.Child>

        <div className="fixed inset-0 z-10 w-screen overflow-y-auto">
          <div className="flex min-h-full items-end justify-center p-4 text-center sm:items-center sm:p-0">
            <Transition.Child
              as={Fragment}
              enter="ease-out duration-300"
              enterFrom="opacity-0 translate-y-4 sm:translate-y-0 sm:scale-95"
              enterTo="opacity-100 translate-y-0 sm:scale-100"
              leave="ease-in duration-200"
              leaveFrom="opacity-100 translate-y-0 sm:scale-100"
              leaveTo="opacity-0 translate-y-4 sm:translate-y-0 sm:scale-95"
            >
              <Dialog.Panel className="relative transform overflow-hidden rounded-lg bg-white px-4 pb-4 pt-5 text-left shadow-xl transition-all sm:my-8 sm:w-full sm:max-w-lg sm:p-6">
                <div>
                  <div className="mt-3 text-center sm:mt-5">
                    <Dialog.Title
                      as="h3"
                      className="text-base font-semibold leading-6 text-gray-900"
                    >
                      {title}
                    </Dialog.Title>
                    {description && (
                      <div className="mt-2">
                        <p className="text-sm text-gray-500">{description}</p>
                      </div>
                    )}
                    {children && <div className="mt-4">{children}</div>}
                  </div>
                </div>
                <div className="mt-5 sm:mt-6 sm:grid sm:grid-flow-row-dense sm:grid-cols-2 sm:gap-3">
                  {onConfirm && (
                    <Button
                      type={formId ? "submit" : "button"}
                      form={formId}
                      onClick={formId ? undefined : onConfirm}
                      className="inline-flex w-full justify-center sm:col-start-2"
                      disabled={isLoading}
                    >
                      {isLoading ? "..." : "Confirm"}
                    </Button>
                  )}
                  <Button
                    type="button"
                    onClick={handleClose}
                    className="mt-3 inline-flex w-full justify-center sm:col-start-1 sm:mt-0"
                    variant="secondary"
                  >
                    Cancel
                  </Button>
                </div>
              </Dialog.Panel>
            </Transition.Child>
          </div>
        </div>
      </Dialog>
    </Transition.Root>
  );
};

export { Modal };
```

## Empty State Component: User-Friendly Zero Data States

Empty state components provide a user-friendly interface when no data is available. This example demonstrates:
1. Clear visual indication of the empty state
2. Actionable guidance for the user
3. Consistent styling with the rest of the application

```typescript
// projects/Projects.empty.tsx
import { PlusIcon } from "@heroicons/react/24/outline";
import { Button } from "@/components/ui/button.tsx";

interface ProjectsEmptyProps {
  onCreate: () => void;
}

const ProjectsEmpty = ({ onCreate }: ProjectsEmptyProps) => {
  return (
    <div className="text-center py-12">
      <svg
        className="mx-auto h-12 w-12 text-gray-400"
        fill="none"
        viewBox="0 0 24 24"
        stroke="currentColor"
        aria-hidden="true"
      >
        <path
          vectorEffect="non-scaling-stroke"
          strokeLinecap="round"
          strokeLinejoin="round"
          strokeWidth={2}
          d="M9 13h6m-3-3v6m-9 1V7a2 2 0 012-2h6l2 2h6a2 2 0 012 2v8a2 2 0 01-2 2H5a2 2 0 01-2-2z"
        />
      </svg>
      <h3 className="mt-2 text-sm font-semibold text-gray-900">No projects</h3>
      <p className="mt-1 text-sm text-gray-500">
        Get started by creating a new project.
      </p>
      <div className="mt-6">
        <Button onClick={onCreate}>
          <PlusIcon className="-ml-0.5 mr-1.5 h-5 w-5" aria-hidden="true" />
          New Project
        </Button>
      </div>
    </div>
  );
};

export { ProjectsEmpty };
```

## Before/After Examples: Refactoring Patterns

These examples demonstrate how to refactor common code patterns to follow our conventions and best practices.

### API Integration: From Direct Calls to TanStack Query

**Before:**
```typescript
// Component with direct API calls
const ProjectsList = () => {
  const [projects, setProjects] = useState<Project[]>([]);
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const fetchProjects = async () => {
      setIsLoading(true);
      setError(null);
      try {
        const response = await axios.get('/api/projects');
        setProjects(response.data.items);
      } catch (err) {
        setError('Failed to fetch projects');
        console.error(err);
      } finally {
        setIsLoading(false);
      }
    };

    fetchProjects();
  }, []);

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;

  return (
    <div>
      <h1>Projects</h1>
      <ul>
        {projects.map(project => (
          <li key={project.uuid}>{project.url}</li>
        ))}
      </ul>
    </div>
  );
};
```

**After:**
```typescript
// API layer
// projects/Projects.api.ts
const fetchProjectList = async (payload: ProjectListGetPayload): Promise<ProjectListGetResponse> => {
  const res: AxiosResponse<ProjectListGetResponse> = await api.get(PROJECTS_ENDPOINT, {
    params: payload,
  });
  return res.data;
};

const projectListQueryOptions = (payload: ProjectListGetPayload) => {
  return queryOptions({
    queryKey: [PROJECTS_QUERY_KEY, payload],
    queryFn: () => fetchProjectList(payload),
  });
};

// Component with TanStack Query
// projects/Projects.page.tsx
const ProjectsPage = () => {
  const projectListPayload: ProjectListGetPayload = {
    ...getDefaultSearchValues<Project>(),
  };

  const projectsQuery = useSuspenseQuery(
    projectListQueryOptions(projectListPayload)
  );

  return (
    <div>
      <h1>Projects</h1>
      <ProjectsTable 
        data={projectsQuery.data.items} 
        onView={handleViewProject}
      />
    </div>
  );
};
```

### Form Handling: From Uncontrolled to TanStack Form

**Before:**
```typescript
// Form with manual state management
const ProjectForm = () => {
  const [url, setUrl] = useState('');
  const [type, setType] = useState('WORDPRESS');
  const [urlError, setUrlError] = useState('');
  
  const validateUrl = (value: string) => {
    if (!value) {
      setUrlError('URL is required');
      return false;
    }
    if (!value.startsWith('http')) {
      setUrlError('URL must start with http:// or https://');
      return false;
    }
    setUrlError('');
    return true;
  };
  
  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    if (!validateUrl(url)) return;
    
    // Submit form
    console.log({ url, type });
  };
  
  return (
    <form onSubmit={handleSubmit}>
      <div>
        <label htmlFor="url">URL</label>
        <input
          id="url"
          value={url}
          onChange={(e) => setUrl(e.target.value)}
          onBlur={() => validateUrl(url)}
        />
        {urlError && <div className="error">{urlError}</div>}
      </div>
      
      <div>
        <label htmlFor="type">Type</label>
        <select
          id="type"
          value={type}
          onChange={(e) => setType(e.target.value)}
        >
          <option value="WORDPRESS">WordPress</option>
          <option value="SHOPIFY">Shopify</option>
          <option value="CUSTOM">Custom</option>
        </select>
      </div>
      
      <button type="submit">Submit</button>
    </form>
  );
};
```

**After:**
```typescript
// Form with TanStack Form and Zod validation
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
      <Field>
        <form.Field
          name="url"
          validators={{
            onChange: z.string().url("Invalid URL"),
            onBlur: z.string().url("Invalid URL"),
          }}
        >
          {(field) => (
            <>
              <Label htmlFor={field.name}>URL</Label>
              <Input
                id={field.name}
                value={field.state.value}
                onChange={(e) => field.handleChange(e.target.value)}
                onBlur={field.handleBlur}
                invalid={field.state.meta.errors.length > 0}
              />
              {field.state.meta.errors.length > 0 && (
                <ErrorMessage>
                  {field.state.meta.errors[0].message}
                </ErrorMessage>
              )}
            </>
          )}
        </form.Field>
      </Field>
      
      <Field>
        <form.Field name="type">
          {(field) => (
            <>
              <Label htmlFor={field.name}>Type</Label>
              <Select
                id={field.name}
                value={field.state.value}
                onChange={(value) => field.handleChange(value)}
              >
                <option value="WORDPRESS">WordPress</option>
                <option value="SHOPIFY">Shopify</option>
                <option value="CUSTOM">Custom</option>
              </Select>
            </>
          )}
        </form.Field>
      </Field>
      
      <form.Subscribe
        selector={(state) => [state.canSubmit, state.isSubmitting]}
        children={([canSubmit, isSubmitting]) => (
          <Button type="submit" disabled={!canSubmit || isSubmitting}>
            {isSubmitting ? "Submitting..." : "Submit"}
          </Button>
        )}
      />
    </form>
  );
};
```

### Component Organization: From Monolithic to Feature-Based

**Before:**
```typescript
// All in one file
// ProjectsPage.tsx
import { useState, useEffect } from 'react';
import axios from 'axios';

interface Project {
  uuid: string;
  url: string;
  type: string;
}

const ProjectsPage = () => {
  const [projects, setProjects] = useState<Project[]>([]);
  const [isLoading, setIsLoading] = useState(false);
  const [isCreateModalOpen, setIsCreateModalOpen] = useState(false);
  const [newProjectUrl, setNewProjectUrl] = useState('');
  const [newProjectType, setNewProjectType] = useState('WORDPRESS');
  
  useEffect(() => {
    fetchProjects();
  }, []);
  
  const fetchProjects = async () => {
    setIsLoading(true);
    try {
      const response = await axios.get('/api/projects');
      setProjects(response.data.items);
    } catch (error) {
      console.error('Failed to fetch projects', error);
    } finally {
      setIsLoading(false);
    }
  };
  
  const createProject = async () => {
    try {
      await axios.post('/api/projects', {
        url: newProjectUrl,
        type: newProjectType
      });
      setIsCreateModalOpen(false);
      setNewProjectUrl('');
      fetchProjects();
    } catch (error) {
      console.error('Failed to create project', error);
    }
  };
  
  const deleteProject = async (uuid: string) => {
    try {
      await axios.delete(`/api/projects/${uuid}`);
      fetchProjects();
    } catch (error) {
      console.error('Failed to delete project', error);
    }
  };
  
  return (
    <div>
      <h1>Projects</h1>
      <button onClick={() => setIsCreateModalOpen(true)}>New Project</button>
      
      {isLoading ? (
        <p>Loading...</p>
      ) : (
        <table>
          <thead>
            <tr>
              <th>URL</th>
              <th>Type</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            {projects.map(project => (
              <tr key={project.uuid}>
                <td>{project.url}</td>
                <td>{project.type}</td>
                <td>
                  <button onClick={() => deleteProject(project.uuid)}>Delete</button>
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      )}
      
      {isCreateModalOpen && (
        <div className="modal">
          <h2>Create Project</h2>
          <div>
            <label>URL</label>
            <input 
              value={newProjectUrl} 
              onChange={e => setNewProjectUrl(e.target.value)} 
            />
          </div>
          <div>
            <label>Type</label>
            <select 
              value={newProjectType} 
              onChange={e => setNewProjectType(e.target.value)}
            >
              <option value="WORDPRESS">WordPress</option>
              <option value="SHOPIFY">Shopify</option>
              <option value="CUSTOM">Custom</option>
            </select>
          </div>
          <div>
            <button onClick={createProject}>Create</button>
            <button onClick={() => setIsCreateModalOpen(false)}>Cancel</button>
          </div>
        </div>
      )}
    </div>
  );
};

export default ProjectsPage;
```

**After:**
Feature-based organization with separate files:

1. **Types (Projects.types.ts)**:
```typescript
import type { ReactElement } from "react";
import { PaginationReq } from "@/types.ts";

type Project = {
  uuid: string;
  url: string;
  type: "WORDPRESS" | "SHOPIFY" | "CUSTOM";
};

interface ProjectListGetPayload extends PaginationReq<Project> {}

interface ProjectCreatePayload extends Omit<Project, "uuid"> {
  url: string;
}

interface ProjectsTableProps {
  data: Project[];
  onDelete?: (project: Project) => void;
}

interface ProjectsFormProps<T extends ProjectCreatePayload> {
  formId?: string;
  onSubmit?: (data: T) => void;
  defaultValues?: T;
}

export type {
  Project,
  ProjectCreatePayload,
  ProjectListGetPayload,
  ProjectsTableProps,
  ProjectsFormProps,
};
```

2. **API (Projects.api.ts)**:
```typescript
import { api } from "@/api.ts";
import { queryOptions, useMutation } from "@tanstack/react-query";
import {
  Project,
  ProjectCreatePayload,
  ProjectListGetPayload,
} from "./Projects.types.ts";
import { PaginatedRes } from "@/types.ts";

const PROJECTS_ENDPOINT = "/projects/";
const PROJECTS_QUERY_KEY = "QUERY_PROJECTS";
const PROJECTS_MUTATION_KEY = "MUTATE_PROJECTS";

const fetchProjectList = async (payload: ProjectListGetPayload) => {
  const res = await api.get(PROJECTS_ENDPOINT, { params: payload });
  return res.data as PaginatedRes<Project>;
};

const projectListQueryOptions = (payload: ProjectListGetPayload) => {
  return queryOptions({
    queryKey: [PROJECTS_QUERY_KEY, payload],
    queryFn: () => fetchProjectList(payload),
  });
};

const createProject = async (payload: ProjectCreatePayload) => {
  const res = await api.post(PROJECTS_ENDPOINT, payload);
  return res.data as Project;
};

const useCreateProject = () => {
  return useMutation({
    mutationKey: [PROJECTS_MUTATION_KEY],
    mutationFn: (payload: ProjectCreatePayload) => createProject(payload),
  });
};

const deleteProject = async (uuid: string) => {
  const res = await api.delete(`${PROJECTS_ENDPOINT}${uuid}/`);
  return res.data;
};

const useDeleteProject = () => {
  return useMutation({
    mutationKey: [PROJECTS_MUTATION_KEY],
    mutationFn: (uuid: string) => deleteProject(uuid),
  });
};

export {
  projectListQueryOptions,
  useCreateProject,
  useDeleteProject,
};
```

3. **Table (Projects.table.tsx)**:
```typescript
import type { Project, ProjectsTableProps } from "./Projects.types.ts";
import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from "@/components/ui/table";

const ProjectsTable = ({ data, onDelete }: ProjectsTableProps) => {
  return (
    <Table>
      <TableHead>
        <TableRow>
          <TableHeader>URL</TableHeader>
          <TableHeader>Type</TableHeader>
          <TableHeader>Actions</TableHeader>
        </TableRow>
      </TableHead>
      <TableBody>
        {data.map((project) => (
          <TableRow key={project.uuid}>
            <TableCell>{project.url}</TableCell>
            <TableCell>{project.type}</TableCell>
            <TableCell>
              <button onClick={() => onDelete?.(project)}>Delete</button>
            </TableCell>
          </TableRow>
        ))}
      </TableBody>
    </Table>
  );
};

export { ProjectsTable };
```

4. **Page (Projects.page.tsx)**:
```typescript
import { useState } from "react";
import { useSuspenseQuery } from "@tanstack/react-query";
import { Project, ProjectCreatePayload } from "./Projects.types.ts";
import { projectListQueryOptions, useCreateProject, useDeleteProject } from "./Projects.api.ts";
import { ProjectsTable } from "./Projects.table.tsx";
import { ProjectsForm } from "./Projects.form.tsx";
import { Modal } from "@/components/common/modal/Modal.tsx";
import { getDefaultSearchValues } from "@/utils.ts";

const ProjectsPage = () => {
  // State
  const [isCreateModalOpen, setIsCreateModalOpen] = useState(false);
  const [isDeleteModalOpen, setIsDeleteModalOpen] = useState(false);
  const [selectedProject, setSelectedProject] = useState<Project | null>(null);
  
  // Queries
  const projectsQuery = useSuspenseQuery(
    projectListQueryOptions(getDefaultSearchValues())
  );
  
  // Mutations
  const createProject = useCreateProject();
  const deleteProject = useDeleteProject();
  
  // Callbacks
  const handleCreate = (payload: ProjectCreatePayload) => {
    // Consider adding alternative user feedback
    createProject.mutate(payload, {
      onSuccess: () => {
        projectsQuery.refetch();
        setIsCreateModalOpen(false);
        // Example: console.log("Project created successfully");
      },
      onError: () => {
        // Example: console.error("Failed to create project");
      },
    });
  };
  
  const handleDelete = () => {
    if (!selectedProject) return;
    
    // Consider adding alternative user feedback
    deleteProject.mutate(selectedProject.uuid, {
      onSuccess: () => {
        projectsQuery.refetch();
        setIsDeleteModalOpen(false);
        setSelectedProject(null);
        // Example: console.log("Project deleted successfully");
      },
      onError: () => {
        // Example: console.error("Failed to delete project");
      },
    });
  };
  
  const onTableItemDelete = (project: Project) => {
    setSelectedProject(project);
    setIsDeleteModalOpen(true);
  };
  
  return (
    <div>
      <h1>Projects</h1>
      <button onClick={() => setIsCreateModalOpen(true)}>New Project</button>
      
      <ProjectsTable 
        data={projectsQuery.data.items} 
        onDelete={onTableItemDelete} 
      />
      
      {isCreateModalOpen && (
        <Modal
          title="Create Project"
          isOpen={isCreateModalOpen}
          setIsOpen={setIsCreateModalOpen}
          formId="create-project-form"
        >
          <ProjectsForm
            formId="create-project-form"
            onSubmit={handleCreate}
          />
        </Modal>
      )}
      
      {isDeleteModalOpen && selectedProject && (
        <Modal
          title="Delete Project"
          description="Are you sure you want to delete this project?"
          isOpen={isDeleteModalOpen}
          setIsOpen={setIsDeleteModalOpen}
          onConfirm={handleDelete}
        />
      )}
    </div>
  );
};

export { ProjectsPage };
```

### Error Handling: From Basic to Comprehensive

**Before:**
```typescript
// Basic error handling
const fetchProjects = async () => {
  try {
    const response = await axios.get('/api/projects');
    return response.data;
  } catch (error) {
    console.error('Failed to fetch projects', error);
    throw error;
  }
};

const ProjectsPage = () => {
  const [projects, setProjects] = useState([]);
  const [error, setError] = useState(null);
  
  useEffect(() => {
    fetchProjects()
      .then(data => setProjects(data.items))
      .catch(err => setError('Failed to load projects'));
  }, []);
  
  if (error) return <div>Error: {error}</div>;
  
  return (
    <div>
      <h1>Projects</h1>
      <ul>
        {projects.map(project => (
          <li key={project.uuid}>{project.url}</li>
        ))}
      </ul>
    </div>
  );
};
```

**After:**
```typescript
// Comprehensive error handling
// Error types
interface ApiError {
  message: string;
  code: string;
  status: number;
}

// API function with structured error handling
const fetchProjects = async (payload: ProjectListGetPayload): Promise<ProjectListGetResponse> => {
  try {
    const res: AxiosResponse<ProjectListGetResponse> = await api.get(PROJECTS_ENDPOINT, { params: payload });
    return res.data;
  } catch (error) {
    if (axios.isAxiosError(error) && error.response) {
      const apiError: ApiError = {
        message: error.response.data?.message || "Failed to fetch projects",
        code: error.response.data?.code || "UNKNOWN_ERROR",
        status: error.response.status,
      };
      console.error("API Error:", apiError);
      throw apiError;
    }
    
    console.error("Unexpected error:", error);
    throw {
      message: "An unexpected error occurred",
      code: "UNKNOWN_ERROR",
      status: 500,
    };
  }
};

// Query with retry logic
const projectListQueryOptions = (payload: ProjectListGetPayload) => {
  return queryOptions({
    queryKey: [PROJECTS_QUERY_KEY, payload],
    queryFn: () => fetchProjects(payload),
    retry: (failureCount, error: ApiError) => {
      // Don't retry for client errors
      if (error.status >= 400 && error.status < 500) {
        return false;
      }
      // Retry server errors up to 3 times
      return failureCount < 3;
    },
  });
};

// Component with error boundary
const ProjectsPageContainer = () => {
  return (
    <ErrorBoundary
      fallback={({ error, resetErrorBoundary }) => (
        <ErrorFallback 
          error={error as ApiError} 
          resetErrorBoundary={resetErrorBoundary}
        />
      )}
    >
      <Suspense fallback={<LoadingSpinner />}>
        <ProjectsPage />
      </Suspense>
    </ErrorBoundary>
  );
};

// Error fallback component
const ErrorFallback = ({ 
  error, 
  resetErrorBoundary 
}: { 
  error: ApiError; 
  resetErrorBoundary: () => void;
}) => {
  // Log error to monitoring service
  useEffect(() => {
    // logErrorToService(error);
  }, [error]);

  return (
    <div className="p-6 bg-red-50 rounded-lg">
      <h3 className="text-lg font-medium text-red-800">
        {error.status === 404 ? "Projects not found" : "Error loading projects"}
      </h3>
      <p className="mt-2 text-sm text-red-700">{error.message}</p>
      <div className="mt-4">
        <Button onClick={resetErrorBoundary} variant="secondary">
          Try again
        </Button>
      </div>
    </div>
  );
};
```

## Error Handling Patterns: Robust Error Management Strategies

Proper error handling is critical for creating robust applications. These examples demonstrate best practices for handling errors in different contexts.

### API Error Handling: Typed Error Responses

This example shows how to implement error handling in API functions with proper error typing and messaging.

```typescript
// Error types
interface ApiError {
  message: string;
  code: string;
  status: number;
}

// API function with error handling
const fetchProject = async (payload: ProjectDetailGetPayload): Promise<ProjectDetailGetResponse> => {
  try {
    const res: AxiosResponse<ProjectDetailGetResponse> = await api.get(
      PROJECTS_ENDPOINT + payload.uuid + "/"
    );
    return res.data;
  } catch (error) {
    // Handle Axios errors
    if (axios.isAxiosError(error) && error.response) {
      const apiError: ApiError = {
        message: error.response.data?.message || "Failed to fetch project",
        code: error.response.data?.code || "UNKNOWN_ERROR",
        status: error.response.status,
      };
      
      // Log the error for debugging
      console.error("API Error:", apiError);
      
      // Throw a formatted error for consistent handling
      throw apiError;
    }
    
    // Handle unexpected errors
    console.error("Unexpected error:", error);
    throw {
      message: "An unexpected error occurred",
      code: "UNKNOWN_ERROR",
      status: 500,
    };
  }
};
```

### Query Error Handling: TanStack Query Error Boundaries

This example demonstrates how to handle query errors with TanStack Query, including error boundaries and fallback UI.

```typescript
// Query with error handling
const projectQueryOptions = (payload: ProjectDetailGetPayload) => {
  return queryOptions({
    queryKey: [PROJECTS_QUERY_KEY, payload],
    queryFn: () => fetchProject(payload), // fetchProject now returns ProjectDetailGetResponse
    retry: (failureCount, error: ApiError) => {
      // Don't retry for client errors (4xx)
      if (error.status >= 400 && error.status < 500) {
        return false;
      }
      // Retry server errors (5xx) up to 3 times
      return failureCount < 3;
    },
  });
};

// Component with error handling
const ProjectDetail = () => {
  const params = useParams();
  
  // Use error boundary for query errors
  return (
    <ErrorBoundary
      fallback={({ error, resetErrorBoundary }) => (
        <ErrorFallback 
          error={error as ApiError} 
          resetErrorBoundary={resetErrorBoundary}
        />
      )}
    >
      <Suspense fallback={<LoadingSpinner />}>
        <ProjectDetailContent uuid={params.uuid} />
      </Suspense>
    </ErrorBoundary>
  );
};

// Error fallback component
const ErrorFallback = ({ 
  error, 
  resetErrorBoundary 
}: { 
  error: ApiError; 
  resetErrorBoundary: () => void;
}) => {
  return (
    <div className="p-6 bg-red-50 rounded-lg">
      <h3 className="text-lg font-medium text-red-800">
        {error.status === 404 ? "Project not found" : "Error loading project"}
      </h3>
      <p className="mt-2 text-sm text-red-700">{error.message}</p>
      <div className="mt-4">
        <Button onClick={resetErrorBoundary} variant="secondary">
          Try again
        </Button>
      </div>
    </div>
  );
};
```

### Form Error Handling: Validation and Submission Errors

This example shows how to handle form validation errors and submission errors.

```typescript
// Form with error handling
const ProjectsForm = <T extends ProjectCreatePayload | ProjectUpdatePayload>({
  formId,
  onSubmit,
  defaultValues,
}: ProjectsFormProps<T>) => {
  // Track server errors separately from form validation
  const [serverError, setServerError] = useState<string | null>(null);
  
  const form = useAppForm({
    defaultValues: {
      url: defaultValues?.url ?? "",
      type: defaultValues?.type ?? "WORDPRESS",
    },
    onSubmit: async ({ value }) => {
      try {
        // Clear any previous server errors
        setServerError(null);
        
        // Submit the form
        await onSubmit?.(value as T);
      } catch (error) {
        // Handle submission errors
        if ((error as ApiError).message) {
          setServerError((error as ApiError).message);
        } else {
          setServerError("An unexpected error occurred. Please try again.");
        }
        
        // Return false to prevent form from resetting on error
        return false;
      }
    },
  });

  return (
    <form id={formId} onSubmit={form.handleSubmit}>
      {/* Display server errors at the top of the form */}
      {serverError && (
        <div className="mb-4 p-3 bg-red-50 border border-red-200 rounded text-red-700 text-sm">
          {serverError}
        </div>
      )}
      
      <form.Field
        name="url"
        validators={{
          onChange: z.string().url("Invalid URL"),
          onBlur: z.string().url("Invalid URL"),
        }}
      >
        {(field) => (
          <div>
            <Label htmlFor={field.name}>URL</Label>
            <Input
              id={field.name}
              value={field.state.value}
              onChange={(e) => field.handleChange(e.target.value)}
              onBlur={field.handleBlur}
              invalid={field.state.meta.errors.length > 0}
            />
            {field.state.meta.errors.length > 0 && (
              <ErrorMessage>
                {field.state.meta.errors[0].message}
              </ErrorMessage>
            )}
          </div>
        )}
      </form.Field>
      
      {/* Form actions */}
      <div className="mt-4 flex justify-end">
        <Button type="submit" disabled={!form.state.canSubmit || form.state.isSubmitting}>
          {form.state.isSubmitting ? "Submitting..." : "Submit"}
        </Button>
      </div>
    </form>
  );
};
```

### Mutation Error Handling: User Feedback for Failed Operations

This example demonstrates how to handle errors in mutation operations with proper user feedback.

```typescript
// Component with mutation error handling
const ProjectsPage = () => {
  // State for tracking specific error messages
  const [deleteError, setDeleteError] = useState<string | null>(null);
  
  // Delete mutation with error handling
  const deleteProject = useDeleteProject();
  
  const onDelete = () => {
    if (selectedProject) {
      // Clear previous errors
      setDeleteError(null);
      
      // Consider alternative loading indication if toast is removed
      // Example: setIsLoading(true);
      
      deleteProject.mutate(selectedProject.uuid, {
        onSuccess: () => {
          // Handle success
          projectsQuery.refetch();
          setIsDeleteModalOpen(false);
          setSelectedProject(null);
          // Example: console.log("Project deleted successfully");
          // Example: setIsLoading(false);
        },
        onError: (error) => {
          // Handle specific error cases
          if ((error as ApiError).status === 403) {
            setDeleteError("You don't have permission to delete this project.");
            // Example: console.error("Permission denied");
          } else if ((error as ApiError).status === 409) {
            setDeleteError("This project is currently in use and cannot be deleted.");
            // Example: console.error("Cannot delete project");
          } else {
            // Generic error handling
            setDeleteError((error as ApiError).message || "Failed to delete project");
            // Example: console.error("Unable to delete project");
          }
          // Example: setIsLoading(false);
        },
      });
    }
  };
  
  // In the modal component, display the error
  return (
    <Modal
      title="Delete Project"
      isOpen={isDeleteModalOpen}
      onConfirm={onDelete}
      onClose={() => {
        setIsDeleteModalOpen(false);
        setDeleteError(null);
      }}
    >
      <p>Are you sure you want to delete this project?</p>
      
      {/* Display error message if present */}
      {deleteError && (
        <div className="mt-4 p-3 bg-red-50 border border-red-200 rounded text-red-700 text-sm">
          {deleteError}
        </div>
      )}
    </Modal>
  );
};
```

### Global Error Handling: Application-Wide Error Boundaries

This example shows how to implement global error handling for uncaught exceptions.

```typescript
// In your main.tsx or App.tsx
import { ErrorBoundary } from 'react-error-boundary';

const GlobalErrorFallback = ({ error, resetErrorBoundary }) => {
  // Log the error to a service like Sentry
  useEffect(() => {
    console.error('Global error:', error);
    // logErrorToService(error);
  }, [error]);

  return (
    <div className="min-h-screen flex items-center justify-center bg-gray-50">
      <div className="max-w-md w-full p-6 bg-white rounded-lg shadow-lg">
        <h2 className="text-2xl font-bold text-red-600 mb-4">
          Something went wrong
        </h2>
        <p className="text-gray-700 mb-4">
          The application encountered an unexpected error. Our team has been notified.
        </p>
        <div className="mt-6">
          <Button onClick={resetErrorBoundary}>
            Try again
          </Button>
        </div>
      </div>
    </div>
  );
};

const App = () => {
  return (
    <ErrorBoundary FallbackComponent={GlobalErrorFallback}>
      <Router>
        {/* Your app routes */}
      </Router>
    </ErrorBoundary>
  );
};
```

## Project Structure: Feature-Based Organization

```
/src
  /features                 # Top-level directory for all features
    /[feature]              # Each feature module resides here (e.g., /features/Projects/)
      index.ts                # Optional barrel file for the feature
      [Feature].types.ts    # Type definitions
      [Feature].api.ts      # API integration
      [Feature].constants.ts # Constants
      [Feature].form.tsx    # Form component
      [Feature].table.tsx   # Table component
      [Feature].detail.tsx  # Detail component
      [Feature].empty.tsx   # Empty state component
  /components               # For shared, non-feature-specific UI and common components
    /ui                     # Shared UI components (e.g., Button, Input)
    /common                 # Shared common components (e.g., Modal, Layout)
  /Pages
    [Feature].page.tsx    # Page components for different routes (e.g., Projects.page.tsx, Contacts.page.tsx)
  /api.ts                   # Global API setup
  /constants.ts             # Global constants
  /types.ts                 # Global type definitions
  /utils.ts                 # Global utility functions
```

## Routing Conventions: Type-Safe Routing with TanStack Router

These examples demonstrate how to implement routing with TanStack Router following the conventions in vite.md.

### TanStack Router Integration

This example shows both basic routes and data-dependent routes with proper type safety and data prefetching.

```typescript
// Basic route without data dependencies
import { createFileRoute } from "@tanstack/react-router";
import { AppPage } from "@/Pages/AppPage.tsx"; // Page components are now in /Pages

export const Route = createFileRoute("/_app/")({
  component: AppPage,
});

// Data-dependent route
// Assuming FeatureDetail is a page component, it would be imported from @/Pages/FeatureDetail.tsx
// import { FeatureDetail } from "@/Pages/FeatureDetail.tsx";
export const Route = createFileRoute("/_app/p/$puuid/")({
  validateSearch: (search?: Record<string, unknown>) => {
    return {
      ...getDefaultSearchValues<Image>(search), // Image type would be imported from its definition file
      sort_by: "url" as keyof Image,
    };
  },
  loaderDeps: ({ search }) => {
    return search;
  },
  loader: async ({ context: { queryClient }, params, deps }) => {
    // Prefetch required data
    // dataQueryOptions and payload are illustrative; specific query options would be imported
    await queryClient.prefetchQuery(dataQueryOptions(payload)); 
  },
  component: FeatureDetail, // FeatureDetail component would be imported from @/Pages/FeatureDetail.tsx
});
```

### Search Parameters

This example shows how to normalize and validate search parameters for consistent pagination, sorting, and filtering.

```typescript
const searchParams = {
  ...getDefaultSearchValues<EntityType>(search),
  sort_by: "name" as keyof EntityType,
};
```

### Data Loading

This example demonstrates how to structure API payloads based on route parameters and search values for data loading.

```typescript
const payload: EntityListGetPayload = {
  uuid: params.uuid,
  limit: deps.limit,
  offset: deps.offset,
  sort_by: deps.sort_by,
  sort_direction: deps.sort_direction,
  filter: deps.filter,
};

await queryClient.prefetchQuery(entityQueryOptions(payload));
```
