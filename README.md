# Codebase Restructuring Proposal

## Current Issues

Based on the analysis of the codebase, I've identified several key issues that need to be addressed:

1. **Inconsistent Routing & Naming**: The routing system uses Pascal case paths with multiple routing files containing conflicting content, making navigation difficult.

2. **Disorganized Component Structure**: Components and pages are scattered throughout the codebase without a clear organizational structure. For example, there are Pages & components folders nested inside a pages folder.

3. **Multiple UI Libraries**: The project uses at least 4 UI libraries simultaneously (Tailwind, Material UI, Mantine, and Ant Design based on package.json), creating inconsistency and bloating the bundle size. 

4. **Underutilized Modern State Management**: TanStack Query is installed but appears underutilized, while Redux is overused for state that could be managed more efficiently with Context API and TanStack Query.

5. **Ignored Path Aliases**: Although tsconfig paths are configured, the codebase still uses relative imports everywhere.

6. **Inconsistent Form Handling**: Formik and Yup are installed, but manual form validation is still being implemented in many places.

7. **Missing Documentation**: There's no clear README explaining the application structure, making onboarding difficult.

## Proposed Solution

### Proposed Folder Structure

```
src/
├── assets/             # Static assets like images, fonts
├── components/         # Reusable UI components
│   ├── common/         # Generic components used across modules
│   ├── sales/          # Components specific to the sales module
│   ├── layout/         # Layout components (header, sidebar, etc.)
│   └── ui/             # UI library components (shadcn implementations)
├── config/             # App configuration files
│   ├── routes.ts       # Centralized route definitions
│   └── constants.ts    # App-wide constants
├── hooks/              # Custom React hooks
│   ├── common/         # Generic hooks
│   └── [module]/       # Module-specific hooks
├── pages/              # Page components, organized by module
│   ├── auth/           # Authentication pages
│   ├── dashboard/      # Dashboard pages
│   └── sales/          # Sales module pages
├── providers/          # Context providers
│   ├── AuthProvider.tsx
│   └── ThemeProvider.tsx
├── services/           # API services
│   ├── api.ts          # Base API configuration
│   ├── common/         # Common service utilities
│   └── [module]/       # Module-specific API calls
├── types/              # TypeScript type definitions
│   ├── common/         # Shared types
│   └── [module]/       # Module-specific types
└── utils/              # Utility functions and helpers
    ├── format.ts       # Formatting utilities
    └── validation.ts   # Validation helpers
```

## Specific Recommendations

### 1. File Naming Conventions

- **Folders**: All lowercase, kebab-case for multi-word folders (e.g., `user-management`)
- **Components**: PascalCase (e.g., `DataTable.tsx`, `UserForm.tsx`)
- **Hooks**: camelCase with 'use' prefix (e.g., `useAuth.ts`, `useDevise.ts`)
- **Services**: camelCase with 'Service' suffix (e.g., `deviseService.ts`)
- **Routes**: All lowercase, kebab-case (e.g., `/finance/taux-change`)

```typescript
// ✅ Components: PascalCase
// src/components/sales/customer/list/DeviseForm.tsx
export const DeviseForm = ({ ... }) => {
  // ...
};

// ✅ Hooks: camelCase with 'use' prefix
// src/hooks/sales/useDevise.ts
export const useDevise = () => {
  // ...
};

// ✅ Services: camelCase with 'Service' suffix
// src/services/sales/deviseService.ts
export const deviseService = {
  getDevises: async () => { /* ... */ },
  // ...
};

// ✅ Types: Descriptive PascalCase
// src/types/sales/finance.ts
export interface Devise {
  id: number;
  code: string;
  nom: string;
  // ...
}

// ✅ Route definitions: kebab-case
// src/config/routes.ts
export const ROUTES = {
  FINANCE: {
    DEVISES: '/sales/devises',
    TAUX_CHANGE: '/sales/taux-change',
    // ...
  },
  // ...
};
```

### 2. UI Library Standardization

I recommend standardizing on a single UI approach:

1. **Use Tailwind CSS as the primary styling solution**: It's already installed and provides excellent flexibility.
2. **Implement shadcn/ui**: This works with Tailwind and provides accessible, customizable components without the overhead of large UI libraries.
3. **Remove redundant UI libraries**: MUI, Mantine, and Ant Design are all solving similar problems. Pick one (preferably shadcn) and remove the others.

