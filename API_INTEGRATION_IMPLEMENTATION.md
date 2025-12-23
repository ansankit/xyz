# API Integration Implementation Documentation

## Overview

This document provides a comprehensive guide to the API integration between the Express.js backend (`apps/api`) and Next.js frontend (`apps/web`) in the Custom Marketing CRM Suite. The integration includes authentication, data fetching, state management, and CRUD operations.

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Dependencies](#dependencies)
3. [Environment Configuration](#environment-configuration)
4. [API Client Setup](#api-client-setup)
5. [TypeScript Types](#typescript-types)
6. [Authentication System](#authentication-system)
7. [State Management](#state-management)
8. [Data Fetching Hooks](#data-fetching-hooks)
9. [Component Integration](#component-integration)
10. [Error Handling](#error-handling)
11. [Loading States](#loading-states)
12. [Protected Routes](#protected-routes)
13. [File Structure](#file-structure)
14. [Usage Examples](#usage-examples)

## Architecture Overview

The integration follows a layered architecture:

```
Frontend (Next.js)          Backend (Express.js)
├── Components              ├── Controllers
├── Hooks (TanStack Query)  ├── Services
├── Context (Auth)          ├── Routes
├── API Client (Axios)      ├── Middleware
└── Types (TypeScript)      └── Database (Prisma)
```

### Key Features

- **JWT Authentication**: Token-based authentication with cookie storage
- **Type Safety**: Full TypeScript integration between frontend and backend
- **State Management**: React Context for auth + TanStack Query for data
- **Error Handling**: Centralized error management with user-friendly messages
- **Loading States**: Comprehensive loading indicators throughout the app
- **Protected Routes**: Route protection based on authentication status

## Dependencies

### Frontend Dependencies Added

```json
{
  "dependencies": {
    "axios": "^1.7.9",
    "js-cookie": "^3.0.5",
    "@tanstack/react-query": "^5.62.8"
  },
  "devDependencies": {
    "@types/js-cookie": "^3.0.6"
  }
}
```

### Installation Command

```bash
npm install axios @tanstack/react-query js-cookie
npm install -D @types/js-cookie
```

## Environment Configuration

### API Configuration

**File:** `apps/web/lib/config.ts`

```typescript
export const API_BASE_URL = process.env.NEXT_PUBLIC_API_BASE_URL || 'http://localhost:4000/api';
```

### Environment Variables

**File:** `apps/web/.env.local`

```env
NEXT_PUBLIC_API_BASE_URL=http://localhost:4000/api
NEXT_PUBLIC_COMPANY_NAME=Innovun Global
```

## API Client Setup

### Axios Configuration

**File:** `apps/web/lib/api/client.ts`

```typescript
import axios, { AxiosInstance, AxiosRequestConfig, AxiosResponse } from 'axios';
import Cookies from 'js-cookie';
import { config } from '../config';
import { ApiError } from './types';

// Create axios instance
const apiClient: AxiosInstance = axios.create({
  baseURL: config.apiUrl,
  timeout: 10000,
  headers: {
    'Content-Type': 'application/json',
  },
});

// Request interceptor to add JWT token
apiClient.interceptors.request.use(
  (config: AxiosRequestConfig) => {
    const token = Cookies.get('auth_token');
    if (token && config.headers) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => {
    return Promise.reject(error);
  }
);

// Response interceptor to handle errors
apiClient.interceptors.response.use(
  (response: AxiosResponse) => {
    return response;
  },
  (error) => {
    if (error.response?.status === 401) {
      // Clear token and redirect to login
      Cookies.remove('auth_token');
      if (typeof window !== 'undefined') {
        window.location.href = '/login';
      }
    }
    
    // Transform error to our ApiError format
    const apiError: ApiError = {
      message: error.response?.data?.error || error.message || 'An error occurred',
      status: error.response?.status || 500,
      code: error.response?.data?.code,
    };
    
    return Promise.reject(apiError);
  }
);

export default apiClient;
```

## TypeScript Types

### API Entity Types

**File:** `apps/web/lib/api/types.ts`

```typescript
export interface User {
  id: number;
  name: string;
  email: string;
  passwordHash: string;
  createdAt: string;
  updatedAt: string;
  permissions?: UserPermissions;
  leads?: Lead[];
  campaigns?: Campaign[];
}

export interface UserPermissions {
  id: number;
  userId: number;
  leadManagement: boolean;
  campaignManagement: boolean;
  chatbotAccess: boolean;
  whatsappCampaign: boolean;
  emailMarketing: boolean;
  systemAdminAccess: boolean;
}

export interface Lead {
  id: number;
  name: string;
  email: string;
  phone?: string;
  source: string;
  status: string;
  score: number;
  ownerId?: number;
  createdAt: string;
  updatedAt: string;
  owner?: User;
  convertedToContact?: Contact;
  campaignMembers?: any[];
  analyticsEvents?: AnalyticsEvent[];
  formSubmissions?: FormSubmission[];
}

export interface Contact {
  id: number;
  name: string;
  email: string;
  phone?: string;
  position?: string;
  accountId?: number;
  createdAt: string;
  updatedAt: string;
  account?: Account;
  convertedLeads?: Lead[];
  campaignMembers?: any[];
}

export interface Campaign {
  id: number;
  name: string;
  description?: string;
  startDate: string;
  endDate?: string;
  createdBy: number;
  createdAt: string;
  updatedAt: string;
  creator?: User;
  campaignMembers?: any[];
  analyticsEvents?: AnalyticsEvent[];
  formSubmissions?: FormSubmission[];
}

export interface Account {
  id: number;
  name: string;
  industry?: string;
  website?: string;
  createdAt: string;
  updatedAt: string;
  contacts?: Contact[];
}

export interface AnalyticsEvent {
  id: number;
  eventType: string;
  eventData: any;
  occurredAt: string;
  campaignId?: number;
  contactId?: number;
  leadId?: number;
  campaign?: Campaign;
  contact?: Contact;
  lead?: Lead;
}

export interface FormSubmission {
  id: number;
  formData: any;
  submittedAt: string;
  campaignId?: number;
  leadId?: number;
  contactId?: number;
  campaign?: Campaign;
  lead?: Lead;
  contact?: Contact;
}

// Authentication Types
export interface LoginPayload {
  email: string;
  password: string;
}

export interface SignupPayload {
  name: string;
  email: string;
  password: string;
}

export interface AuthResponse {
  token: string;
  user: User;
}

export interface UpdateUserPermissionsPayload {
  id: number;
  permissions: Partial<Omit<UserPermissions, 'id' | 'userId'>>;
}
```

## Authentication System

### AuthContext Implementation

**File:** `apps/web/contexts/AuthContext.tsx`

```typescript
"use client";

import React, { createContext, useContext, useState, useEffect, useCallback } from 'react';
import Cookies from 'js-cookie';
import { authService } from '../lib/api/services';
import { User, LoginPayload, SignupPayload } from '../lib/api/types';
import { AxiosError } from 'axios';

interface AuthContextType {
  user: User | null;
  token: string | null;
  login: (payload: LoginPayload) => Promise<void>;
  signup: (payload: SignupPayload) => Promise<void>;
  logout: () => void;
  isLoading: boolean;
  error: string | null;
  clearError: () => void;
}

const AuthContext = createContext<AuthContextType | undefined>(undefined);

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [token, setToken] = useState<string | null>(null);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  const clearError = useCallback(() => {
    setError(null);
  }, []);

  const loadUserFromCookies = useCallback(async () => {
    setIsLoading(true);
    const storedToken = Cookies.get('token');
    if (storedToken) {
      setToken(storedToken);
      try {
        const userData = await authService.getMe();
        setUser(userData);
      } catch (err) {
        console.error('Failed to fetch user data from token:', err);
        Cookies.remove('token');
        setToken(null);
        setUser(null);
      }
    }
    setIsLoading(false);
  }, []);

  useEffect(() => {
    loadUserFromCookies();
  }, [loadUserFromCookies]);

  const login = async (payload: LoginPayload) => {
    setError(null);
    try {
      const response = await authService.login(payload);
      Cookies.set('token', response.token, { expires: 7 });
      setToken(response.token);
      setUser(response.user);
    } catch (err) {
      const axiosError = err as AxiosError;
      setError(axiosError.response?.data?.error || 'Login failed');
      throw err;
    }
  };

  const signup = async (payload: SignupPayload) => {
    setError(null);
    try {
      const response = await authService.signup(payload);
      Cookies.set('token', response.token, { expires: 7 });
      setToken(response.token);
      setUser(response.user);
    } catch (err) {
      const axiosError = err as AxiosError;
      setError(axiosError.response?.data?.error || 'Signup failed');
      throw err;
    }
  };

  const logout = () => {
    Cookies.remove('token');
    setToken(null);
    setUser(null);
    setError(null);
  };

  return (
    <AuthContext.Provider value={{ user, token, login, signup, logout, isLoading, error, clearError }}>
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth() {
  const context = useContext(AuthContext);
  if (context === undefined) {
    throw new Error('useAuth must be used within an AuthProvider');
  }
  return context;
}
```

## State Management

### TanStack Query Setup

**File:** `apps/web/lib/queryClient.ts`

```typescript
import { QueryClient } from '@tanstack/react-query';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5, // 5 minutes
      refetchOnWindowFocus: false,
      retry: 2,
    },
  },
});
```

**File:** `apps/web/providers/QueryProvider.tsx`

```typescript
"use client";

import React from 'react';
import { QueryClientProvider } from '@tanstack/react-query';
import { queryClient } from '../lib/queryClient';

export function QueryProvider({ children }: { children: React.ReactNode }) {
  return (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  );
}
```

## Data Fetching Hooks

### Lead Management Hooks

**File:** `apps/web/hooks/useLeads.ts`

```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { leadService } from '../lib/api/services';
import { Lead, LeadFilters } from '../lib/api/types';

// Query keys
export const leadKeys = {
  all: ['leads'] as const,
  lists: () => [...leadKeys.all, 'list'] as const,
  list: (filters: LeadFilters) => [...leadKeys.lists(), filters] as const,
  details: () => [...leadKeys.all, 'detail'] as const,
  detail: (id: number) => [...leadKeys.details(), id] as const,
  complete: () => [...leadKeys.all, 'complete'] as const,
};

// Basic CRUD hooks
export function useLeads(filters?: LeadFilters) {
  return useQuery({
    queryKey: leadKeys.list(filters || {}),
    queryFn: () => leadService.getAllLeads(filters),
  });
}

export function useLeadsComplete() {
  return useQuery({
    queryKey: leadKeys.complete(),
    queryFn: leadService.getAllLeadsComplete,
  });
}

export function useLead(id: number) {
  return useQuery({
    queryKey: leadKeys.detail(id),
    queryFn: () => leadService.getLeadById(id),
    enabled: !!id,
  });
}

// Filter and search hooks
export function useFilterLeads(filters: LeadFilters) {
  return useQuery({
    queryKey: [...leadKeys.lists(), 'filter', filters],
    queryFn: () => leadService.filterLeads(filters),
  });
}

export function useSearchLeads(query: string) {
  return useQuery({
    queryKey: [...leadKeys.lists(), 'search', query],
    queryFn: () => leadService.searchLeads(query),
    enabled: !!query && query.length > 2,
  });
}

export function useLeadsByStatus(status: string) {
  return useQuery({
    queryKey: [...leadKeys.lists(), 'status', status],
    queryFn: () => leadService.getLeadsByStatus(status),
    enabled: !!status,
  });
}

// Assignment and conversion hooks
export function useAssignLead() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: ({ id, userId }: { id: number; userId: number }) => 
      leadService.assignLead(id, userId),
    onSuccess: (updatedLead) => {
      queryClient.setQueryData(leadKeys.detail(updatedLead.id), updatedLead);
      queryClient.invalidateQueries({ queryKey: leadKeys.lists() });
    },
  });
}

export function useConvertLead() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: leadService.convertLead,
    onSuccess: (_, leadId) => {
      queryClient.removeQueries({ queryKey: leadKeys.detail(leadId) });
      queryClient.invalidateQueries({ queryKey: leadKeys.lists() });
      queryClient.invalidateQueries({ queryKey: ['contacts'] });
    },
  });
}

// CRUD mutations
export function useCreateLead() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: leadService.createLead,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: leadKeys.all });
    },
  });
}

export function useUpdateLead() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: ({ id, data }: { id: number; data: Partial<Lead> }) =>
      leadService.updateLead(id, data),
    onSuccess: (updatedLead) => {
      queryClient.setQueryData(leadKeys.detail(updatedLead.id), updatedLead);
      queryClient.invalidateQueries({ queryKey: leadKeys.lists() });
    },
  });
}

export function useDeleteLead() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: leadService.deleteLead,
    onSuccess: (_, deletedId) => {
      queryClient.removeQueries({ queryKey: leadKeys.detail(deletedId) });
      queryClient.invalidateQueries({ queryKey: leadKeys.lists() });
    },
  });
}
```

### Analytics Hooks

**File:** `apps/web/hooks/useAnalytics.ts`

```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { analyticsService } from '../lib/api/services';
import { AnalyticsEvent } from '../lib/api/types';

// Query keys
export const analyticsKeys = {
  all: ['analytics', 'events'] as const,
  lists: () => [...analyticsKeys.all, 'list'] as const,
  list: (params?: any) => [...analyticsKeys.lists(), params || {}] as const,
  details: () => [...analyticsKeys.all, 'detail'] as const,
  detail: (id: number) => [...analyticsKeys.details(), id] as const,
  byCampaign: (id: number) => [...analyticsKeys.all, 'campaign', id] as const,
  byContact: (id: number) => [...analyticsKeys.all, 'contact', id] as const,
  byLead: (id: number) => [...analyticsKeys.all, 'lead', id] as const,
};

// Hooks for fetching analytics events
export function useAnalytics(params?: {
  campaignId?: number;
  contactId?: number;
  leadId?: number;
  eventType?: string;
}) {
  return useQuery({
    queryKey: analyticsKeys.list(params),
    queryFn: () => analyticsService.getAllEvents(params),
  });
}

export function useAnalyticsByLead(leadId: number) {
  return useQuery({
    queryKey: analyticsKeys.byLead(leadId),
    queryFn: () => analyticsService.getEventsByLead(leadId),
    enabled: !!leadId,
  });
}

export function useCreateAnalyticsEvent() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: analyticsService.createEvent,
    onSuccess: (newEvent) => {
      queryClient.invalidateQueries({ queryKey: analyticsKeys.all });
      if (newEvent.campaignId) {
        queryClient.invalidateQueries({ queryKey: analyticsKeys.byCampaign(newEvent.campaignId) });
      }
      if (newEvent.contactId) {
        queryClient.invalidateQueries({ queryKey: analyticsKeys.byContact(newEvent.contactId) });
      }
      if (newEvent.leadId) {
        queryClient.invalidateQueries({ queryKey: analyticsKeys.byLead(newEvent.leadId) });
      }
    },
  });
}
```

### Webhook and Health Hooks

**File:** `apps/web/hooks/useWebhook.ts`

```typescript
import { useQuery, useMutation } from '@tanstack/react-query';
import { webhookService, healthService } from '../lib/api/services';
import { WebhookPayload, WebhookResponse } from '../lib/api/types';

// Query keys
export const webhookKeys = {
  health: ['health'] as const,
  webhookTest: ['webhook', 'test'] as const,
};

// Health check hook
export function useHealth() {
  return useQuery({
    queryKey: webhookKeys.health,
    queryFn: healthService.checkHealth,
    refetchInterval: 30000, // Refetch every 30 seconds
    staleTime: 10000, // Consider data stale after 10 seconds
  });
}

// Webhook test hook
export function useWebhookTest() {
  return useMutation({
    mutationFn: webhookService.testLandingiWebhook,
  });
}

// Webhook payload submission hook
export function useWebhookSubmission() {
  return useMutation({
    mutationFn: (payload: WebhookPayload) => webhookService.handleLandingiWebhook(payload),
  });
}
```

### API Service Functions

**File:** `apps/web/lib/api/services.ts`

```typescript
import apiClient from './client';
import {
  LoginPayload,
  SignupPayload,
  AuthResponse,
  User,
  Lead,
  Contact,
  Campaign,
  AnalyticsEvent,
  UpdateUserPermissionsPayload,
} from './types';

// Auth Services
export const authService = {
  login: async (payload: LoginPayload): Promise<AuthResponse> => {
    const response = await apiClient.post('/auth/login', payload);
    return response.data;
  },
  signup: async (payload: SignupPayload): Promise<AuthResponse> => {
    const response = await apiClient.post('/auth/signup', payload);
    return response.data;
  },
  getMe: async (): Promise<User> => {
    const response = await apiClient.get('/auth/me');
    return response.data;
  },
};

// User Services
export const userService = {
  getAllUsers: async (): Promise<User[]> => {
    const response = await apiClient.get('/users');
    return response.data;
  },
  getUserById: async (id: number): Promise<User> => {
    const response = await apiClient.get(`/users/${id}`);
    return response.data;
  },
  updateUser: async (id: number, data: Partial<User>): Promise<User> => {
    const response = await apiClient.put(`/users/${id}`, data);
    return response.data;
  },
  deleteUser: async (id: number): Promise<void> => {
    await apiClient.delete(`/users/${id}`);
  },
  updateUserPermissions: async (payload: UpdateUserPermissionsPayload): Promise<User> => {
    const response = await apiClient.put(`/users/${payload.id}/permissions`, payload.permissions);
    return response.data;
  },
};

// Lead Services
export const leadService = {
  getAllLeads: async (): Promise<Lead[]> => {
    const response = await apiClient.get('/leads');
    return response.data;
  },
  getAllLeadsComplete: async (): Promise<Lead[]> => {
    const response = await apiClient.get('/leads/all');
    return response.data;
  },
  getLeadById: async (id: number): Promise<Lead> => {
    const response = await apiClient.get(`/leads/${id}`);
    return response.data;
  },
  createLead: async (data: Omit<Lead, 'id' | 'createdAt' | 'updatedAt'>): Promise<Lead> => {
    const response = await apiClient.post('/leads', data);
    return response.data;
  },
  updateLead: async (id: number, data: Partial<Lead>): Promise<Lead> => {
    const response = await apiClient.put(`/leads/${id}`, data);
    return response.data;
  },
  deleteLead: async (id: number): Promise<void> => {
    await apiClient.delete(`/leads/${id}`);
  },
};

// Contact Services
export const contactService = {
  getAllContacts: async (): Promise<Contact[]> => {
    const response = await apiClient.get('/contacts');
    return response.data;
  },
  getContactById: async (id: number): Promise<Contact> => {
    const response = await apiClient.get(`/contacts/${id}`);
    return response.data;
  },
  createContact: async (data: Omit<Contact, 'id' | 'createdAt' | 'updatedAt'>): Promise<Contact> => {
    const response = await apiClient.post('/contacts', data);
    return response.data;
  },
  updateContact: async (id: number, data: Partial<Contact>): Promise<Contact> => {
    const response = await apiClient.put(`/contacts/${id}`, data);
    return response.data;
  },
  deleteContact: async (id: number): Promise<void> => {
    await apiClient.delete(`/contacts/${id}`);
  },
};

// Campaign Services
export const campaignService = {
  getAllCampaigns: async (): Promise<Campaign[]> => {
    const response = await apiClient.get('/campaigns');
    return response.data;
  },
  getCampaignById: async (id: number): Promise<Campaign> => {
    const response = await apiClient.get(`/campaigns/${id}`);
    return response.data;
  },
  createCampaign: async (data: Omit<Campaign, 'id' | 'createdAt' | 'updatedAt'>): Promise<Campaign> => {
    const response = await apiClient.post('/campaigns', data);
    return response.data;
  },
  updateCampaign: async (id: number, data: Partial<Campaign>): Promise<Campaign> => {
    const response = await apiClient.put(`/campaigns/${id}`, data);
    return response.data;
  },
  deleteCampaign: async (id: number): Promise<void> => {
    await apiClient.delete(`/campaigns/${id}`);
  },
};

// Analytics Services
export const analyticsService = {
  getAllEvents: async (params?: any): Promise<AnalyticsEvent[]> => {
    const response = await apiClient.get('/analytics/events', { params });
    return response.data;
  },
  getEventById: async (id: number): Promise<AnalyticsEvent> => {
    const response = await apiClient.get(`/analytics/events/${id}`);
    return response.data;
  },
  createEvent: async (data: Omit<AnalyticsEvent, 'id' | 'occurredAt'>): Promise<AnalyticsEvent> => {
    const response = await apiClient.post('/analytics/events', data);
    return response.data;
  },
};
```

## Component Integration

### Root Layout Integration

**File:** `apps/web/app/layout.tsx`

```typescript
import type { Metadata } from "next";
import localFont from "next/font/local";
import "@repo/ui/globals.css";
import { HeaderWrapper } from "../components/header-wrapper";
import { AppSidebar } from "../components/appSidebar";
import { AuthProvider } from "../contexts/AuthContext";
import { QueryProvider } from "../providers/QueryProvider";
import logo from './assets/images/logos/logo_v1.png';
import Image from "next/image";

const geistSans = localFont({
  src: "./fonts/GeistVF.woff",
  variable: "--font-geist-sans",
});
const geistMono = localFont({
  src: "./fonts/GeistMonoVF.woff",
  variable: "--font-geist-mono",
});

export const metadata: Metadata = {
  title: "Create Next App",
  description: "Generated by create next app",
};

export default function RootLayout({
  children,
}: Readonly<{
  children: React.ReactNode;
}>) {
  return (
    <html lang="en">
      <body className={`${geistSans.variable} ${geistMono.variable}`}>
        <QueryProvider>
          <AuthProvider>
            <AppLayout>
              {children}
            </AppLayout>
          </AuthProvider>
        </QueryProvider>
      </body>
    </html>
  );
}

function AppLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="h-screen flex flex-col">
      <HeaderWrapper
        notificationCount={3}
        icon={<Image src={logo} alt="logo" width={200} height={200} />}
      />
      <div className="flex flex-1 overflow-hidden">
        <div className="flex-shrink-0">
          <AppSidebar />
        </div>
        <main className="flex-1 p-6 overflow-y-auto">
          {children}
        </main>
      </div>
    </div>
  );
}
```

### Protected Route Component

**File:** `apps/web/components/ProtectedRoute.tsx`

```typescript
"use client";

import { useEffect } from 'react';
import { useRouter } from 'next/navigation';
import { useAuth } from '../contexts/AuthContext';

interface ProtectedRouteProps {
  children: React.ReactNode;
  fallback?: React.ReactNode;
}

export function ProtectedRoute({ children, fallback }: ProtectedRouteProps) {
  const { user, isLoading } = useAuth();
  const router = useRouter();

  useEffect(() => {
    if (!isLoading && !user) {
      router.push('/login');
    }
  }, [user, isLoading, router]);

  if (isLoading) {
    return (
      fallback || (
        <div className="flex items-center justify-center min-h-screen">
          <div className="animate-spin rounded-full h-32 w-32 border-b-2 border-primary"></div>
        </div>
      )
    );
  }

  if (!user) {
    return null;
  }

  return <>{children}</>;
}
```

## Error Handling

### Centralized Error Management

The application implements comprehensive error handling:

1. **API Level**: Axios interceptors handle HTTP errors
2. **Service Level**: Service functions catch and format errors
3. **Hook Level**: TanStack Query provides error states
4. **Component Level**: Components display user-friendly error messages

### Error Display Examples

```typescript
// In components
const { data: leads = [], isLoading, error } = useLeads();

if (error) {
  return (
    <div className="text-center">
      <h2 className="text-2xl font-bold text-red-600 mb-2">Error Loading Data</h2>
      <p className="text-muted-foreground">
        {error.message || 'An error occurred while loading data'}
      </p>
    </div>
  );
}
```

## Loading States

### Loading State Implementation

The application provides comprehensive loading states:

1. **Authentication Loading**: While checking user status
2. **Data Fetching Loading**: During API calls
3. **Mutation Loading**: During create/update/delete operations
4. **Form Submission Loading**: During login/signup

### Loading State Examples

```typescript
// Data fetching loading
if (isLoading) {
  return (
    <div className="flex items-center justify-center min-h-[400px]">
      <div className="animate-spin rounded-full h-32 w-32 border-b-2 border-primary"></div>
    </div>
  );
}

// Form submission loading
<Button type="submit" disabled={isSubmitting}>
  {isSubmitting ? 'Logging in...' : 'Login'}
</Button>

// Mutation loading
const createLeadMutation = useCreateLead();
// createLeadMutation.isPending - for loading state
```

## File Structure

```
apps/web/
├── lib/
│   ├── api/
│   │   ├── client.ts          # Axios configuration
│   │   ├── services.ts        # API service functions
│   │   └── types.ts          # TypeScript interfaces
│   ├── config.ts             # Environment configuration
│   └── queryClient.ts        # TanStack Query setup
├── contexts/
│   └── AuthContext.tsx       # Authentication context
├── providers/
│   └── QueryProvider.tsx     # Query client provider
├── hooks/
│   ├── useLeads.ts          # Lead management hooks
│   ├── useContacts.ts       # Contact management hooks
│   ├── useUsers.ts          # User management hooks
│   └── useCampaigns.ts      # Campaign management hooks
├── components/
│   ├── ProtectedRoute.tsx   # Route protection
│   ├── login-form.tsx       # Login form with API integration
│   ├── signup-form.tsx      # Signup form with API integration
│   └── lead-management-dashboard.tsx # Dashboard with real data
└── app/
    ├── layout.tsx           # Root layout with providers
    ├── login/page.tsx       # Login page
    ├── signup/page.tsx      # Signup page
    └── lead-management/page.tsx # Protected dashboard
```

## Admin Panel Integration

### Admin Page

**File:** `apps/web/app/admin/page.tsx`

The admin panel provides system health monitoring, webhook testing, and user permission management:

```typescript
"use client";

import React, { useState } from 'react';
import { Button } from '@repo/ui/components/ui/button';
import { Card, CardContent, CardHeader, CardTitle } from '@repo/ui/components/ui/card';
import { Tabs, TabsContent, TabsList, TabsTrigger } from '@repo/ui/components/ui/tabs';
import { useUsers, useUpdateUserPermissions } from '../../hooks/useUsers';
import { useWebhookTest, useHealth } from '../../hooks/useWebhook';
import { useAuth } from '../../contexts/AuthContext';
import { ProtectedRoute } from '../../components/ProtectedRoute';

export default function AdminPage() {
  const { user } = useAuth();
  const [webhookPayload, setWebhookPayload] = useState('{"test": "data"}');
  
  // Hooks
  const { data: users = [], isLoading: usersLoading } = useUsers();
  const updatePermissionsMutation = useUpdateUserPermissions();
  const webhookTestMutation = useWebhookTest();
  const { data: health, isLoading: healthLoading } = useHealth();

  // Check if user has admin access
  const isAdmin = user?.permissions?.systemAdminAccess;

  if (!isAdmin) {
    return (
      <div className="p-6">
        <Card>
          <CardContent className="p-6">
            <h2 className="text-2xl font-bold text-red-600 mb-2">Access Denied</h2>
            <p className="text-muted-foreground">
              You don't have permission to access the admin panel.
            </p>
          </CardContent>
        </Card>
      </div>
    );
  }

  const handleTestWebhook = async () => {
    try {
      await webhookTestMutation.mutateAsync();
    } catch (error) {
      console.error('Webhook test failed:', error);
    }
  };

  const handleUpdatePermissions = async (userId: number, permissions: any) => {
    try {
      await updatePermissionsMutation.mutateAsync({ id: userId, permissions });
    } catch (error) {
      console.error('Failed to update permissions:', error);
    }
  };

  return (
    <ProtectedRoute>
      <div className="p-6">
        <h1 className="text-3xl font-bold mb-6">Admin Panel</h1>
        
        <Tabs defaultValue="health" className="space-y-6">
          <TabsList>
            <TabsTrigger value="health">System Health</TabsTrigger>
            <TabsTrigger value="webhooks">Webhook Testing</TabsTrigger>
            <TabsTrigger value="users">User Management</TabsTrigger>
          </TabsList>

          <TabsContent value="health">
            <Card>
              <CardHeader>
                <CardTitle>System Health</CardTitle>
              </CardHeader>
              <CardContent>
                {healthLoading ? (
                  <p>Loading health status...</p>
                ) : health ? (
                  <div className="space-y-2">
                    <div className="flex items-center gap-2">
                      <Badge variant={health.status === 'ok' ? 'default' : 'destructive'}>
                        {health.status}
                      </Badge>
                      <span>API Status</span>
                    </div>
                    <div className="flex items-center gap-2">
                      <Badge variant={health.database === 'connected' ? 'default' : 'destructive'}>
                        {health.database}
                      </Badge>
                      <span>Database Status</span>
                    </div>
                  </div>
                ) : (
                  <p className="text-red-600">Failed to load health status</p>
                )}
              </CardContent>
            </Card>
          </TabsContent>

          <TabsContent value="webhooks">
            <Card>
              <CardHeader>
                <CardTitle>Webhook Testing</CardTitle>
              </CardHeader>
              <CardContent className="space-y-4">
                <Button 
                  onClick={handleTestWebhook}
                  disabled={webhookTestMutation.isPending}
                >
                  {webhookTestMutation.isPending ? 'Testing...' : 'Test Landingi Webhook'}
                </Button>
                {webhookTestMutation.data && (
                  <div className="mt-4 p-4 bg-green-50 border border-green-200 rounded">
                    <h4 className="font-semibold text-green-800">Webhook Test Result:</h4>
                    <pre className="text-sm text-green-700 mt-2">
                      {JSON.stringify(webhookTestMutation.data, null, 2)}
                    </pre>
                  </div>
                )}
              </CardContent>
            </Card>
          </TabsContent>

          <TabsContent value="users">
            <Card>
              <CardHeader>
                <CardTitle>User Management</CardTitle>
              </CardHeader>
              <CardContent>
                {usersLoading ? (
                  <p>Loading users...</p>
                ) : (
                  <div className="space-y-4">
                    {users.map((user) => (
                      <div key={user.id} className="border rounded p-4">
                        <div className="flex justify-between items-start">
                          <div>
                            <h3 className="font-semibold">{user.name}</h3>
                            <p className="text-sm text-muted-foreground">{user.email}</p>
                          </div>
                          <div className="flex gap-2">
                            <Button
                              size="sm"
                              variant="outline"
                              onClick={() => handleUpdatePermissions(user.id, {
                                leadManagement: !user.permissions?.leadManagement
                              })}
                            >
                              Toggle Lead Management
                            </Button>
                            <Button
                              size="sm"
                              variant="outline"
                              onClick={() => handleUpdatePermissions(user.id, {
                                systemAdminAccess: !user.permissions?.systemAdminAccess
                              })}
                            >
                              Toggle Admin Access
                            </Button>
                          </div>
                        </div>
                      </div>
                    ))}
                  </div>
                )}
              </CardContent>
            </Card>
          </TabsContent>
        </Tabs>
      </div>
    </ProtectedRoute>
  );
}
```

## Usage Examples

### Using Authentication

```typescript
import { useAuth } from '../contexts/AuthContext';

function MyComponent() {
  const { user, login, logout, isLoading } = useAuth();

  const handleLogin = async () => {
    try {
      await login({ email: 'user@example.com', password: 'password' });
      // User is now logged in
    } catch (error) {
      // Handle login error
    }
  };

  if (isLoading) return <div>Loading...</div>;
  if (!user) return <div>Please log in</div>;

  return (
    <div>
      <p>Welcome, {user.name}!</p>
      <button onClick={logout}>Logout</button>
    </div>
  );
}
```

### Using Advanced Lead Management

```typescript
import { useLeads, useSearchLeads, useAssignLead, useConvertLead } from '../hooks/useLeads';

function LeadsComponent() {
  const [searchQuery, setSearchQuery] = useState('');
  const [selectedLeads, setSelectedLeads] = useState<number[]>([]);
  
  const { data: leads = [], isLoading, error } = useLeads();
  const { data: searchResults = [] } = useSearchLeads(searchQuery);
  const assignLeadMutation = useAssignLead();
  const convertLeadMutation = useConvertLead();

  const handleAssignLead = async (leadId: number, userId: number) => {
    try {
      await assignLeadMutation.mutateAsync({ id: leadId, userId });
    } catch (error) {
      console.error('Failed to assign lead:', error);
    }
  };

  const handleConvertLead = async (leadId: number) => {
    try {
      await convertLeadMutation.mutateAsync(leadId);
    } catch (error) {
      console.error('Failed to convert lead:', error);
    }
  };

  const displayLeads = searchQuery ? searchResults : leads;

  if (isLoading) return <div>Loading leads...</div>;
  if (error) return <div>Error: {error.message}</div>;

  return (
    <div>
      <input
        type="text"
        placeholder="Search leads..."
        value={searchQuery}
        onChange={(e) => setSearchQuery(e.target.value)}
      />
      {displayLeads.map(lead => (
        <div key={lead.id} className="border p-4 rounded">
          <h3>{lead.name}</h3>
          <p>{lead.email}</p>
          <p>Status: {lead.status}</p>
          <p>Score: {lead.score}</p>
          <div className="flex gap-2 mt-2">
            <button 
              onClick={() => handleAssignLead(lead.id, 1)}
              disabled={assignLeadMutation.isPending}
            >
              Assign
            </button>
            <button 
              onClick={() => handleConvertLead(lead.id)}
              disabled={convertLeadMutation.isPending}
            >
              Convert
            </button>
          </div>
        </div>
      ))}
    </div>
  );
}
```

### Using Analytics

```typescript
import { useAnalyticsByLead, useCreateAnalyticsEvent } from '../hooks/useAnalytics';

function LeadAnalytics({ leadId }: { leadId: number }) {
  const { data: events = [], isLoading } = useAnalyticsByLead(leadId);
  const createEventMutation = useCreateAnalyticsEvent();

  const handleCreateEvent = async (eventType: string, eventData: any) => {
    try {
      await createEventMutation.mutateAsync({
        leadId,
        eventType,
        eventData,
        campaignId: undefined,
        contactId: undefined
      });
    } catch (error) {
      console.error('Failed to create event:', error);
    }
  };

  if (isLoading) return <div>Loading analytics...</div>;

  return (
    <div>
      <h3>Analytics Events</h3>
      {events.map(event => (
        <div key={event.id} className="border p-2 rounded mb-2">
          <p><strong>Type:</strong> {event.eventType}</p>
          <p><strong>Data:</strong> {JSON.stringify(event.eventData)}</p>
          <p><strong>Time:</strong> {new Date(event.occurredAt).toLocaleString()}</p>
        </div>
      ))}
      <button 
        onClick={() => handleCreateEvent('manual_note', { note: 'User added a note' })}
        disabled={createEventMutation.isPending}
      >
        Add Note
      </button>
    </div>
  );
}
```

### Using Health Monitoring

```typescript
import { useHealth } from '../hooks/useWebhook';

function HealthStatus() {
  const { data: health, isLoading, error } = useHealth();

  if (isLoading) return <div>Checking system health...</div>;
  if (error) return <div className="text-red-600">Health check failed</div>;

  return (
    <div className="flex items-center gap-2">
      <div className={`w-3 h-3 rounded-full ${
        health?.status === 'ok' ? 'bg-green-500' : 'bg-red-500'
      }`} />
      <span>
        {health?.status === 'ok' ? 'System Online' : 'System Issues'}
      </span>
    </div>
  );
}
```

### Using Protected Routes

```typescript
import { ProtectedRoute } from '../components/ProtectedRoute';

function DashboardPage() {
  return (
    <ProtectedRoute>
      <div>This content is only visible to authenticated users</div>
    </ProtectedRoute>
  );
}
```

## Key Benefits

1. **Type Safety**: Full TypeScript integration ensures compile-time error checking
2. **Performance**: TanStack Query provides caching, background updates, and optimistic updates
3. **User Experience**: Comprehensive loading states and error handling
4. **Security**: JWT token management with automatic attachment and refresh
5. **Maintainability**: Clean separation of concerns with hooks and services
6. **Scalability**: Easy to add new API endpoints and data fetching patterns

## Future Enhancements

1. **Token Refresh**: Implement automatic token refresh before expiration
2. **Offline Support**: Add offline capabilities with TanStack Query
3. **Real-time Updates**: Integrate WebSocket connections for live data
4. **Advanced Caching**: Implement more sophisticated caching strategies
5. **Error Recovery**: Add retry mechanisms for failed requests
6. **Performance Monitoring**: Add API performance tracking and analytics

This implementation provides a robust foundation for the CRM application with excellent developer experience and user experience.


