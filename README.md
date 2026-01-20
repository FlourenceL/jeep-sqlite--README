# Ionic React + SQLite Setup Guide

A comprehensive guide to setting up an Ionic React project with SQLite using @capacitor-community/sqlite and jeep-sqlite for cross-platform local database storage.

## Prerequisites

- Node.js (v16 or higher)
- npm or yarn
- Ionic CLI (`npm install -g @ionic/cli`)
- Android Studio (for Android development)
- Xcode (for iOS development, macOS only)

## Step 1: Create a New Ionic React Project

```bash
ionic start my-sqlite-app blank --type=react --capacitor
cd my-sqlite-app
```

## Step 2: Install Required Dependencies

```bash
npm install @capacitor-community/sqlite
npm install react-sqlite-hook
npm install jeep-sqlite
```

## Step 3: Configure Capacitor

Sync the Capacitor configuration:

```bash
npx cap sync
```

## Step 4: Platform-Specific Setup

### For Web (Development)

Register the jeep-sqlite custom element in your `src/main.tsx` or `src/index.tsx`:

```typescript
import React from 'react';
import { createRoot } from 'react-dom/client';
import App from './App';
import { Capacitor } from '@capacitor/core';
import { JeepSqlite } from 'jeep-sqlite/dist/components/jeep-sqlite';

// Register jeep-sqlite web component for web platform
if (Capacitor.getPlatform() === 'web') {
  customElements.define('jeep-sqlite', JeepSqlite);
  window.addEventListener('DOMContentLoaded', async () => {
    const jeepEl = document.createElement('jeep-sqlite');
    document.body.appendChild(jeepEl);
    await customElements.whenDefined('jeep-sqlite');
  });
}

const container = document.getElementById('root');
const root = createRoot(container!);
root.render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

### For Android

1. Add the plugin to your Android project:

```bash
npx cap add android
npx cap sync android
```

2. No additional configuration needed - the plugin works out of the box.

### For iOS

1. Add the plugin to your iOS project:

```bash
npx cap add ios
npx cap sync ios
```

2. No additional configuration needed - the plugin works out of the box.

## Step 5: Create a SQLite Hook

Create a custom hook to manage SQLite operations. Create `src/hooks/useSQLite.tsx`:

```typescript
import { useEffect, useState } from 'react';
import { CapacitorSQLite, SQLiteConnection, SQLiteDBConnection } from '@capacitor-community/sqlite';
import { Capacitor } from '@capacitor/core';

export const useSQLite = () => {
  const [db, setDb] = useState<SQLiteDBConnection | null>(null);
  const [initialized, setInitialized] = useState(false);

  useEffect(() => {
    const initDB = async () => {
      try {
        const platform = Capacitor.getPlatform();
        const sqlite = new SQLiteConnection(CapacitorSQLite);

        // For web platform, check if jeep-sqlite is ready
        if (platform === 'web') {
          const jeepSqliteEl = document.querySelector('jeep-sqlite');
          if (jeepSqliteEl) {
            await customElements.whenDefined('jeep-sqlite');
            await sqlite.initWebStore();
          }
        }

        // Create or open database
        const dbName = 'myAppDB';
        const db = await sqlite.createConnection(
          dbName,
          false,
          'no-encryption',
          1,
          false
        );

        await db.open();

        // Create tables
        await db.execute(`
          CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            email TEXT UNIQUE NOT NULL,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP
          );
        `);

        setDb(db);
        setInitialized(true);
      } catch (error) {
        console.error('Error initializing database:', error);
      }
    };

    initDB();
  }, []);

  return { db, initialized };
};
```

## Step 6: Create Database Service

Create `src/services/DatabaseService.ts` for database operations:

```typescript
import { SQLiteDBConnection } from '@capacitor-community/sqlite';

export class DatabaseService {
  constructor(private db: SQLiteDBConnection) {}

  // Insert a user
  async addUser(name: string, email: string) {
    const query = `INSERT INTO users (name, email) VALUES (?, ?)`;
    const result = await this.db.run(query, [name, email]);
    return result;
  }

  // Get all users
  async getUsers() {
    const query = `SELECT * FROM users ORDER BY created_at DESC`;
    const result = await this.db.query(query);
    return result.values || [];
  }

  // Get user by ID
  async getUserById(id: number) {
    const query = `SELECT * FROM users WHERE id = ?`;
    const result = await this.db.query(query, [id]);
    return result.values?.[0] || null;
  }

