# Service Account Setup Guide

This guide explains how to set up service account authentication for your Python microservice.

## Prerequisites

- Node.js and npm installed
- Python 3.7+ installed
- Python packages: `requests`, `pyjwt`

## Setup Steps

### 1. Configure Environment Variables

Add these variables to your `.env` file:

```
# Service JWT settings (choose a strong secret)
SERVICE_JWT_SECRET=your_secure_service_jwt_secret_key_here
SERVICE_JWT_EXPIRES_IN=30d
```

### 2. Run Database Migrations (if not already done)

```bash
npm run migration:up
```

### 3. Seed the Service Account and Roles

This command will:
- Create the 'admin', 'user', and 'service' roles if they don't exist
- Create a service account named 'python-microservice' 
- Generate a JWT token for the service account
- Save credentials to the `temp` directory

```bash
# Run only the service account seeder (recommended)
npm run seed:service

# Or run all seeders (including roles and service account)
npm run seed
```

### 4. Get the Generated Token

After running the seed command, check the generated file:

```bash
cat temp/service-account-token.txt
```

Copy the JWT token for use in your Python service.

### 5. Use the Token in Your Python Service

There are two ways to use the token:

#### Option 1: Environment Variable

```bash
export SERVICE_API_TOKEN="your_jwt_token_here"
python your_service.py
```

#### Option 2: Pass Directly to Your Client

```python
from service_client import ServiceAPIClient

client = ServiceAPIClient(
    base_url="http://localhost:3000",
    service_token="your_jwt_token_here"
)
```

## Testing the Integration

You can use the example Python client to test your integration:

```bash
# Copy the JWT token first
python docs/python-service-client.py "your_jwt_token_here"
```

## Security Considerations

- Store the JWT token securely (use environment variables or a secret manager)
- Do not commit tokens to source control
- Rotate tokens periodically
- Use HTTPS for all API communication

## Regenerating Tokens

If you need to regenerate the token for an existing service account:

```bash
npm run seed:service
```

This will generate a new token for the existing service account and save it to the temp directory. 