DashDeck SSO Integration Guide
This guide explains how to integrate DashDeck Single Sign-On (SSO) into your application, allowing users to launch your app directly from DashDeck with automatic authentication and access to student data.

Overview
DashDeck SSO uses an OAuth 2.0 Authorization Code Flow:

User clicks your app link in DashDeck
DashDeck redirects to your app with a ?code= parameter
Your app exchanges the code for a JWT token (within 60 seconds)
The JWT token is stored and used for authenticated API calls
User is logged in and can fetch their DashDeck students
┌─────────────┐    1. Click app link    ┌─────────────┐
│   DashDeck  │ ─────────────────────► │  Your App   │
│  Dashboard  │    ?code=abc123        │  Frontend   │
└─────────────┘                        └──────┬──────┘
                                              │
                                              │ 2. Exchange code
                                              ▼
                                       ┌──────────────┐
                                       │  Your App    │
                                       │  Backend     │
                                       └──────┬───────┘
                                              │
                                              │ 3. POST /api/sso/exchange
                                              ▼
                                       ┌──────────────┐
                                       │  DashDeck    │
                                       │  API         │
                                       └──────┬───────┘
                                              │
                                              │ 4. Return JWT token
                                              ▼
                                       ┌──────────────┐
                                       │  Store token │
                                       │  in storage  │
                                       └──────────────┘

Environment Variables Required
Add these to your .env or Replit Secrets:

# DashDeck SSO Configuration
DASHDECK_API_URL=https://your-dashdeck-instance.com
DASHDECK_SSO_SECRET=your-shared-secret-key

Variable	Description
DASHDECK_API_URL	Base URL of your DashDeck instance
DASHDECK_SSO_SECRET	Shared secret for HMAC signature verification (get from DashDeck admin)
Backend Setup
1. Create the SSO Service
Create a file server/services/dashdeckSSO.ts:

import crypto from 'crypto';
interface DashDeckTokenResponse {
  token: string;
  user: {
    id: string;
    email: string;
    name: string;
    role: string;
  };
  establishment?: {
    id: string;
    name: string;
    subdomain: string;
  };
}
interface DashDeckStudent {
  studentId: string;
  firstName: string;
  lastName: string;
  yearGroup: number;
  dateOfBirth?: string;
}
class DashDeckSSOService {
  private apiUrl: string;
  private ssoSecret: string;
  constructor() {
    this.apiUrl = process.env.DASHDECK_API_URL || '';
    this.ssoSecret = process.env.DASHDECK_SSO_SECRET || '';
  }
  isConfigured(): boolean {
    return !!(this.apiUrl && this.ssoSecret);
  }
  // Generate HMAC signature for secure API calls
  private generateSignature(payload: string): string {
    return crypto
      .createHmac('sha256', this.ssoSecret)
      .update(payload)
      .digest('hex');
  }
  // Exchange authorization code for JWT token
  async exchangeCode(code: string, redirectUri: string): Promise<DashDeckTokenResponse> {
    const timestamp = Date.now().toString();
    const payload = `${code}:${timestamp}`;
    const signature = this.generateSignature(payload);
    const response = await fetch(`${this.apiUrl}/api/sso/exchange`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-SSO-Signature': signature,
        'X-SSO-Timestamp': timestamp,
      },
      body: JSON.stringify({
        code,
        redirect_uri: redirectUri,
        app_id: 'your-app-id', // Your registered app ID in DashDeck
      }),
    });
    if (!response.ok) {
      const error = await response.text();
      throw new Error(`Token exchange failed: ${error}`);
    }
    return response.json();
  }
  // Fetch students using JWT token
  async fetchStudents(token: string): Promise<DashDeckStudent[]> {
    const timestamp = Date.now().toString();
    const signature = this.generateSignature(`students:${timestamp}`);
    const response = await fetch(`${this.apiUrl}/api/sso/students`, {
      method: 'GET',
      headers: {
        'Authorization': `Bearer ${token}`,
        'X-SSO-Signature': signature,
        'X-SSO-Timestamp': timestamp,
      },
    });
    if (!response.ok) {
      throw new Error('Failed to fetch students from DashDeck');
    }
    const data = await response.json();
    return data.students || [];
  }
}
export const dashDeckSSO = new DashDeckSSOService();

2. Add API Routes
Add these routes to your server/routes.ts:

