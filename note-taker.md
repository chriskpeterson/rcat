# Building a Tiered Note-Taking App with React Native, Firebase, and RevenueCat

A complete guide to creating a simple note-taking app with Free, Pro, and Premium subscription tiers using RevenueCat SDK.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Setting Up Your Development Environment](#setting-up-your-development-environment)
3. [Creating the React Native Project](#creating-the-react-native-project)
4. [Setting Up Firebase](#setting-up-firebase)
5. [Building the Basic Note-Taking App](#building-the-basic-note-taking-app)
6. [Setting Up RevenueCat](#setting-up-revenuecat)
7. [Configuring App Store Connect & Google Play Console](#configuring-app-store-connect--google-play-console)
8. [Integrating RevenueCat SDK](#integrating-revenuecat-sdk)
9. [Implementing Subscription Tiers](#implementing-subscription-tiers)
10. [Testing Your Implementation](#testing-your-implementation)
11. [Common Pitfalls and Solutions](#common-pitfalls-and-solutions)

---

## Prerequisites

Before starting, ensure you have:
- A Mac computer (for iOS development)
- An Apple Developer Account ($99/year)
- A Google Play Developer Account ($25 one-time)
- Basic knowledge of JavaScript and React
- A code editor (VS Code recommended)

---

## Setting Up Your Development Environment

### 1. Install Node.js

Download and install Node.js (LTS version) from [nodejs.org](https://nodejs.org)

Verify installation:
```bash
node --version
npm --version
```

### 2. Install Watchman (macOS)

```bash
brew install watchman
```

### 3. Install Xcode (for iOS)

1. Download Xcode from the Mac App Store
2. Install Command Line Tools:
```bash
xcode-select --install
```

### 4. Install CocoaPods

```bash
sudo gem install cocoapods
```

### 5. Set Up Android Studio (for Android)

1. Download Android Studio from [developer.android.com](https://developer.android.com/studio)
2. During installation, ensure these components are selected:
   - Android SDK
   - Android SDK Platform
   - Android Virtual Device
3. Configure environment variables in `~/.zshrc` or `~/.bash_profile`:

```bash
export ANDROID_HOME=$HOME/Library/Android/sdk
export PATH=$PATH:$ANDROID_HOME/emulator
export PATH=$PATH:$ANDROID_HOME/tools
export PATH=$PATH:$ANDROID_HOME/tools/bin
export PATH=$PATH:$ANDROID_HOME/platform-tools
```

Apply changes:
```bash
source ~/.zshrc
```

---

## Creating the React Native Project

### 1. Create New Project

```bash
npx react-native@latest init NotesApp
cd NotesApp
```

### 2. Test the Installation

For iOS:
```bash
npx react-native run-ios
```

For Android:
```bash
npx react-native run-android
```

---

## Setting Up Firebase

### 1. Create Firebase Project

1. Go to [Firebase Console](https://console.firebase.google.com)
2. Click "Add project"
3. Name it "NotesApp" and follow the setup wizard
4. Disable Google Analytics (optional for this tutorial)

### 2. Add iOS App to Firebase

1. In Firebase Console, click the iOS icon
2. Enter your iOS bundle ID (e.g., `com.yourname.notesapp`)
   - Find this in Xcode: Open `ios/NotesApp.xcworkspace`, select your project, see "Bundle Identifier"
3. Download `GoogleService-Info.plist`
4. Open Xcode: `open ios/NotesApp.xcworkspace`
5. Drag `GoogleService-Info.plist` into the project (make sure "Copy items if needed" is checked)

### 3. Add Android App to Firebase

1. In Firebase Console, click the Android icon
2. Enter your Android package name (e.g., `com.notesapp`)
   - Find this in `android/app/build.gradle` under `applicationId`
3. Download `google-services.json`
4. Place it in `android/app/`

### 4. Enable Firestore Database

1. In Firebase Console, go to "Firestore Database"
2. Click "Create database"
3. Start in test mode (for development)
4. Choose a location close to your users

### 5. Install Firebase Dependencies

```bash
npm install @react-native-firebase/app @react-native-firebase/firestore @react-native-firebase/auth
```

### 6. Configure iOS

Add to `ios/Podfile` (before `use_react_native!`):

```ruby
$RNFirebaseAsStaticFramework = true
```

Install pods:
```bash
cd ios
pod install
cd ..
```

### 7. Configure Android

In `android/build.gradle`, add:

```gradle
buildscript {
  dependencies {
    // ... other dependencies
    classpath 'com.google.gms:google-services:4.4.0'
  }
}
```

In `android/app/build.gradle`, add at the bottom:

```gradle
apply plugin: 'com.google.gms.google-services'
```

---

## Building the Basic Note-Taking App

### 1. Project Structure

Create the following folder structure:

```
NotesApp/
├── src/
│   ├── components/
│   │   ├── NotesList.js
│   │   ├── NoteEditor.js
│   │   └── PaywallModal.js
│   ├── screens/
│   │   ├── HomeScreen.js
│   │   ├── EditorScreen.js
│   │   └── SettingsScreen.js
│   ├── services/
│   │   ├── firebase.js
│   │   └── revenuecat.js
│   ├── context/
│   │   └── SubscriptionContext.js
│   └── utils/
│       └── constants.js
```

### 2. Install Additional Dependencies

```bash
npm install @react-navigation/native @react-navigation/stack react-native-gesture-handler react-native-reanimated react-native-screens react-native-safe-area-context @react-native-community/masked-view react-native-vector-icons
```

For iOS:
```bash
cd ios && pod install && cd ..
```

### 3. Create Firebase Service

Create `src/services/firebase.js`:

```javascript
import firestore from '@react-native-firebase/firestore';
import auth from '@react-native-firebase/auth';

class FirebaseService {
  constructor() {
    this.notesCollection = firestore().collection('notes');
  }

  async signInAnonymously() {
    try {
      await auth().signInAnonymously();
      return auth().currentUser;
    } catch (error) {
      console.error('Anonymous sign-in error:', error);
      throw error;
    }
  }

  getCurrentUser() {
    return auth().currentUser;
  }

  async createNote(title, content) {
    const user = this.getCurrentUser();
    if (!user) throw new Error('No user signed in');

    return await this.notesCollection.add({
      title,
      content,
      userId: user.uid,
      createdAt: firestore.FieldValue.serverTimestamp(),
      updatedAt: firestore.FieldValue.serverTimestamp(),
    });
  }

  async updateNote(noteId, title, content) {
    return await this.notesCollection.doc(noteId).update({
      title,
      content,
      updatedAt: firestore.FieldValue.serverTimestamp(),
    });
  }

  async deleteNote(noteId) {
    return await this.notesCollection.doc(noteId).delete();
  }

  getUserNotes() {
    const user = this.getCurrentUser();
    if (!user) throw new Error('No user signed in');

    return this.notesCollection
      .where('userId', '==', user.uid)
      .orderBy('updatedAt', 'desc');
  }
}

export default new FirebaseService();
```

### 4. Create Constants

Create `src/utils/constants.js`:

```javascript
export const SUBSCRIPTION_TIERS = {
  FREE: 'free',
  PRO: 'pro',
  PREMIUM: 'premium',
};

export const TIER_LIMITS = {
  [SUBSCRIPTION_TIERS.FREE]: {
    maxNotes: 10,
    features: ['Basic notes', 'Text formatting'],
  },
  [SUBSCRIPTION_TIERS.PRO]: {
    maxNotes: 100,
    features: ['Basic notes', 'Text formatting', 'Image attachments', 'Cloud sync'],
  },
  [SUBSCRIPTION_TIERS.PREMIUM]: {
    maxNotes: Infinity,
    features: ['All Pro features', 'Voice notes', 'Collaboration', 'Priority support'],
  },
};
```

### 5. Create Home Screen

Create `src/screens/HomeScreen.js`:

```javascript
import React, { useEffect, useState } from 'react';
import {
  View,
  FlatList,
  TouchableOpacity,
  Text,
  StyleSheet,
  Alert,
} from 'react-native';
import FirebaseService from '../services/firebase';
import { SUBSCRIPTION_TIERS, TIER_LIMITS } from '../utils/constants';

const HomeScreen = ({ navigation }) => {
  const [notes, setNotes] = useState([]);
  const [currentTier, setCurrentTier] = useState(SUBSCRIPTION_TIERS.FREE);

  useEffect(() => {
    const signIn = async () => {
      await FirebaseService.signInAnonymously();
    };
    signIn();
  }, []);

  useEffect(() => {
    const unsubscribe = FirebaseService.getUserNotes().onSnapshot(
      (snapshot) => {
        const notesData = snapshot.docs.map((doc) => ({
          id: doc.id,
          ...doc.data(),
        }));
        setNotes(notesData);
      },
      (error) => {
        console.error('Error fetching notes:', error);
      }
    );

    return () => unsubscribe();
  }, []);

  const handleAddNote = () => {
    const limit = TIER_LIMITS[currentTier].maxNotes;
    
    if (notes.length >= limit) {
      Alert.alert(
        'Note Limit Reached',
        `You've reached the maximum of ${limit} notes for your ${currentTier} plan. Upgrade to create more notes!`,
        [
          { text: 'Cancel', style: 'cancel' },
          { text: 'Upgrade', onPress: () => navigation.navigate('Settings') },
        ]
      );
      return;
    }

    navigation.navigate('Editor', { mode: 'create' });
  };

  const handleEditNote = (note) => {
    navigation.navigate('Editor', { mode: 'edit', note });
  };

  const renderNote = ({ item }) => (
    <TouchableOpacity
      style={styles.noteItem}
      onPress={() => handleEditNote(item)}
    >
      <Text style={styles.noteTitle}>{item.title || 'Untitled'}</Text>
      <Text style={styles.notePreview} numberOfLines={2}>
        {item.content}
      </Text>
    </TouchableOpacity>
  );

  return (
    <View style={styles.container}>
      <View style={styles.header}>
        <Text style={styles.headerTitle}>My Notes</Text>
        <Text style={styles.tierBadge}>{currentTier.toUpperCase()}</Text>
      </View>
      
      <Text style={styles.noteCount}>
        {notes.length} / {TIER_LIMITS[currentTier].maxNotes === Infinity ? '∞' : TIER_LIMITS[currentTier].maxNotes} notes
      </Text>

      <FlatList
        data={notes}
        renderItem={renderNote}
        keyExtractor={(item) => item.id}
        contentContainerStyle={styles.listContent}
      />

      <TouchableOpacity style={styles.addButton} onPress={handleAddNote}>
        <Text style={styles.addButtonText}>+ New Note</Text>
      </TouchableOpacity>
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#f5f5f5',
  },
  header: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    padding: 16,
    backgroundColor: '#fff',
    borderBottomWidth: 1,
    borderBottomColor: '#e0e0e0',
  },
  headerTitle: {
    fontSize: 24,
    fontWeight: 'bold',
  },
  tierBadge: {
    backgroundColor: '#007AFF',
    color: '#fff',
    paddingHorizontal: 12,
    paddingVertical: 4,
    borderRadius: 12,
    fontSize: 12,
    fontWeight: 'bold',
  },
  noteCount: {
    padding: 16,
    color: '#666',
  },
  listContent: {
    padding: 16,
  },
  noteItem: {
    backgroundColor: '#fff',
    padding: 16,
    marginBottom: 12,
    borderRadius: 8,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 1 },
    shadowOpacity: 0.1,
    shadowRadius: 2,
    elevation: 2,
  },
  noteTitle: {
    fontSize: 18,
    fontWeight: '600',
    marginBottom: 8,
  },
  notePreview: {
    fontSize: 14,
    color: '#666',
  },
  addButton: {
    position: 'absolute',
    bottom: 24,
    right: 24,
    backgroundColor: '#007AFF',
    paddingHorizontal: 24,
    paddingVertical: 16,
    borderRadius: 30,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.25,
    shadowRadius: 4,
    elevation: 5,
  },
  addButtonText: {
    color: '#fff',
    fontSize: 16,
    fontWeight: '600',
  },
});

export default HomeScreen;
```

### 6. Create Editor Screen

Create `src/screens/EditorScreen.js`:

```javascript
import React, { useState } from 'react';
import {
  View,
  TextInput,
  StyleSheet,
  TouchableOpacity,
  Text,
  Alert,
} from 'react-native';
import FirebaseService from '../services/firebase';

const EditorScreen = ({ route, navigation }) => {
  const { mode, note } = route.params;
  const [title, setTitle] = useState(note?.title || '');
  const [content, setContent] = useState(note?.content || '');

  const handleSave = async () => {
    try {
      if (mode === 'create') {
        await FirebaseService.createNote(title, content);
      } else {
        await FirebaseService.updateNote(note.id, title, content);
      }
      navigation.goBack();
    } catch (error) {
      Alert.alert('Error', 'Failed to save note');
      console.error('Save error:', error);
    }
  };

  const handleDelete = async () => {
    Alert.alert(
      'Delete Note',
      'Are you sure you want to delete this note?',
      [
        { text: 'Cancel', style: 'cancel' },
        {
          text: 'Delete',
          style: 'destructive',
          onPress: async () => {
            try {
              await FirebaseService.deleteNote(note.id);
              navigation.goBack();
            } catch (error) {
              Alert.alert('Error', 'Failed to delete note');
            }
          },
        },
      ]
    );
  };

  return (
    <View style={styles.container}>
      <View style={styles.header}>
        <TouchableOpacity onPress={() => navigation.goBack()}>
          <Text style={styles.cancelButton}>Cancel</Text>
        </TouchableOpacity>
        <TouchableOpacity onPress={handleSave}>
          <Text style={styles.saveButton}>Save</Text>
        </TouchableOpacity>
      </View>

      <TextInput
        style={styles.titleInput}
        placeholder="Title"
        value={title}
        onChangeText={setTitle}
        placeholderTextColor="#999"
      />

      <TextInput
        style={styles.contentInput}
        placeholder="Start typing..."
        value={content}
        onChangeText={setContent}
        multiline
        textAlignVertical="top"
        placeholderTextColor="#999"
      />

      {mode === 'edit' && (
        <TouchableOpacity style={styles.deleteButton} onPress={handleDelete}>
          <Text style={styles.deleteButtonText}>Delete Note</Text>
        </TouchableOpacity>
      )}
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#fff',
  },
  header: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    padding: 16,
    borderBottomWidth: 1,
    borderBottomColor: '#e0e0e0',
  },
  cancelButton: {
    color: '#007AFF',
    fontSize: 16,
  },
  saveButton: {
    color: '#007AFF',
    fontSize: 16,
    fontWeight: '600',
  },
  titleInput: {
    fontSize: 24,
    fontWeight: 'bold',
    padding: 16,
    borderBottomWidth: 1,
    borderBottomColor: '#e0e0e0',
  },
  contentInput: {
    flex: 1,
    fontSize: 16,
    padding: 16,
  },
  deleteButton: {
    margin: 16,
    padding: 16,
    backgroundColor: '#FF3B30',
    borderRadius: 8,
    alignItems: 'center',
  },
  deleteButtonText: {
    color: '#fff',
    fontSize: 16,
    fontWeight: '600',
  },
});

export default EditorScreen;
```

---

## Setting Up RevenueCat

### 1. Create RevenueCat Account

1. Go to [RevenueCat](https://www.revenuecat.com) and create a free account
2. Click "Create new app"
3. Enter your app name: "NotesApp"

### 2. Configure App in RevenueCat Dashboard

**For iOS:**
1. Click "iOS" in your project
2. Enter your Bundle ID (from Xcode)
3. Generate an App-Specific Shared Secret:
   - Go to [App Store Connect](https://appstoreconnect.apple.com)
   - Select your app
   - Go to "App Information" → "App-Specific Shared Secret"
   - Generate and copy the secret
4. Paste the shared secret in RevenueCat

**For Android:**
1. Click "Android" in your project
2. Enter your package name
3. Set up Google Play Service Credentials:
   - Go to [Google Play Console](https://play.google.com/console)
   - Go to "Setup" → "API access"
   - Create a service account and download the JSON key
4. Upload the JSON file to RevenueCat

### 3. Create Products in RevenueCat

1. Go to "Products" in RevenueCat dashboard
2. Create the following products:

**Pro Monthly:**
- Identifier: `pro_monthly`
- Type: Subscription

**Pro Yearly:**
- Identifier: `pro_yearly`
- Type: Subscription

**Premium Monthly:**
- Identifier: `premium_monthly`
- Type: Subscription

**Premium Yearly:**
- Identifier: `premium_yearly`
- Type: Subscription

### 4. Create Entitlements

Entitlements represent features users get access to.

1. Go to "Entitlements" in RevenueCat
2. Create two entitlements:

**pro:**
- Attach products: `pro_monthly`, `pro_yearly`

**premium:**
- Attach products: `premium_monthly`, `premium_yearly`

### 5. Create Offerings

Offerings are how you present products to users.

1. Go to "Offerings"
2. Create "Default Offering"
3. Add packages:
   - **Pro Package**: Add `pro_monthly` and `pro_yearly`
   - **Premium Package**: Add `premium_monthly` and `premium_yearly`

---

## Configuring App Store Connect & Google Play Console

### App Store Connect (iOS)

1. Go to [App Store Connect](https://appstoreconnect.apple.com)
2. Click "My Apps" → "+"  → "New App"
3. Fill in app information
4. Go to "Features" → "In-App Purchases"
5. Create Auto-Renewable Subscriptions:

**Pro Monthly:**
- Product ID: `pro_monthly`
- Reference Name: Pro Monthly
- Subscription Duration: 1 month
- Price: $4.99

**Pro Yearly:**
- Product ID: `pro_yearly`
- Reference Name: Pro Yearly
- Subscription Duration: 1 year
- Price: $49.99

**Premium Monthly:**
- Product ID: `premium_monthly`
- Reference Name: Premium Monthly
- Subscription Duration: 1 month
- Price: $9.99

**Premium Yearly:**
- Product ID: `premium_yearly`
- Reference Name: Premium Yearly
- Subscription Duration: 1 year
- Price: $99.99

6. Create a Subscription Group (e.g., "NotesApp Subscriptions")
7. Add all subscriptions to this group
8. Set up subscription levels (Premium > Pro)

### Google Play Console (Android)

1. Go to [Google Play Console](https://play.google.com/console)
2. Create a new app
3. Go to "Monetize" → "Subscriptions"
4. Create the same subscription products:
   - `pro_monthly` - $4.99/month
   - `pro_yearly` - $49.99/year
   - `premium_monthly` - $9.99/month
   - `premium_yearly` - $99.99/year

5. Create a subscription group and add all subscriptions

**⚠️ CRITICAL PITFALL #1:**
Product IDs in App Store Connect and Google Play Console MUST exactly match the identifiers you created in RevenueCat. Case-sensitive!

---

## Integrating RevenueCat SDK

### 1. Install RevenueCat SDK

```bash
npm install react-native-purchases
```

For iOS:
```bash
cd ios && pod install && cd ..
```

### 2. Get API Keys

1. In RevenueCat dashboard, go to your project
2. Go to "API keys" in settings
3. Copy your **Public API Key** for iOS
4. Copy your **Public API Key** for Android

**⚠️ CRITICAL PITFALL #2:**
Use the PUBLIC API key, NOT the secret key. The secret key should never be in your app code.

### 3. Create RevenueCat Service

Create `src/services/revenuecat.js`:

```javascript
import Purchases from 'react-native-purchases';
import { Platform } from 'react-native';

const API_KEYS = {
  ios: 'appl_YOUR_IOS_API_KEY',
  android: 'goog_YOUR_ANDROID_API_KEY',
};

class RevenueCatService {
  async initialize(userId) {
    try {
      const apiKey = Platform.select({
        ios: API_KEYS.ios,
        android: API_KEYS.android,
      });

      await Purchases.configure({ apiKey, appUserID: userId });
      
      console.log('RevenueCat initialized successfully');
    } catch (error) {
      console.error('RevenueCat initialization error:', error);
      throw error;
    }
  }

  async getOfferings() {
    try {
      const offerings = await Purchases.getOfferings();
      return offerings;
    } catch (error) {
      console.error('Error fetching offerings:', error);
      throw error;
    }
  }

  async purchasePackage(packageToPurchase) {
    try {
      const { customerInfo } = await Purchases.purchasePackage(packageToPurchase);
      return customerInfo;
    } catch (error) {
      if (error.userCancelled) {
        console.log('User cancelled purchase');
        return null;
      }
      console.error('Purchase error:', error);
      throw error;
    }
  }

  async restorePurchases() {
    try {
      const customerInfo = await Purchases.restorePurchases();
      return customerInfo;
    } catch (error) {
      console.error('Restore purchases error:', error);
      throw error;
    }
  }

  async getCustomerInfo() {
    try {
      const customerInfo = await Purchases.getCustomerInfo();
      return customerInfo;
    } catch (error) {
      console.error('Error getting customer info:', error);
      throw error;
    }
  }

  checkEntitlement(customerInfo, entitlementId) {
    return (
      customerInfo?.entitlements?.active?.[entitlementId]?.isActive === true
    );
  }

  getCurrentTier(customerInfo) {
    if (this.checkEntitlement(customerInfo, 'premium')) {
      return 'premium';
    }
    if (this.checkEntitlement(customerInfo, 'pro')) {
      return 'pro';
    }
    return 'free';
  }
}

export default new RevenueCatService();
```

**⚠️ CRITICAL PITFALL #3:**
Replace `'appl_YOUR_IOS_API_KEY'` and `'goog_YOUR_ANDROID_API_KEY'` with your actual API keys from RevenueCat dashboard.

### 4. Create Subscription Context

Create `src/context/SubscriptionContext.js`:

```javascript
import React, { createContext, useContext, useState, useEffect } from 'react';
import RevenueCatService from '../services/revenuecat';
import FirebaseService from '../services/firebase';
import { SUBSCRIPTION_TIERS } from '../utils/constants';

const SubscriptionContext = createContext();

export const useSubscription = () => {
  const context = useContext(SubscriptionContext);
  if (!context) {
    throw new Error('useSubscription must be used within SubscriptionProvider');
  }
  return context;
};

export const SubscriptionProvider = ({ children }) => {
  const [currentTier, setCurrentTier] = useState(SUBSCRIPTION_TIERS.FREE);
  const [customerInfo, setCustomerInfo] = useState(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    initializeRevenueCat();
  }, []);

  const initializeRevenueCat = async () => {
    try {
      const user = FirebaseService.getCurrentUser();
      if (!user) {
        console.log('No user signed in, skipping RevenueCat init');
        setIsLoading(false);
        return;
      }

      await RevenueCatService.initialize(user.uid);
      await updateCustomerInfo();
    } catch (error) {
      console.error('Failed to initialize RevenueCat:', error);
      setIsLoading(false);
    }
  };

  const updateCustomerInfo = async () => {
    try {
      const info = await RevenueCatService.getCustomerInfo();
      setCustomerInfo(info);
      const tier = RevenueCatService.getCurrentTier(info);
      setCurrentTier(tier);
      setIsLoading(false);
    } catch (error) {
      console.error('Failed to get customer info:', error);
      setIsLoading(false);
    }
  };

  const purchasePackage = async (pkg) => {
    try {
      const info = await RevenueCatService.purchasePackage(pkg);
      if (info) {
        await updateCustomerInfo();
        return true;
      }
      return false;
    } catch (error) {
      console.error('Purchase failed:', error);
      throw error;
    }
  };

  const restorePurchases = async () => {
    try {
      await RevenueCatService.restorePurchases();
      await updateCustomerInfo();
    } catch (error) {
      console.error('Restore failed:', error);
      throw error;
    }
  };

  const value = {
    currentTier,
    customerInfo,
    isLoading,
    purchasePackage,
    restorePurchases,
    updateCustomerInfo,
  };

  return (
    <SubscriptionContext.Provider value={value}>
      {children}
    </SubscriptionContext.Provider>
  );
};
```

### 5. Create Paywall Modal

Create `src/components/PaywallModal.js`:

```javascript
import React, { useState, useEffect } from 'react';
import {
  View,
  Text,
  Modal,
  StyleSheet,
  TouchableOpacity,
  ScrollView,
  ActivityIndicator,
  Alert,
} from 'react-native';
import RevenueCatService from '../services/revenuecat';
import { useSubscription } from '../context/SubscriptionContext';
import { TIER_LIMITS } from '../utils/constants';

const PaywallModal = ({ visible, onClose }) => {
  const [offerings, setOfferings] = useState(null);
  const [loading, setLoading] = useState(true);
  const [purchasing, setPurchasing] = useState(false);
  const { purchasePackage } = useSubscription();

  useEffect(() => {
    if (visible) {
      loadOfferings();
    }
  }, [visible]);

  const loadOfferings = async () => {
    try {
      setLoading(true);
      const offers = await RevenueCatService.getOfferings();
      setOfferings(offers);
    } catch (error) {
      Alert.alert('Error', 'Failed to load subscription options');
    } finally {
      setLoading(false);
    }
  };

  const handlePurchase = async (pkg) => {
    try {
      setPurchasing(true);
      const success = await purchasePackage(pkg);
      if (success) {
        Alert.alert('Success', 'Subscription activated!');
        onClose();
      }
    } catch (error) {
      Alert.alert('Purchase Failed', error.message);
    } finally {
      setPurchasing(false);
    }
  };

  const renderPackage = (pkg, tierName, tierLimits) => {
    const product = pkg.product;
    const priceString = product.priceString;
    const period = product.subscriptionPeriod;

    return (
      <View key={pkg.identifier} style={styles.packageContainer}>
        <View style={styles.packageHeader}>
          <Text style={styles.tierName}>{tierName}</Text>
          <Text style={styles.price}>{priceString}</Text>
          <Text style={styles.period}>/{period}</Text>
        </View>

        <View style={styles.featuresContainer}>
          {tierLimits.features.map((feature, index) => (
            <Text key={index} style={styles.feature}>
              ✓ {feature}
            </Text>
          ))}
          <Text style={styles.feature}>
            ✓ Up to {tierLimits.maxNotes === Infinity ? 'unlimited' : tierLimits.maxNotes} notes
          </Text>
        </View>

        <TouchableOpacity
          style={styles.purchaseButton}
          onPress={() => handlePurchase(pkg)}
          disabled={purchasing}
        >
          <Text style={styles.purchaseButtonText}>
            {purchasing ? 'Processing...' : 'Subscribe'}
          </Text>
        </TouchableOpacity>
      </View>
    );
  };

  if (loading) {
    return (
      <Modal visible={visible} animationType="slide" transparent>
        <View style={styles.modalContainer}>
          <View style={styles.contentContainer}>
            <ActivityIndicator size="large" color="#007AFF" />
          </View>
        </View>
      </Modal>
    );
  }

  const currentOffering = offerings?.current;
  const proPackage = currentOffering?.availablePackages?.find(
    (pkg) => pkg.product.identifier.includes('pro')
  );
  const premiumPackage = currentOffering?.availablePackages?.find(
    (pkg) => pkg.product.identifier.includes('premium')
  );

  return (
    <Modal visible={visible} animationType="slide" transparent>
      <View style={styles.modalContainer}>
        <View style={styles.contentContainer}>
          <View style={styles.header}>
            <Text style={styles.title}>Upgrade Your Plan</Text>
            <TouchableOpacity onPress={onClose}>
              <Text style={styles.closeButton}>✕</Text>
            </TouchableOpacity>
          </View>

          <ScrollView style={styles.scrollView}>
            {proPackage && renderPackage(proPackage, 'Pro', TIER_LIMITS.pro)}
            {premiumPackage && renderPackage(premiumPackage, 'Premium', TIER_LIMITS.premium)}
          </ScrollView>

          <Text style={styles.disclaimer}>
            Subscriptions auto-renew. Cancel anytime.
          </Text>
        </View>
      </View>
    </Modal>
  );
};

const styles = StyleSheet.create({
  modalContainer: {
    flex: 1,
    backgroundColor: 'rgba(0, 0, 0, 0.5)',
    justifyContent: 'flex-end',
  },
  contentContainer: {
    backgroundColor: '#fff',
    borderTopLeftRadius: 20,
    borderTopRightRadius: 20,
    maxHeight: '90%',
  },
  header: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    padding: 20,
    borderBottomWidth: 1,
    borderBottomColor: '#e0e0e0',
  },
  title: {
    fontSize: 24,
    fontWeight: 'bold',
  },
  closeButton: {
    fontSize: 28,
    color: '#666',
  },
  scrollView: {
    padding: 20,
  },
  packageContainer: {
    backgroundColor: '#f9f9f9',
    borderRadius: 12,
    padding: 20,
    marginBottom: 16,
    borderWidth: 2,
    borderColor: '#007AFF',
  },
  packageHeader: {
    marginBottom: 16,
  },
  tierName: {
    fontSize: 22,
    fontWeight: 'bold',
    marginBottom: 8,
  },
  price: {
    fontSize: 32,
    fontWeight: 'bold',
    color: '#007AFF',
  },
  period: {
    fontSize: 16,
    color: '#666',
  },
  featuresContainer: {
    marginBottom: 16,
  },
  feature: {
    fontSize: 16,
    marginBottom: 8,
    color: '#333',
  },
  purchaseButton: {
    backgroundColor: '#007AFF',
    padding: 16,
    borderRadius: 8,
    alignItems: 'center',
  },
  purchaseButtonText: {
    color: '#fff',
    fontSize: 18,
    fontWeight: '600',
  },
  disclaimer: {
    textAlign: 'center',
    color: '#666',
    fontSize: 12,
    padding: 16,
  },
});

export default PaywallModal;
```

**⚠️ CRITICAL PITFALL #4:**
The package finding logic (`pkg.product.identifier.includes('pro')`) assumes your product IDs contain 'pro' or 'premium'. Adjust this logic to match your actual product IDs.

### 6. Update App.js

Replace your `App.js`:

```javascript
import React, { useEffect, useState } from 'react';
import { NavigationContainer } from '@react-navigation/native';
import { createStackNavigator } from '@react-navigation/stack';
import HomeScreen from './src/screens/HomeScreen';
import EditorScreen from './src/screens/EditorScreen';
import SettingsScreen from './src/screens/SettingsScreen';
import FirebaseService from './src/services/firebase';
import { SubscriptionProvider } from './src/context/SubscriptionContext';
import { ActivityIndicator, View } from 'react-native';

const Stack = createStackNavigator();

function App() {
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const initialize = async () => {
      try {
        await FirebaseService.signInAnonymously();
      } catch (error) {
        console.error('Initialization error:', error);
      } finally {
        setLoading(false);
      }
    };

    initialize();
  }, []);

  if (loading) {
    return (
      <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
        <ActivityIndicator size="large" />
      </View>
    );
  }

  return (
    <SubscriptionProvider>
      <NavigationContainer>
        <Stack.Navigator>
          <Stack.Screen
            name="Home"
            component={HomeScreen}
            options={{ headerShown: false }}
          />
          <Stack.Screen
            name="Editor"
            component={EditorScreen}
            options={{ headerShown: false }}
          />
          <Stack.Screen
            name="Settings"
            component={SettingsScreen}
            options={{ title: 'Settings' }}
          />
        </Stack.Navigator>
      </NavigationContainer>
    </SubscriptionProvider>
  );
}

