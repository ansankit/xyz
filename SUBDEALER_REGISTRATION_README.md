# Subdealer Registration Feature - Implementation Guide

## Overview

This document provides step-by-step instructions for setting up and testing the subdealer registration feature. The feature allows subdealers to register themselves by providing their phone number and GST details, with OTP verification via SMS.

## Quick Summary

**Route**: `/new-registration-subdealer` (unprotected, public access)

**Flow**:
1. User enters 10-digit phone number → Validates and proceeds
2. User enters 15-character GST number → Validates and enables "Fetch GST Details" button
3. System fetches GST details from API → Prefills form fields
4. User clicks "Generate OTP" → OTP sent via SMS (or logged in dev mode)
5. User enters 6-digit OTP → Verifies and creates subdealer record in database
6. Success message displayed with registration details

**Key Files**:
- Frontend: `apps/web/app/new-registration-subdealer/page.tsx`
- Form Component: `apps/web/components/subdealer-registration-form.tsx`
- Backend Controller: `apps/api/src/controllers/subdealer.controller.ts`
- Routes: `apps/api/src/routes/subdealer.routes.ts`
- Services: `apps/api/src/services/gst.service.ts`, `apps/api/src/services/sms.service.ts`

## Features Implemented

- **Progressive Form Flow**: Multi-step registration form with progressive field revelation
- **Phone Number Validation**: 10-digit Indian phone number validation
- **GST Number Validation**: 15-character alphanumeric GST number validation
- **GST API Integration**: Mock GST API service (ready to replace with real API)
- **OTP Verification**: SMS-based OTP verification using Gupshup
- **Database Storage**: Complete subdealer information storage with verification status

## Next Steps

### 1. Database Migration

Run the Prisma migration to create the new database tables:

```bash
cd packages/db
npx prisma migrate dev --name add_subdealer_models
npx prisma generate
```

This will create two new tables:
- `subdealers` - Stores subdealer information
- `subdealer_otps` - Stores OTP records for verification

### 2. Environment Variables Setup

Create or update the `apps/api/.env` file with the following environment variables:

```env
# Gupshup SMS Configuration
GUPSHUP_SMS_API_KEY=your_gupshup_sms_api_key_here
GUPSHUP_SMS_APP_ID=your_app_id_here
GUPSHUP_SMS_SENDER_ID=GUPSHUP

# GST API Configuration (gstincheck.co.in)
# API Endpoint: https://sheet.gstincheck.co.in/check/{api-key}/{gstin-number}
# Get your API key from: https://gstincheck.co.in/
GST_API_KEY=your_gst_api_key_here
# Alternative variable names (if you use a different name):
# GSTIN_CHECK_API_KEY=your_gst_api_key_here
# GSTINCHECK_API_KEY=your_gst_api_key_here

# Required: Database connection (if not already set)
DATABASE_URL=postgresql://user:password@localhost:5432/dbname
DIRECT_URL=postgresql://user:password@localhost:5432/dbname

# Required: Encryption key for secure storage (if not already set)
ENCRYPTION_KEY=your_32_character_encryption_key_here
```

**Important Notes**: 
- **GST API Key**: Add `GST_API_KEY` to your `.env` file. The service will use the real API if the key is present, otherwise it falls back to mock data (development only).
- Make sure the `.env` file is in the `apps/api/` directory
- Never commit `.env` files to version control
- The `ENCRYPTION_KEY` should be a 32-character string for AES-256-GCM encryption
- In development mode, if GST API fails, it will automatically fall back to mock data

#### Getting Gupshup SMS API Key and App ID

1. Sign up for a Gupshup account at https://www.gupshup.io/
2. Log in to your dashboard
3. Navigate to API section → Create App
   - Provide an app name
   - This creates a unique App ID
4. Generate API Key:
   - Go to API section in dashboard
   - Create/generate your API key
   - This key authenticates your requests
5. Get Sender ID:
   - Use default 'GUPSHUP' or request a custom sender ID
6. Add to `.env` file:
   - `GUPSHUP_SMS_API_KEY` - Your API key
   - `GUPSHUP_SMS_APP_ID` - Your App ID (from step 3)
   - `GUPSHUP_SMS_SENDER_ID` - Sender ID (default: GUPSHUP)

#### Getting GST API Key (gstincheck.co.in)

1. Visit https://gstincheck.co.in/
2. Sign up for an account or log in
3. Navigate to API section or dashboard
4. Generate or copy your API key
5. Add it to your `.env` file as `GST_API_KEY`

