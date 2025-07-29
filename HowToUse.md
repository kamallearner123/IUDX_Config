## 1. Prepare the Environment:
- Ensure Python 3.8+, kubectl, and helm are installed on the control node.
- Install required Python packages: pip install pyyaml psutil requests.
- Create the iudx_config.yaml file with the appropriate configuration.

## 2. Run the script:
- $python iudx_deployment.py
## 3. Script's functionality:
- Node Readiness: Checks CPU, memory, disk, and kubectl connectivity.
- Configuration Validation: Ensures all required configuration sections are present.
- Rancher/Kubernetes Deployment: Deploys Rancher using Helm and waits for cluster readiness.
- Component Deployment: Deploys immudb, PostgreSQL, and IUDX components (AAA server, Catalogue server) in parallel using Helm.
- Monitoring: Starts a background thread to monitor the health of components via an HTTP endpoint.
- Status Reporting: Logs the status of all components (Running, NotReady, Unknown).

## 4. Monitoring Module:
- The monitoring module periodically checks the specified endpoint (e.g., http://localhost:8080/status) and logs the health status.
- Customize the endpoint in the configuration file as needed.

## 5. Notes:
- IUDX Components: The script assumes Helm charts for IUDX components (AAA server, Catalogue server) are available. Replace iudx/aaa-server and iudx/catalogue-server with actual chart names or use kubectl apply for custom manifests if needed.
immudb and PostgreSQL: Deployed using Helm charts from their respective repositories (e.g., Bitnami for PostgreSQL).
- Error Handling: The script includes robust error handling and will abort if any critical step fails.
- Scalability: The ThreadPoolExecutor is used for parallel component deployment to reduce time.
- Monitoring: The monitoring module is basic; enhance it by integrating with Prometheus or Grafana for production use.
- References: The script aligns with IUDX deployment practices from GitHub repositories and Kubernetes deployment guides.github.comphoenixnap.comgithub.com