export default App;
```

### 7. Create Settings Screen

Create `src/screens/SettingsScreen.js`:

```javascript
import React, { useState } from 'react';
import {
  View,
  Text,
  StyleSheet,
  TouchableOpacity,
  Alert,
} from 'react-native';
import { useSubscription } from '../context/SubscriptionContext';
import PaywallModal from '../components/PaywallModal';
import { TIER_LIMITS } from '../utils/constants';

const SettingsScreen = () => {
  const [paywallVisible, setPaywallVisible] = useState(false);
  const { currentTier, restorePurchases } = useSubscription();

  const handleRestore = async () => {
    try {
      await restorePurchases();
      Alert.alert('Success', 'Purchases restored successfully');
    } catch (error) {
      Alert.alert('Error', 'Failed to restore purchases');
    }
  };

  const currentLimits = TIER_LIMITS[currentTier];

  return (
    <View style={styles.container}>
      <View style={styles.section}>
        <Text style={styles.sectionTitle}>Current Plan</Text>
        <View style={styles.planCard}>
          <Text style={styles.planName}>{currentTier.toUpperCase()}</Text>
          <Text style={styles.planDetail}>
            {currentLimits.maxNotes === Infinity ? 'Unlimited' : currentLimits.maxNotes} notes
          </Text>
          {currentLimits.features.map((feature, index) => (
            <Text key={index} style={styles.feature}>
              ✓ {feature}
            </Text>
          ))}
        </View>
      </View>

      {currentTier !== 'premium' && (
        <TouchableOpacity
          style={styles.upgradeButton}
          onPress={() => setPaywallVisible(true)}
        >
          <Text style={styles.upgradeButtonText}>
            {currentTier === 'free' ? 'Upgrade to Pro or Premium' : 'Upgrade to Premium'}
          </Text>
        </TouchableOpacity>
      )}

      <TouchableOpacity style={styles.restoreButton} onPress={handleRestore}>
        <Text style={styles.restoreButtonText}>Restore Purchases</Text>
      </TouchableOpacity>

      <PaywallModal
        visible={paywallVisible}
        onClose={() => setPaywallVisible(false)}
      />
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#f5f5f5',
    padding: 16,
  },
  section: {
    marginBottom: 24,
  },
  sectionTitle: {
    fontSize: 18,
    fontWeight: '600',
    marginBottom: 12,
    color: '#333',
  },
  planCard: {
    backgroundColor: '#fff',
    padding: 20,
    borderRadius: 12,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
    elevation: 3,
  },
  planName: {
    fontSize: 24,
    fontWeight: 'bold',
    marginBottom: 8,
    color: '#007AFF',
  },
  planDetail: {
    fontSize: 16,
    color: '#666',
    marginBottom: 12,
  },
  feature: {
    fontSize: 14,
    color: '#333',
    marginBottom: 6,
  },
  upgradeButton: {
    backgroundColor: '#007AFF',
    padding: 16,
    borderRadius: 8,
    alignItems: 'center',
    marginBottom: 12,
  },
  upgradeButtonText: {
    color: '#fff',
    fontSize: 16,
    fontWeight: '600',
  },
  restoreButton: {
    backgroundColor: '#fff',
    padding: 16,
    borderRadius: 8,
    alignItems: 'center',
    borderWidth: 1,
    borderColor: '#007AFF',
  },
  restoreButtonText: {
    color: '#007AFF',
    fontSize: 16,
    fontWeight: '600',
  },
});

