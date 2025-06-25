# Feature Request: Service Account Impersonation Support

**Date**: 25 June 2025  
**Status**: Proposed Enhancement  
**Priority**: Medium  

## Overview

Currently, the GCP MCP server relies exclusively on Application Default Credentials (ADC) configured via `gcloud auth application-default login`. This limits the application to using the permissions of the authenticated user's credentials. Adding service account impersonation support would enable more flexible and secure authentication patterns.

## Current Authentication Implementation

The application currently uses a simple GoogleAuth configuration:

```typescript
const auth = new GoogleAuth({
  scopes: ['https://www.googleapis.com/auth/cloud-platform']
});
```

This approach:
- Uses ADC (typically user credentials from `gcloud auth application-default login`)
- Has no support for assuming different identities
- Cannot leverage service account impersonation
- Limits security isolation and permission management

## Proposed Feature: Service Account Impersonation

### Benefits

1. **Enhanced Security**: Use service accounts with minimal required permissions instead of broad user credentials
2. **Better Access Control**: Different operations can use different service accounts with specific IAM roles
3. **Compliance**: Meets enterprise requirements for service-to-service authentication
4. **Audit Trail**: Service account usage provides clearer audit logs than user account usage
5. **Automation Friendly**: Enables better integration with CI/CD and automated workflows

### Technical Implementation

#### 1. Configuration Options

Add support for multiple configuration methods:

**Environment Variables:**
```bash
export GCP_IMPERSONATE_SERVICE_ACCOUNT="worker-sa@project-id.iam.gserviceaccount.com"
export GCP_IMPERSONATION_CHAIN="worker-sa@project-id.iam.gserviceaccount.com,another-sa@project-id.iam.gserviceaccount.com"
```

**Configuration File (optional):**
```json
{
  "authentication": {
    "impersonateServiceAccount": "worker-sa@project-id.iam.gserviceaccount.com",
    "impersonationChain": ["worker-sa@project-id.iam.gserviceaccount.com"],
    "delegateChain": []
  }
}
```

**Command Line Arguments:**
```bash
npx gcp-mcp --impersonate-service-account=worker-sa@project-id.iam.gserviceaccount.com
```

#### 2. Code Changes Required

**Update GoogleAuth initialization:**
```typescript
const initializeAuth = async (impersonateServiceAccount?: string) => {
  try {
    const authConfig: GoogleAuthOptions = {
      scopes: ['https://www.googleapis.com/auth/cloud-platform']
    };

    // Add impersonation if specified
    if (impersonateServiceAccount) {
      authConfig.impersonationChain = [impersonateServiceAccount];
    }

    const auth = new GoogleAuth(authConfig);
    return await retry(async () => await auth.getClient());
  } catch (error) {
    console.error('Failed to initialize authentication:', error);
    throw error;
  }
};
```

**Add configuration parsing:**
```typescript
interface AuthConfig {
  impersonateServiceAccount?: string;
  impersonationChain?: string[];
}

const getAuthConfig = (): AuthConfig => {
  return {
    impersonateServiceAccount: process.env.GCP_IMPERSONATE_SERVICE_ACCOUNT,
    impersonationChain: process.env.GCP_IMPERSONATION_CHAIN?.split(',')
  };
};
```

#### 3. MCP Server Configuration Updates

**Claude Desktop Configuration:**
```json
{
  "mcpServers": {
    "gcp": {
      "command": "npx",
      "args": ["-y", "gcp-mcp"],
      "env": {
        "GCP_IMPERSONATE_SERVICE_ACCOUNT": "worker-sa@project-id.iam.gserviceaccount.com"
      }
    }
  }
}
```

### Prerequisites for Users

To use service account impersonation, users need:

1. **Source Credentials**: ADC or service account key with impersonation permissions
2. **IAM Permissions**: The `roles/iam.serviceAccountTokenCreator` role on the target service account
3. **Target Service Account**: A service account with appropriate GCP resource permissions

**Example IAM Setup:**
```bash
# Grant impersonation permission to user
gcloud iam service-accounts add-iam-policy-binding \
    worker-sa@project-id.iam.gserviceaccount.com \
    --member="user:your-email@company.com" \
    --role="roles/iam.serviceAccountTokenCreator"

# Grant GCP permissions to the service account
gcloud projects add-iam-policy-binding project-id \
    --member="serviceAccount:worker-sa@project-id.iam.gserviceaccount.com" \
    --role="roles/compute.viewer"
```

