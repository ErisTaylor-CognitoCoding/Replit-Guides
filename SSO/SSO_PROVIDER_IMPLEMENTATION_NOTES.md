mplement SSO Provider for CognitoCoding DashDeck
Overview
Implement a secure OAuth 2.0-style authorization code flow so users can SSO from this DashDeck into other Cognito Coding apps (like Curriculum Creator). This implementation is based on a working reference implementation in Education DashDeck.

What We're Building
User Flow:

User clicks "Curriculum Creator" in DashDeck
DashDeck generates a one-time authorization code (60-second expiry)
User is redirected to: https://curriculumcreator.cognitocoding.com?code=abc123...
Curriculum Creator calls back to DashDeck to exchange code for JWT token
DashDeck validates code and returns JWT + user data + branding
User is logged into Curriculum Creator with their DashDeck account
Implementation Steps
Step 1: Database Schema
Add a table for authorization codes:

// In shared/schema.ts (or your schema file)
import { pgTable, serial, varchar, boolean, timestamp } from "drizzle-orm/pg-core";
export const ssoAuthorizationCodes = pgTable("sso_authorization_codes", {
  id: serial("id").primaryKey(),
  code: varchar("code", { length: 64 }).notNull().unique(),
  userId: varchar("user_id").notNull(),
  establishmentId: varchar("establishment_id").notNull(),
  redirectUrl: text("redirect_url").notNull(),
  used: boolean("used").notNull().default(false),
  expiresAt: timestamp("expires_at").notNull(),
  createdAt: timestamp("created_at").notNull().defaultNow(),
});
export type SSOAuthorizationCode = typeof ssoAuthorizationCodes.$inferSelect;
export type InsertSSOAuthorizationCode = typeof ssoAuthorizationCodes.$inferInsert;

Run migration:

npm run db:push

Step 2: SSO Service Layer
Create server/sso.ts:

import jwt from "jsonwebtoken";
import { randomBytes } from "crypto";
import { storage } from "./storage";
import type { User, Establishment } from "@shared/schema";
const SECRET_KEY = process.env.SSO_SECRET_KEY || "your-secret-key-change-in-production";
const TOKEN_EXPIRY = "15m";
const AUTH_CODE_EXPIRY_SECONDS = 60;
interface SSOTokenPayload {
  userId: string;
  email: string;
  firstName?: string;
  lastName?: string;
  establishmentId?: string;
  establishmentName?: string;
  establishmentLogo?: string;
  establishmentColorScheme?: {
    primary?: string;
    accent?: string;
    background?: string;
    foreground?: string;
  } | null;
  iat?: number;
  exp?: number;
}
export function generateSSOToken(user: User, establishment?: Establishment | null): string {
  if (!user.email) {
    throw new Error("User email is required for SSO token generation");
  }
  const payload: SSOTokenPayload = {
    userId: user.id,
    email: user.email,
    firstName: user.firstName ?? undefined,
    lastName: user.lastName ?? undefined,
    establishmentId: user.establishmentId ?? undefined,
    establishmentName: establishment?.name ?? undefined,
    establishmentLogo: establishment?.logoUrl ?? undefined,
    establishmentColorScheme: establishment?.colorScheme ?? undefined,
  };
  return jwt.sign(payload, SECRET_KEY, { expiresIn: TOKEN_EXPIRY });
}
export function verifySSOToken(token: string): SSOTokenPayload | null {
  try {
    const decoded = jwt.verify(token, SECRET_KEY) as jwt.JwtPayload;
    return {
      userId: decoded.userId as string,
      email: decoded.email as string,
      firstName: decoded.firstName as string | undefined,
      lastName: decoded.lastName as string | undefined,
      establishmentId: decoded.establishmentId as string | undefined,
      establishmentName: decoded.establishmentName as string | undefined,
      establishmentLogo: decoded.establishmentLogo as string | undefined,
      establishmentColorScheme: decoded.establishmentColorScheme as any,
      iat: decoded.iat,
      exp: decoded.exp,
    };
  } catch (error) {
    console.error("SSO token verification failed:", error);
    return null;
  }
}
export async function generateAuthorizationCode(
  user: User,
  establishmentId: string,
  redirectUrl: string
): Promise<string> {
  const code = randomBytes(32).toString("hex");
  const expiresAt = new Date(Date.now() + AUTH_CODE_EXPIRY_SECONDS * 1000);
  
  await storage.createSSOAuthorizationCode({
    code,
    userId: user.id,
    establishmentId,
    redirectUrl,
    used: false,
    expiresAt,
  });
  
  return code;
}
export async function exchangeCodeForToken(
  code: string
): Promise<{ token: string; user: User; establishment: Establishment } | null> {
  // Atomically fetch and mark the code as used in a single database operation
  // This prevents race conditions where two simultaneous redemptions could both succeed
  const authCode = await storage.atomicallyRedeemAuthorizationCode(code);
  
  if (!authCode) {
    console.error("Authorization code not found, already used, or expired");
    return null;
  }
  
  // Code has been atomically validated and marked as used - now fetch user and establishment
  const user = await storage.getUser(authCode.userId);
  const establishment = await storage.getEstablishment(authCode.establishmentId);
  
  if (!user || !establishment) {
    console.error("User or establishment not found for authorization code");
    return null;
  }
  
  const token = generateSSOToken(user, establishment);
  
  return { token, user, establishment };
}

