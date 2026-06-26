# Tenna-Cloud Offline-First System Implementation

## Overview
This document summarizes the offline-first system implemented for Tenna-Cloud using Dexie.js (IndexedDB wrapper) and Service Workers with Background Sync API.

## Components Implemented

### 1. Local Database (`offline/db.ts`)
- Dexie.js database instance with schema matching Supabase tables
- Tables: clinics, profiles, patients, appointments, invoices, subscriptions, syncQueue
- Proper indexing and relationships
- TypeScript interfaces for all entities

### 2. Offline Service (`offline/service.ts`)
- OnlineService` with the following capabilities:
  - **Online/Offline Detection**: Uses `navigator.onLine` and event listeners
  - **Automatic Initialization**: Sets up event listeners on module load
  - **Bidirectional Sync**:
    - Local → Server: Saves changes to IndexedDB immediately, queues for sync when online
    - Server → Local: Pulls latest data from server when online
  - **Conflict Resolution**: Server wins in case of conflicts (can be enhanced)
  - **Sync Queue**: IndexedDB table to track pending operations when offline
  - **Table-Specific Sync**: Individual methods for syncing each table type
  - **Save Locally**: Unified method for saving entities with automatic queuing
  - **Get Local Data**: Always-available local data access

### 3. Enhanced Services
#### Patient Service (`services/patient-service.ts`)
- All CRUD operations now use offline-first approach
- Read operations try local DB first (with freshness check), fallback to server
- Write operations save to local DB immediately, then sync to server when online
- Search functionality works offline with local filtering

#### Appointment Service (`services/appointment-service.ts`)
- All CRUD operations now use offline-first approach
- Read operations try local DB first (with freshness check), fallback to server
- Write operations save to local DB immediately, then sync to server when online
- Relationship loading (patient/doctor data) cached locally
- Date-based filtering works offline

### 4. Service Worker (`public/sw.js`)
- **Caching Strategy**: Cache-first for static assets, network-first for API requests
- **Offline Page**: Serves `/offline.html` when offline navigation requests fail
- **Background Sync**: Listens for `sync` and `periodicsync` events
- **Client Communication**: Uses BroadcastChannel to notify clients of sync status
- **Registration**: Automatic registration via layout.tsx script

### 5. PWA Support
- **Manifest** (`public/manifest.json`): App metadata for installation
- **Icons**: Placeholder icons for PWA installation
- **Offline Page** (`public/offline.html`): User-friendly offline experience

### 6. Integration
- **Layout Updates** (`app/layout.tsx`): 
  - Added manifest link
  - Added service worker registration script
- **Existing Components**: No changes needed - they automatically benefit from offline-first services

## Key Features

### 1. Seamless Offline Experience
- Users can continue working when internet connection is lost
- All changes saved locally to IndexedDB immediately
- UI remains responsive with local data access

### 2. Automatic Synchronization
- Changes automatically sync to server when connection is restored
- Background sync using Service Worker API
- Periodic sync for keeping data fresh
- Manual retry capability via UI

### 3. Data Consistency
- Local DB serves as single source of truth during offline periods
- Conflict resolution favors server data (can be customized)
- Transactional updates for related data (patient/doctor with appointments)

### 4. Performance Benefits
- Fast reads from local IndexedDB (no network latency)
- Reduced server load during peak usage
- Immediate UI feedback for user actions

### 5. Robust Error Handling
- Graceful degradation to local-only mode when server unavailable
- Detailed logging for debugging
- Queue persistence across page reloads

## Usage

### For Developers
The offline system is transparent - existing service calls work exactly the same:
```javascript
// These calls now work offline-first
await patientService.getPatients();
await patientService.createPatient(patientData);
await appointmentService.getAppointmentsByDate(date);
```

### For Users
1. Work normally when online
2. Continue working when offline - all changes saved locally
3. See offline indicator when connection is lost
4. Changes automatically sync when connection returns
5. Receive notifications about sync status

## API Contracts
All service methods maintain the same interface as before:
- `getPatients(): Promise<Patient[]>`
- `createPatient(data): Promise<Patient>`
- `updatePatient(id, data): Promise<Patient>`
- `deletePatient(id): Promise<void>`
- `getAppointments(): Promise<Appointment[]>`
- `createAppointment(data): Promise<Appointment>`
- etc.

## Future Enhancements
1. **Conflict Resolution Strategies**: Implement custom conflict resolution per entity type
2. **Selective Sync**: Allow users to choose what data to sync when on metered connections
3. **Progress Indicators**: Visual feedback for sync operations
4. **Schema Versioning**: Handle database schema migrations
5. **Encryption**: Encrypt sensitive data in local storage for security
6. **Cleanup Policies**: Automatic removal of old local data to prevent unlimited growth

## Files Modified/Added
```
offline/
├── db.ts              # Dexie.js database schema and instance
├── service.ts         # Offline-first service with sync capabilities
public/
├── sw.js              # Service worker for caching and background sync
├─manifest.json        # PWA manifest
├─offline.html         # Offline fallback page
└─icons/               # PWA icons (placeholders)
src/
├─services/
│  ├─patient-service.ts    # Enhanced with offline-first
│  └─appointment-service.ts # Enhanced with offline-first
├─layout.tsx           # Added PWA manifest and SW registration
└─components/          # Updated table/form components (no logic changes)
```

## Testing
To test the offline functionality:
1. Run the application normally
2. Open DevTools > Application > Service Workers to verify registration
3. Go offline (DevTools > Network > Offline checkbox)
4. Perform CRUD operations - they should succeed locally
5. Go back online - changes should sync to server
6. Verify data consistency between local and server stores

## Notes
- The service worker caches static assets using a cache-first strategy
- API requests use network-first strategy with fallback to cache
- Background sync triggers when network connectivity changes
- Periodic sync runs every 15 minutes when supported by browser
- All data operations are type-safe with TypeScript interfaces