export default SettingsScreen;
```

---

## Implementing Subscription Tiers

### Update HomeScreen to Use Subscription Context

Update `src/screens/HomeScreen.js` to use the subscription context:

```javascript
// Add this import at the top
import { useSubscription } from '../context/SubscriptionContext';

// Inside the component, replace the currentTier state with:
const { currentTier } = useSubscription();

// Remove this line:
// const [currentTier, setCurrentTier] = useState(SUBSCRIPTION_TIERS.FREE);
```

**⚠️ CRITICAL PITFALL #5:**
Make sure to initialize RevenueCat AFTER Firebase authentication completes. RevenueCat needs a user ID to work properly. If you initialize before auth, it won't work correctly.

---

## Testing Your Implementation

### 1. iOS Sandbox Testing

**Create Sandbox Test Account:**
1. Go to [App Store Connect](https://appstoreconnect.apple.com)
2. Go to "Users and Access" → "Sandbox Testers"
3. Click "+" to add a tester
4. Create a unique email (e.g., `test1@example.com`)
5. Set password and other details

**Test on Device:**
1. Sign out of App Store on your test device
2. Build and run your app:
```bash
npx react-native run-ios
```
3. Trigger a purchase in your app
4. When prompted, sign in with your sandbox account
5. Complete the test purchase (it's free in sandbox)

**⚠️ CRITICAL PITFALL #6:**
You CANNOT test subscriptions in the iOS Simulator. You must use a real device for testing RevenueCat subscriptions.

### 2. Android Testing

**Set Up Test License:**
1. In Google Play Console, go to "Setup" → "License testing"
2. Add your test Google account email
3. Set response to "Licensed"

**Create Internal Test Track:**
1. Go to "Testing" → "Internal testing"
2. Create a new release
3. Upload your APK/AAB
4. Add yourself as a tester

**Test on Device:**
```bash
npx react-native run-android
```

**⚠️ CRITICAL PITFALL #7:**
For Android, you must upload at least one version to Internal Testing in Google Play Console before in-app purchases will work, even in testing.

### 3. Verify Integration in RevenueCat

After making test purchases:
1. Go to RevenueCat dashboard
2. Click "Customers"
3. Search for your test user ID (Firebase UID)
4. Verify that:
   - The customer appears
   - Active entitlements are shown
   - Purchase history is recorded

### 4. Test Scenarios

Test these scenarios thoroughly:

**Free Tier:**
- [ ] Can create notes up to limit (10)
- [ ] Gets paywall when exceeding limit
- [ ] Can view/edit existing notes

**Purchase Flow:**
- [ ] Paywall displays correctly
- [ ] Can select different tiers
- [ ] Purchase completes successfully
- [ ] Entitlements update immediately
- [ ] Note limit increases after purchase

**Pro Tier:**
- [ ] Can create up to 100 notes
- [ ] All Pro features accessible

**Premium Tier:**
- [ ] Unlimited note creation
- [ ] All Premium features accessible

**Restore Purchases:**
- [ ] Delete and reinstall app
- [ ] Sign in with same Firebase user
- [ ] Click "Restore Purchases"
- [ ] Subscriptions restore correctly

**Subscription Management:**
- [ ] Can cancel subscription (in App Store/Play Store)
- [ ] App correctly handles expired subscriptions
- [ ] App correctly handles subscription downgrades

---

## Common Pitfalls and Solutions

### Pitfall #1: Product ID Mismatches
**Problem:** RevenueCat can't find products, purchases fail
**Solution:** Ensure product IDs match EXACTLY (case-sensitive) across:
- App Store Connect / Google Play Console
- RevenueCat dashboard
- Your code

### Pitfall #2: Using Secret API Key
**Problem:** Security vulnerability, app may be rejected
**Solution:** Always use PUBLIC API keys in your app, never secret keys

### Pitfall #3: Not Waiting for Contracts
**Problem:** "Agreements not signed" errors
**Solution:** In App Store Connect, ensure all agreements are signed under "Agreements, Tax, and Banking"

### Pitfall #4: Testing on Simulator (iOS)
**Problem:** Purchases fail silently
**Solution:** Always test on real devices. Simulator doesn't support StoreKit properly

### Pitfall #5: Missing Shared Secret (iOS)
**Problem:** Purchase validation fails
**Solution:** Generate and add App-Specific Shared Secret in RevenueCat dashboard

### Pitfall #6: Google Play Service Account Issues
**Problem:** Android purchases don't validate
**Solution:** 
- Ensure service account has correct permissions
- Wait 24-48 hours after creating service account
- Verify JSON key is uploaded correctly to RevenueCat

### Pitfall #7: Subscription Status Not Updating
**Problem:** User purchases but tier doesn't change
**Solution:** 
- Call `getCustomerInfo()` after successful purchase
- Implement proper error handling
- Check RevenueCat dashboard for webhook deliveries

### Pitfall #8: Revenue Cat Initialization Before Auth
**Problem:** RevenueCat can't identify user, subscriptions don't persist
**Solution:** Always initialize RevenueCat AFTER Firebase authentication completes

### Pitfall #9: Firestore Security Rules
**Problem:** Users can't read/write notes
**Solution:** Update Firestore rules:
```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /notes/{noteId} {
      allow read, write: if request.auth != null && 
                           request.auth.uid == resource.data.userId;
      allow create: if request.auth != null && 
                       request.auth.uid == request.resource.data.userId;
    }
  }
}
```

### Pitfall #10: App Not Configured for In-App Purchases
**Problem:** "In-App Purchase Not Available" error
**Solution:**
- iOS: Ensure Capability "In-App Purchase" is enabled in Xcode
- Android: Ensure proper permissions in AndroidManifest.xml

### Pitfall #11: Subscription Groups Not Set Up
**Problem:** Can't purchase, subscription levels don't work correctly
**Solution:** Create subscription groups in App Store Connect and add all subscriptions to the group

### Pitfall #12: Not Handling Purchase Cancellation
**Problem:** App crashes or shows errors when user cancels
**Solution:** Check for `error.userCancelled` and handle gracefully

### Pitfall #13: Testing with Production API Keys
**Problem:** Test data pollutes production analytics
**Solution:** Use separate RevenueCat projects for development and production

### Pitfall #14: Not Implementing Restore Purchases
**Problem:** Users lose access after reinstalling
**Solution:** Always provide a "Restore Purchases" button (required by Apple)

### Pitfall #15: Ignoring Webhook Events
**Problem:** Subscription status updates delayed or missed
**Solution:** Implement server-side webhook handling for real-time updates (advanced)

---

## Next Steps

Congratulations! You've built a fully functional note-taking app with subscription tiers. Here are some enhancements to consider:

1. **Add Rich Text Editing:** Implement markdown or rich text support
2. **Add Categories/Tags:** Organize notes by category
3. **Implement Search:** Add note search functionality
4. **Add Cloud Sync:** Ensure notes sync across devices (Firebase handles this automatically)
5. **Add Sharing:** Allow users to share notes
6. **Implement Analytics:** Track user behavior with Firebase Analytics
7. **Add Push Notifications:** Remind users about subscription expirations
8. **Add Offline Support:** Cache notes locally for offline access
9. **Implement Promo Codes:** Offer promotional discounts
10. **Add Family Sharing:** Support subscription sharing (iOS)

---

## Debugging Tips

### Enable Debug Logging

In `src/services/revenuecat.js`, add after initialization:

```javascript
import Purchases, { LOG_LEVEL } from 'react-native-purchases';