### Error Handling and User Experience

1. **Clear Error Messages**: When impersonation fails, provide specific guidance:
   ```
   Error: Failed to impersonate service account 'worker-sa@project-id.iam.gserviceaccount.com'
   
   Possible causes:
   - Missing 'roles/iam.serviceAccountTokenCreator' permission
   - Service account does not exist
   - Invalid service account email format
   
   To grant impersonation permission:
   gcloud iam service-accounts add-iam-policy-binding worker-sa@project-id.iam.gserviceaccount.com --member="user:$(gcloud config get-value account)" --role="roles/iam.serviceAccountTokenCreator"
   ```

2. **Validation**: Validate service account email format before attempting impersonation

3. **Fallback Behavior**: If impersonation is configured but fails, decide whether to:
   - Fall back to ADC (with warning)
   - Fail completely (recommended for security)

### Documentation Updates

#### README.md Updates

Add new section after "GCP Setup":

```markdown
### Authentication Methods

#### Application Default Credentials (Default)
Set up application default credentials using `gcloud auth application-default login`

#### Service Account Impersonation (Recommended for Production)
For enhanced security, you can configure the MCP server to impersonate a service account:

1. Set up impersonation permissions:
```bash
gcloud iam service-accounts add-iam-policy-binding \
    your-service-account@project-id.iam.gserviceaccount.com \
    --member="user:$(gcloud config get-value account)" \
    --role="roles/iam.serviceAccountTokenCreator"
```

2. Configure impersonation via environment variable:
```bash
export GCP_IMPERSONATE_SERVICE_ACCOUNT="your-service-account@project-id.iam.gserviceaccount.com"
```

3. Or add to your MCP configuration:
```json
{
  "mcpServers": {
    "gcp": {
      "command": "npx",
      "args": ["-y", "gcp-mcp"],
      "env": {
        "GCP_IMPERSONATE_SERVICE_ACCOUNT": "your-service-account@project-id.iam.gserviceaccount.com"
      }
    }
  }
}
```
```

### Testing Strategy

1. **Unit Tests**: Test authentication initialization with and without impersonation
2. **Integration Tests**: Test actual GCP API calls using impersonated credentials
3. **Error Case Testing**: Test various failure scenarios (missing permissions, invalid service accounts)
4. **Documentation Testing**: Verify setup instructions work for new users

### Security Considerations

1. **Principle of Least Privilege**: Encourage users to create service accounts with minimal required permissions
2. **Credential Exposure**: Ensure impersonated credentials are not logged or exposed
3. **Permission Validation**: Consider adding a tool to validate that the impersonated service account has required permissions
4. **Audit Logging**: Document how impersonation affects GCP audit logs

### Backwards Compatibility

- Default behavior remains unchanged (uses ADC)
- New impersonation features are opt-in
- Existing configurations continue to work without modification
- No breaking changes to the MCP interface

### Implementation Priority

**Phase 1: Basic Impersonation**
- Environment variable support (`GCP_IMPERSONATE_SERVICE_ACCOUNT`)
- Basic error handling and validation
- Documentation updates

**Phase 2: Advanced Features**
- Impersonation chain support
- Configuration file support
- Command line argument support
- Enhanced error messages and troubleshooting

**Phase 3: Additional Security Features**
- Permission validation tools
- Multiple service account profiles
- Dynamic service account selection based on operation type

### Related Issues

This feature addresses common enterprise requirements:
- Service-to-service authentication
- Security compliance requirements
- Automated workflow integration
- Multi-tenant scenarios where different operations need different permissions

### Implementation Estimate

- **Development**: 2-3 days
- **Testing**: 1-2 days  
- **Documentation**: 1 day
- **Total**: 4-6 days

### Alternative Solutions

1. **Service Account Key Files**: Support for key file authentication (less secure, not recommended)
2. **Workload Identity**: For Kubernetes environments (specific use case)
3. **External Credential Sources**: Support for external identity providers (complex implementation)

## Conclusion

Adding service account impersonation support would significantly enhance the security and flexibility of the GCP MCP server while maintaining backwards compatibility. The implementation is straightforward using existing GoogleAuth library capabilities, and the feature would enable enterprise adoption scenarios that are currently not possible.