```typescript
// ❌ Before: Using multiple UI libraries
import { TextField, Button } from "@mui/material";
import { Select } from "@mantine/core";
import { DatePicker } from "antd";

const DeviseForm = () => {
  return (
    <form>
      <TextField label="Code" />
      <Select label="Type" data={[]} />
      <DatePicker />
      <Button variant="contained">Submit</Button>
    </form>
  );
};

// ✅ After: Using shadcn/ui with Tailwind
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { 
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue 
} from "@/components/ui/select";
import { DatePicker } from "@/components/ui/date-picker";

const DeviseForm = () => {
  return (
    <form className="space-y-4">
      <div>
        <label className="text-sm font-medium">Code</label>
        <Input />
      </div>
      <div>
        <label className="text-sm font-medium">Type</label>
        <Select>
          <SelectTrigger>
            <SelectValue placeholder="Select type" />
          </SelectTrigger>
          <SelectContent>
            <SelectItem value="type1">Type 1</SelectItem>
            <SelectItem value="type2">Type 2</SelectItem>
          </SelectContent>
        </Select>
      </div>
      <DatePicker />
      <Button>Submit</Button>
    </form>
  );
};
```

### 3. State Management Modernization

Transition from Redux for server data to a combination of React Context and TanStack Query:

```typescript
// Redux approach: Multiple files, complex setup(Just an example)
// store/actions/devise.js
export const FETCH_DEVISES_REQUEST = 'FETCH_DEVISES_REQUEST';
export const FETCH_DEVISES_SUCCESS = 'FETCH_DEVISES_SUCCESS';
export const FETCH_DEVISES_FAILURE = 'FETCH_DEVISES_FAILURE';

export const fetchDevisesRequest = () => ({ type: FETCH_DEVISES_REQUEST });
export const fetchDevisesSuccess = (data) => ({ type: FETCH_DEVISES_SUCCESS, payload: data });
export const fetchDevisesFailure = (error) => ({ type: FETCH_DEVISES_FAILURE, payload: error });

export const fetchDevises = () => async (dispatch) => {
  dispatch(fetchDevisesRequest());
  try {
    const response = await api.get('/finance/devises/');
    dispatch(fetchDevisesSuccess(response.data));
  } catch (error) {
    dispatch(fetchDevisesFailure(error.message));
  }
};

// store/reducers/devise.js
const initialState = {
  devises: [],
  loading: false,
  error: null,
};

export const deviseReducer = (state = initialState, action) => {
  switch (action.type) {
    case FETCH_DEVISES_REQUEST:
      return { ...state, loading: true };
    case FETCH_DEVISES_SUCCESS:
      return { ...state, loading: false, devises: action.payload };
    case FETCH_DEVISES_FAILURE:
      return { ...state, loading: false, error: action.payload };
    default:
      return state;
  }
};

// Component usage
import { useDispatch, useSelector } from 'react-redux';
import { fetchDevises } from '../store/actions/devise';

const DevisesPage = () => {
  const dispatch = useDispatch();
  const { devises, loading, error } = useSelector((state) => state.devise);

  useEffect(() => {
    dispatch(fetchDevises());
  }, [dispatch]);

  // ...
};

// ====================================================

// TanStack Query approach: Clean, declarative, with caching
// hooks/sales/useDevise.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { deviseService } from '@/services/deviseService';

export const useDevise = (searchContent?: string) => {
  const queryClient = useQueryClient();

  const devises = useQuery({
    queryKey: ['devises', searchContent],
    queryFn: () => deviseService.getDevises(searchContent),
  });

  const createDevise = useMutation({
    mutationFn: (data) => deviseService.createDevise(data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['devises'] });
    },
  });

  return {
    devises,
    createDevise,
  };
};

// Component usage
import { useDevise } from '@/hooks/useDevise';

const DevisesPage = () => {
  const { devises, createDevise } = useDevise();

  if (devises.isLoading) return <p>Loading...</p>;
  if (devises.error) return <p>Error: {devises.error.message}</p>;

  const handleCreate = (data) => {
    createDevise.mutate(data);
  };

  // ...
};

// Context for app-wide state (theme, auth, etc.)
// providers/AuthProvider.tsx
import { createContext, useContext, useState } from 'react';

const AuthContext = createContext(null);

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  
  const login = async (credentials) => {
    // Login logic
  };
  
  const logout = async () => {
    // Logout logic
  };
  
  return (
    <AuthContext.Provider value={{ user, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
};

export const useAuth = () => useContext(AuthContext);
```