// In initialize function:
Purchases.setLogLevel(LOG_LEVEL.DEBUG);
```

### Common Debug Commands

**Check RevenueCat Customer Info:**
```javascript
const info = await Purchases.getCustomerInfo();
console.log('Customer Info:', JSON.stringify(info, null, 2));
```

**Check Active Entitlements:**
```javascript
const info = await Purchases.getCustomerInfo();
console.log('Active Entitlements:', Object.keys(info.entitlements.active));
```

**Check Available Products:**
```javascript
const offerings = await Purchases.getOfferings();
console.log('Current Offering:', offerings.current);
console.log('Available Packages:', offerings.current?.availablePackages);
```

---

## Resources

- [RevenueCat Documentation](https://docs.revenuecat.com/)
- [React Native Firebase Docs](https://rnfirebase.io/)
- [App Store Connect Help](https://help.apple.com/app-store-connect/)
- [Google Play Console Help](https://support.google.com/googleplay/android-developer/)
- [React Native Documentation](https://reactnative.dev/docs/getting-started)

---

## Support

If you encounter issues:

1. Check the [RevenueCat Community](https://community.revenuecat.com/)
2. Review [Firebase Support](https://firebase.google.com/support)
3. Check your implementation against this guide
4. Review console logs for specific error messages
5. Test on real devices, not simulators

Good luck with your app! 🚀