**API Endpoint Format**: `https://sheet.gstincheck.co.in/check/{api-key}/{gstin-number}`

**Note**: If the API key is not set, the system will use mock data in development mode. In production, make sure to set the API key.

**Note**: In development mode, if the SMS API key is not configured, OTPs will be logged to the console instead of being sent via SMS.

### 3. Optional: Configure SMS Provider in Database

Alternatively, you can configure the SMS API key through the Integration Manager in the UI:

1. Go to `/integration-manager` in your application
2. Add a new integration with provider type `sms`
3. Enter your Gupshup API key (it will be encrypted and stored securely)

### 4. Start the Application

```bash
# Start the backend API
cd apps/api
npm run dev

# Start the frontend (in a new terminal)
cd apps/web
npm run dev
```

### 5. Access the Registration Page

Navigate to the subdealer registration page:
```
http://localhost:3000/new-registration-subdealer
```

## API Endpoints

### 1. Fetch GST Details

**Endpoint**: `POST /api/subdealer/fetch-gst`

**Request Body**:
```json
{
  "gstNumber": "27AABCU9603R1ZM"
}
```

**Response**:
```json
{
  "success": true,
  "data": {
    "legalName": "Sample Business",
    "tradeName": "Trade Name",
    "address": "123 Business Street",
    "city": "Mumbai",
    "state": "Maharashtra",
    "pincode": "110001",
    "panNumber": "AABCU9603R",
    "registrationDate": "2020-01-15T00:00:00.000Z",
    "businessType": "Private Limited Company",
    "status": "Active",
    "jurisdiction": "Maharashtra - Ward 1"
  }
}
```

**Rate Limit**: 20 requests per 15 minutes per GST number

### 2. Generate OTP

**Endpoint**: `POST /api/subdealer/generate-otp`

**Request Body**:
```json
{
  "phone": "9876543210"
}
```

**Response**:
```json
{
  "success": true,
  "message": "OTP sent successfully"
}
```

**Rate Limit**: 5 requests per 15 minutes per phone number

### 3. Verify OTP and Register

**Endpoint**: `POST /api/subdealer/verify-otp`

**Request Body**:
```json
{
  "phone": "9876543210",
  "otp": "123456",
  "gstDetails": {
    "gstNumber": "27AABCU9603R1ZM",
    "legalName": "Sample Business",
    "tradeName": "Trade Name",
    "address": "123 Business Street",
    "city": "Mumbai",
    "state": "Maharashtra",
    "pincode": "110001",
    "panNumber": "AABCU9603R",
    "registrationDate": "2020-01-15T00:00:00.000Z",
    "businessType": "Private Limited Company",
    "status": "Active",
    "jurisdiction": "Maharashtra - Ward 1",
    "email": "optional@example.com"
  }
}
```

**Response**:
```json
{
  "success": true,
  "message": "Subdealer registered successfully",
  "data": {
    "id": 1,
    "phone": "9876543210",
    "gstNumber": "27AABCU9603R1ZM",
    "legalName": "Sample Business"
  }
}
```

**Rate Limit**: 10 requests per 10 minutes per phone number

## Quick Start Checklist

- [ ] Run database migration: `npx prisma migrate dev --name add_subdealer_models`
- [ ] Generate Prisma client: `npx prisma generate`
- [ ] Set up environment variables in `apps/api/.env`
- [ ] Configure Gupshup SMS API key (or use dev mode for console logging)
- [ ] Start backend server: `cd apps/api && npm run dev`
- [ ] Start frontend server: `cd apps/web && npm run dev`
- [ ] Navigate to `http://localhost:3000/new-registration-subdealer`
- [ ] Test the complete registration flow

## Testing Guide

### Manual Testing Steps

1. **Test Phone Number Validation**
   - Enter a phone number with less than 10 digits → Should show error
   - Enter exactly 10 digits → Should auto-advance to step 2
   - Enter a phone number starting with 0-5 → Should show error

2. **Test GST Number Validation**
   - Enter a GST number with less than 15 characters → Should show error
   - Enter exactly 15 characters → "Fetch GST Details" button should be enabled
   - Enter invalid characters → Should show error

3. **Test GST Details Fetch**
   - Enter a valid 15-character GST number
   - Click "Fetch GST Details"
   - Verify that GST details are prefilled in the form

