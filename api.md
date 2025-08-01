# API Documentation

This document describes the API endpoints, request parameters, and response formats for the SoybeanAdmin project.

## Base Configuration

### Request Configuration
- **Base URL**: Configured via environment variable `VITE_SERVICE_BASE_URL`
- **Success Code**: `0000` (configurable via `VITE_SERVICE_SUCCESS_CODE`)
- **Authentication**: Bearer token in Authorization header
- **Content Type**: `application/json`

### Response Format
All API responses follow a standard format:

```typescript
interface Response<T = unknown> {
  /** Response code */
  code: string;
  /** Response message */
  msg: string;
  /** Response data */
  data: T;
}
```

### Error Handling
- **Logout Codes**: Configurable via `VITE_SERVICE_LOGOUT_CODES`
- **Modal Logout Codes**: Configurable via `VITE_SERVICE_MODAL_LOGOUT_CODES`
- **Expired Token Codes**: Configurable via `VITE_SERVICE_EXPIRED_TOKEN_CODES`

## Authentication APIs

### Login
**Endpoint**: `POST /auth/login`

**Request Parameters**:
```typescript
{
  userName: string;  // User name
  password: string;  // Password
}
```

**Response**:
```typescript
interface LoginToken {
  token: string;        // Access token
  refreshToken: string; // Refresh token
}
```

**Example**:
```json
// Request
{
  "userName": "admin",
  "password": "123456"
}

// Response
{
  "code": "0000",
  "msg": "Success",
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIs...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIs..."
  }
}
```

### Get User Info
**Endpoint**: `GET /auth/getUserInfo`

**Request Parameters**: None (requires Authorization header)

**Response**:
```typescript
interface UserInfo {
  userId: string;    // User ID
  userName: string;  // User name
  roles: string[];   // User roles
  buttons: string[]; // Available button permissions
}
```

**Example**:
```json
// Response
{
  "code": "0000",
  "msg": "Success",
  "data": {
    "userId": "1",
    "userName": "admin",
    "roles": ["admin", "user"],
    "buttons": ["add", "edit", "delete"]
  }
}
```

### Refresh Token
**Endpoint**: `POST /auth/refreshToken`

**Request Parameters**:
```typescript
{
  refreshToken: string; // Refresh token
}
```

**Response**:
```typescript
interface LoginToken {
  token: string;        // New access token
  refreshToken: string; // New refresh token
}
```

### Custom Backend Error
**Endpoint**: `GET /auth/error`

**Request Parameters**:
```typescript
{
  code: string; // Error code
  msg: string;  // Error message
}
```

**Response**: Custom error response for testing purposes

## Route APIs

### Get Constant Routes
**Endpoint**: `GET /route/getConstantRoutes`

**Request Parameters**: None

**Response**:
```typescript
interface MenuRoute {
  id: string;
  name: string;
  path: string;
  component: string;
  meta: {
    title: string;
    i18nKey?: string;
    icon?: string;
    order?: number;
    constant?: boolean;
    hideInMenu?: boolean;
    keepAlive?: boolean;
  };
  children?: MenuRoute[];
}

type Response = MenuRoute[];
```

### Get User Routes
**Endpoint**: `GET /route/getUserRoutes`

**Request Parameters**: None (requires Authorization header)

**Response**:
```typescript
interface UserRoute {
  routes: MenuRoute[];  // User-accessible routes
  home: string;        // Default home route key
}
```

**Example**:
```json
// Response
{
  "code": "0000",
  "msg": "Success",
  "data": {
    "routes": [
      {
        "id": "1",
        "name": "home",
        "path": "/home",
        "component": "layout.base$view.home",
        "meta": {
          "title": "Home",
          "i18nKey": "route.home",
          "icon": "mdi:monitor-dashboard",
          "order": 1
        }
      }
    ],
    "home": "home"
  }
}
```

### Check Route Existence
**Endpoint**: `GET /route/isRouteExist`

**Request Parameters**:
```typescript
{
  routeName: string; // Route name to check
}
```

**Response**:
```typescript
boolean // true if route exists, false otherwise
```

## Common Types

### Pagination
```typescript
interface PaginatingCommonParams {
  current: number; // Current page number
  size: number;    // Page size
  total: number;   // Total count
}

interface PaginatingQueryRecord<T = any> extends PaginatingCommonParams {
  records: T[]; // Data records
}

type CommonSearchParams = Pick<PaginatingCommonParams, 'current' | 'size'>;
```

### Common Record
```typescript
interface CommonRecord<T = any> {
  id: number;          // Record ID
  createBy: string;    // Creator
  createTime: string;  // Creation time
  updateBy: string;    // Updater
  updateTime: string;  // Update time
  status: '1' | '2' | null; // Status: 1=enabled, 2=disabled
}
```

### Enable Status
```typescript
type EnableStatus = '1' | '2'; // '1': enabled, '2': disabled
```

## Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `VITE_SERVICE_BASE_URL` | API base URL | - |
| `VITE_SERVICE_SUCCESS_CODE` | Success response code | `0000` |
| `VITE_SERVICE_LOGOUT_CODES` | Codes that trigger logout | - |
| `VITE_SERVICE_MODAL_LOGOUT_CODES` | Codes that show logout modal | - |
| `VITE_SERVICE_EXPIRED_TOKEN_CODES` | Codes for token expiration | - |
| `VITE_HTTP_PROXY` | Enable HTTP proxy in development | `Y` |

## Request Interceptors

### Authorization
All requests (except login) automatically include:
```
Authorization: Bearer <token>
```

### ApiFox Integration
Development requests include ApiFox token header:
```
apifoxToken: XL299LiMEDZ0H5h3A29PxwQXdMJqWyY2
```

## Error Codes

The system handles various error scenarios:

1. **Logout Codes**: Automatically logout and redirect to login page
2. **Modal Logout Codes**: Show logout confirmation modal
3. **Expired Token Codes**: Automatically refresh token and retry request
4. **Backend Errors**: Display error message to user

## Usage Notes

1. All timestamps are in ISO string format
2. User roles and buttons are arrays of strings
3. Route components use dot notation (e.g., `layout.base$view.home`)
4. Meta properties support internationalization via i18nKey
5. Routes can be nested with children array
6. Status fields use string values ('1', '2') not boolean