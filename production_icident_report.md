## Root cause analysis
The issue seems to be caused by occasional failures in the "app3" container., which is part of the application running behind HAProxy.
Initial investigations indicate that the failure may be related to resource exhaustion, container configuration, or issues with how HAProxy is handling container health checks and load balancing.

The issue was traced to a hardcoded logic in the "app3" container's application code that triggers a 503 Service Unavailable error when a specific environment variable is set. Upon setting the environment variable for "app3," the application fails and returns a 503 HTTP status code. This issue led to the intermittent failure observed in the HAProxy status report, where the "app3" container would fail after three requests and then return to a healthy state.
```
// Health endpoint
const healthHandler = (): Response => {
  if (appName === "app3") {   
    healthCheckFailCount++;

    if (healthCheckFailCount % 3 === 0) {
      console.log(`${appName} - Health check failed`);
      return new Response("Service Unavailable", { status: 503 });
    }
  }
  return new Response("OK");
};


// Main request handler
const handler = (req: Request): Response => {
  console.log(`${appName} - Request received: ${req.url}`);
  if (req.url.indexOf('/health') > -1) {
    return healthHandler();
  }

  if (appName === "app3") {
    return new Response("Internal Server Error", { status: 500 });
  }
```

## Incident Summary
 The "app3" container fails intermittently with a 503 Service Unavailable error when an environment variable is set to APP_NAME=app3. This failure results in some production machines becoming unresponsive, which in turn affects the customer experience and leads to service disruptions. The issue was caused by hardcoded behavior in the application code, where the environment variable triggers a failure.
 
## Impact Assessment
- **Affected Services:** "app3" container and its associated services.
- **Production Impact:** Some production machines are unresponsive due to the 503 error being returned, leading to service disruptions.
- **Customer Impact:** Customers experience delays or disruptions in service availability as a result of the container returning 503 errors.
  
## Incident Timeline

### Detection
The issue was first identified when the HAProxy status page indicated that the "app3" container was marked as unhealthy after returning multiple 503 errors following every third request. The failure pattern was intermittent, which made it challenging to immediately pinpoint the root cause.

### Escalation
After further investigation, the issue was escalated to the development  team once it was found that the problem was related to the environment variable being set, which caused the application to return 503 errors.

### Mitigation
A temporary workaround was implemented by unsetting the problematic environment variable in the container, which restored normal functionality. The application returned to a healthy state, and production machines were no longer unresponsive.

### Resolution
The root cause was identified as hardcoded behavior in the application, where setting the environment variable caused the application to intentionally return a 503 error. The application code was updated to handle this scenario more gracefully without causing a service failure. The environment variable handling was reviewed, and the code was modified to ensure that setting the variable no longer results in an error.
 
##  Actions Taken
- **HAProxy Configuration:** Checked HAProxy configuration for any issues with timeouts or request handling. Adjusted the timeout settings to ensure that requests are given enough time to be processed.
- **Container Logs and Network:** Investigated the container logs to check for any errors or misconfigurations. All services were found to be running on the same `bridge` network, which did not appear to be causing the issue.
- **Dockerfile Inspection:** Reviewed the Dockerfile to investigate further.
- The application code was updated to handle the environment variable more robustly, ensuring it does not result in a 503 error.
- HAProxy configurations were checked and validated to ensure proper health check intervals and retry behavior.
- **Temporary workaround will be implemented until a bug fix is raised:**
  
     **Solution 1:** <br>
     **Started the application with a different app name or Remove enviroment variable for app3**: The container was redeployed with a modified configuration for app name  = app_3 to temporarily bypass the issue related to the environment variable or we can remove the enviroment variable which help to start with  'unknown' as per code support this intiaizaltion [when enviroment variable is not set].
  
  You can view the related pull request [here] <br>
      **Start with differnt enviromemnt variable name for app3 :  (https://github.com/username/repository/pull/PR_number). <br>
      **Remove enviroment variable: (https://github.com/username/repository/pull/PR_number).
  
     **Solution 2:** <br>
     **Removed all affected containers (`app3` replicas) from the HAProxy load balancer**: Since the application is running with three replicas same docker file, all containers using the affected enviroment variable [app3] were temporarily removed from the load balancer to prevent them from receiving traffic. This maintained high availability by ensuring that only healthy containers continued serving requests.
 You can view the related pull request [here](https://github.com/username/repository/pull/PR_number).
  
##  Follow-Up Actions
 **Engage with the development team**: Connect with the responsible development team to showcase the issue, explain the impact on production, and request a bug fix to resolve the hardcoded environment variable logic in the application code. This will prevent future occurrences of the 503 error when the environment variable is set.
- **Code Review:** Conduct a thorough review of environment variable handling in the application to prevent similar issues in the future.
- **Testing and Validation:** Implement unit and integration tests for environment variable handling to ensure that no hardcoded errors occur when variables are set.
- **Monitoring and Alerts:** Enhance monitoring to detect configuration issues with environment variables before they impact production.
- **Documentation:** Update application documentation to clearly define how environment variables should be configured and handled in the production environment.

## Lessons Learned
- **Configuration Management:** Avoid hardcoding behavior based on environment variables, as this can lead to unexpected service failures. Implement more flexible handling and validation of environment variables.
- **Proactive Monitoring:** Implement more comprehensive monitoring on environment variable configurations to detect potential issues early.
- **Clear Documentation:** Ensure proper documentation and guidelines are in place for managing configuration settings, including environment variables, to prevent misconfigurations.