Install dependency:

npm install jsonwebtoken
npm install -D @types/jsonwebtoken

Step 3: Storage Layer Methods
Add to your storage interface and implementation:

// In IStorage interface
createSSOAuthorizationCode(code: InsertSSOAuthorizationCode): Promise<SSOAuthorizationCode>;
getSSOAuthorizationCode(code: string): Promise<SSOAuthorizationCode | undefined>;
atomicallyRedeemAuthorizationCode(code: string): Promise<SSOAuthorizationCode | null>;
cleanupExpiredSSOCodes(): Promise<void>;
// In PostgresStorage implementation
async createSSOAuthorizationCode(code: InsertSSOAuthorizationCode): Promise<SSOAuthorizationCode> {
  const [result] = await db.insert(ssoAuthorizationCodesTable)
    .values(code)
    .returning();
  return result;
}
async getSSOAuthorizationCode(code: string): Promise<SSOAuthorizationCode | undefined> {
  const result = await db.select()
    .from(ssoAuthorizationCodesTable)
    .where(eq(ssoAuthorizationCodesTable.code, code))
    .limit(1);
  return result[0];
}
async atomicallyRedeemAuthorizationCode(code: string): Promise<SSOAuthorizationCode | null> {
  const now = new Date();
  
  // Single atomic operation: fetch and mark the code as used
  // Only succeeds if code exists, is unused, and not expired
  const result = await db.update(ssoAuthorizationCodesTable)
    .set({ used: true })
    .where(and(
      eq(ssoAuthorizationCodesTable.code, code),
      eq(ssoAuthorizationCodesTable.used, false),
      lt(now, ssoAuthorizationCodesTable.expiresAt)
    ))
    .returning();
  
  return result.length > 0 ? result[0] : null;
}
async cleanupExpiredSSOCodes(): Promise<void> {
  const now = new Date();
  await db.delete(ssoAuthorizationCodesTable)
    .where(lt(ssoAuthorizationCodesTable.expiresAt, now));
}

Step 4: API Routes
Add to your routes file:

import { generateAuthorizationCode, exchangeCodeForToken, verifySSOToken } from "./sso";
// SSO Launch - Generate authorization code
app.post("/api/sso/launch", isAuthenticated, async (req: any, res) => {
  try {
    const userId = req.user.claims.sub; // Or however you get user ID
    const { appUrl } = req.body;
    
    if (!appUrl) {
      res.status(400).json({ error: "App URL is required" });
      return;
    }
    
    const user = await storage.getUser(userId);
    if (!user) {
      res.status(404).json({ error: "User not found" });
      return;
    }
    if (!req.establishment) {
      res.status(400).json({ error: "No establishment found" });
      return;
    }
    // Clean up expired codes
    await storage.cleanupExpiredSSOCodes();
    
    // Generate authorization code (expires in 60 seconds)
    const code = await generateAuthorizationCode(
      user,
      req.establishment.id,
      appUrl
    );
    
    // Return redirect URL with code
    const redirectUrl = `${appUrl}?code=${code}`;
    res.json({ redirectUrl });
  } catch (error) {
    console.error("SSO launch error:", error);
    res.status(500).json({ error: "Failed to generate authorization code" });
  }
});
// SSO Exchange - Trade code for token (called by other apps)
app.post("/api/sso/exchange", async (req, res) => {
  try {
    const { code } = req.body;
    
    if (!code) {
      res.status(400).json({ error: "Authorization code is required" });
      return;
    }
    const result = await exchangeCodeForToken(code);
    
    if (!result) {
      res.status(401).json({ error: "Invalid, expired, or already used authorization code" });
      return;
    }
    // Return token and SANITIZED user/establishment data
    // NEVER expose OAuth tokens (googleAccessToken, googleRefreshToken)
    res.json({
      token: result.token,
      user: {
        id: result.user.id,
        email: result.user.email,
        firstName: result.user.firstName,
        lastName: result.user.lastName,
        profileImageUrl: result.user.profileImageUrl,
        role: result.user.role,
        // Deliberately omit OAuth tokens
      },
      establishment: {
        id: result.establishment.id,
        name: result.establishment.name,
        logoUrl: result.establishment.logoUrl,
        colorScheme: result.establishment.colorScheme,
      }
    });
  } catch (error) {
    console.error("SSO exchange error:", error);
    res.status(500).json({ error: "Failed to exchange authorization code" });
  }
});
// SSO Verify - Check if token is valid
app.post("/api/sso/verify", async (req, res) => {
  try {
    const { token } = req.body;
    
    if (!token) {
      res.status(400).json({ error: "Token is required" });
      return;
    }
    const payload = verifySSOToken(token);
    
    if (!payload) {
      res.status(401).json({ valid: false, error: "Invalid or expired token" });
      return;
    }
    res.json({
      valid: true,
      userId: payload.userId,
      email: payload.email,
      firstName: payload.firstName,
      lastName: payload.lastName,
      establishmentId: payload.establishmentId,
      establishmentName: payload.establishmentName,
      establishmentLogo: payload.establishmentLogo,
      establishmentColorScheme: payload.establishmentColorScheme,
      expiresAt: payload.exp,
    });
  } catch (error) {
    console.error("SSO verification error:", error);
    res.status(500).json({ error: "Failed to verify token" });
  }
});