import { dashDeckSSO } from './services/dashdeckSSO';
// Check if DashDeck SSO is configured
app.get('/api/dashdeck/status', (req, res) => {
  res.json({
    configured: dashDeckSSO.isConfigured(),
    redirectUrl: process.env.DASHDECK_API_URL 
      ? `${process.env.DASHDECK_API_URL}/apps/your-app` 
      : null,
  });
});
// Exchange authorization code for token
app.post('/api/dashdeck/exchange', async (req, res) => {
  try {
    const { code, redirect_uri } = req.body;
    
    if (!code) {
      return res.status(400).json({ error: 'Authorization code required' });
    }
    const result = await dashDeckSSO.exchangeCode(code, redirect_uri);
    res.json(result);
  } catch (error: any) {
    console.error('DashDeck token exchange error:', error);
    res.status(401).json({ error: error.message });
  }
});
// Fetch students (requires token in Authorization header)
app.get('/api/dashdeck/fetch-students', async (req, res) => {
  try {
    const authHeader = req.headers.authorization;
    if (!authHeader?.startsWith('Bearer ')) {
      return res.status(401).json({ error: 'DashDeck token required' });
    }
    const token = authHeader.substring(7);
    const students = await dashDeckSSO.fetchStudents(token);
    res.json({ students });
  } catch (error: any) {
    console.error('DashDeck fetch students error:', error);
    res.status(500).json({ error: error.message });
  }
});

Frontend Setup
1. Create the DashDeck Context
Create client/src/contexts/DashDeckContext.tsx:

import { createContext, useContext, useState, useEffect, ReactNode } from 'react';
// Types
export interface DashDeckUser {
  id: string;
  email: string;
  name: string;
  role: string;
}
export interface DashDeckStudent {
  studentId: string;
  firstName: string;
  lastName: string;
  yearGroup: number;
}
interface DashDeckContextType {
  isAuthenticated: boolean;
  user: DashDeckUser | null;
  token: string | null;
  fetchStudents: () => Promise<DashDeckStudent[]>;
  logout: () => void;
}
const DashDeckContext = createContext<DashDeckContextType | undefined>(undefined);
// Storage keys
const TOKEN_KEY = 'dashdeck_token';
const USER_KEY = 'dashdeck_user';
export function DashDeckProvider({ children }: { children: ReactNode }) {
  const [token, setToken] = useState<string | null>(() => 
    localStorage.getItem(TOKEN_KEY)
  );
  const [user, setUser] = useState<DashDeckUser | null>(() => {
    const stored = localStorage.getItem(USER_KEY);
    return stored ? JSON.parse(stored) : null;
  });
  // Save/clear token and user when they change
  useEffect(() => {
    if (token) {
      localStorage.setItem(TOKEN_KEY, token);
    } else {
      localStorage.removeItem(TOKEN_KEY);
    }
  }, [token]);
  useEffect(() => {
    if (user) {
      localStorage.setItem(USER_KEY, JSON.stringify(user));
    } else {
      localStorage.removeItem(USER_KEY);
    }
  }, [user]);
  // Public method to set auth data (called after code exchange)
  const setAuthData = (newToken: string, newUser: DashDeckUser) => {
    setToken(newToken);
    setUser(newUser);
  };
  // Fetch students from DashDeck
  const fetchStudents = async (): Promise<DashDeckStudent[]> => {
    if (!token) {
      throw new Error('Not authenticated with DashDeck');
    }
    const response = await fetch('/api/dashdeck/fetch-students', {
      headers: {
        'Authorization': `Bearer ${token}`,
      },
    });
    if (!response.ok) {
      if (response.status === 401) {
        logout(); // Token expired
        throw new Error('Session expired. Please launch from DashDeck again.');
      }
      throw new Error('Failed to fetch students');
    }
    const data = await response.json();
    return data.students || [];
  };
  // Logout
  const logout = () => {
    setToken(null);
    setUser(null);
    localStorage.removeItem(TOKEN_KEY);
    localStorage.removeItem(USER_KEY);
  };
  // Expose setAuthData for the SSO hook
  (window as any).__dashDeckSetAuth = setAuthData;
  return (
    <DashDeckContext.Provider value={{
      isAuthenticated: !!token,
      user,
      token,
      fetchStudents,
      logout,
    }}>
      {children}
    </DashDeckContext.Provider>
  );
}
export function useDashDeck() {
  const context = useContext(DashDeckContext);
  if (!context) {
    throw new Error('useDashDeck must be used within DashDeckProvider');
  }
  return context;
}

2. Create the SSO Hook
Create client/src/hooks/useDashDeckSSO.ts:

import { useEffect } from 'react';
const PROCESSED_KEY = 'dashdeck_code_processed';
export function useDashDeckSSO() {
  useEffect(() => {
    const handleSSOCode = async () => {
      // Get code from URL
      const urlParams = new URLSearchParams(window.location.search);
      const code = urlParams.get('code');
      
      if (!code) return;
      // Prevent duplicate processing (React StrictMode, re-renders)
      const processedCode = sessionStorage.getItem(PROCESSED_KEY);
      if (processedCode === code) {
        // Already processed - just clean URL
        window.history.replaceState({}, '', window.location.pathname);
        return;
      }
      // Mark as being processed
      sessionStorage.setItem(PROCESSED_KEY, code);
      try {
        // Exchange code for token
        const response = await fetch('/api/dashdeck/exchange', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            code,
            redirect_uri: window.location.origin,
          }),
        });
        if (!response.ok) {
          const error = await response.json();
          throw new Error(error.error || 'Token exchange failed');
        }
        const data = await response.json();
        
        // Store auth data using the context's exposed method
        const setAuth = (window as any).__dashDeckSetAuth;
        if (setAuth && data.token && data.user) {
          setAuth(data.token, data.user);
        }
        // Clean URL
        window.history.replaceState({}, '', window.location.pathname);
        
      } catch (error) {
        console.error('DashDeck SSO error:', error);
        sessionStorage.removeItem(PROCESSED_KEY);
      }
    };
    handleSSOCode();
  }, []);
}