4. **Test OTP Generation**
   - After GST details are fetched, click "Generate OTP"
   - Check console (in dev mode) or SMS for OTP
   - Verify that step 4 (OTP input) appears

5. **Test OTP Verification**
   - Enter the OTP received
   - Click "Verify OTP & Register"
   - Verify success message appears

6. **Test Duplicate Prevention**
   - Try registering with the same phone number twice → Should show error
   - Try registering with the same GST number twice → Should show error

### Testing with cURL

```bash
# 1. Fetch GST Details
curl -X POST http://localhost:4000/api/subdealer/fetch-gst \
  -H "Content-Type: application/json" \
  -d '{"gstNumber": "27AABCU9603R1ZM"}'

# 2. Generate OTP
curl -X POST http://localhost:4000/api/subdealer/generate-otp \
  -H "Content-Type: application/json" \
  -d '{"phone": "9876543210"}'

# 3. Verify OTP (use the OTP from console/SMS)
curl -X POST http://localhost:4000/api/subdealer/verify-otp \
  -H "Content-Type: application/json" \
  -d '{
    "phone": "9876543210",
    "otp": "123456",
    "gstDetails": {
      "gstNumber": "27AABCU9603R1ZM",
      "legalName": "Test Business",
      "tradeName": "Test Trade",
      "address": "123 Test St",
      "city": "Mumbai",
      "state": "Maharashtra",
      "pincode": "400001",
      "panNumber": "AABCU9603R",
      "registrationDate": "2020-01-15",
      "businessType": "Private Limited",
      "status": "Active",
      "jurisdiction": "Maharashtra - Ward 1"
    }
  }'
```

## Database Schema

### Subdealer Table

| Field | Type | Description |
|-------|------|-------------|
| id | Int | Primary key |
| phone | String (unique) | 10-digit phone number |
| gstNumber | String (unique) | 15-character GST number |
| email | String? | Optional email address |
| legalName | String | Legal business name |
| tradeName | String? | Trade name |
| address | String? | Business address |
| city | String? | City |
| state | String? | State |
| pincode | String? | Pincode |
| panNumber | String? | PAN number |
| registrationDate | DateTime? | GST registration date |
| businessType | String? | Type of business |
| status | String? | GST status |
| jurisdiction | String? | GST jurisdiction |
| phoneVerified | Boolean | Phone verification status |
| verifiedAt | DateTime? | Verification timestamp |
| createdAt | DateTime | Creation timestamp |
| updatedAt | DateTime | Last update timestamp |

### SubdealerOTP Table

| Field | Type | Description |
|-------|------|-------------|
| id | Int | Primary key |
| subdealerId | Int? | Reference to subdealer (nullable until verification) |
| phone | String | Phone number |
| otpHash | String | Hashed OTP |
| expiresAt | DateTime | OTP expiration time |
| attempts | Int | Number of verification attempts |
| usedAt | DateTime? | When OTP was used |
| createdAt | DateTime | Creation timestamp |
| updatedAt | DateTime | Last update timestamp |

## GST API Integration

### Current Implementation

The system is now integrated with **gstincheck.co.in** GST API. 

**API Endpoint**: `https://sheet.gstincheck.co.in/check/{api-key}/{gstin-number}`

**Behavior**:
- If `GST_API_KEY` is set in `.env`, it uses the real API
- If API key is not set, it falls back to mock data (development mode only)
- If API fails in development mode, it falls back to mock data
- In production, API errors will be thrown (no fallback)

### How It Works

1. User enters GST number
2. System validates format (15 characters)
3. If `GST_API_KEY` is configured:
   - Makes API call to `https://sheet.gstincheck.co.in/check/{api-key}/{gstin-number}`
   - Maps API response to our `GstDetails` interface
   - Returns real GST data
4. If API key is not configured:
   - Returns mock data based on GST number pattern
   - Allows testing without API setup

### API Response Mapping

The service maps the API response fields:
- `lgnm` → `legalName`
- `tradeNam` → `tradeName`
- `adr[0]` → address components (building, street, locality, etc.)
- `rgdt` → `registrationDate`
- `ctb` → `businessType`
- `sts` → `status`
- `stj` → `jurisdiction`

### Switching to a Different GST API Provider

If you want to use a different GST API provider:

1. **Update GST Service** (`apps/api/src/services/gst.service.ts`):

