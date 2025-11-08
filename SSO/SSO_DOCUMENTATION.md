Education DashDeck SSO Documentation
Overview
Education DashDeck implements a secure OAuth 2.0-style authorization code flow for Single Sign-On (SSO) with other Cognito Coding applications. This allows users to seamlessly access multiple Cognito apps without re-authenticating, while maintaining establishment branding and user context across applications.

Security Features
Why Authorization Code Flow?
The authorization code flow provides several critical security advantages over passing tokens directly in URLs:

No Token Exposure in URLs: JWT tokens never appear in browser address bars, preventing leakage through:

Browser history
Server access logs
HTTP Referer headers
Screenshot/screen sharing
One-Time Use Codes: Authorization codes can only be redeemed once, preventing replay attacks

Short Expiry: Authorization codes expire after 60 seconds, minimizing the window for interception

Server-to-Server Exchange: The actual JWT token is only transmitted in a secure server-to-server request

OAuth Token Protection: User's Google OAuth tokens (googleAccessToken, googleRefreshToken) are never exposed to integrated applications

Architecture
Flow Diagram
User                    DashDeck                 Cognito App
  |                        |                          |
  |---(1) Click App------->|                          |
  |                        |                          |
  |                   Generate Code                   |
  |                   (60s expiry)                    |
  |                        |                          |
  |<---(2) Redirect URL----|                          |
  |     with code          |                          |
  |                        |                          |
  |---(3) Navigate with code------------------->|     |
  |                        |                     |     |
  |                        |<---(4) Exchange Code-----|
  |                        |                     |     |
  |                   Validate Code              |     |
  |                   Mark as Used               |     |
  |                   Generate JWT               |     |
  |                        |                     |     |
  |                        |---(5) Return Token------>|
  |                        |     + User Data     |     |
  |                        |     + Branding      |     |
  |                        |                     |     |
  |<---(6) App loads with user context-----------|     |

Step-by-Step Process
Step 1: User Initiates SSO
User clicks on a Cognito app from their DashDeck dashboard.

Step 2: Generate Authorization Code
DashDeck generates a one-time authorization code that:

Expires in 60 seconds
Is linked to the user and establishment
Stores the target app's redirect URL
Is marked as unused in the database
Step 3: Redirect to App
User is redirected to the Cognito app with the authorization code in the URL:

https://codinghub.cognitocoding.com?code=abc123...

Step 4: Server-to-Server Exchange
The Cognito app's backend makes a secure POST request to DashDeck:

POST https://dashdeck.app/api/sso/exchange
Content-Type: application/json
{
  "code": "abc123..."
}

Step 5: Atomic Validation & Token Generation
DashDeck performs an atomic database operation that:

Validates the code exists
Checks it hasn't been used
Verifies it hasn't expired
Marks it as used (all in one transaction)
If successful, DashDeck:

Generates a JWT token (15-minute expiry)
Returns sanitized user data
Returns establishment branding information
Step 6: App Authenticates User
The Cognito app stores the JWT token and uses it to:

Identify the user
Apply establishment branding
Make authorized requests
API Endpoints
1. Launch SSO Flow
Endpoint: POST /api/sso/launch

Authentication: Required (DashDeck session)

Request Body:

{
  "appUrl": "https://codinghub.cognitocoding.com"
}

Response:

{
  "redirectUrl": "https://codinghub.cognitocoding.com?code=64_character_hex_code"
}

Frontend Usage:

const response = await apiRequest("/api/sso/launch", "POST", { 
  appUrl: "https://codinghub.cognitocoding.com" 
});
const { redirectUrl } = await response.json();
window.open(redirectUrl, "_blank");

2. Exchange Authorization Code
Endpoint: POST /api/sso/exchange

Authentication: Not required (public endpoint for app servers)

Request Body:

{
  "code": "64_character_hex_code"
}

Success Response (200):

{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": "user_123",
    "email": "teacher@school.com",
    "firstName": "Jane",
    "lastName": "Smith",
    "profileImageUrl": "https://...",
    "role": "teacher"
  },
  "establishment": {
    "id": "my-school",
    "name": "My School",
    "logoUrl": "https://...",
    "colorScheme": {
      "primary": "210 100% 50%",
      "accent": "45 100% 60%"
    }
  }
}

Error Response (401):

{
  "error": "Invalid, expired, or already used authorization code"
}

Important: This endpoint deliberately excludes OAuth tokens:

googleAccessToken ❌ Never exposed
googleRefreshToken ❌ Never exposed
googleTokenExpiry ❌ Never exposed
3. Verify SSO Token
Endpoint: POST /api/sso/verify

Authentication: Not required

Request Body:

{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}

Success Response (200):

{
  "valid": true,
  "userId": "user_123",
  "email": "teacher@school.com",
  "firstName": "Jane",
  "establishmentId": "my-school",
  "establishmentName": "My School",
  "establishmentLogo": "https://...",
  "establishmentColorScheme": {
    "primary": "210 100% 50%",
    "accent": "45 100% 60%"
  },
  "expiresAt": 1699564800
}

Database Schema
Authorization Codes Table
CREATE TABLE sso_authorization_codes (
  id SERIAL PRIMARY KEY,
  code VARCHAR(64) UNIQUE NOT NULL,
  user_id VARCHAR NOT NULL,
  establishment_id VARCHAR NOT NULL,
  redirect_url TEXT NOT NULL,
  used BOOLEAN NOT NULL DEFAULT false,
  expires_at TIMESTAMP NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT now()
);

Atomic Redemption Query
The critical security feature is the atomic redemption query:

