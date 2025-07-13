# Service Account Authentication Guide

This guide explains how to authenticate your Python microservice with our NestJS API using service accounts and JWT tokens.

## Overview

Our system uses JWT (JSON Web Tokens) for service-to-service authentication. Your Python service will need to:

1. Request a service account token (done by administrators)
2. Use this token in authenticated requests

## Setup in NestJS Backend

An administrator must:

1. Create a service account through the admin interface
2. Generate a JWT token for your service 
3. Share this token with your Python service securely

## Python Client Implementation

Here's how to implement JWT authentication in your Python client:

```python
import requests
import jwt
import time
import os
from typing import Dict, Any, Optional

class APIClient:
    def __init__(self, base_url: str, service_token: Optional[str] = None):
        self.base_url = base_url.rstrip('/')
        self.service_token = service_token or os.environ.get('SERVICE_API_TOKEN')
        
        if not self.service_token:
            raise ValueError("Service token is required")
            
    def _get_headers(self) -> Dict[str, str]:
        """Create authorization headers with JWT token."""
        return {
            'Authorization': f'Bearer {self.service_token}',
            'Content-Type': 'application/json'
        }
    
    def make_request(self, method: str, path: str, data: Any = None) -> Dict[str, Any]:
        """Make an authenticated request to the API."""
        url = f"{self.base_url}/{path.lstrip('/')}"
        headers = self._get_headers()
        
        response = requests.request(
            method=method,
            url=url,
            headers=headers,
            json=data if data else None
        )
        
        response.raise_for_status()
        return response.json()
    
    # Example methods
    def get_projects(self) -> Dict[str, Any]:
        """Get all projects."""
        return self.make_request('GET', 'projects')
    
    def create_document(self, project_id: str, title: str, content: str) -> Dict[str, Any]:
        """Create a new document."""
        data = {
            'projectId': project_id,
            'title': title,
            'content': content
        }
        return self.make_request('POST', 'documents', data)


# Usage example
if __name__ == "__main__":
    # Get token from environment or configuration
    token = os.environ.get('SERVICE_API_TOKEN')
    
    client = APIClient(
        base_url="https://api.yourdomain.com",
        service_token=token
    )
    
    # Make authenticated requests
    try:
        projects = client.get_projects()
        print(f"Found {len(projects)} projects")
    except Exception as e:
        print(f"Error: {e}")
```

## Token Security

Best practices for handling the service token:

1. Never commit tokens to source control
2. Store tokens in environment variables or a secure secret manager
3. Rotate tokens periodically for security
4. Limit token permissions to only what's needed
5. Use HTTPS for all API communication

## Token Validation and Debugging

To validate a token or check its expiration:

```python
import jwt
from datetime import datetime

def decode_jwt(token: str) -> dict:
    """Decode a JWT token without verification to inspect its contents."""
    # Note: This doesn't verify the signature, just decodes the payload
    return jwt.decode(token, options={"verify_signature": False})

def is_token_expired(token: str) -> bool:
    """Check if a token is expired."""
    try:
        payload = decode_jwt(token)
        expiry = datetime.fromtimestamp(payload['exp'])
        return datetime.now() > expiry
    except (jwt.PyJWTError, KeyError):
        return True

# Example usage
token = os.environ.get('SERVICE_API_TOKEN')
payload = decode_jwt(token)
print(f"Token payload: {payload}")
print(f"Token expired: {is_token_expired(token)}")
```

## Rate Limiting

Be aware that service accounts may be subject to rate limiting. Implement appropriate retry logic and respect rate limits. 