```typescript
async fetchGstDetails(gstNumber: string): Promise<GstDetails> {
  const normalized = gstNumber.trim().toUpperCase();
  
  // Replace with your actual GST API call
  const response = await fetch(`${process.env.GST_API_URL}/gst/${normalized}`, {
    method: 'GET',
    headers: {
      'Authorization': `Bearer ${process.env.GST_API_KEY}`,
      'Content-Type': 'application/json',
    },
  });
  
  if (!response.ok) {
    throw new Error('Failed to fetch GST details');
  }
  
  const data = await response.json();
  
  // Map the API response to GstDetails interface
  return {
    legalName: data.legal_name || data.legalName,
    tradeName: data.trade_name || data.tradeName,
    address: data.address,
    city: data.city,
    state: data.state,
    pincode: data.pincode,
    panNumber: data.pan || data.panNumber,
    registrationDate: data.registration_date || data.registrationDate,
    businessType: data.business_type || data.businessType,
    status: data.status,
    jurisdiction: data.jurisdiction,
  };
}
```

2. **Add GST API Environment Variables**:
```env
GST_API_URL=https://your-gst-api-url.com
GST_API_KEY=your_gst_api_key
```

## Development Mode

### SMS OTP in Development

When `NODE_ENV=development` and the Gupshup API key is not configured or SMS sending fails, the system will:
1. Log the OTP to the console: `[DEV MODE] OTP for {phone}: {otp}`
2. Still return success to allow testing without SMS service
3. Store the OTP in the database as normal

**Example Console Output**:
```
[DEV MODE] OTP for 9876543210: 123456
```

This allows you to test the complete flow without setting up SMS service immediately.

## Troubleshooting

### OTP Not Being Sent

1. **Check Environment Variables**: Verify `GUPSHUP_SMS_API_KEY` is set correctly
2. **Check Console Logs**: In development mode, OTPs are logged to console
3. **Check Gupshup Account**: Ensure your Gupshup account has SMS credits
4. **Check Phone Number Format**: Ensure phone number is in correct format (10 digits)

### GST Details Not Fetching

1. **Check GST Number Format**: Must be exactly 15 characters, alphanumeric
2. **Check API Response**: Verify the mock API is returning data correctly
3. **Check Console Logs**: Look for error messages in the browser console

### Database Errors

1. **Run Migrations**: Ensure you've run `npx prisma migrate dev`
2. **Check Database Connection**: Verify DATABASE_URL in `.env` is correct
3. **Check Prisma Client**: Run `npx prisma generate` to regenerate Prisma client

### Rate Limiting Issues

If you hit rate limits during testing:
- OTP generation: Wait 15 minutes between requests for the same phone number
- GST fetching: Wait if you've exceeded 20 requests in 15 minutes
- OTP verification: Maximum 10 attempts per 10 minutes per phone number

## Security Considerations

1. **OTP Expiration**: OTPs expire after 10 minutes
2. **Attempt Limiting**: Maximum 5 verification attempts per OTP
3. **Rate Limiting**: All endpoints have rate limiting to prevent abuse
4. **Input Validation**: All inputs are validated on both client and server
5. **Data Encryption**: Sensitive data like OTPs are hashed before storage

## Future Enhancements

1. **Email Verification**: Add email verification alongside phone verification
2. **Admin Dashboard**: Create an admin interface to view/manage subdealers
3. **Real GST API**: Replace mock GST API with actual GST API integration
4. **Document Upload**: Add capability to upload business documents
5. **Status Management**: Add approval/rejection workflow for subdealer registrations
6. **Notifications**: Send email/SMS notifications on registration status changes

## Support

For issues or questions:
1. Check the troubleshooting section above
2. Review the API endpoint documentation
3. Check console logs for error messages
4. Verify all environment variables are set correctly

## File Structure

```
apps/
├── api/
│   ├── src/
│   │   ├── controllers/
│   │   │   └── subdealer.controller.ts
│   │   ├── routes/
│   │   │   └── subdealer.routes.ts
│   │   ├── services/
│   │   │   ├── gst.service.ts
│   │   │   └── sms.service.ts
│   │   └── utils/
│   │       └── validators.ts (updated)
├── web/
│   ├── app/
│   │   └── new-registration-subdealer/
│   │       └── page.tsx
│   ├── components/
│   │   └── subdealer-registration-form.tsx
│   └── lib/
│       ├── api/
│       │   ├── services.ts (updated)
│       │   └── types.ts (updated)
│       └── validation/
│           └── subdealer.ts
packages/
└── db/
    └── prisma/
        └── schema.prisma (updated)
```

