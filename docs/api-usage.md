# Akkuea REST API Documentation

## Table of Contents

- [Quick Start](#quick-start)
- [Authentication](#authentication)
- [Base Configuration](#base-configuration)
- [Authentication Endpoints](#authentication-endpoints)
- [Users Endpoints](#users-endpoints)
- [Properties Endpoints](#properties-endpoints)
- [Lending Endpoints](#lending-endpoints)
- [KYC Endpoints](#kyc-endpoints)
- [Notifications Endpoints](#notifications-endpoints)
- [Oracle Endpoints](#oracle-endpoints)
- [Error Handling](#error-handling)
- [Rate Limiting](#rate-limiting)
- [Common Workflows](#common-workflows)

---

## Quick Start

### Base URL

```
https://api.akkuea.com
```

For local development:

```
http://localhost:3001
```

## Authentication

The API uses a **Stellar wallet challenge-response** flow to issue JWTs.

### 1. Request a challenge (nonce)

```bash
curl -X POST http://localhost:3001/auth/challenge \
  -H "Content-Type: application/json" \
  -d '{"stellarAddress": "GBXGQJWVLWOYHFLVTKWV5FGHA3LNYY2JQKM7OAJAUEQFU6LPCSEFVXON"}'
```

**Response:**
```json
{
  "nonce": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "expiresAt": 1714006800000
}
```

### 2. Sign the nonce and create a session

```bash
curl -X POST http://localhost:3001/auth/session \
  -H "Content-Type: application/json" \
  -d '{
    "stellarAddress": "GBXGQJWVLWOYHFLVTKWV5FGHA3LNYY2JQKM7OAJAUEQFU6LPCSEFVXON",
    "signature": "base64-encoded-ed25519-signature"
  }'
```

**Response:**
```json
{
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "user": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "walletAddress": "GBXGQJWVLWOYHFLVTKWV5FGHA3LNYY2JQKM7OAJAUEQFU6LPCSEFVXON",
    "displayName": "Alice"
  }
}
```

### 3. Use the JWT on protected requests

Include the token in the `Authorization` header:

```bash
curl -H "Authorization: Bearer <token>" http://localhost:3001/users/me
```

---

## Health

```bash
curl http://localhost:3001/health
```

**Response:**
```json
{
  "status": "healthy",
  "timestamp": "2026-06-24T12:00:00.000Z",
  "version": "1.0.0",
  "services": {
    "database": {
      "healthy": true,
      "latency": 3
    }
  }
}
```

---

## Auth (`/auth`)

### POST `/auth/challenge` — Get a nonce to sign

```bash
curl -X POST http://localhost:3001/auth/challenge \
  -H "Content-Type: application/json" \
  -d '{"stellarAddress": "GBXGQJWVLWOYHFLVTKWV5FGHA3LNYY2JQKM7OAJAUEQFU6LPCSEFVXON"}'
```

**Response (`200`):**
```json
{
  "nonce": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "expiresAt": 1714006800000
}
```

| Status | Error | Description |
|---|---|---|
| 400 | `BAD_REQUEST` | Invalid Stellar address format |
| 429 | `RATE_LIMITED` | Too many requests |

### POST `/auth/session` — Verify signature, receive JWT

```bash
curl -X POST http://localhost:3001/auth/session \
  -H "Content-Type: application/json" \
  -d '{
    "stellarAddress": "GBXGQJWVLWOYHFLVTKWV5FGHA3LNYY2JQKM7OAJAUEQFU6LPCSEFVXON",
    "signature": "base64-encoded-ed25519-signature"
  }'
```

**Response (`200`):**
```json
{
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "user": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "walletAddress": "GBXGQJWVLWOYHFLVTKWV5FGHA3LNYY2JQKM7OAJAUEQFU6LPCSEFVXON",
    "displayName": "Alice"
  }
}
```

| Status | Error | Description |
|---|---|---|
| 400 | `BAD_REQUEST` | Invalid Stellar address or missing signature |
| 401 | `UNAUTHORIZED` | Challenge not found, expired, or signature invalid |
| 429 | `RATE_LIMITED` | Too many requests |

---

## Properties (`/properties`)

### GET `/properties` — List properties (paginated, filterable)

```bash
curl "http://localhost:3001/properties?page=1&limit=10&propertyType=residential&country=US"
```

**Response (`200`):**
```json
{
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440001",
      "name": "Sunset Villa",
      "description": "A luxury beachfront property in Malibu.",
      "propertyType": "residential",
      "location": {
        "address": "123 Ocean Drive",
        "city": "Malibu",
        "country": "US",
        "postalCode": "90265"
      },
      "totalValue": "2500000.00",
      "totalShares": 10000,
      "availableShares": 6500,
      "pricePerShare": "250.00",
      "images": ["https://storage.akkuea.com/properties/sunset-villa-1.jpg"],
      "verified": true,
      "listedAt": "2026-01-15T10:00:00.000Z",
      "owner": "GBXGQJWVLWOYHFLVTKWV5FGHA3LNYY2JQKM7OAJAUEQFU6LPCSEFVXON"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 10,
    "total": 42,
    "totalPages": 5
  }
}
```

| Query Parameter | Type | Description |
|---|---|---|
| `page` | number | Page number (default 1) |
| `limit` | number | Items per page (default 20) |
| `propertyType` | enum | `residential`, `commercial`, `industrial`, `land`, `mixed` |
| `country` | string | Filter by country code |
| `minPrice` | number | Minimum total value |
| `maxPrice` | number | Maximum total value |
| `verified` | boolean | Filter by verification status |

### GET `/properties/:id` — Get a single property

```bash
curl http://localhost:3001/properties/550e8400-e29b-41d4-a716-446655440001
```

**Response (`200`):**
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440001",
  "name": "Sunset Villa",
  "description": "A luxury beachfront property in Malibu.",
  "propertyType": "residential",
  "location": {
    "address": "123 Ocean Drive",
    "city": "Malibu",
    "country": "US"
  },
  "totalValue": "2500000.00",
  "totalShares": 10000,
  "availableShares": 6500,
  "pricePerShare": "250.00",
  "images": [],
  "verified": true,
  "listedAt": "2026-01-15T10:00:00.000Z",
  "owner": "GBXGQJWVLWOYHFLVTKWV5FGHA3LNYY2JQKM7OAJAUEQFU6LPCSEFVXON"
}
```

| Status | Error | Description |
|---|---|---|
| 404 | `NOT_FOUND` | Property does not exist |

### POST `/properties` — Create a property (JWT required)

```bash
curl -X POST http://localhost:3001/properties \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{
    "name": "Downtown Office Tower",
    "description": "Class A office space in the financial district.",
    "propertyType": "commercial",
    "location": {
      "address": "100 Market Street",
      "city": "San Francisco",
      "country": "US"
    },
    "totalValue": "5000000.00",
    "totalShares": 20000,
    "pricePerShare": "250.00",
    "images": ["https://storage.akkuea.com/properties/office-tower-1.jpg"]
  }'
```

**Response (`201`):**
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440002",
  "name": "Downtown Office Tower",
  "description": "Class A office space in the financial district.",
  "propertyType": "commercial",
  "location": {
    "address": "100 Market Street",
    "city": "San Francisco",
    "country": "US"
  },
  "totalValue": "5000000.00",
  "totalShares": 20000,
  "availableShares": 20000,
  "pricePerShare": "250.00",
  "images": ["https://storage.akkuea.com/properties/office-tower-1.jpg"],
  "verified": false,
  "listedAt": "2026-06-24T12:00:00.000Z",
  "owner": "GBXGQJWVLWOYHFLVTKWV5FGHA3LNYY2JQKM7OAJAUEQFU6LPCSEFVXON"
}
```

| Status | Error | Description |
|---|---|---|
| 400 | `BAD_REQUEST` | Validation error in request body |
| 401 | `UNAUTHORIZED` | Missing or invalid JWT |

### POST `/properties/:id/buy-shares` — Buy property shares (JWT required)

```bash
curl -X POST http://localhost:3001/properties/550e8400-e29b-41d4-a716-446655440001/buy-shares \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{
    "buyer": "GBXGQJWVLWOYHFLVTKWV5FGHA3LNYY2JQKM7OAJAUEQFU6LPCSEFVXON",
    "shares": 10
  }'
```

**Response (`200`):**
```json
{
  "transactionHash": "a1b2c3d4e5f67890abcdef1234567890abcdef1234567890abcdef1234567890",
  "newBalance": 10
}
```

| Status | Error | Description |
|---|---|---|
| 400 | `BAD_REQUEST` | Insufficient shares available or invalid input |
| 401 | `UNAUTHORIZED` | Missing or invalid JWT |
| 404 | `NOT_FOUND` | Property does not exist |

---

## Lending (`/lending`)

### GET `/lending/pools` — List lending pools

```bash
curl "http://localhost:3001/lending/pools?page=1&limit=20&isActive=true"
```

**Response (`200`):**
```json
{
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440010",
      "name": "USDC Liquidity Pool",
      "asset": "USDC",
      "assetAddress": "GBXGQJWVLWOYHFLVTKWV5FGHA3LNYY2JQKM7OAJAUEQFU6LPCSEFVXON",
      "totalDeposits": "1000000.0000000",
      "totalBorrows": "650000.0000000",
      "availableLiquidity": "350000.0000000",
      "utilizationRate": "65.00",
      "supplyAPY": "5.20",
      "borrowAPY": "8.50",
      "collateralFactor": "0.75",
      "liquidationThreshold": "0.80",
      "liquidationPenalty": "0.05",
      "reserveFactor": 1000,
      "isActive": true,
      "isPaused": false,
      "createdAt": "2026-01-01T00:00:00.000Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 3,
    "totalPages": 1
  }
}
```

| Query Parameter | Type | Description |
|---|---|---|
| `page` | number | Page number (default 1) |
| `limit` | number | Items per page (default 20) |
| `asset` | string | Filter by asset symbol (e.g. `USDC`) |
| `isActive` | boolean | Filter by active status |

### GET `/lending/pools/:id` — Get a single pool

```bash
curl http://localhost:3001/lending/pools/550e8400-e29b-41d4-a716-446655440010
```

**Response (`200`):**
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440010",
  "name": "USDC Liquidity Pool",
  "asset": "USDC",
  "assetAddress": "GBXGQJWVLWOYHFLVTKWV5FGHA3LNYY2JQKM7OAJAUEQFU6LPCSEFVXON",
  "totalDeposits": "1000000.0000000",
  "totalBorrows": "650000.0000000",
  "availableLiquidity": "350000.0000000",
  "utilizationRate": "65.00",
  "supplyAPY": "5.20",
  "borrowAPY": "8.50",
  "collateralFactor": "0.75",
  "liquidationThreshold": "0.80",
  "liquidationPenalty": "0.05",
  "reserveFactor": 1000,
  "isActive": true,
  "isPaused": false,
  "createdAt": "2026-01-01T00:00:00.000Z"
}
```

### POST `/lending/pools/:id/deposit` — Deposit into a pool (JWT required)

```bash
curl -X POST http://localhost:3001/lending/pools/550e8400-e29b-41d4-a716-446655440010/deposit \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{"amount": "5000.00"}'
```

**Response (`201`):**
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440020",
  "poolId": "550e8400-e29b-41d4-a716-446655440010",
  "depositorId": "550e8400-e29b-41d4-a716-446655440000",
  "amount": "5000.0000000",
  "shares": "5000.0000000",
  "depositedAt": "2026-06-24T12:00:00.000Z",
  "lastAccrualAt": "2026-06-24T12:00:00.000Z",
  "accruedInterest": "0.0000000"
}
```

### POST `/lending/pools/:id/withdraw` — Withdraw from a pool (JWT required)

```bash
curl -X POST http://localhost:3001/lending/pools/550e8400-e29b-41d4-a716-446655440010/withdraw \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{"amount": "1000.00"}'
```

**Response (`200`):**
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440020",
  "poolId": "550e8400-e29b-41d4-a716-446655440010",
  "depositorId": "550e8400-e29b-41d4-a716-446655440000",
  "amount": "4000.0000000",
  "shares": "4000.0000000",
  "depositedAt": "2026-06-24T12:00:00.000Z",
  "lastAccrualAt": "2026-06-24T12:00:00.000Z",
  "accruedInterest": "12.3400000"
}
```

### POST `/lending/pools/:id/borrow` — Borrow from a pool (JWT required)

```bash
curl -X POST http://localhost:3001/lending/pools/550e8400-e29b-41d4-a716-446655440010/borrow \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{
    "borrowAmount": "2000.00",
    "collateralAmount": "3000.00",
    "collateralAsset": "GBXGQJWVLWOYHFLVTKWV5FGHA3LNYY2JQKM7OAJAUEQFU6LPCSEFVXON"
  }'
```

**Response (`201`):**
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440030",
  "poolId": "550e8400-e29b-41d4-a716-446655440010",
  "borrowerId": "550e8400-e29b-41d4-a716-446655440000",
  "principal": "2000.0000000",
  "accruedInterest": "0.0000000",
  "collateralAmount": "3000.0000000",
  "collateralAsset": "GBXGQJWVLWOYHFLVTKWV5FGHA3LNYY2JQKM7OAJAUEQFU6LPCSEFVXON",
  "healthFactor": "1.5000",
  "borrowedAt": "2026-06-24T12:00:00.000Z",
  "lastAccrualAt": "2026-06-24T12:00:00.000Z"
}
```

| Status | Error | Description |
|---|---|---|
| 400 | `BAD_REQUEST` | Validation error or insufficient collateral |
| 401 | `UNAUTHORIZED` | Missing or invalid JWT |
| 404 | `NOT_FOUND` | Pool does not exist |

---

## KYC (`/kyc`)

### GET `/kyc/status/:userId` — Get KYC status (JWT + ownership required)

```bash
curl -H "Authorization: Bearer <token>" \
  http://localhost:3001/kyc/status/550e8400-e29b-41d4-a716-446655440000
```

**Response (`200`):**
```json
{
  "status": "verified",
  "documents": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440040",
      "userId": "550e8400-e29b-41d4-a716-446655440000",
      "type": "passport",
      "fileName": "passport.pdf",
      "fileUrl": "https://storage.akkuea.com/kyc/passport.pdf",
      "status": "approved",
      "uploadedAt": "2026-01-20T10:00:00.000Z",
      "reviewedAt": "2026-01-21T14:00:00.000Z",
      "documentUrl": "/kyc/file/550e8400-e29b-41d4-a716-446655440040"
    }
  ]
}
```

| Status | Error | Description |
|---|---|---|
| 401 | `UNAUTHORIZED` | Missing or invalid JWT |
| 403 | `FORBIDDEN` | Cannot access another user's KYC data |

### POST `/kyc/upload` — Upload a KYC document (JWT required, multipart/form-data)

```bash
curl -X POST http://localhost:3001/kyc/upload \
  -H "Authorization: Bearer <token>" \
  -F "file=@passport.pdf" \
  -F "documentType=passport"
```

**Response (`201`):**
```json
{
  "documentId": "550e8400-e29b-41d4-a716-446655440040",
  "submissionId": "550e8400-e29b-41d4-a716-446655440000"
}
```

| Status | Error | Description |
|---|---|---|
| 400 | `BAD_REQUEST` | Invalid file type or missing fields |
| 401 | `UNAUTHORIZED` | Missing or invalid JWT |

---

## Notifications (`/notifications`)

### GET `/notifications` — List user notifications (JWT required)

```bash
curl -H "Authorization: Bearer <token>" \
  "http://localhost:3001/notifications?limit=10&offset=0"
```

**Response (`200`):**
```json
{
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440050",
      "userId": "550e8400-e29b-41d4-a716-446655440000",
      "eventType": "VERIFICATION_APPROVED",
      "title": "KYC Approved",
      "message": "Your identity verification has been approved.",
      "channel": "IN_APP",
      "isRead": false,
      "createdAt": "2026-01-21T14:00:00.000Z",
      "updatedAt": "2026-01-21T14:00:00.000Z"
    }
  ],
  "pagination": {
    "limit": 10,
    "offset": 0
  }
}
```

### PATCH `/notifications/:id/read` — Mark a notification as read (JWT required)

```bash
curl -X PATCH http://localhost:3001/notifications/550e8400-e29b-41d4-a716-446655440050/read \
  -H "Authorization: Bearer <token>"
```

**Response (`200`):**
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440050",
  "userId": "550e8400-e29b-41d4-a716-446655440000",
  "eventType": "VERIFICATION_APPROVED",
  "title": "KYC Approved",
  "message": "Your identity verification has been approved.",
  "channel": "IN_APP",
  "isRead": true,
  "readAt": "2026-06-24T12:00:00.000Z",
  "createdAt": "2026-01-21T14:00:00.000Z",
  "updatedAt": "2026-06-24T12:00:00.000Z"
}
```

| Status | Error | Description |
|---|---|---|
| 401 | `UNAUTHORIZED` | Missing or invalid JWT |
| 404 | `NOT_FOUND` | Notification does not exist |

### POST `/notifications/read-all` — Mark all notifications as read (JWT required)

```bash
curl -X POST http://localhost:3001/notifications/read-all \
  -H "Authorization: Bearer <token>"
```

**Response (`200`):**
```json
{
  "message": "3 notifications marked as read",
  "count": 3
}
```

---

## Error Response Format

All errors follow a consistent structure defined in `apps/api/src/utils/errors.ts`:

```json
{
  "success": false,
  "error": "ERROR_CODE",
  "message": "Human-readable description of the error",
  "statusCode": 400,
  "timestamp": "2026-06-24T12:00:00.000Z"
}
```

### Error Classes

| HTTP Status | Error Code | Class | Description |
|---|---|---|---|
| 400 | `BAD_REQUEST` | `BadRequestError` | Invalid input, malformed request body |
| 401 | `UNAUTHORIZED` | `UnauthorizedError` | Missing, invalid, or expired JWT |
| 403 | `FORBIDDEN` | `ForbiddenError` | Insufficient permissions |
| 404 | `NOT_FOUND` | `NotFoundError` | Resource does not exist |
| 422 | `VALIDATION_ERROR` | `ApiError.validation` | Schema validation failure |
| 429 | `RATE_LIMITED` | — | Too many requests (rate limited) |
| 500 | `INTERNAL_ERROR` | `AppError` | Unexpected server error |

### Validation Error Example (400)

```json
{
  "success": false,
  "error": "Validation Error",
  "message": "Invalid input parameters",
  "statusCode": 400,
  "timestamp": "2026-06-24T12:00:00.000Z"
}
```

### Authentication Error Example (401)

```json
{
  "success": false,
  "error": "UNAUTHORIZED",
  "message": "Authentication required",
  "statusCode": 401,
  "timestamp": "2026-06-24T12:00:00.000Z"
}
```

### Not Found Error Example (404)

```json
{
  "success": false,
  "error": "NOT_FOUND",
  "message": "Property not found",
  "statusCode": 404,
  "timestamp": "2026-06-24T12:00:00.000Z"
}
```
