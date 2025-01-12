# Web Service Frontend Design Document

## Overview

A Next.js-based frontend application for the Agent Network Platform with two separate domains:
- `www.xxx.ai`: Public landing page and early access
- `console.xxx.ai`: Protected console application

## Architecture

### Domain Structure
```plaintext
www.xxx.ai/                    # Public Landing Site
├── page.tsx                   # Landing page with early access form
└── (auth)/                    # Auth route group
    └── login/                 # Login page
        └── page.tsx

console.xxx.ai/                # Protected Console
├── page.tsx                   # Dashboard
├── (auth)/                    # Auth route group
│   ├── register/             # Invitation-only registration
│   │   └── page.tsx    
│   ├── verify/              # Email verification
│   │   └── page.tsx
│   └── reset/              # Password reset
│       └── page.tsx
└── dashboard/              # Protected routes
```

## Navigation Flow

### Domain-based User Journey
```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'primaryColor': '#A5B4FC', 'primaryTextColor': '#312E81', 'primaryBorderColor': '#4338CA', 'lineColor': '#6366F1', 'secondaryColor': '#E0E7FF', 'tertiaryColor': '#EEF2FF', 'noteTextColor': '#1E1B4B', 'noteBorderColor': '#4338CA', 'noteBkgColor': '#E0E7FF', 'activationBorderColor': '#4338CA', 'activationBkgColor': '#EEF2FF', 'sequenceNumberColor': '#312E81'}}}%%
graph TD
    subgraph WWW["Public Landing Site"]
        Landing["Landing Page"]
        EarlyForm["Early Access Form"]
        Landing <--> EarlyForm
    end
    
    subgraph REVIEW["Admin Review"]
        Review["Human Review"]
        Email["Send Invitation Email"]
        Review --> Email
    end
    
    subgraph CONSOLE["Protected Console"]
        Login["Login Page"]
        Register["Registration Page:<br/>Invitation Required"]
        Verify["Email Verification"]
        Dashboard["Dashboard"]
    end
    
    EarlyForm --> Review
    Email --> Register
    Landing -->|Login Button| Login
    Register --> Verify
    Verify --> Dashboard
    Login --> Dashboard
    Dashboard -->|Logout| Landing
    
    classDef www fill:#E0E7FF,stroke:#4338CA,color:#312E81,stroke-width:2px
    classDef console fill:#EEF2FF,stroke:#4338CA,color:#312E81,stroke-width:2px
    classDef review fill:#F3E8FF,stroke:#6B21A8,color:#1E1B4B,stroke-width:2px
    
    class Landing,EarlyForm www
    class Login,Register,Verify,Dashboard console
    class Review,Email review
```

### Registration Access Control Flow
```mermaid
%%{init: {
  'theme': 'base',
  'themeVariables': {
    'primaryColor': '#818CF8',
    'primaryTextColor': '#312E81',
    'primaryBorderColor': '#4338CA',
    'lineColor': '#6366F1',
    'secondaryColor': '#E0E7FF',
    'tertiaryColor': '#EEF2FF',
    'noteTextColor': '#1E1B4B',
    'noteBorderColor': '#4338CA',
    'noteBkgColor': '#E0E7FF'
  }
}}%%
sequenceDiagram
    participant U as User
    participant W as www.xxx.ai
    participant C as console.xxx.ai
    participant A as Control API
    
    U->>W: Access Landing Page
    W->>W: Display Landing Page<br>with Early Access Form
    
    U->>W: Submit Early Access Form
    W->>A: POST /early-access/apply
    A-->>W: Application Received
    
    Note over U,A: Admin Review Process
    
    A->>U: Send Invitation Email
    Note over U,A: Contains:<br>- Invitation Code<br>- Registration Link to console.xxx.ai
    
    U->>C: Access Registration Page<br>via Email Link
    C->>A: Verify Invitation Code
    
    alt Valid Invitation Code
        A-->>C: Code Valid
        C->>C: Show Registration Form
        U->>C: Complete Registration
    else Invalid/No Invitation Code
        A-->>C: Code Invalid
        C->>W: Redirect to www.xxx.ai
    end
```