#### Advantages of TanStack Query + Context

1. **Reduced Boilerplate**: No need for actions, reducers, and middleware
2. **Automatic Caching**: Built-in data caching with configurable stale times
3. **Optimistic Updates**: Easier implementation of optimistic UI updates
4. **Parallel Queries**: Simplified handling of multiple concurrent requests
5. **Refetching Strategies**: Advanced refetching on window focus, network reconnection
6. **Pagination & Infinite Scrolling**: Built-in support for common data fetching patterns
7. **Type Safety**: Better TypeScript integration
8. **DevTools**: Excellent debugging tools

### 4. Path Alias Implementation

```typescript
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@components/*": ["src/components/*"],
      "@hooks/*": ["src/hooks/*"],
      "@pages/*": ["src/pages/*"],
      "@services/*": ["src/services/*"],
      "@types/*": ["src/types/*"],
      "@utils/*": ["src/utils/*"]
    }
    // other options...
  }
}

// vite.config.js
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
      '@components': path.resolve(__dirname, './src/components'),
      '@hooks': path.resolve(__dirname, './src/hooks'),
      '@pages': path.resolve(__dirname, './src/pages'),
      '@services': path.resolve(__dirname, './src/services'),
      '@types': path.resolve(__dirname, './src/types'),
      '@utils': path.resolve(__dirname, './src/utils')
    }
  }
});

// ❌ Before: Relative imports
import { DeviseForm } from '../../../../components/sales/DeviseForm';
import { useDevise } from '../../../hooks/useDevise';
import { formatCurrency } from '../../../utils/format';

// ✅ After: Absolute imports with aliases
import { DeviseForm } from '@components/sales/DeviseForm';
import { useDevise } from '@hooks/useDevise';
import { formatCurrency } from '@utils/format';
```

### 5. Form Handling Standardization

```typescript
// ❌ Before: Manual form handling(AddProductModal for example)
import { useState } from 'react';
import { TextField, Button } from '@mui/material';

const DeviseForm = ({ onSubmit }) => {
  const [code, setCode] = useState('');
  const [nom, setNom] = useState('');
  const [codeError, setCodeError] = useState('');
  const [nomError, setNomError] = useState('');

  const handleSubmit = (e) => {
    e.preventDefault();
    
    let isValid = true;
    
    if (!code) {
      setCodeError('Code is required');
      isValid = false;
    } else {
      setCodeError('');
    }
    
    if (!nom) {
      setNomError('Name is required');
      isValid = false;
    } else {
      setNomError('');
    }
    
    if (isValid) {
      onSubmit({ code, nom });
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <TextField 
        value={code}
        onChange={(e) => setCode(e.target.value)}
        error={!!codeError}
        helperText={codeError}
        label="Code"
      />
      <TextField 
        value={nom}
        onChange={(e) => setNom(e.target.value)}
        error={!!nomError}
        helperText={nomError}
        label="Name"
      />
      <Button type="submit">Submit</Button>
    </form>
  );
};

// ✅ After: Using Formik and Yup
import { Formik, Form, Field, ErrorMessage } from 'formik';
import * as Yup from 'yup';
import { Input } from '@/components/ui/input';
import { Button } from '@/components/ui/button';

// Define validation schema
const DeviseSchema = Yup.object().shape({
  code: Yup.string()
    .required('Code is required')
    .max(3, 'Code must be 3 characters or less'),
  nom: Yup.string()
    .required('Name is required')
    .max(50, 'Name must be 50 characters or less')
});

const DeviseForm = ({ onSubmit, initialValues = { code: '', nom: '' } }) => {
  return (
    <Formik
      initialValues={initialValues}
      validationSchema={DeviseSchema}
      onSubmit={onSubmit}
    >
      {({ errors, touched }) => (
        <Form className="space-y-4">
          <div>
            <label className="block text-sm font-medium">Code</label>
            <Field 
              name="code" 
              as={Input}
              className={errors.code && touched.code ? 'border-red-500' : ''}
            />
            <ErrorMessage 
              name="code" 
              component="div" 
              className="text-sm text-red-500 mt-1" 
            />
          </div>
          
          <div>
            <label className="block text-sm font-medium">Name</label>
            <Field 
              name="nom" 
              as={Input}
              className={errors.nom && touched.nom ? 'border-red-500' : ''}
            />
            <ErrorMessage 
              name="nom" 
              component="div" 
              className="text-sm text-red-500 mt-1" 
            />
          </div>
          
          <Button type="submit">Submit</Button>
        </Form>
      )}
    </Formik>
  );
};
```

