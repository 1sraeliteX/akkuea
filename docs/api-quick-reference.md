# Akkuea REST API - Quick Reference

## Base Information

```
Base URL: https://api.akkuea.com
Local Dev: http://localhost:3001
Auth: Bearer JWT Token (from Stellar wallet signature)
```

## Authentication Flow

```
1. POST /auth/challenge          → Get nonce to sign
2. Sign nonce with Stellar wallet
3. POST /auth/session           → Submit signed challenge, get JWT
4. Use JWT in Authorization header for protected endpoints
```

## Endpoint Summary

### Auth
| Method | Endpoint | Auth | Purpose |
|--------|----------|------|---------|
| POST | `/auth/challenge` | No | Get challenge nonce |
| POST | `/auth/session` | No | Verify signature & get JWT |

### Users
| Method | Endpoint | Auth | Purpose |
|--------|----------|------|---------|
| POST | `/users` | No | Create new user |
| GET | `/users/:id` | No | Get user by ID |
| GET | `/users/wallet/:address` | No | Get user by wallet |
| GET | `/users/me` | Yes | Get current user |
| PATCH | `/users/me` | Yes | Update current user |

### Properties
| Method | Endpoint | Auth | Purpose |
|--------|----------|------|---------|
| GET | `/properties` | No | List properties (filterable) |
| GET | `/properties/:id` | No | Get property details |
| POST | `/properties` | Yes | Create property |
| PUT | `/properties/:id` | Yes | Update property |
| DELETE | `/properties/:id` | Yes | Delete property |
| POST | `/properties/:id/buy-shares` | Yes | Purchase shares |
| GET | `/properties/:id/shares/:owner` | No | Get user shares |

### Lending
| Method | Endpoint | Auth | Purpose |
|--------|----------|------|---------|
| GET | `/lending/pools` | No | List pools |
| GET | `/lending/pools/:id` | No | Get pool details |
| POST | `/lending/pools/:id/deposit` | Yes | Deposit funds |
| POST | `/lending/pools/:id/withdraw` | Yes | Withdraw funds |
| POST | `/lending/pools/:id/borrow` | Yes | Borrow with collateral |
| POST | `/lending/pools/:id/repay` | Yes | Repay loan |
| GET | `/lending/pools/:id/user/:address/summary` | No | Get position summary |

### KYC
| Method | Endpoint | Auth | Purpose |
|--------|----------|------|---------|
| GET | `/kyc/status/:userId` | Yes | Get KYC status |
| POST | `/kyc/upload` | Yes | Upload KYC document |
| POST | `/kyc/submit` | Yes | Submit KYC verification |
| GET | `/kyc/documents/:userId` | Yes | List user documents |
| POST | `/kyc/verify/:documentId` | Internal | Verify document (admin) |

### Notifications
| Method | Endpoint | Auth | Purpose |
|--------|----------|------|---------|
| GET | `/notifications` | Yes | Get notifications |
| GET | `/notifications/unread-count` | Yes | Get unread count |
| GET | `/notifications/:id` | Yes | Get notification |
| PATCH | `/notifications/:id/read` | Yes | Mark as read |
| POST | `/notifications/read-multiple` | Yes | Mark multiple as read |
| POST | `/notifications/read-all` | Yes | Mark all as read |
| DELETE | `/notifications/:id` | Yes | Delete notification |

### Oracle
| Method | Endpoint | Auth | Purpose |
|--------|----------|------|---------|
| POST | `/oracle/valuations` | No | Ingest valuation |
| GET | `/oracle/valuations/:propertyId` | No | Get latest valuation |
| GET | `/oracle/valuations/:propertyId/history` | No | Get valuation history |

## HTTP Status Codes

| Code | Error Code | Meaning |
|------|-----------|---------|
| 200 | - | OK |
| 201 | - | Created |
| 400 | BAD_REQUEST | Invalid input |
| 401 | UNAUTHORIZED | Missing/invalid token |
| 403 | FORBIDDEN | Permission denied |
| 404 | NOT_FOUND | Resource not found |
| 429 | RATE_LIMITED | Too many requests |
| 500 | INTERNAL_ERROR | Server error |

## Rate Limits

- **Challenge/Session**: 5 req/min per IP
- **Property Listing**: 10 req/min per user
- **Lending Operations**: 20 req/min per user
- **KYC Upload**: 5 req/min per user

## Common cURL Headers

```bash
# With authentication
-H "Authorization: Bearer $TOKEN"

# Content type
-H "Content-Type: application/json"

# File upload
-F "file=@/path/to/file"
```

## Example: Complete Auth Flow

```bash
# 1. Get challenge
curl -X POST http://localhost:3001/auth/challenge \
  -H "Content-Type: application/json" \
  -d '{"stellarAddress":"GBR..."}'

# 2. Sign with wallet and get JWT
curl -X POST http://localhost:3001/auth/session \
  -H "Content-Type: application/json" \
  -d "{\"stellarAddress\":\"GBR...\",\"signature\":\"SIGNED_CHALLENGE\"}"

# 3. Use JWT
curl http://localhost:3001/users/me \
  -H "Authorization: Bearer $TOKEN"
```

## Error Response Format

```json
{
  "success": false,
  "error": "ERROR_CODE",
  "message": "Description",
  "statusCode": 400,
  "timestamp": "2025-06-27T10:30:45.123Z"
}
```

## Development Tips

- Store JWT in environment variable: `export TOKEN="..."`
- Use `jq` for JSON formatting: `curl ... | jq`
- Use `-v` flag for verbose output
- Use `-X` to specify HTTP method
- Use `-d` for request body data
- Use `-F` for file uploads