UPDATE sso_authorization_codes
SET used = true
WHERE code = $1 
  AND used = false 
  AND expires_at > now()
RETURNING *;

This single query ensures:

Only unused codes can be redeemed
Only non-expired codes work
Race conditions are impossible (two simultaneous requests can't both succeed)
JWT Token Structure
Payload
interface SSOTokenPayload {
  userId: string;
  email: string;
  firstName?: string;
  establishmentId?: string;
  establishmentName?: string;
  establishmentLogo?: string;
  establishmentColorScheme?: {
    primary?: string;
    accent?: string;
    background?: string;
    foreground?: string;
  };
  iat: number;  // Issued at timestamp
  exp: number;  // Expiration timestamp (15 minutes)
}

Generation
jwt.sign(payload, SECRET_KEY, { expiresIn: "15m" });

Verification
const decoded = jwt.verify(token, SECRET_KEY);

Implementation Guide for Cognito Apps
Backend Setup
Receive Authorization Code
When DashDeck redirects to your app, extract the code from the URL:

// Express.js example
app.get("/", (req, res) => {
  const code = req.query.code;
  if (code) {
    // Exchange code for token
    exchangeCodeForToken(code);
  }
});

Exchange Code for Token
Make a server-to-server request:

async function exchangeCodeForToken(code: string) {
  const response = await fetch("https://dashdeck.app/api/sso/exchange", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ code })
  });
  
  if (!response.ok) {
    throw new Error("Invalid authorization code");
  }
  
  const { token, user, establishment } = await response.json();
  
  // Store token in session
  // Apply establishment branding
  // Authenticate user
  
  return { token, user, establishment };
}

Verify Token on Subsequent Requests
async function verifyToken(token: string) {
  const response = await fetch("https://dashdeck.app/api/sso/verify", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ token })
  });
  
  const result = await response.json();
  return result.valid ? result : null;
}

Frontend Implementation
Apply Establishment Branding
function applyBranding(establishment: Establishment) {
  if (establishment.colorScheme) {
    document.documentElement.style.setProperty(
      '--primary', 
      establishment.colorScheme.primary
    );
    document.documentElement.style.setProperty(
      '--accent', 
      establishment.colorScheme.accent
    );
  }
}

Display User Information
function displayUser(user: User) {
  return (
    <div>
      <img src={user.profileImageUrl} alt={user.firstName} />
      <span>{user.firstName} {user.lastName}</span>
      <Badge>{user.role}</Badge>
    </div>
  );
}

Security Best Practices
For DashDeck Administrators
Secret Key Management

Store SECRET_KEY securely in environment variables
Rotate keys periodically
Never commit keys to version control
HTTPS Only

All SSO endpoints must use HTTPS in production
Never allow SSO over HTTP
Monitoring

Monitor for unusual redemption patterns
Log failed exchange attempts
Set up alerts for repeated failures
For Cognito App Developers
Never Store Long-Lived Tokens

JWT tokens expire in 15 minutes
Re-verify on each session if needed
Validate All Token Data

Don't trust token payload without verification
Always verify tokens server-side
Handle Expiration Gracefully

Redirect to DashDeck for re-authentication
Show clear error messages
Common Issues & Troubleshooting
Issue: "Authorization code already used"
Cause: Code was redeemed multiple times

Solutions:

Check for duplicate redemption requests
Ensure app doesn't retry the exchange on failure
Verify no client-side code is attempting redemption
Issue: "Authorization code expired"
Cause: More than 60 seconds elapsed between generation and exchange

Solutions:

Ensure the exchange happens immediately after redirect
Check for slow network conditions
Verify server time synchronization
Issue: "Invalid token"
Cause: Token has expired (>15 minutes old) or was modified

Solutions:

Generate a new authorization code
Verify SECRET_KEY matches between systems
Check token hasn't been tampered with
Issue: Race Condition Errors
Cause: Multiple simultaneous exchange attempts

Solutions:

The atomic database operation prevents this
If still occurring, check for application-level retries
Review network configuration for duplicate requests
Performance Considerations
Code Cleanup
Authorization codes are automatically cleaned up:

// Runs before each code generation
await storage.cleanupExpiredSSOCodes();

This prevents database bloat from expired codes.

Token Caching
Cognito apps should cache verified tokens to avoid repeated verification:

const tokenCache = new Map<string, { payload: any, expiresAt: number }>();
function getCachedToken(token: string) {
  const cached = tokenCache.get(token);
  if (cached && cached.expiresAt > Date.now()) {
    return cached.payload;
  }
  return null;
}

Testing
Manual Testing Flow
Log into DashDeck
Click on a Cognito app
Verify you're redirected with a ?code= parameter
Verify the app exchanges the code successfully
Verify you're logged into the app
Try using the same code again (should fail)
Security Testing
Test Code Reuse Prevention

Exchange a code once successfully
Try to exchange it again (should fail with 401)
Test Expiration

Generate a code
Wait 61 seconds
Try to exchange (should fail)
Test Race Conditions

Generate a code
Make two simultaneous exchange requests
Only one should succeed
Additional Resources
JWT Documentation: https://jwt.io/
OAuth 2.0 Authorization Code Flow: https://oauth.net/2/grant-types/authorization-code/
Drizzle ORM Documentation: https://orm.drizzle.team/
Support
For issues or questions about the SSO implementation, contact the DashDeck development team or refer to the codebase:

SSO Logic: server/sso.ts
Storage Layer: server/storage.ts
API Routes: server/routes.ts
Database Schema: shared/schema.ts
