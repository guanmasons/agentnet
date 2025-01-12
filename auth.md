# Authentication and Authorization Design

## Overview

### Cognito User Pools
User Pools serve as our user directory and authentication system. They:
- Handle user registration and authentication
- Manage user attributes and verification
- Issue JWT tokens for identity verification
- Integrate with SES for email verification

### Cognito Identity Pools
Identity Pools provide secure AWS service access. They:
- Issue temporary AWS credentials
- Map authenticated users to IAM roles
- Enable fine-grained access control
- Integrate with User Pools for authentication

### System Architecture

```mermaid
%%{init: {
  'theme': 'base',
  'themeVariables': {
    'primaryColor': '#4F46E5',
    'primaryTextColor': '#000000',
    'primaryBorderColor': '#4338CA',
    'lineColor': '#6366F1',
    'secondaryColor': '#E2E8F0',
    'tertiaryColor': '#F1F5F9',
    'textColor': '#000000',
    'mainBkg': '#ffffff'
  }
}}%%
graph TB
    subgraph "Authentication Layer"
        direction LR
        UP[Cognito User Pool<br>us-west-2_abc123]
        SES[Simple Email Service]
        UP --> SES
    end

    subgraph "Authorization Layer"
        direction LR
        IP[Identity Pool<br>us-west-2:67890xyz]
        IAM[IAM Roles]
        IP --> IAM
    end

    subgraph "Data Layer"
        direction LR
        DDB[DynamoDB Tables]
    end

    UP --> IP
    IAM --> DDB

    classDef default fill:#F8FAFC,stroke:#475569,stroke-width:2px
    classDef auth fill:#E0E7FF,stroke:#4338CA,stroke-width:2px
    classDef authz fill:#DCFCE7,stroke:#166534,stroke-width:2px
    classDef data fill:#F3E8FF,stroke:#6B21A8,stroke-width:2px
    
    class UP,SES auth
    class IP,IAM authz
    class DDB data
```

## Authentication Flows

### System Setup Flow

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
    'noteBkgColor': '#E0E7FF',
    'activationBorderColor': '#4338CA',
    'activationBkgColor': '#EEF2FF',
    'sequenceNumberColor': '#312E81'
  }
}}%%
sequenceDiagram
    participant A as AWS Admin
    participant C as Cognito
    participant S as SES
    participant I as Identity Pool
    participant D as DynamoDB

    rect rgb(224, 231, 255)
        Note over A,D: Initial System Setup Flow

        A->>C: Create User Pool
        Note over C: Request:<br/>Region: us-west-2<br/>EmailConfiguration: verify@yourdomain.com

        C-->>A: User Pool Created
        Note over C: Response:<br/>UserPoolId: us-west-2_abc123<br/>Region: us-west-2

        A->>C: Create App Client
        Note over C: Request:<br/>UserPoolId: us-west-2_abc123

        C-->>A: App Client Created
        Note over C: Response:<br/>ClientId: 3ab12345xyz<br/>ClientSecret: 67890xyz...

        A->>S: Configure SES
        Note over S: Request:<br/>EmailIdentity: verify@yourdomain.com

        S-->>A: SES Configured
        Note over S: Response:<br/>IdentityVerified: true<br/>EmailEnabled: true

        A->>I: Create Identity Pool
        Note over I: Request:<br/>UserPoolId: us-west-2_abc123<br/>ClientId: 3ab12345xyz

        I-->>A: Identity Pool Created
        Note over I: Response:<br/>IdentityPoolId: us-west-2:67890xyz

        A->>D: Create DynamoDB Tables
        Note over D: Request:<br/>TableNames: Users, Agents<br/>PrimaryKey: user_id

        D-->>A: Tables Created
        Note over D: Response:<br/>TableStatus: ACTIVE<br/>StreamEnabled: true
    end
