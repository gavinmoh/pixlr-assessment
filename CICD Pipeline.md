# Challenge 4: CI/CD Pipeline Design

In this scenario, I will assume that the web application is built using a JavaScript framework (e.g., Next.js, create-react-app), and the pipeline deploys to an existing Kubernetes deployment named `deployment/web-app`.

## Tools Selection

Nowadays, most popular Git repository services offer comprehensive solutions as CI/CD platforms. In many cases, third-party CI/CD platforms such as Jenkins are unnecessary. In this scenario, I will use GitHub Actions for the following reasons:

- GitHub is the most popular Git repository service, so there's a good chance most team members are familiar with it.
- It seamlessly integrates with GitHub, simplifying setup and eliminating the need for external tools.
- GitHub Actions offers a generous amount of free minutes and storage, even for private repositories, making it a cost-effective choice.

- **Version Control**: GitHub
- **CI/CD Platform**: GitHub Actions
- **Container Registry**: GitHub Container Registry
- **Testing Framework**: Jest for unit testing and Playwright for end-to-end testing
- **Vulnerability Scanner**: Trivy

## Pipeline Stages

1. **PR Merged**
	- A pull request is merged into the master branch, triggering a GitHub Action workflow.

2. **Test**
	- This stage sets up a test environment, including a database service.
	- It executes unit testing followed by end-to-end testing.
	- For languages such as JavaScript, Ruby, and Python, there's no need to go through the build stage to compile the code.
	- Test results and code coverage reports are generated and stored in GitHub Artifacts for review purposes.

3. **Build**
	- This stage sets up the environment for building the Docker image. Environment variables and secrets are set up based on the environment defined in the GitHub repository settings.
	- It compiles a production build.
	- It builds and pushes a Docker image to GitHub Container Registry.
	- The Docker image is scanned for vulnerabilities and security issues using Trivy.
4. **Deployment**
	- Updates the Kubernetes deployment using `kubectl set image deployment/web-app web-app=<latest version tag/sha256>`.
	- Rolls out the deployment using `kubectl rollout restart deployment/web-app`.

### Rollbacks

When defining the GitHub Action workflow, each stage is set to depend on the previous stage, ensuring that if a stage fails, the pipeline won't proceed. Notifications with the workflow results are provided, allowing appropriate actions to be taken.

```yml 
# .github/workflows/pipeline.yml 
jobs: 
	test: 
    # test job definition
	build: 
		needs: [test] 
		# build job definition
	deploy: 
		needs: [build] 
    # deploy job definition
```

When running `kubectl rollout restart deployment/web-app` in the deployment stage, Kubernetes creates pods and ensures the new versions are healthy before directing traffic to them and removing the previous versions. If a pod fails to start or fails a health check, no traffic is directed to it, acting as a failsafe for the deployment. Kubernetes doesn't offer blue/green deployment out of the box. For this strategy, manual intervention or additional tools like Spinnaker are needed.

After deployment, if necessary, rollbacks can be performed in several ways:
- Git revert and push to GitHub to trigger the pipeline. Although time-consuming, this method, combined with branch protection rules, requires a PR for the rollback, serving as documentation and an audit trail. It's fully automated and requires no intervention from a DevOps engineer.
- Rollback by issuing `kubectl rollout undo deployment/web-app`. Kubernetes maintains revision history of deployments, allowing for rollback to previous revisions. A DevOps engineer can execute this command, or a GitHub Action workflow triggered by `workflow_dispatch` event can be defined, enabling developers to perform the rollback directly in GitHub.

## Security Considerations

**Docker Image Scanning**: We incorporate scanning in the build stage using Trivy. The pipeline fails if the vulnerability scan doesn't pass, preventing deployment of vulnerable code.

**Secret Management**: GitHub environment is used in the build stage to securely manage secrets such as API keys, credentials and access tokens. For runtime secrets and environment variables required by the web application, Kubernetes Secret or an external secret management system can be used.

**Access Control**: Access to resources and secrets is controlled by service accounts and RBAC in Kubernetes. We create a service account with the necessary permissions to update the deployment and use it in the GitHub Action workflow. This practice ensures that only the necessary resources are accessed and that the pipeline is secure.

In conclusion, I have outlined a CI/CD pipeline that automates the test, build, and deployment process for a Dockerized web application. Leveraging GitHub Actions, with its tight integration with GitHub, simplifies setup and enhances security through practices like Docker image scanning, secret management, and role-based access control.