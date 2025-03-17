# React Native with Offline Capabilities for Healthcare Platform

## Overview

This document outlines the implementation of React Native for the mobile application component of the Healthcare Platform targeting Sub-Saharan Africa and Mali. The implementation prioritizes offline functionality, performance optimization for low-resource environments, and seamless synchronization with backend services.

## Table of Contents

1. [Setup and Installation](#setup-and-installation)
2. [Project Structure](#project-structure)
3. [Offline Architecture](#offline-architecture)
4. [Authentication](#authentication)
5. [Data Management](#data-management)
6. [UI Components](#ui-components)
7. [Navigation](#navigation)
8. [Biometric Integration](#biometric-integration)
9. [Performance Optimization](#performance-optimization)
10. [Testing Strategy](#testing-strategy)
11. [Deployment](#deployment)

## Setup and Installation

### Prerequisites

- Node.js 18.x or later
- React Native CLI or Expo (we recommend Expo for easier setup)
- Android Studio (for Android development)
- Xcode (for iOS development, macOS only)

### Creating a New Project

```bash
# Using Expo (recommended for simpler setup)
npx create-expo-app healthcare-mobile --template

# Or using React Native CLI
npx react-native init HealthcareMobile --template react-native-template-typescript
```

### Essential Dependencies

```bash
# Offline and storage
npm install @nozbe/watermelondb
npm install @react-native-async-storage/async-storage

# UI components
npm install react-native-paper
npm install react-native-vector-icons

# Navigation
npm install @react-navigation/native
npm install @react-navigation/stack
npm install @react-navigation/bottom-tabs

# Biometrics
npm install react-native-biometrics

# Network handling
npm install @react-native-community/netinfo

# Internationalization
npm install i18next react-i18next
```

## Project Structure

Recommended project structure for the healthcare mobile app:

```
healthcare-mobile/
├── src/
│   ├── api/                # API integration
│   │   ├── client.ts       # API client setup
│   │   ├── sync.ts         # Synchronization logic
│   │   └── endpoints/      # API endpoint handlers
│   ├── components/         # Reusable UI components
│   │   ├── common/         # Shared components
│   │   ├── forms/          # Form components
│   │   └── patients/       # Patient-specific components
│   ├── database/           # WatermelonDB setup
│   │   ├── models/         # Database models
│   │   ├── schemas/        # Database schemas
│   │   └── index.ts        # Database initialization
│   ├── hooks/              # Custom React hooks
│   ├── navigation/         # Navigation configuration
│   │   ├── AppNavigator.tsx # Main navigator
│   │   ├── AuthNavigator.tsx # Authentication navigation
│   │   └── MainNavigator.tsx # Main app navigation
│   ├── screens/            # Screen components
│   │   ├── auth/           # Authentication screens
│   │   ├── patients/       # Patient management screens
│   │   └── prescriptions/  # Prescription screens
│   ├── services/           # Business logic
│   │   ├── auth.ts         # Authentication service
│   │   ├── patients.ts     # Patient service
│   │   └── sync.ts         # Sync service
│   ├── store/              # State management
│   │   ├── actions/        # Redux actions
│   │   ├── reducers/       # Redux reducers
│   │   └── index.ts        # Store configuration
│   ├── utils/              # Utility functions
│   │   ├── connectivity.ts # Network utilities
│   │   ├── encryption.ts   # Encryption helpers
│   │   └── validators.ts   # Form validation
│   ├── i18n/               # Internationalization
│   │   ├── en/             # English translations
│   │   ├── fr/             # French translations
│   │   └── index.ts        # i18n configuration
│   └── App.tsx             # Root component
├── .env                    # Environment variables
├── app.json                # App configuration
└── tsconfig.json           # TypeScript configuration
```

## Offline Architecture

### WatermelonDB Setup

WatermelonDB is an efficient, offline-first database for React Native that enables robust offline capabilities with seamless synchronization.

```typescript
// src/database/index.ts
import { Database } from '@nozbe/watermelondb'
import SQLiteAdapter from '@nozbe/watermelondb/adapters/sqlite'

import { schema } from './schemas'
import { Patient, Prescription, SyncLog } from './models'

// Initialize database adapter
const adapter = new SQLiteAdapter({
  schema,
  // Use SQLite for Android and iOS
  dbName: 'healthcareDB',
  // Optional migrations
  migrations: [],
  // Use foreign keys
  useWebWorker: false,
})

// Initialize database with models
export const database = new Database({
  adapter,
  modelClasses: [
    Patient,
    Prescription,
    SyncLog,
  ],
})
```

### Schema Definition

```typescript
// src/database/schemas/index.ts
import { appSchema, tableSchema } from '@nozbe/watermelondb'

export const schema = appSchema({
  version: 1,
  tables: [
    tableSchema({
      name: 'patients',
      columns: [
        { name: 'server_id', type: 'string', isOptional: true },
        { name: 'name', type: 'string' },
        { name: 'date_of_birth', type: 'string' },
        { name: 'gender', type: 'string' },
        { name: 'contact_number', type: 'string', isOptional: true },
        { name: 'address', type: 'string', isOptional: true },
        { name: 'medical_history', type: 'string', isOptional: true },
        { name: 'last_sync_at', type: 'number', isOptional: true },
        { name: 'created_at', type: 'number' },
        { name: 'updated_at', type: 'number' },
        { name: 'sync_status', type: 'string', isOptional: true },
      ],
    }),
    tableSchema({
      name: 'prescriptions',
      columns: [
        { name: 'server_id', type: 'string', isOptional: true },
        { name: 'patient_id', type: 'string', isIndexed: true },
        { name: 'medication', type: 'string' },
        { name: 'dosage', type: 'string' },
        { name: 'frequency', type: 'string' },
        { name: 'start_date', type: 'number' },
        { name: 'end_date', type: 'number', isOptional: true },
        { name: 'notes', type: 'string', isOptional: true },
        { name: 'last_sync_at', type: 'number', isOptional: true },
        { name: 'created_at', type: 'number' },
        { name: 'updated_at', type: 'number' },
        { name: 'sync_status', type: 'string', isOptional: true },
      ],
    }),
    tableSchema({
      name: 'sync_logs',
      columns: [
        { name: 'entity_type', type: 'string' },
        { name: 'entity_id', type: 'string' },
        { name: 'action', type: 'string' },
        { name: 'status', type: 'string' },
        { name: 'error', type: 'string', isOptional: true },
        { name: 'created_at', type: 'number' },
      ],
    }),
  ],
})
```

### Database Models

```typescript
// src/database/models/Patient.ts
import { Model } from '@nozbe/watermelondb'
import { field, date, readonly, text } from '@nozbe/watermelondb/decorators'

export default class Patient extends Model {
  static table = 'patients'
  
  @text('server_id') serverId
  @text('name') name
  @text('date_of_birth') dateOfBirth
  @text('gender') gender
  @text('contact_number') contactNumber
  @text('address') address
  @text('medical_history') medicalHistory
  @text('sync_status') syncStatus
  @date('last_sync_at') lastSyncAt
  @readonly @date('created_at') createdAt
  @readonly @date('updated_at') updatedAt
}

// src/database/models/Prescription.ts
import { Model } from '@nozbe/watermelondb'
import { field, date, readonly, text, relation } from '@nozbe/watermelondb/decorators'

export default class Prescription extends Model {
  static table = 'prescriptions'
  
  @text('server_id') serverId
  @text('patient_id') patientId
  @text('medication') medication
  @text('dosage') dosage
  @text('frequency') frequency
  @date('start_date') startDate
  @date('end_date') endDate
  @text('notes') notes
  @text('sync_status') syncStatus
  @date('last_sync_at') lastSyncAt
  @readonly @date('created_at') createdAt
  @readonly @date('updated_at') updatedAt
  
  @relation('patients', 'patient_id') patient
}
```

### Synchronization Service

```typescript
// src/services/sync.ts
import NetInfo from '@react-native-community/netinfo'
import { database } from '../database'
import { apiClient } from '../api/client'

export class SyncService {
  // Track sync status
  private isSyncing = false
  
  // Initialize with database
  constructor() {
    // Set up network change listeners
    NetInfo.addEventListener(state => {
      if (state.isConnected && state.isInternetReachable) {
        this.syncWhenOnline()
      }
    })
  }
  
  // Attempt to sync when online
  private async syncWhenOnline() {
    const state = await NetInfo.fetch()
    if (state.isConnected && state.isInternetReachable) {
      this.synchronize()
    }
  }
  
  // Main synchronization function
  public async synchronize() {
    if (this.isSyncing) return
    
    try {
      this.isSyncing = true
      
      // Get all records that need to be synced
      await database.action(async () => {
        // 1. Push local changes to server
        await this.pushChanges()
        
        // 2. Pull changes from server
        await this.pullChanges()
      })
    } catch (error) {
      console.error('Sync error:', error)
      // Log sync error for retry
    } finally {
      this.isSyncing = false
    }
  }
  
  // Push local changes to server
  private async pushChanges() {
    // Get patients to sync
    const patientsToSync = await database.get('patients')
      .query(Q.where('sync_status', Q.oneOf(['pending', 'update', 'delete'])))
      .fetch()
    
    // Get prescriptions to sync
    const prescriptionsToSync = await database.get('prescriptions')
      .query(Q.where('sync_status', Q.oneOf(['pending', 'update', 'delete'])))
      .fetch()
    
    // Push patients
    for (const patient of patientsToSync) {
      await this.syncPatient(patient)
    }
    
    // Push prescriptions
    for (const prescription of prescriptionsToSync) {
      await this.syncPrescription(prescription)
    }
  }
  
  // Pull changes from server
  private async pullChanges() {
    // Get last sync timestamp
    const lastSync = await this.getLastSyncTimestamp()
    
    // Fetch changes from server
    const { patients, prescriptions } = await apiClient.fetchChanges(lastSync)
    
    // Apply patient changes
    for (const patientData of patients) {
      await this.applyPatientChange(patientData)
    }
    
    // Apply prescription changes
    for (const prescriptionData of prescriptions) {
      await this.applyPrescriptionChange(prescriptionData)
    }
    
    // Update last sync timestamp
    await this.updateLastSyncTimestamp()
  }
  
  // Sync a single patient
  private async syncPatient(patient) {
    try {
      if (patient.syncStatus === 'delete') {
        // Delete from server
        if (patient.serverId) {
          await apiClient.deletePatient(patient.serverId)
        }
        // Delete locally
        await patient.destroyPermanently()
      } else {
        // Create or update
        const patientData = {
          id: patient.serverId,
          name: patient.name,
          dateOfBirth: patient.dateOfBirth,
          gender: patient.gender,
          contactNumber: patient.contactNumber,
          address: patient.address,
          medicalHistory: patient.medicalHistory,
        }
        
        let serverPatient
        if (patient.syncStatus === 'pending') {
          // Create new patient on server
          serverPatient = await apiClient.createPatient(patientData)
        } else {
          // Update existing patient
          serverPatient = await apiClient.updatePatient(patient.serverId, patientData)
        }
        
        // Update local record with server data
        await patient.update(p => {
          p.serverId = serverPatient.id
          p.syncStatus = 'synced'
          p.lastSyncAt = new Date()
        })
      }
    } catch (error) {
      console.error('Error syncing patient:', error)
      // Log sync error for retry
    }
  }
  
  // Similar function for prescriptions
  private async syncPrescription(prescription) {
    // Implementation similar to syncPatient
  }
  
  // Apply changes from server to local database
  private async applyPatientChange(patientData) {
    // Find local patient by server ID
    const localPatient = await database.get('patients')
      .query(Q.where('server_id', patientData.id))
      .fetch()
    
    if (localPatient.length > 0) {
      // Update existing patient
      await localPatient[0].update(p => {
        p.name = patientData.name
        p.dateOfBirth = patientData.dateOfBirth
        p.gender = patientData.gender
        p.contactNumber = patientData.contactNumber
        p.address = patientData.address
        p.medicalHistory = patientData.medicalHistory
        p.syncStatus = 'synced'
        p.lastSyncAt = new Date()
      })
    } else {
      // Create new patient locally
      await database.get('patients').create(p => {
        p.serverId = patientData.id
        p.name = patientData.name
        p.dateOfBirth = patientData.dateOfBirth
        p.gender = patientData.gender
        p.contactNumber = patientData.contactNumber
        p.address = patientData.address
        p.medicalHistory = patientData.medicalHistory
        p.syncStatus = 'synced'
        p.lastSyncAt = new Date()
      })
    }
  }
}

// Initialize and export sync service
export const syncService = new SyncService()
```

## Authentication

### Biometric Authentication Implementation

```typescript
// src/services/auth.ts
import AsyncStorage from '@react-native-async-storage/async-storage'
import ReactNativeBiometrics from 'react-native-biometrics'
import { apiClient } from '../api/client'

const rnBiometrics = new ReactNativeBiometrics()

export class AuthService {
  // Login with credentials
  async login(username, password) {
    try {
      // Online authentication
      const authResult = await apiClient.login(username, password)
      
      // Store tokens securely
      await this.storeAuthTokens(authResult.tokens)
      
      // Set up biometrics if available
      await this.setupBiometrics(username, password)
      
      return authResult.user
    } catch (error) {
      // Try offline authentication if network error
      if (!navigator.onLine) {
        return this.offlineLogin(username, password)
      }
      throw error
    }
  }
  
  // Setup biometric authentication
  async setupBiometrics(username, password) {
    const { available } = await rnBiometrics.isSensorAvailable()
    
    if (available) {
      // Create secure encryption key with biometrics
      const { publicKey } = await rnBiometrics.createKeys()
      
      // Encrypt credentials with public key
      // This is simplified - in a real app, use a proper encryption method
      const encryptedCredentials = JSON.stringify({
        username,
        password,
      })
      
      // Store encrypted credentials
      await AsyncStorage.setItem('biometric_credentials', encryptedCredentials)
      await AsyncStorage.setItem('biometric_public_key', publicKey)
    }
  }
  
  // Login with biometrics
  async biometricLogin() {
    const encryptedCredentials = await AsyncStorage.getItem('biometric_credentials')
    const publicKey = await AsyncStorage.getItem('biometric_public_key')
    
    if (!encryptedCredentials || !publicKey) {
      throw new Error('Biometric login not set up')
    }
    
    // Prompt for biometric authentication
    const { success } = await rnBiometrics.simplePrompt({
      promptMessage: 'Authenticate to continue',
    })
    
    if (success) {
      // Decrypt credentials
      // In a real app, use proper decryption
      const credentials = JSON.parse(encryptedCredentials)
      
      // Login with decrypted credentials
      return this.login(credentials.username, credentials.password)
    }
    
    throw new Error('Biometric authentication failed')
  }
  
  // Offline login when no network
  async offlineLogin(username, password) {
    // Get stored hash from secure storage
    const storedHash = await AsyncStorage.getItem(`offline_auth_${username}`)
    
    if (!storedHash) {
      throw new Error('No offline credentials stored')
    }
    
    // Verify password hash
    // In a real app, use a proper hashing algorithm
    const passwordHash = password // This should be hashed
    
    if (passwordHash !== storedHash) {
      throw new Error('Invalid credentials')
    }
    
    // Get cached user data
    const userData = await AsyncStorage.getItem(`offline_user_${username}`)
    
    if (!userData) {
      throw new Error('No offline user data')
    }
    
    return JSON.parse(userData)
  }
  
  // Store authentication tokens
  async storeAuthTokens(tokens) {
    await AsyncStorage.setItem('auth_tokens', JSON.stringify(tokens))
  }
  
  // Get stored authentication tokens
  async getAuthTokens() {
    const tokens = await AsyncStorage.getItem('auth_tokens')
    return tokens ? JSON.parse(tokens) : null
  }
  
  // Refresh tokens
  async refreshTokens() {
    const tokens = await this.getAuthTokens()
    
    if (!tokens || !tokens.refreshToken) {
      throw new Error('No refresh token available')
    }
    
    try {
      const newTokens = await apiClient.refreshTokens(tokens.refreshToken)
      await this.storeAuthTokens(newTokens)
      return newTokens
    } catch (error) {
      // Clear tokens if refresh fails
      await AsyncStorage.removeItem('auth_tokens')
      throw error
    }
  }
  
  // Logout
  async logout() {
    // Clear all tokens and credentials
    await AsyncStorage.removeItem('auth_tokens')
    await AsyncStorage.removeItem('biometric_credentials')
    await AsyncStorage.removeItem('biometric_public_key')
  }
}

// Initialize and export auth service
export const authService = new AuthService()
```

## Data Management

### Patient Management Service

```typescript
// src/services/patients.ts
import { database } from '../database'
import { syncService } from './sync'

export class PatientService {
  // Create a new patient
  async createPatient(patientData) {
    try {
      let newPatient
      
      await database.action(async () => {
        newPatient = await database.get('patients').create(patient => {
          patient.name = patientData.name
          patient.dateOfBirth = patientData.dateOfBirth
          patient.gender = patientData.gender
          patient.contactNumber = patientData.contactNumber || ''
          patient.address = patientData.address || ''
          patient.medicalHistory = patientData.medicalHistory || ''
          patient.syncStatus = 'pending'
        })
      })
      
      // Try to sync immediately if online
      syncService.synchronize()
      
      return newPatient
    } catch (error) {
      console.error('Error creating patient:', error)
      throw error
    }
  }
  
  // Get all patients
  async getPatients() {
    return database.get('patients').query().fetch()
  }
  
  // Get patient by ID
  async getPatient(id) {
    return database.get('patients').find(id)
  }
  
  // Update patient
  async updatePatient(id, patientData) {
    let updatedPatient
    
    await database.action(async () => {
      const patient = await database.get('patients').find(id)
      
      updatedPatient = await patient.update(p => {
        p.name = patientData.name
        p.dateOfBirth = patientData.dateOfBirth
        p.gender = patientData.gender
        p.contactNumber = patientData.contactNumber || p.contactNumber
        p.address = patientData.address || p.address
        p.medicalHistory = patientData.medicalHistory || p.medicalHistory
        p.syncStatus = 'update'
      })
    })
    
    // Try to sync immediately if online
    syncService.synchronize()
    
    return updatedPatient
  }
  
  // Delete patient
  async deletePatient(id) {
    await database.action(async () => {
      const patient = await database.get('patients').find(id)
      
      // Mark for deletion (will be removed after sync)
      await patient.update(p => {
        p.syncStatus = 'delete'
      })
      
      // Also mark related prescriptions for deletion
      const prescriptions = await database.get('prescriptions')
        .query(Q.where('patient_id', id))
        .fetch()
      
      for (const prescription of prescriptions) {
        await prescription.update(p => {
          p.syncStatus = 'delete'
        })
      }
    })
    
    // Try to sync immediately if online
    syncService.synchronize()
  }
  
  // Search patients by name
  async searchPatients(query) {
    return database.get('patients')
      .query(Q.where('name', Q.like(`%${query}%`)))
      .fetch()
  }
}

// Initialize and export patient service
export const patientService = new PatientService()
```

## UI Components

### Custom UI Components with React Native Paper

```typescript
// src/components/common/OfflineIndicator.tsx
import React from 'react'
import { View, StyleSheet } from 'react-native'
import { Banner, Text } from 'react-native-paper'
import { useNetInfo } from '@react-native-community/netinfo'
import { useTranslation } from 'react-i18next'

export const OfflineIndicator = () => {
  const netInfo = useNetInfo()
  const { t } = useTranslation()
  
  if (netInfo.isConnected === false) {
    return (
      <Banner
        visible={true}
        icon="wifi-off"
        actions={[]}
        style={styles.banner}
      >
        <Text style={styles.text}>{t('offline.working_offline')}</Text>
      </Banner>
    )
  }
  
  return null
}

const styles = StyleSheet.create({
  banner: {
    backgroundColor: '#FFF9C4',
  },
  text: {
    color: '#5D4037',
  },
})

// src/components/forms/PatientForm.tsx
import React, { useState } from 'react'
import { ScrollView, StyleSheet, View } from 'react-native'
import { TextInput, Button, HelperText, RadioButton } from 'react-native-paper'
import { useTranslation } from 'react-i18next'

export const PatientForm = ({ onSubmit, initialValues = {}, isEditing = false }) => {
  const { t } = useTranslation()
  const [values, setValues] = useState({
    name: initialValues.name || '',
    dateOfBirth: initialValues.dateOfBirth || '',
    gender: initialValues.gender || '',
    contactNumber: initialValues.contactNumber || '',
    address: initialValues.address || '',
    medicalHistory: initialValues.medicalHistory || '',
  })
  
  const [errors, setErrors] = useState({})
  
  const handleChange = (field, value) => {
    setValues({ ...values, [field]: value })
    // Clear error when field is edited
    if (errors[field]) {
      setErrors({ ...errors, [field]: undefined })
    }
  }
  
  const validate = () => {
    const newErrors = {}
    
    if (!values.name) {
      newErrors.name = t('validation.name_required')
    }
    
    if (!values.dateOfBirth) {
      newErrors.dateOfBirth = t('validation.dob_required')
    }
    
    if (!values.gender) {
      newErrors.gender = t('validation.gender_required')
    }
    
    setErrors(newErrors)
    return Object.keys(newErrors).length === 0
  }
  
  const handleSubmit = () => {
    if (validate()) {
      onSubmit(values)
    }
  }
  
  return (
    <ScrollView style={styles.container}>
      <TextInput
        label={t('patient.name')}
        value={values.name}
        onChangeText={(text) => handleChange('name', text)}
        style={styles.input}
        error={!!errors.name}
      />
      {errors.name && <HelperText type="error">{errors.name}</HelperText>}
      
      <TextInput
        label={t('patient.date_of_birth')}
        value={values.dateOfBirth}
        onChangeText={(text) => handleChange('dateOfBirth', text)}
        style={styles.input}
        error={!!errors.dateOfBirth}
      />
      {errors.dateOfBirth && <HelperText type="error">{errors.dateOfBirth}</HelperText>}
      
      <View style={styles.genderContainer}>
        <RadioButton.Group
          onValueChange={(value) => handleChange('gender', value)}
          value={values.gender}
        >
          <View style={styles.radioRow}>
            <RadioButton.Item label={t('gender.male')} value="male" />
            <RadioButton.Item label={t('gender.female')} value="female" />
            <RadioButton.Item label={t('gender.other')} value="other" />
          </View>
        </RadioButton.Group>
        {errors.gender && <HelperText type="error">{errors.gender}</HelperText>}
      </View>
      
      <TextInput
        label={t('patient.contact_number')}
        value={values.contactNumber}
        onChangeText={(text) => handleChange('contactNumber', text)}
        style={styles.input}
        keyboardType="phone-pad"
      />
      
      <TextInput
        label={t('patient.address')}
        value={values.address}
        onChangeText={(text) => handleChange('address', text)}
        style={styles.input}
        multiline
      />
      
      <TextInput
        label={t('patient.medical_history')}
        value={values.medicalHistory}
        onChangeText={(text) => handleChange('medicalHistory', text)}
        style={styles.textArea}
        multiline
        numberOfLines={4}
      />
      
      <Button
        mode="contained"
        onPress={handleSubmit}
        style={styles.button}
      >
        {isEditing ? t('common.update') : t('common.save')}
      </Button>
    </ScrollView>
  )
}

const styles = StyleSheet.create({
  container: {
    padding: 16,
  },
  input: {
    marginBottom: 8,
  },
  textArea: {
    marginBottom: 16,
    height: 100,
  },
  genderContainer: {
    marginBottom: 16,
  },
  radioRow: {
    flexDirection: 'row',
    alignItems: 'center',
  },
  button: {
    marginTop: 16,
  },
})
```

## Navigation

### Navigation Setup

```typescript
// src/navigation/AppNavigator.tsx
import React, { useEffect, useState } from 'react'
import { NavigationContainer } from '@react-navigation/native'
import AsyncStorage from '@react-native-async-storage/async-storage'
import { ActivityIndicator, View } from 'react-native'

import { AuthNavigator } from './AuthNavigator'
import { MainNavigator } from './MainNavigator'

export const AppNavigator = () => {
  const [isLoading, setIsLoading] = useState(true)
  const [isAuthenticated, setIsAuthenticated] = useState(false)
  
  useEffect(() => {
    // Check if user is authenticated
    const checkAuth = async () => {
      try {
        const tokens = await AsyncStorage.getItem('auth_tokens')
        setIsAuthenticated(!!tokens)
      } catch (error) {
        console.error('Error checking auth status:', error)
      } finally {
        setIsLoading(false)
      }
    }
    
    checkAuth()
  }, [])
  
  if (isLoading) {
    return (
      <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
        <ActivityIndicator size="large" />
      </View>
    )
  }
  
  return (
    <NavigationContainer>
      {isAuthenticated ? <MainNavigator /> : <AuthNavigator />}
    </NavigationContainer>
  )
}

// src/navigation/MainNavigator.tsx
import React from 'react'
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs'
import { createStackNavigator } from '@react-navigation/stack'
import { IconButton } from 'react-native-paper'
import { useTranslation } from 'react-i18next'

import { HomeScreen } from '../screens/HomeScreen'
import { PatientsScreen } from '../screens/patients/PatientsScreen'
import { PatientDetailScreen } from '../screens/patients/PatientDetailScreen'
import { AddPatientScreen } from '../screens/patients/AddPatientScreen'
import { EditPatientScreen } from '../screens/patients/EditPatientScreen'
import { PrescriptionsScreen } from '../screens/prescriptions/PrescriptionsScreen'
import { AddPrescriptionScreen } from '../screens/prescriptions/AddPrescriptionScreen'
import { SettingsScreen } from '../screens/SettingsScreen'

const Tab = createBottomTabNavigator()
const Stack = createStackNavigator()

const PatientsStack = () => {
  return (
    <Stack.Navigator>
      <Stack.Screen name="PatientsList" component={PatientsScreen} />
      <Stack.Screen name="PatientDetail" component={PatientDetailScreen} />
      <Stack.Screen name="AddPatient" component={AddPatientScreen} />
      <Stack.Screen name="EditPatient" component={EditPatientScreen} />
    </Stack.Navigator>
  )
}

const PrescriptionsStack = () => {
  return (
    <Stack.Navigator>
      <Stack.Screen name="PrescriptionsList" component={PrescriptionsScreen} />
      <Stack.Screen name="AddPrescription" component={AddPrescriptionScreen} />
    </Stack.Navigator>
  )
}

export const MainNavigator = () => {
  const { t } = useTranslation()
  
  return (
    <Tab.Navigator>
      <Tab.Screen
        name="Home"
        component={HomeScreen}
        options={{
          tabBarLabel: t('navigation.home'),
          tabBarIcon: ({ color, size }) => (
            <IconButton icon="home" color={color} size={size} />
          ),
        }}
      />
      <Tab.Screen
        name="Patients"
        component={PatientsStack}
        options={{
          tabBarLabel: t('navigation.patients'),
          tabBarIcon: ({ color, size }) => (
            <IconButton icon="account-multiple" color={color} size={size} />
          ),
          headerShown: false,
        }}
      />
      <Tab.Screen
        name="Prescriptions"
        component={PrescriptionsStack}
        options={{
          tabBarLabel: t('navigation.prescriptions'),
          tabBarIcon: ({ color, size }) => (
            <IconButton icon="file-document" color={color} size={size} />
          ),
          headerShown: false,
        }}
      />
      <Tab.Screen
        name="Settings"
        component={SettingsScreen}
        options={{
          tabBarLabel: t('navigation.settings'),
          tabBarIcon: ({ color, size }) => (
            <IconButton icon="cog" color={color} size={size} />
          ),
        }}
      />
    </Tab.Navigator>
  )
}
```

## Biometric Integration

### Biometric Authentication Component

```typescript
// src/components/auth/BiometricPrompt.tsx
import React, { useEffect, useState } from 'react'
import { View, StyleSheet } from 'react-native'
import { Button, Text } from 'react-native-paper'
import ReactNativeBiometrics from 'react-native-biometrics'
import { useTranslation } from 'react-i18next'
import AsyncStorage from '@react-native-async-storage/async-storage'

const rnBiometrics = new ReactNativeBiometrics()

interface BiometricPromptProps {
  onSuccess: () => void
  onCancel: () => void
}

export const BiometricPrompt = ({ onSuccess, onCancel }: BiometricPromptProps) => {
  const { t } = useTranslation()
  const [isBiometricAvailable, setIsBiometricAvailable] = useState(false)
  const [biometricType, setBiometricType] = useState<string>('')
  
  useEffect(() => {
    checkBiometricAvailability()
  }, [])
  
  const checkBiometricAvailability = async () => {
    try {
      // Check if device supports biometrics
      const { available, biometryType } = await rnBiometrics.isSensorAvailable()
      
      // Check if user has set up biometric login
      const hasSetup = await AsyncStorage.getItem('biometric_credentials')
      
      setIsBiometricAvailable(available && !!hasSetup)
      setBiometricType(biometryType || '')
    } catch (error) {
      console.error('Error checking biometric availability:', error)
      setIsBiometricAvailable(false)
    }
  }
  
  const handleBiometricAuthentication = async () => {
    try {
      const { success } = await rnBiometrics.simplePrompt({
        promptMessage: t('auth.biometric_prompt'),
        cancelButtonText: t('common.cancel'),
      })
      
      if (success) {
        onSuccess()
      }
    } catch (error) {
      console.error('Biometric authentication error:', error)
    }
  }
  
  if (!isBiometricAvailable) {
    return null
  }
  
  const biometricLabel = biometricType === 'FaceID' 
    ? t('auth.use_face_id') 
    : biometricType === 'TouchID' 
      ? t('auth.use_touch_id') 
      : t('auth.use_biometrics')
  
  return (
    <View style={styles.container}>
      <Text style={styles.text}>{t('auth.login_faster')}</Text>
      <Button
        mode="contained"
        icon={biometricType === 'FaceID' ? 'face-recognition' : 'fingerprint'}
        onPress={handleBiometricAuthentication}
        style={styles.button}
      >
        {biometricLabel}
      </Button>
      <Button
        mode="text"
        onPress={onCancel}
      >
        {t('auth.use_password')}
      </Button>
    </View>
  )
}

const styles = StyleSheet.create({
  container: {
    padding: 16,
    alignItems: 'center',
  },
  text: {
    marginBottom: 16,
    textAlign: 'center',
  },
  button: {
    marginBottom: 8,
    width: '100%',
  },
})
```

## Performance Optimization

### Optimizing for Low-Resource Devices

```typescript
// src/utils/performance.ts
import { InteractionManager } from 'react-native'

// Run heavy operations after animations complete
export const runAfterInteractions = (task: () => any): Promise<any> => {
  return new Promise((resolve) => {
    InteractionManager.runAfterInteractions(() => {
      const result = task()
      resolve(result)
    })
  })
}

// Chunk array processing to avoid UI blocking
export const processArrayInChunks = async (
  array: any[],
  chunkSize: number,
  processor: (item: any) => any
): Promise<any[]> => {
  const results: any[] = []
  
  for (let i = 0; i < array.length; i += chunkSize) {
    const chunk = array.slice(i, i + chunkSize)
    
    // Process chunk
    const chunkResults = await Promise.all(chunk.map(processor))
    results.push(...chunkResults)
    
    // Yield to UI thread before processing next chunk
    await new Promise(resolve => setTimeout(resolve, 0))
  }
  
  return results
}

// Optimize image loading for list items
import { PixelRatio } from 'react-native'

export const getOptimizedImageSize = (originalWidth: number, originalHeight: number): { width: number, height: number } => {
  const pixelRatio = PixelRatio.get()
  
  // Scale down images for low-end devices
  if (pixelRatio < 2) {
    return {
      width: originalWidth / 2,
      height: originalHeight / 2,
    }
  }
  
  return {
    width: originalWidth,
    height: originalHeight,
  }
}

// Cache management for limited storage
import AsyncStorage from '@react-native-async-storage/async-storage'

export const cleanupImageCache = async (): Promise<void> => {
  try {
    // Get cache info
    const cacheInfo = await AsyncStorage.getItem('image_cache_info')
    
    if (!cacheInfo) return
    
    const { items, size, lastCleanup } = JSON.parse(cacheInfo)
    const now = Date.now()
    
    // Clean up cache if over 50MB or last cleaned more than 3 days ago
    if (size > 50 * 1024 * 1024 || now - lastCleanup > 3 * 24 * 60 * 60 * 1000) {
      // Sort items by last accessed time
      const sortedItems = items.sort((a, b) => a.lastAccessed - b.lastAccessed)
      
      // Remove oldest 30% of items
      const itemsToRemove = sortedItems.slice(0, Math.floor(sortedItems.length * 0.3))
      
      for (const item of itemsToRemove) {
        await AsyncStorage.removeItem(`image_cache_${item.key}`)
      }
      
      // Update cache info
      await AsyncStorage.setItem('image_cache_info', JSON.stringify({
        items: sortedItems.slice(Math.floor(sortedItems.length * 0.3)),
        size: size - itemsToRemove.reduce((acc, item) => acc + item.size, 0),
        lastCleanup: now,
      }))
    }
  } catch (error) {
    console.error('Error cleaning up image cache:', error)
  }
}
```

## Testing Strategy

### Testing Offline Capabilities

```typescript
// src/utils/testing.ts
import AsyncStorage from '@react-native-async-storage/async-storage'
import NetInfo from '@react-native-community/netinfo'
import { database } from '../database'

// Simulate offline mode
export const goOffline = (): void => {
  // Mock NetInfo to return offline status
  jest.spyOn(NetInfo, 'fetch').mockImplementation(() => 
    Promise.resolve({
      isConnected: false,
      isInternetReachable: false,
      type: 'none',
      details: null,
    })
  )
  
  // Simulate offline window.navigator
  Object.defineProperty(navigator, 'onLine', {
    configurable: true,
    value: false,
  })
}

// Simulate coming back online
export const goOnline = (): void => {
  // Restore NetInfo
  jest.spyOn(NetInfo, 'fetch').mockImplementation(() => 
    Promise.resolve({
      isConnected: true,
      isInternetReachable: true,
      type: 'wifi',
      details: null,
    })
  )
  
  // Restore online window.navigator
  Object.defineProperty(navigator, 'onLine', {
    configurable: true,
    value: true,
  })
}

// Reset test database
export const resetDatabase = async (): Promise<void> => {
  await database.action(async () => {
    await database.unsafeResetDatabase()
  })
}

// Clear AsyncStorage
export const clearStorage = async (): Promise<void> => {
  await AsyncStorage.clear()
}

// Initialize test data
export const setupTestData = async (): Promise<void> => {
  await database.action(async () => {
    // Create test patients
    await database.get('patients').create(patient => {
      patient.name = 'Test Patient'
      patient.dateOfBirth = '1990-01-01'
      patient.gender = 'male'
      patient.contactNumber = '1234567890'
      patient.syncStatus = 'synced'
      patient.serverId = 'server_id_1'
    })
    
    // Create test prescriptions
    await database.get('prescriptions').create(prescription => {
      prescription.patientId = 'server_id_1'
      prescription.medication = 'Test Medication'
      prescription.dosage = '10mg'
      prescription.frequency = 'twice daily'
      prescription.startDate = new Date()
      prescription.syncStatus = 'synced'
      prescription.serverId = 'server_rx_1'
    })
  })
}
```

## Deployment

### Building for Production

```bash
# For Expo projects
expo build:android
expo build:ios

# For React Native CLI projects
# Android
cd android && ./gradlew assembleRelease

# iOS (requires Mac)
cd ios && xcodebuild -workspace YourApp.xcworkspace -scheme YourApp -configuration Release
```

### CI/CD Setup with GitHub Actions

```yaml
# .github/workflows/build.yml
name: Build and Test

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      - name: Install Dependencies
        run: npm ci
      - name: Run Tests
        run: npm test

  build-android:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      - name: Setup Java
        uses: actions/setup-java@v2
        with:
          distribution: 'adopt'
          java-version: '11'
      - name: Install Dependencies
        run: npm ci
      - name: Build Android
        run: |
          cd android
          ./gradlew assembleRelease
      - name: Upload APK
        uses: actions/upload-artifact@v2
        with:
          name: app-release
          path: android/app/build/outputs/apk/release/app-release.apk
```

## Conclusion

This implementation guide provides a comprehensive approach to developing the React Native mobile application for the Healthcare Platform. The architecture prioritizes offline-first capabilities, multi-language support, and performance optimization for low-resource environments, making it well-suited for deployment in Sub-Saharan Africa and Mali.

## Resources

- [React Native Documentation](https://reactnative.dev/docs/getting-started)
- [WatermelonDB Documentation](https://nozbe.github.io/WatermelonDB/index.html)
- [React Native Paper](https://callstack.github.io/react-native-paper/)
- [React Navigation](https://reactnavigation.org/docs/getting-started)
- [React Native Biometrics](https://github.com/SelfLender/react-native-biometrics)