```

### User Registration Flow

```mermaid
%%{init: {
  'theme': 'base',
  'themeVariables': {
    'primaryColor': '#A5B4FC',
    'primaryTextColor': '#312E81',
    'primaryBorderColor': '#4338CA',
    'lineColor': '#6366F1',
    'secondaryColor': '#E0E7FF',
    'tertiaryColor': '#EEF2FF',
    'noteTextColor': '#1E1B4B',
    'noteBorderColor': '#4338CA',
    'noteBkgColor': '#E0E7FF',
    'activationBorderColor': '#4338CA',
    'activationBkgColor': '#EEF2FF',
    'sequenceNumberColor': '#312E81'
  }
}}%%
sequenceDiagram
    participant U as User Browser
    participant F as Frontend
    participant C as Cognito
    participant S as SES
    participant D as DynamoDB
    
    rect rgb(224, 231, 255)
        Note over U,D: User Registration Flow
        
        U->>F: Submit Registration Form
        Note over F: Request:<br/>Email: alice@example.com<br/>Password: ****

        F->>C: SignUp API Call
        Note over C: Request:<br/>ClientId: 3ab12345xyz<br/>Username: alice@example.com<br/>Password: ****

        C-->>F: SignUp Response
        Note over C: Response:<br/>UserSub: d7e87e6d-8704-4e91-b7f3-1234abcd<br/>UserConfirmed: false
        
        C->>S: Send Verification Email
        Note over S: Request:<br/>To: alice@example.com<br/>From: verify@yourdomain.com<br/>Template: VERIFY_EMAIL
        
        S-->>U: Verification Code Email
        Note over S: Email Content:<br/>Code: 123456<br/>ExpiresIn: 24h
        
        U->>F: Submit Verification Code
        Note over F: Request:<br/>Username: alice@example.com<br/>Code: 123456
        
        F->>C: ConfirmSignUp API Call
        Note over C: Request:<br/>ClientId: 3ab12345xyz<br/>Username: alice@example.com<br/>ConfirmationCode: 123456
        
        C-->>F: Confirmation Response
        Note over C: Response:<br/>Success: true<br/>UserStatus: CONFIRMED
        
        F->>D: Create User Record
        Note over D: Request:<br/>TableName: Users<br/>Item: {<br/>  user_id: d7e87e6d-8704-4e91-b7f3-1234abcd,<br/>  email: alice@example.com,<br/>  created_at: timestamp<br/>}
        
        D-->>F: User Record Created
        Note over D: Response:<br/>Success: true
        
        F-->>U: Registration Complete
        Note over F: Response:<br/>Status: SUCCESS<br/>RedirectTo: /login
    end
```

### User Authentication Flow

```mermaid
%%{init: {
  'theme': 'base',
  'themeVariables': {
    'primaryColor': '#A5B4FC',
    'primaryTextColor': '#312E81',
    'primaryBorderColor': '#4338CA',
    'lineColor': '#6366F1',
    'secondaryColor': '#E0E7FF',
    'tertiaryColor': '#EEF2FF',
    'noteTextColor': '#1E1B4B',
    'noteBorderColor': '#4338CA',
    'noteBkgColor': '#E0E7FF',
    'activationBorderColor': '#4338CA',
    'activationBkgColor': '#EEF2FF',
    'sequenceNumberColor': '#312E81'
  }
}}%%
sequenceDiagram
    participant U as User Browser
    participant F as Frontend
    participant C as Cognito
    participant I as Identity Pool
    participant D as DynamoDB

    rect rgb(224, 231, 255)
        Note over U,D: Token to Credentials Flow
        
        U->>F: Login Request
        Note over F: Request:<br/>Username: alice@example.com<br/>Password: ****
        
        F->>C: InitiateAuth
        Note over C: Request:<br/>ClientId: 3ab12345xyz<br/>AuthFlow: USER_PASSWORD_AUTH<br/>AuthParameters: {<br/>  USERNAME: alice@example.com,<br/>  PASSWORD: ****<br/>}
        
        C-->>F: Auth Response
        Note over F: Response:<br/>ID Token: eyJhbGci...<br/>Access Token: eyJhbG...<br/>Refresh Token: eyJjdH...
        
        F->>I: GetId Request
        Note over F: Request:<br/>IdentityPoolId: us-west-2:67890xyz<br/>Logins: {<br/>  cognito-idp.us-west-2.amazonaws.com/us-west-2_abc123:<br/>  ID Token<br/>}
        
        I-->>F: GetId Response
        Note over F: Response:<br/>IdentityId: us-west-2:def456
        
        F->>I: GetCredentials Request
        Note over F: Request:<br/>IdentityId: us-west-2:def456<br/>Logins: {<br/>  cognito-idp.us-west-2.amazonaws.com/us-west-2_abc123:<br/>  ID Token<br/>}
        
        I-->>F: GetCredentials Response
        Note over F: Response:<br/>AccessKeyId: ASIA...<br/>SecretAccessKey: xyzABC...<br/>SessionToken: IQoJb3...<br/>Expiration: 1h
    end

    rect rgb(236, 253, 243)
        Note over U,D: Resource Access Flow
        
        U->>F: Request My Agents
        Note over F: Request:<br/>Path: /api/agents<br/>Method: GET
        
        F->>D: AWS SDK Request
        Note over F: Request:<br/>Operation: Query<br/>TableName: Agents<br/>KeyCondition: owner_id = d7e87e6d-8704...<br/>Credentials: {<br/>  AccessKeyId: ASIA...,<br/>  SecretAccessKey: xyzABC...,<br/>  SessionToken: IQoJb3...<br/>}
        
        D-->>F: Query Response
        Note over D: Response:<br/>Items: [{<br/>  agent_id: "agent_123",<br/>  owner_id: "d7e87e6d-8704...",<br/>  name: "My Agent",<br/>  status: "active"<br/>}]
        
        F-->>U: Display Agents
        Note over F: Response:<br/>Agents: [{<br/>  id: "agent_123",<br/>  name: "My Agent",<br/>  status: "active"<br/>}]
    end
