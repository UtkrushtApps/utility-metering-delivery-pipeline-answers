# Solution Steps

1. Replace the existing Jenkinsfile with a change-aware pipeline that: (1) detects which files changed in the current commit, (2) maps those changes to impacted services, and (3) stores the impacted service list for later stages.

2. Implement impact detection rules: if any file under shared/ changed, mark all four services as impacted; otherwise mark only the service whose directory under services/<service>/ changed.

3. Tag all Docker images with the exact current commit SHA (e.g., docker build -t <service>:${COMMIT_SHA} ...). Never use a moving tag like latest for CI builds.

4. Build only the impacted services, and run those builds concurrently using Jenkins parallel. Each parallel branch should build exactly one service image.

5. Restrict deployments to the main branch only (when { branch 'main' }). Add a manual input gate so releases are controlled rather than automatic.

6. Use the controller-managed secret store for the deployment target: bind the sandbox-id credential to SANDBOX_ID via withCredentials([string(credentialsId: 'sandbox-id', ...)]). The deploy_script.py already reads SANDBOX_ID from the environment.

7. Deploy only the impacted services, calling deploy_script.py with the commit SHA as the image/tag value and with the correct compose tag environment variable name for each service (METER_INGEST_TAG, USAGE_RATING_TAG, BILLING_EXPORT_TAG, CUSTOMER_ALERT_TAG).

8. Verify behavior with three scenarios: (1) change only one service -> only its image rebuilds and deploys; (2) change shared/ -> all four images rebuild and redeploy; (3) changing multiple independent services -> Jenkins shows the build stages running in parallel.