### Authentication Flow
```mermaid
%%{init: {
  'theme': 'base',
  'themeVariables': {
    'primaryColor': '#818CF8',
    'primaryTextColor': '#312E81',
    'primaryBorderColor': '#4338CA',
    'lineColor': '#6366F1',
    'secondaryColor': '#E0E7FF',
    'tertiaryColor': '#EEF2FF',
    'noteTextColor': '#1E1B4B',
    'noteBorderColor': '#4338CA',
    'noteBkgColor': '#E0E7FF'
  }
}}%%
sequenceDiagram
    participant U as User
    participant W as www.xxx.ai
    participant C as console.xxx.ai
    participant A as Control API
    
    U->>W: Access Landing Page
    U->>W: Click Login
    W->>C: Redirect to console.xxx.ai/login
    
    U->>C: Submit Login Form
    C->>A: POST /auth/login
    A-->>C: Auth Success + Tokens
    
    C->>C: Store Tokens
    C->>C: Redirect to Dashboard
    
    U->>C: Click Logout
    C->>C: Clear Tokens
    C->>W: Redirect to www.xxx.ai
```

## Frontend-Backend Interface

### Domain-specific Endpoints

#### www.xxx.ai Endpoints

| Endpoint | Method | Purpose | Request Data | Response |
|----------|--------|----------|--------------|-----------|
| `/early-access/apply` | POST | Submit early access application | name, company, position, email, use_case | application_id |

#### console.xxx.ai Endpoints

| Endpoint | Method | Purpose | Request Data | Response |
|----------|--------|----------|--------------|-----------|
| `/auth/register` | POST | Register with invitation | email, password, invitation_code | user_id, requiresConfirmation |
| `/auth/verify` | POST | Verify email | email, code | success status |
| `/auth/login` | POST | Authenticate user | email, password | tokens, user data |
| `/auth/reset` | POST | Reset password | email, code, new_password | success status |

## Security Implementation

### Registration Security
1. **Invitation Code Protection**
   - Registration page only accessible via email link
   - Server-side invitation code validation
   - One-time use invitation codes
   - Code expiration handling
   - Rate limiting on code verification

2. **Domain Security**
   - Cross-Origin Resource Sharing (CORS) configuration
   - Protected routes on console.xxx.ai
   - Secure session management
   - HTTP-only cookies for tokens

### Environment Configuration

```env
# www.xxx.ai (.env)
NEXT_PUBLIC_CONSOLE_URL=https://console.xxx.ai
NEXT_PUBLIC_CONTROL_API_URL=https://api.xxx.ai

# console.xxx.ai (.env)
NEXT_PUBLIC_WWW_URL=https://www.xxx.ai
NEXT_PUBLIC_CONTROL_API_URL=https://api.xxx.ai
NEXT_PUBLIC_AUTH_DOMAIN=xxx.ai
NEXT_PUBLIC_USER_POOL_ID=us-west-2_abc123
NEXT_PUBLIC_USER_POOL_CLIENT_ID=client123xyz
```

## Middleware Implementation

### Registration Access Control
```typescript
// console.xxx.ai/middleware.ts
export async function middleware(request: NextRequest) {
  // Only apply to registration page
  if (request.nextUrl.pathname.startsWith('/auth/register')) {
    const invitationCode = request.nextUrl.searchParams.get('code')
    
    if (!invitationCode) {
      return NextResponse.redirect('https://www.xxx.ai')
    }
    
    // Verify invitation code with API
    const isValid = await verifyInvitationCode(invitationCode)
    if (!isValid) {
      return NextResponse.redirect('https://www.xxx.ai')
    }
  }
}

export const config = {
  matcher: '/auth/register'
}
```
