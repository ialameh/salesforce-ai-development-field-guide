# P5. Prompts for CI/CD

AI-assisted Salesforce CI/CD has two distinct failure modes: AI generating pipeline configuration that is insecure, and AI generating deployment configurations that break in production. This chapter covers how to prompt for CI/CD pipelines that are safe, reliable, and maintainable.

## What breaks AI-generated CI/CD

AI generates pipeline YAML that looks correct and has these common problems:

1. **Missing approval gates** — the pipeline deploys to staging and production without manual approval
2. **No scratch org validation** — deployments go directly to the authenticated org without testing in a fresh org
3. **Destructive changes without review** — destructiveChanges.xml is auto-applied
4. **No test enforcement** — tests run but failures do not block deployment
5. **Secrets in pipeline variables** — API keys or credentials stored in plaintext pipeline variables
6. **Production deploy on main branch push** — main branch merges auto-deploy to production without review

## CI/CD prompt constraints

```
Must:
- Manual approval gate between staging and production
- Run all tests (not just a subset) before any deployment
- Use a scratch org for integration testing, not the same org that runs CI
- Validate destructiveChanges.xml with a human review step before applying
- Store secrets in encrypted pipeline secrets (GitHub Secrets, Bitbucket Pipelines variables), never in plaintext
- Fail the pipeline if any test class has < 75% coverage (Salesforce minimum)
- Use sf project deploy validate before deploying to any org

Must not:
- Auto-deploy to production on main branch push
- Use password or API key in pipeline YAML (use secrets reference)
- Deploy destructiveChanges.xml without explicit approval
- Skip tests for faster pipeline run (tests are the safety net)
- Deploy to production orgs from personal machines (only from CI/CD)
```

## Pipeline generation prompts

```
You are developer-AI. Write a GitHub Actions CI/CD pipeline for a Salesforce project.

Project: <name>
Org type: <scratch org / sandbox / production>
Deployment flow: feature branch -> PR -> dev org -> staging -> production

Pipeline requirements:
1. Trigger: push to feature branches (test and build), PR open/change (validate), main branch push (deploy to staging)
2. Jobs:
   - lint-and-test: run sfdx-project.yml validations, Apex static analysis, test execution
   - deploy-to-dev: sf project deploy start --target-org dev --validate-only (dry run)
   - deploy-to-staging: manual approval required, then sf project deploy start --target-org staging
   - deploy-to-production: manual approval required, then sf project deploy start --target-org production

Secrets management:
- Store sfdx auth URL as GitHub Actions secret: SF_AUTH_URL_DEV, SF_AUTH_URL_STAGING, SF_AUTH_URL_PROD
- Reference secrets in pipeline: ${{ secrets.SF_AUTH_URL_DEV }}
- Never echo or log secrets values

Approval gates:
- staging deployment requires GitHub environment protection rule with required reviewers
- production deployment requires GitHub environment protection rule with 2 required reviewers

Test requirements:
- Run: sf project deploy validate --test-level RunLocalTests
- Pipeline fails if any test fails or if coverage drops below 75%
- Generate test result summary as CI artifact

Write the complete .github/workflows/sf-ci.yml file.
```

## Deployment configuration prompts

```
You are developer-AI. Write sf project deploy configuration for <org type>.

Org: <dev scratch / staging / production>

Configuration requirements:
1. deployment: <directory or package version>
2. test: RunLocalTests (required for production deploys)
3. ignore-warnings: false (do not ignore warnings in production)
4. purge-on-delete: false (for destructive deploys, require confirmation)
5. Allow missing members: false (do not skip missing components in a package)

For destructive deployments:
1. Validate destructiveChanges.xml separately before applying
2. Require manual approval for destructive deploys (do not auto-apply)
3. Run a test deploy to a scratch org first before applying destructiveChanges to production

Write the deploy-config.json or command syntax for sf project deploy start.
```

## Scratch org creation in CI prompts

```
You are developer-AI. Write a pipeline job that creates and uses a scratch org for integration testing.

Job: scratch-org-test

Steps:
1. Authenticate to dev hub: sf org login access-token --instance-url https://login.salesforce.com
2. Create scratch org: sf org create scratch --definition-file config/project-scratch-def.json --alias ci-test --set-default --wait 10
3. Push source: sf project deploy start --source-dir force-app --target-org ci-test
4. Run tests: sf project deploy validate --test-level RunLocalTests --target-org ci-test
5. Delete scratch org: sf org delete scratch --target-org ci-test --no-prompt

Constraint: The scratch org must be deleted even if the tests fail (add finally block or continue-on-error: false)
Timeout: Set job timeout to 30 minutes (scratch org creation can take 3-5 minutes)

Write the pipeline job YAML.
```

## What this chapter covered

- The five CI/CD failure modes for AI-generated pipelines
- The non-negotiable CI/CD constraints
- Prompts for pipeline generation, deployment configuration, and scratch org CI jobs
- Why manual approval gates are not optional

## References

- [Salesforce CI/CD Best Practices](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_guide/sfdx_cli_userguide_org_commands.htm)
- [GitHub Actions Security](https://docs.github.com/en/actions/security-guides/secrets-management)
- [Salesforce Scratch Orgs](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_guide/sfdx_cli_userguide_scratch_orgs.htm)