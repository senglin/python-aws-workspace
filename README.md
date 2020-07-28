# An AWS Development Environment for Python

## Purpose
1. Provides a template for building Python Applications for AWS using Visual Studio Code, along with the associated tools.
2. A containerized development environment using the [Remote-Containers](https://code.visualstudio.com/docs/remote/containers) extension.

## Requirements
1. [Visual Studio Code](https://code.visualstudio.com/)
2. [Remote - Containers extension for Visual Studio Code](https://aka.ms/vscode-remote/download/extension)
3. [Docker](https://docs.docker.com/install/#supported-platforms)

## Installed Frameworks
1. AWS CLI
2. AWS SAM CLI
3. AWS Fargate CLI (forked version at [senglin/fargatecli](https://github.com/senglin/fargatecli))
4. AWS Toolkit for VS Code
5. Docker (Docker In Docker)
6. Git 
7. Python 3.8
8. Node v14.2.0
9. Ruby
10. Curl, Wget, Vim

## Developing inside a Container
1. Clone this repository or copy just the `.devcontainer` folder into your own local git repository folder. You can also run the following script in your terminal.
```bash
bash <(wget -qO- https://raw.githubusercontent.com/senglin/python-aws-workspace/master/install.sh)
```
2. Start VS Code, and in a new window, click on the quick actions Status Bar item in the lower left corner.
3. Select **Remote-Containers:Reopen in Container**.
4. From the menu, select **Terminal -> New Terminal**.  You can now use the installed tools to clone, develop and deploy to AWS.

## Expectation
1. Development in performed using a non-root user account (without sudo capabilities) - i.e. `vscode` user.
2. The container application inherits host's AWS Configuration. The  `~/.aws` folder is accessibled from the container.
3. The container app reuses ssh private key configuration from the host.  SSH-AGENT settings are forwarded to the container application.

## Variants of containerization
The behavior of the `docker` command in the container is configured via `VSC_DIND` environment variables, in `services.devcontainer.environment` sectopm of the `docker-compose.yml` file.
| Value     | Description                                                                                         |
|-----------|-----------------------------------------------------------------------------------------------------|
| `root`    | Docker CLI in container accesses host Docker Engine (root access required)                          |
| `nonroot` | Docker CLI in container accesses host Docker Eingine (nonroot access method)                        |
| `dind`    | Start Docker Engine within container when the container starts up. CLI uses container Docker Engine |