3. Wire Up in App.tsx
Update your App.tsx:

import { DashDeckProvider, useDashDeck } from './contexts/DashDeckContext';
import { useDashDeckSSO } from './hooks/useDashDeckSSO';
// Component that handles SSO and routing
function AppContent() {
  // Process SSO code from URL
  useDashDeckSSO();
  
  const { isAuthenticated: isDashDeckAuth } = useDashDeck();
  const { isAuthenticated: isOtherAuth } = useYourOtherAuth(); // e.g., Replit Auth
  
  // Allow access if authenticated via either method
  const isAuthenticated = isDashDeckAuth || isOtherAuth;
  if (!isAuthenticated) {
    return <LandingPage />;
  }
  return <MainApp />;
}
// Wrap with provider
function App() {
  return (
    <DashDeckProvider>
      <AppContent />
    </DashDeckProvider>
  );
}
export default App;

Using Student Data
Once set up, you can fetch and use DashDeck students anywhere in your app:

import { useDashDeck } from '@/contexts/DashDeckContext';
function StudentPicker() {
  const { fetchStudents, isAuthenticated } = useDashDeck();
  const [students, setStudents] = useState([]);
  const [loading, setLoading] = useState(false);
  const handleLoadStudents = async () => {
    if (!isAuthenticated) {
      alert('Please launch from DashDeck to import students');
      return;
    }
    setLoading(true);
    try {
      const studentList = await fetchStudents();
      setStudents(studentList);
    } catch (error) {
      console.error('Failed to load students:', error);
    } finally {
      setLoading(false);
    }
  };
  return (
    <div>
      <button onClick={handleLoadStudents} disabled={loading}>
        {loading ? 'Loading...' : 'Load Students from DashDeck'}
      </button>
      
      {students.map(student => (
        <div key={student.studentId}>
          {student.firstName} {student.lastName} - Year {student.yearGroup}
        </div>
      ))}
    </div>
  );
}

DashDeck Admin Setup
On the DashDeck side, you need to register your app:

Go to DashDeck Admin Panel → Apps/Integrations
Add New App with these settings:
App Name: Your App Name (e.g., "CodingHub")
App ID: Unique identifier (e.g., "codinghub")
Redirect URL: Your app's URL (e.g., https://codinghub.yourdomain.app)
SSO Secret: Generate a secure random string (use same value in your app's env vars)
Enable permissions:
read:students - To fetch student list
read:user - To get user info
Save and note the SSO Secret
Security Checklist
 SSO Secret is stored securely (environment variable, not in code)
 HMAC signatures verify all API requests
 Authorization codes expire after 60 seconds
 JWT tokens are stored in localStorage (cleared on logout)
 Token expiry is handled gracefully (re-auth prompt)
 HTTPS is enforced for all SSO traffic
Troubleshooting
Issue	Solution
"Code expired" error	Authorization codes only last 60 seconds. User must click link in DashDeck again.
Infinite redirect loop	Check sessionStorage tracking - ensure processed codes aren't re-processed
"Token exchange failed"	Verify SSO Secret matches between DashDeck and your app
Students not loading	Check Authorization header is being sent with Bearer token
CORS errors	Ensure DashDeck allows your app's origin
Quick Start Checklist
 Add environment variables (DASHDECK_API_URL, DASHDECK_SSO_SECRET)
 Create SSO service file (server/services/dashdeckSSO.ts)
 Add API routes (/api/dashdeck/status, /exchange, /fetch-students)
 Create DashDeck context (client/src/contexts/DashDeckContext.tsx)
 Create SSO hook (client/src/hooks/useDashDeckSSO.ts)
 Wire up provider and hook in App.tsx
 Register app in DashDeck admin panel
 Test: Click app link in DashDeck → Auto-login → Fetch students
Files Reference
File	Purpose
server/services/dashdeckSSO.ts	Backend SSO service (code exchange, student fetch)
server/routes.ts	API endpoints for SSO
client/src/contexts/DashDeckContext.tsx	React context for auth state
client/src/hooks/useDashDeckSSO.ts	Hook to process URL codes
client/src/App.tsx	Provider setup and auth gating
Last updated: November 2024
