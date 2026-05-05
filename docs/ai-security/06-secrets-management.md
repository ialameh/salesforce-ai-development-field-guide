# S6. Secrets Management in AI-Assisted Projects

AI-generated code that includes callouts needs credentials. Credentials must not be hardcoded, must not appear in prompts, and must not be stored in version control. This chapter covers the Salesforce-specific secrets management patterns that work in AI-assisted development.

## The credential exposure failure

AI generates Apex callout code with hardcoded credentials:

```apex
// AI-generated (broken - credentials exposed)
HttpRequest req = new HttpRequest();
req.setEndpoint('https://api.example.com');
req.setHeader('Authorization', 'Bearer sk_prod_1234567890abcdef');
```

This happens because the AI defaults to the simplest credential pattern when you say "authenticate to the ERP system." The fix is to always specify Named Credentials as the auth mechanism and to never mention the actual credential value in any prompt.

## Named Credentials as the solution

Named Credentials are the Salesforce-native way to store callout credentials. They store the endpoint URL, authentication type, and credentials securely in the org's metadata. Apex code references Named Credentials by name, never by value.

```
// Correct pattern - uses Named Credentials by name
HttpRequest req = new HttpRequest();
req.setEndpoint('callout:ErpIntegrationNamedCredential/opportunities');
req.setMethod('POST');
// No Authorization header needed - Named Credential handles auth
```

Benefits:
- Credentials stored securely in org, not in code
- Not in version control
- Not visible in AI provider logs when the code is pasted in prompts
- Rotation by admin without code changes
- Works with OAuth, basic auth, and certificate auth

## Prompt for Named Credentials correctly

When prompting developer-AI to generate callout code:

```
Generate callout to: callout:ErpIntegrationNamedCredential/orders
Method: POST
Auth: Named Credential (ErpIntegrationNamedCredential) handles authentication automatically
No: do not include Authorization headers, do not mention any API keys or tokens
Use: req.setEndpoint('callout:ErpIntegrationNamedCredential/orders')
```

The "No:" instruction is critical. It tells the AI not to generate auth headers or credential values.

## Custom Metadata for configuration secrets

Beyond callout credentials, AI-assisted development may need configuration values that should not be in code:

- API rate limits
- Feature toggle values
- Environment-specific URLs
- Integration retry counts

Use Custom Metadata types to store these:

```xml
<!-- ErpIntegrationConfig__mdt -->
<fields>
    <field>
        <name>Max_Retries__c</name>
        <type>Number</type>
        <value>3</value>
    </field>
    <field>
        <name>Rate_Limit_Per_Minute__c</name>
        <type>Number</type>
        <value>100</value>
    </field>
</fields>
```

```
// AI prompt: "Read rate limit from Custom Metadata: ErpIntegrationConfig__mdt.Max_Retries__c"
// Do not say: "Use retry count of 3" (hardcoded)
```

## CI/CD secrets management

CI/CD pipelines need secrets (auth URLs, access tokens) for deployment. These must be stored in the CI/CD secrets store:

```
GitHub Actions:
- Settings -> Secrets and Variables -> Actions
- Add: SF_AUTH_URL_PROD (encrypted auth URL for production)
- Reference in workflow: ${{ secrets.SF_AUTH_URL_PROD }}

Bitbucket Pipelines:
- Repository settings -> Pipelines -> Secure variables
- Add: SF_AUTH_URL_PROD
- Reference in pipeline: $BITBUCKET_SECRET_SF_AUTH_URL_PROD
```

Never store secrets in pipeline YAML files or in version control. Never echo or log secret values in pipeline output.

## What NOT to do with secrets

```
Never:
- Paste a credential value into any AI tool prompt
- Store credentials in Custom Labels or Custom Settings (they are not encrypted)
- Store credentials in comments in code (AI might reference them)
- Put credentials in static variables or caches (they can be accessed via debug logs)
- Log credentials in System.debug statements or custom log objects
- Use the same credential in production and non-production orgs
```

## Verifying no secrets in code

Add a secret scanner to CI/CD:

```
GitHub Secret Scanning:
- GitHub automatically scans repositories for secrets
- Enable push protection: Blocks push if secrets are detected
- Add custom patterns for Salesforce-specific secrets (named credential names are not secrets, but auth tokens are)

Static analysis:
- PMD or ApexNav (Apex static analysis) can detect hardcoded strings that look like credentials
- Add static analysis to CI/CD pipeline to fail on hardcoded credential patterns
```

## What this chapter covered

- The credential exposure failure mode and how it happens
- Named Credentials as the solution for callout auth
- How to prompt correctly for Named Credentials (and what to forbid)
- Custom Metadata for configuration secrets (not hardcoded)
- CI/CD secrets management (GitHub Secrets, Bitbucket Pipelines variables)
- What NOT to do with secrets
- Verifying no secrets in code (GitHub Secret Scanning, static analysis)

## References

- [Named Credentials](https://developer.salesforce.com/docs/atlas.en-us.226.0/apex_code犬.content/apex_code_named_credentials.htm)
- [Custom Metadata Types](https://developer.salesforce.com/docs/atlas.en-us.226.0/apex_code犬.content/apex_code_custom_metadata.htm)
- [GitHub Secret Scanning](https://docs.github.com/en/code-security/secret-scanning)