### 6. Routing Structure

```typescript
// src/config/routes.ts - Centralized route definitions
export const ROUTES = {
  HOME: '/',
  AUTH: {
    LOGIN: '/auth/login',
    REGISTER: '/auth/register',
    FORGOT_PASSWORD: '/auth/forgot-password'
  },
  DASHBOARD: '/dashboard',
  FINANCE: {
    ROOT: '/finance',
    DEVISES: '/finance/devises',
    TAUX_CHANGE: '/finance/taux-change',
    TAUX_CHANGE_DETAILS: (id: string | number) => `/finance/taux-change/${id}`
  },
  USER: {
    PROFILE: '/user/profile',
    SETTINGS: '/user/settings'
  }
};

// src/App.tsx - Main routing setup
import { BrowserRouter, Routes, Route, Navigate } from 'react-router-dom';
import { ROUTES } from '@/config/routes';

// Layouts
import { MainLayout } from '@/components/layout/MainLayout';
import { AuthLayout } from '@/components/layout/AuthLayout';

// Auth pages
import { LoginPage } from '@/pages/auth/LoginPage';
import { RegisterPage } from '@/pages/auth/RegisterPage';
import { ForgotPasswordPage } from '@/pages/auth/ForgotPasswordPage';

// Main app pages
import { DashboardPage } from '@/pages/dashboard/DashboardPage';
import { DevisesPage } from '@/pages/finance/DevisesPage';
import { TauxChangePage } from '@/pages/finance/TauxChangePage';
import { TauxChangeDetailsPage } from '@/pages/finance/TauxChangeDetailsPage';

// Guards
import { ProtectedRoute } from '@/components/auth/ProtectedRoute';

const App = () => {
  return (
    <BrowserRouter>
      <Routes>
        {/* Auth routes */}
        <Route element={<AuthLayout />}>
          <Route path={ROUTES.AUTH.LOGIN} element={<LoginPage />} />
          <Route path={ROUTES.AUTH.REGISTER} element={<RegisterPage />} />
          <Route path={ROUTES.AUTH.FORGOT_PASSWORD} element={<ForgotPasswordPage />} />
        </Route>
        
        {/* Protected routes */}
        <Route element={<ProtectedRoute><MainLayout /></ProtectedRoute>}>
          <Route path={ROUTES.HOME} element={<Navigate to={ROUTES.DASHBOARD} />} />
          <Route path={ROUTES.DASHBOARD} element={<DashboardPage />} />
          
          {/* Finance module */}
          <Route path={ROUTES.FINANCE.ROOT} element={<Navigate to={ROUTES.FINANCE.DEVISES} />} />
          <Route path={ROUTES.FINANCE.DEVISES} element={<DevisesPage />} />
          <Route path={ROUTES.FINANCE.TAUX_CHANGE} element={<TauxChangePage />} />
          <Route path={`${ROUTES.FINANCE.TAUX_CHANGE}/:id`} element={<TauxChangeDetailsPage />} />
        </Route>
        
        {/* 404 route */}
        <Route path="*" element={<Navigate to={ROUTES.HOME} />} />
      </Routes>
    </BrowserRouter>
  );
};

export default App;

// Usage in components or hooks
import { useNavigate } from 'react-router-dom';
import { ROUTES } from '@/config/routes';

const EditButton = ({ id }) => {
  const navigate = useNavigate();
  
  const handleClick = () => {
    navigate(ROUTES.FINANCE.TAUX_CHANGE_DETAILS(id));
  };
  
  return <button onClick={handleClick}>Edit</button>;
};
```