```

## Token and Credential Management

### JWT Tokens
1. **ID Token**: Contains user identity
   ```json
   {
       "sub": "d7e87e6d-8704-4e91-b7f3-1234abcd",
       "email": "alice@example.com",
       "token_use": "id",
       "iss": "https://cognito-idp.us-west-2.amazonaws.com/us-west-2_abc123",
       "exp": 1704980400,
       "iat": 1704976800,
       "auth_time": 1704976800
   }
   ```

2. **Access Token**: Contains permissions
   ```json
   {
       "sub": "d7e87e6d-8704-4e91-b7f3-1234abcd",
       "device_key": "us-west-2_device_123xyz",
       "token_use": "access",
       "scope": "aws.cognito.signin.user.admin",
       "iss": "https://cognito-idp.us-west-2.amazonaws.com/us-west-2_abc123",
       "exp": 1704980400,
       "iat": 1704976800,
       "jti": "87e6d-8704-4e91-b7f3",
       "client_id": "client123xyz"
   }
   ```

3. **Refresh Token**: Used to obtain new access tokens
   ```json
   {
       "sub": "d7e87e6d-8704-4e91-b7f3-1234abcd",
       "device_key": "us-west-2_device_123xyz",
       "token_use": "refresh",
       "iss": "https://cognito-idp.us-west-2.amazonaws.com/us-west-2_abc123",
       "exp": 1705581600,
       "iat": 1704976800
   }
   ```

### AWS Credentials
```json
{
    "AccessKeyId": "ASIA...",
    "SecretAccessKey": "xyzABC...",
    "SessionToken": "IQoJb3...",
    "Expiration": 3600
}
```

### Cognito ClientId Considerations

#### When ClientId is Required

The Cognito App Client ID is only required for identity operations with Cognito:
- User registration (SignUp, ConfirmSignUp)
- User authentication (InitiateAuth)
- Password reset flows
- Token refresh operations

After successful authentication, subsequent operations use JWT tokens and temporary AWS credentials instead of ClientId.

#### Security Note

The ClientId is a public identifier (like an OAuth client ID) and not a secret:
- Can be safely exposed in frontend code
- Security is handled through other mechanisms (tokens, credentials, CORS, etc.)

#### Vercel Configuration

Configure ClientId in Vercel using environment variables:
1. In Vercel dashboard: Settings → Environment Variables
2. Add CLIENT_ID (e.g `USER_POOL_CLIENT_ID`) with your Cognito App Client ID
3. Deploy your application

## Access Control Implementation

#### Overview
The platform implements row-level security using Amazon Cognito and DynamoDB to ensure users can only access their own resources.

#### DynamoDB Access Control

1. **User Data Access**
   ```javascript
   {
       "Effect": "Allow",
       "Action": [
           "dynamodb:GetItem",
           "dynamodb:PutItem",
           "dynamodb:Query"
       ],
       "Resource": "arn:aws:dynamodb:${region}:${account}:table/Users",
       "Condition": {
           "ForAllValues:StringEquals": {
               "dynamodb:LeadingKeys": ["${aws:PrincipalTag/user_id}"]
           }
       }
   }
   ```
   - Users can only access their own profile using their Cognito user ID
   - The user_id is mapped from Cognito user attributes to PrincipalTag
   - Row-level security enforced through IAM policy conditions

2. **Agent Data Protection**
   ```javascript
   {
       "Effect": "Allow",
       "Action": [
           "dynamodb:GetItem",
           "dynamodb:PutItem",
           "dynamodb:Query",
           "dynamodb:UpdateItem",
           "dynamodb:DeleteItem"
       ],
       "Resource": "arn:aws:dynamodb:${region}:${account}:table/Agents",
       "Condition": {
           "ForAllValues:StringEquals": {
               "dynamodb:LeadingKeys": ["${aws:PrincipalTag/user_id}"]
           }
       }
   }
   ```
   - Agent records include owner_id matching Cognito user ID
   - Users can only access agents where they are the owner
   - All agent operations (create, read, update, delete) are protected

### Protection Mechanisms

1. **Identity Verification**
   - Every request requires a valid JWT token
   - Tokens contain the user's unique identifier
   - Tokens expire automatically after a set period
   - Invalid or expired tokens are immediately rejected

2. **Resource Ownership**
   - All resources (users, agents) have an owner field
   - Database queries automatically filter by owner
   - Ownership is verified before any modification
   - Unauthorized access attempts are logged
