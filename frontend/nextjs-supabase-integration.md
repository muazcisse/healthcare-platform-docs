# Next.js with Supabase Integration for Healthcare Platform

## Overview

This document provides comprehensive guidance on implementing Next.js with Supabase integration for the Healthcare Platform project targeting Sub-Saharan Africa and Mali. The integration focuses on offline-first capabilities, multi-language support, and FHIR compliance.

## Table of Contents

1. [Setup and Installation](#setup-and-installation)
2. [Project Structure](#project-structure)
3. [Authentication](#authentication)
4. [Offline Capabilities](#offline-capabilities)
5. [Database Operations](#database-operations)
6. [Real-time Updates](#real-time-updates)
7. [File Storage](#file-storage)
8. [Internationalization](#internationalization)
9. [Performance Optimization](#performance-optimization)
10. [Deployment](#deployment)

## Setup and Installation

### Prerequisites

- Node.js 18.x or later
- npm or yarn
- Supabase account

### Creating a New Project

```bash
# Create a new Next.js project
npx create-next-app@latest healthcare-platform --typescript

# Navigate to project directory
cd healthcare-platform

# Install Supabase client libraries
npm install @supabase/supabase-js @supabase/auth-helpers-nextjs
```

### Environment Configuration

Create a `.env.local` file in the root of your project:

```
NEXT_PUBLIC_SUPABASE_URL=your_supabase_project_url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_supabase_anon_key
```

## Project Structure

Recommended project structure for the healthcare platform:

```
healthcare-platform/
├── components/           # Reusable UI components
│   ├── common/           # General components
│   ├── forms/            # Form components
│   ├── layouts/          # Layout components
│   └── patients/         # Patient-specific components
├── lib/                  # Utility functions and libraries
│   ├── supabase.ts       # Supabase client initialization
│   ├── fhir.ts           # FHIR helper functions
│   └── offline.ts        # Offline capability utilities
├── models/               # TypeScript interfaces/types
├── pages/                # Next.js pages
│   ├── api/              # API routes
│   ├── auth/             # Authentication pages
│   ├── patients/         # Patient management
│   ├── prescriptions/    # Prescription management
│   └── _app.tsx          # Custom App component
├── public/               # Static files
│   ├── locales/          # Translation files
│   └── worker.js         # Service worker for offline
├── styles/               # Global styles
└── utils/                # Helper functions
```

## Authentication

### Supabase Client Setup

```typescript
// lib/supabase.ts
import { createClient } from '@supabase/supabase-js'

const supabaseUrl = process.env.NEXT_PUBLIC_SUPABASE_URL as string
const supabaseAnonKey = process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY as string

export const supabase = createClient(supabaseUrl, supabaseAnonKey, {
  auth: {
    persistSession: true,
    autoRefreshToken: true,
  },
  db: {
    schema: 'healthcare'
  }
})
```

### Authentication with Offline Support

Implement biometric authentication with offline fallback:

```typescript
// lib/auth.ts
import { supabase } from './supabase'

export async function loginWithEmail(email: string, password: string) {
  try {
    const { data, error } = await supabase.auth.signInWithPassword({
      email,
      password
    })
    
    if (error) throw error
    
    // Store credentials securely for offline authentication
    await storeOfflineCredentials(email, password)
    
    return { user: data.user, session: data.session }
  } catch (error) {
    console.error('Login error:', error)
    
    // Try offline authentication if online fails
    if (!navigator.onLine) {
      return attemptOfflineLogin(email, password)
    }
    
    throw error
  }
}

async function storeOfflineCredentials(email: string, password: string) {
  // Store hashed credentials in IndexedDB for offline use
  // Use a secure encryption method
}

async function attemptOfflineLogin(email: string, password: string) {
  // Verify against stored credentials
  // Return cached user data
}
```

## Offline Capabilities

### Service Worker Setup

Create a service worker for offline caching:

```javascript
// public/worker.js
const CACHE_NAME = 'healthcare-platform-v1'
const urlsToCache = [
  '/',
  '/offline',
  '/styles/globals.css',
  // Add other critical assets
]

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then((cache) => cache.addAll(urlsToCache))
  )
})

self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request)
      .then((response) => {
        // Return cached response if available
        if (response) {
          return response
        }
        
        // Clone the request
        const fetchRequest = event.request.clone()
        
        return fetch(fetchRequest).then((response) => {
          // Check for valid response
          if (!response || response.status !== 200 || response.type !== 'basic') {
            return response
          }
          
          // Clone the response
          const responseToCache = response.clone()
          
          caches.open(CACHE_NAME).then((cache) => {
            cache.put(event.request, responseToCache)
          })
          
          return response
        }).catch(() => {
          // Return fallback for HTML pages
          if (event.request.headers.get('accept').includes('text/html')) {
            return caches.match('/offline')
          }
        })
      })
  )
})
```

Register the service worker:

```typescript
// pages/_app.tsx
import { useEffect } from 'react'

function MyApp({ Component, pageProps }) {
  useEffect(() => {
    if ('serviceWorker' in navigator) {
      navigator.serviceWorker.register('/worker.js')
        .then((registration) => {
          console.log('Service Worker registered with scope:', registration.scope)
        })
        .catch((error) => {
          console.error('Service Worker registration failed:', error)
        })
    }
  }, [])
  
  return <Component {...pageProps} />
}

export default MyApp
```

### IndexedDB for Offline Data Storage

Set up IndexedDB for offline data persistence:

```typescript
// lib/indexedDB.ts
import { openDB } from 'idb'

export const initDB = async () => {
  return openDB('healthcare-db', 1, {
    upgrade(db) {
      // Create stores for different data types
      if (!db.objectStoreNames.contains('patients')) {
        db.createObjectStore('patients', { keyPath: 'id' })
      }
      
      if (!db.objectStoreNames.contains('prescriptions')) {
        db.createObjectStore('prescriptions', { keyPath: 'id' })
      }
      
      if (!db.objectStoreNames.contains('syncQueue')) {
        db.createObjectStore('syncQueue', { keyPath: 'id', autoIncrement: true })
      }
    }
  })
}

// Example function to save patient data offline
export const savePatientOffline = async (patient) => {
  const db = await initDB()
  await db.put('patients', patient)
}

// Example function to queue operations for sync
export const queueForSync = async (operation) => {
  const db = await initDB()
  await db.add('syncQueue', {
    ...operation,
    timestamp: Date.now(),
    syncStatus: 'pending'
  })
}
```

## Database Operations

### CRUD Operations with Offline Support

```typescript
// lib/patients.ts
import { supabase } from './supabase'
import { savePatientOffline, queueForSync } from './indexedDB'

export async function createPatient(patientData) {
  try {
    if (!navigator.onLine) {
      // Store locally and queue for sync
      const tempId = `temp_${Date.now()}`
      const patient = { ...patientData, id: tempId, syncStatus: 'pending' }
      
      await savePatientOffline(patient)
      await queueForSync({
        type: 'create',
        table: 'patients',
        data: patientData
      })
      
      return { data: patient, error: null }
    }
    
    // Online operation
    const { data, error } = await supabase
      .from('patients')
      .insert(patientData)
      .select()
      .single()
    
    if (error) throw error
    
    // Cache data for offline use
    await savePatientOffline(data)
    
    return { data, error: null }
  } catch (error) {
    console.error('Error creating patient:', error)
    return { data: null, error }
  }
}

export async function getPatient(id) {
  try {
    // Try online first
    if (navigator.onLine) {
      const { data, error } = await supabase
        .from('patients')
        .select('*')
        .eq('id', id)
        .single()
      
      if (error) throw error
      
      // Cache data for offline use
      await savePatientOffline(data)
      
      return { data, error: null }
    }
    
    // Fallback to offline data
    const db = await initDB()
    const patient = await db.get('patients', id)
    
    return { data: patient, error: null }
  } catch (error) {
    console.error('Error fetching patient:', error)
    
    // Try offline data as fallback
    try {
      const db = await initDB()
      const patient = await db.get('patients', id)
      
      return { data: patient, error: null }
    } catch (offlineError) {
      return { data: null, error }
    }
  }
}
```

## Real-time Updates

### Implementing Real-time Subscriptions

```typescript
// components/PrescriptionList.tsx
import { useEffect, useState } from 'react'
import { supabase } from '../lib/supabase'

export default function PrescriptionList() {
  const [prescriptions, setPrescriptions] = useState([])
  
  useEffect(() => {
    // Fetch initial data
    fetchPrescriptions()
    
    // Set up real-time subscription
    const subscription = supabase
      .channel('public:prescriptions')
      .on('postgres_changes', 
        { event: '*', schema: 'public', table: 'prescriptions' }, 
        (payload) => {
          handleRealtimeUpdate(payload)
        }
      )
      .subscribe()
    
    return () => {
      supabase.removeChannel(subscription)
    }
  }, [])
  
  const fetchPrescriptions = async () => {
    const { data, error } = await supabase
      .from('prescriptions')
      .select('*')
      .order('created_at', { ascending: false })
    
    if (error) {
      console.error('Error fetching prescriptions:', error)
      return
    }
    
    setPrescriptions(data)
  }
  
  const handleRealtimeUpdate = (payload) => {
    const { eventType, new: newRecord, old: oldRecord } = payload
    
    if (eventType === 'INSERT') {
      setPrescriptions(prev => [newRecord, ...prev])
    } else if (eventType === 'UPDATE') {
      setPrescriptions(prev => 
        prev.map(item => item.id === newRecord.id ? newRecord : item)
      )
    } else if (eventType === 'DELETE') {
      setPrescriptions(prev => 
        prev.filter(item => item.id !== oldRecord.id)
      )
    }
  }
  
  return (
    <div>
      <h2>Prescriptions</h2>
      <ul>
        {prescriptions.map(prescription => (
          <li key={prescription.id}>
            {prescription.medication} - {prescription.dosage}
          </li>
        ))}
      </ul>
    </div>
  )
}
```

## File Storage

### Managing Medical Documents and Images

```typescript
// lib/storage.ts
import { supabase } from './supabase'
import { queueForSync } from './indexedDB'

export async function uploadMedicalDocument(file, patientId) {
  try {
    if (!navigator.onLine) {
      // Queue for upload when online
      await queueForSync({
        type: 'upload',
        file,
        path: `medical-records/${patientId}/${file.name}`
      })
      
      return { data: { path: `medical-records/${patientId}/${file.name}` }, error: null }
    }
    
    const fileExt = file.name.split('.').pop()
    const fileName = `${Math.random().toString(36).substring(2, 15)}.${fileExt}`
    const filePath = `medical-records/${patientId}/${fileName}`
    
    const { data, error } = await supabase.storage
      .from('documents')
      .upload(filePath, file)
    
    if (error) throw error
    
    return { data, error: null }
  } catch (error) {
    console.error('Error uploading document:', error)
    return { data: null, error }
  }
}

export async function getDocumentUrl(path) {
  const { data } = await supabase.storage.from('documents').getPublicUrl(path)
  return data.publicUrl
}
```

## Internationalization

### Setting Up Multi-language Support

Install required packages:

```bash
npm install next-i18next
```

Configure i18n:

```typescript
// next-i18next.config.js
module.exports = {
  i18n: {
    defaultLocale: 'en',
    locales: ['en', 'fr'],
    localeDetection: true,
  },
  localePath: './public/locales',
}
```

Create translation files:

```json
// public/locales/en/common.json
{
  "welcome": "Welcome to Healthcare Platform",
  "login": "Login",
  "register": "Register",
  "patientManagement": "Patient Management",
  "prescriptions": "Prescriptions",
  "offline": "You are currently offline"
}

// public/locales/fr/common.json
{
  "welcome": "Bienvenue sur la Plateforme de Santé",
  "login": "Connexion",
  "register": "S'inscrire",
  "patientManagement": "Gestion des Patients",
  "prescriptions": "Ordonnances",
  "offline": "Vous êtes actuellement hors ligne"
}
```

Use translations in components:

```typescript
// pages/index.tsx
import { useTranslation } from 'next-i18next'
import { serverSideTranslations } from 'next-i18next/serverSideTranslations'

export default function Home() {
  const { t } = useTranslation('common')
  
  return (
    <div>
      <h1>{t('welcome')}</h1>
      <p>{!navigator.onLine && t('offline')}</p>
    </div>
  )
}

export async function getStaticProps({ locale }) {
  return {
    props: {
      ...(await serverSideTranslations(locale, ['common'])),
    },
  }
}
```

## Performance Optimization

### Implementing for Low-Resource Environments

1. **Code Splitting**: Reduce initial load time

```typescript
// Use dynamic imports for large components
import dynamic from 'next/dynamic'

const PrescriptionEditor = dynamic(
  () => import('../components/PrescriptionEditor'),
  { 
    loading: () => <p>Loading editor...</p>,
    ssr: false // Disable server rendering if component uses browser APIs
  }
)
```

2. **Image Optimization**: Optimize images for low-bandwidth environments

```typescript
// components/PatientPhoto.tsx
import Image from 'next/image'

export default function PatientPhoto({ src, alt }) {
  return (
    <div className="patient-photo">
      <Image 
        src={src} 
        alt={alt}
        width={150}
        height={150}
        placeholder="blur"
        blurDataURL="data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='40' height='40' viewBox='0 0 40 40'%3E%3Ccircle cx='20' cy='20' r='20' fill='%23f2f2f2'/%3E%3C/svg%3E"
        priority={false}
      />
    </div>
  )
}
```

3. **Progressive Web App Setup**: Enhance offline experience

```json
// public/manifest.json
{
  "name": "Healthcare Platform",
  "short_name": "Healthcare",
  "theme_color": "#1E88E5",
  "background_color": "#ffffff",
  "display": "standalone",
  "scope": "/",
  "start_url": "/",
  "icons": [
    {
      "src": "icons/icon-72x72.png",
      "sizes": "72x72",
      "type": "image/png"
    },
    // Add other icon sizes
    {
      "src": "icons/icon-512x512.png",
      "sizes": "512x512",
      "type": "image/png"
    }
  ]
}
```

Add to `_document.tsx`:

```typescript
// pages/_document.tsx
import Document, { Html, Head, Main, NextScript } from 'next/document'

class MyDocument extends Document {
  render() {
    return (
      <Html>
        <Head>
          <link rel="manifest" href="/manifest.json" />
          <meta name="theme-color" content="#1E88E5" />
        </Head>
        <body>
          <Main />
          <NextScript />
        </body>
      </Html>
    )
  }
}

export default MyDocument
```

## Deployment

### Preparing for Production

1. **Environment Variable Configuration**:

Create `.env.production` for production settings:

```
NEXT_PUBLIC_SUPABASE_URL=your_production_supabase_url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_production_supabase_anon_key
```

2. **Build and Export**:

```bash
# Build for production
npm run build

# For static export if needed
npm run export
```

3. **Server Setup for PWA Support**:

When deploying to a server, ensure proper caching headers:

```nginx
# Example Nginx configuration
location / {
  try_files $uri $uri/ /index.html;
}

location /service-worker.js {
  add_header Cache-Control "no-cache";
  proxy_pass http://localhost:3000;
}

location /_next/static {
  add_header Cache-Control "public, max-age=31536000, immutable";
  proxy_pass http://localhost:3000;
}
```

## Conclusion

This implementation guide provides a comprehensive foundation for building the Next.js with Supabase frontend for the Healthcare Platform. The architecture prioritizes offline-first capabilities, multi-language support, and performance optimization for low-resource environments, making it well-suited for deployment in Sub-Saharan Africa and Mali.

## Resources

- [Next.js Documentation](https://nextjs.org/docs)
- [Supabase Documentation](https://supabase.com/docs)
- [Next.js with Supabase Auth Helpers](https://github.com/supabase/auth-helpers)
- [IndexedDB API Documentation](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API)
- [Service Workers API](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API)