## README Template

```markdown
# Project Name

## Overview
Brief description of the project and its purpose.

## Tech Stack
- React 18
- TypeScript
- Vite
- TanStack Query
- Tailwind CSS
- shadcn/ui
- React Router
- Formik & Yup

## Getting Started

### Prerequisites
- Node.js (v20+)

### Installation
```bash
# Clone the repository
git clone <repository-url>

# Install dependencies
npm install

# Start development server
npm run dev
```

## Project Structure

```
src/
├── assets/             # Static assets like images, fonts
├── components/         # Reusable UI components
│   ├── common/         # Generic components used across modules
│   ├── [module]/       # Module-specific components
│   ├── layout/         # Layout components
│   └── ui/             # UI library components
├── config/             # App configuration files
│   ├── routes.ts       # Centralized route definitions
│   └── constants.ts    # App-wide constants
├── hooks/              # Custom React hooks
│   ├── common/         # Generic hooks
│   └── [module]/       # Module-specific hooks
├── pages/              # Page components, organized by module
│   ├── auth/           # Authentication pages
│   ├── dashboard/      # Dashboard pages
│   └── [module]/       # Module-specific pages
├── providers/          # Context providers
├── services/           # API services
├── types/              # TypeScript type definitions
└── utils/              # Utility functions and helpers
```

## Naming Conventions

- **Folders**: Lowercase, kebab-case for multi-word folders (e.g., `user-management`)
- **Components**: PascalCase (e.g., `DataTable.tsx`, `UserForm.tsx`)
- **Hooks**: camelCase with 'use' prefix (e.g., `useAuth.ts`, `useDevise.ts`)
- **Services**: camelCase with 'Service' suffix (e.g., `deviseService.ts`)
- **Routes**: All lowercase, kebab-case (e.g., `/finance/taux-change`)

## State Management

This project uses a combination of React Context API (for global UI state) and TanStack Query (for server state). This provides several benefits:

- Simpler state management with less boilerplate
- Built-in caching and data synchronization
- Better TypeScript integration
- Improved performance with automatic re-fetching

## Development Guidelines

### Component Guidelines
- Keep components small and focused on a single responsibility
- Use composition over inheritance
- Extract reusable logic into custom hooks
- Use TypeScript interfaces for prop definitions

### API Calls
- All API calls should go through the service layer
- Use TanStack Query for data fetching and mutations
- Handle loading and error states consistently

### Styling
- Use Tailwind CSS for styling
- Use shadcn/ui components when possible
- Maintain consistent spacing and color usage

## Available Scripts
```
- `npm run dev` - Start development server
- `npm run staging` - Start staging server
- `npm run prod` - Start production server
- `npm run build` - Build for production
- `npm run preview` - Preview production build locally
```

## Form Handling Approach

For all forms in the application, use Formik combined with Yup for validation:

1. Define a Yup schema for validation
2. Use Formik to handle form state and submission
3. Leverage shadcn/ui components within Formik
4. Extract common validation logic into reusable schemas

## Module Structure Example

Example of a very simple module structure (Sales module):


```
src/
├── components/
│   └── sales/
│       ├── DeviseForm.tsx
│       ├── DeviseTable.tsx
│       ├── TauxChangeForm.tsx
│       └── TauxChangeTable.tsx
├── hooks/
│   └── sales/
│       ├── useDevise.ts
│       └── useTauxChange.ts
├── pages/
│   └── sales/
│       ├── DevisesPage.tsx
│       ├── TauxChangePage.tsx
│       └── TauxChangeDetailsPage.tsx
├── services/
│   └── sales/
│       ├── deviseService.ts
│       └── tauxChangeService.ts
└── types/
    └── sales/
        ├── devise.ts
        ├── tauxChange.ts
        └── index.ts
```


This structure provides clear organization, making it easy for developers to locate files related to a specific feature. And honestly, everyone working on the Frontend can be shipping at least a module(UI+Integration) in a day given the swagger is working endpoints are tested.

At least in my opinion, no shades :)
```