  // Update user
  async updateUser(id: number, name: string, email: string) {
    const query = `UPDATE users SET name = ?, email = ? WHERE id = ?`;
    const result = await this.db.run(query, [name, email, id]);
    return result;
  }

  // Delete user
  async deleteUser(id: number) {
    const query = `DELETE FROM users WHERE id = ?`;
    const result = await this.db.run(query, [id]);
    return result;
  }
}
```

## Step 7: Example Usage in a Component

Update `src/pages/Home.tsx` to use the database:

```typescript
import React, { useEffect, useState } from 'react';
import {
  IonContent,
  IonHeader,
  IonPage,
  IonTitle,
  IonToolbar,
  IonButton,
  IonInput,
  IonItem,
  IonLabel,
  IonList,
} from '@ionic/react';
import { useSQLite } from '../hooks/useSQLite';
import { DatabaseService } from '../services/DatabaseService';

const Home: React.FC = () => {
  const { db, initialized } = useSQLite();
  const [users, setUsers] = useState<any[]>([]);
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');

  useEffect(() => {
    if (initialized && db) {
      loadUsers();
    }
  }, [initialized, db]);

  const loadUsers = async () => {
    if (!db) return;
    const dbService = new DatabaseService(db);
    const users = await dbService.getUsers();
    setUsers(users);
  };

  const handleAddUser = async () => {
    if (!db || !name || !email) return;
    const dbService = new DatabaseService(db);
    await dbService.addUser(name, email);
    setName('');
    setEmail('');
    loadUsers();
  };

  const handleDeleteUser = async (id: number) => {
    if (!db) return;
    const dbService = new DatabaseService(db);
    await dbService.deleteUser(id);
    loadUsers();
  };

  return (
    <IonPage>
      <IonHeader>
        <IonToolbar>
          <IonTitle>SQLite Demo</IonTitle>
        </IonToolbar>
      </IonHeader>
      <IonContent className="ion-padding">
        {!initialized ? (
          <p>Initializing database...</p>
        ) : (
          <>
            <IonItem>
              <IonLabel position="floating">Name</IonLabel>
              <IonInput
                value={name}
                onIonChange={(e) => setName(e.detail.value!)}
              />
            </IonItem>
            <IonItem>
              <IonLabel position="floating">Email</IonLabel>
              <IonInput
                value={email}
                onIonChange={(e) => setEmail(e.detail.value!)}
              />
            </IonItem>
            <IonButton expand="block" onClick={handleAddUser}>
              Add User
            </IonButton>

            <IonList>
              {users.map((user) => (
                <IonItem key={user.id}>
                  <IonLabel>
                    <h2>{user.name}</h2>
                    <p>{user.email}</p>
                  </IonLabel>
                  <IonButton
                    color="danger"
                    onClick={() => handleDeleteUser(user.id)}
                  >
                    Delete
                  </IonButton>
                </IonItem>
              ))}
            </IonList>
          </>
        )}
      </IonContent>
    </IonPage>
  );
};

export default Home;
```

## Step 8: Run Your Application

### Web (Development)

```bash
ionic serve
```

### Android

```bash
npx cap run android
```

### iOS

```bash
npx cap run ios
```

## Important Notes

- **Web Platform**: Uses IndexedDB under the hood via jeep-sqlite
- **Native Platforms**: Uses native SQLite databases
- **Data Persistence**: Data persists across app restarts on all platforms
- **Database Location**: 
  - Web: IndexedDB in the browser
  - Android: `/data/data/[app-id]/databases/`
  - iOS: `Library/Application Support/databases/`

## Common Issues and Solutions

### Issue: jeep-sqlite not defined
**Solution**: Ensure the custom element is registered before app initialization in `main.tsx`

### Issue: Database not persisting on web
**Solution**: Call `await sqlite.initWebStore()` before creating connections on web platform

### Issue: Database locked
**Solution**: Always close database connections properly when unmounting components

## Additional Resources

- [Capacitor SQLite Documentation](https://github.com/capacitor-community/sqlite)
- [jeep-sqlite Documentation](https://github.com/jepiqueau/jeep-sqlite)
- [Ionic React Documentation](https://ionicframework.com/docs/react)

## Next Steps

- Implement migrations for database schema changes
- Add TypeScript interfaces for type safety
- Create a context provider for global database access
- Implement data encryption for sensitive information
- Add database backup and restore functionality