Step 5: Frontend - App Launcher Component
Create a component that shows available apps and launches SSO:

import { useQuery } from "@tanstack/react-query";
import { apiRequest } from "@/lib/queryClient";
import { Card, CardHeader, CardTitle, CardContent } from "@/components/ui/card";
import { Badge } from "@/components/ui/badge";
import { useToast } from "@/hooks/use-toast";
const cognitoApps = [
  {
    name: "Curriculum Creator",
    url: "https://curriculumcreator.cognitocoding.com",
    appKey: "curriculum-creator",
  },
  {
    name: "Award Tracker",
    url: "https://awardtracker.cognitocoding.com",
    appKey: "award-tracker",
  },
  // Add more apps...
];
export default function CognitoApps() {
  const { toast } = useToast();
  const handleAppClick = async (appUrl: string) => {
    try {
      // Call backend to generate authorization code
      const response = await apiRequest("/api/sso/launch", "POST", { appUrl });
      const data = await response.json();
      
      if (!data.redirectUrl) {
        throw new Error("No redirect URL returned");
      }
      
      // Open app with authorization code
      window.open(data.redirectUrl, "_blank");
    } catch (error) {
      toast({
        title: "Error",
        description: "Failed to launch app. Please try again.",
        variant: "destructive",
      });
    }
  };
  return (
    <Card>
      <CardHeader>
        <CardTitle>Cognito Coding Apps</CardTitle>
      </CardHeader>
      <CardContent>
        <div className="space-y-3">
          {cognitoApps.map((app) => (
            <div
              key={app.appKey}
              onClick={() => handleAppClick(app.url)}
              className="flex items-center justify-between p-4 rounded-md border hover-elevate cursor-pointer"
            >
              <div>
                <h4 className="font-medium">{app.name}</h4>
              </div>
              <Badge>Launch</Badge>
            </div>
          ))}
        </div>
      </CardContent>
    </Card>
  );
}

Step 6: Environment Variables
Add to your Replit Secrets or .env:

SSO_SECRET_KEY=your-random-secret-key-change-this-in-production

Generate a secure secret:

node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"

Security Features
✅ One-time codes: Authorization codes can only be used once
✅ 60-second expiry: Short-lived codes minimize interception risk
✅ Atomic redemption: Race-condition proof database operation
✅ No token in URLs: JWT tokens never appear in browser history
✅ OAuth protection: User's Google tokens never exposed to apps
✅ Server-to-server exchange: Tokens only transmitted securely

Testing
Add the CognitoApps component to your dashboard
Click "Curriculum Creator"
Should open: https://curriculumcreator.cognitocoding.com?code=abc123...
Check browser network tab - code exchange should return token + user data
What Other Apps Need
The notes I created in IMPLEMENTATION_NOTES_FOR_CURRICULUM_CREATOR.md show what Curriculum Creator and other apps need to implement on THEIR side to consume this SSO.

Summary
This implements DashDeck as an SSO Provider that:

Generates authorization codes when users click apps
Exchanges codes for JWT tokens (server-to-server)
Verifies tokens for ongoing authentication
Provides user data + establishment branding to apps
Other apps become SSO Consumers that receive codes and exchange them for authentication.
