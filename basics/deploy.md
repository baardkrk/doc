# Deploy your application

This section will take you through the deployment of your application, using the _NAIS deploy_ tool.

NAIS deploy enables you to deploy your application into [any cluster](../#nais-clusters) from any continuous integration platform, including GitHub Actions, CircleCI, Travis CI and Jenkins.

Note: Using _naisd_ or _JIRA Autodeploy_ to deploy your application? These mechanisms are deprecated and are going to be shut down. See [migration guide from naisd to Naiserator](../legacy/migrating-from-naisd.md). All information on the current page relates to Naiserator compatible `nais.yaml` files. You can also read the [naisd user documentation](../legacy/naisd.md).

### How it works

1. Create a deployment request using [deployment-cli](https://github.com/navikt/deployment-cli) in your build pipeline. This request is sent to Github's [deployment API](https://developer.github.com/v3/repos/deployments/) and is then forwarded to NAIS deploy.
2. NAIS deploy verifies the integrity and authenticity of the deployment, assumes the identity of the deploying team, and applies your _Kubernetes resources_ into the specified [cluster](../#nais-clusters).
3. If you deployed any _Application_ or _Deployment_ resources, NAIS deploy will wait until these are rolled out successfully, or a timeout occurs.
4. The deployment status is continually posted back to Github and is available through their API, enabling integration with your pipeline or monitoring setup.

### Summary

1. Make sure the [prerequisites](deploy.md#prerequisites) are met before attempting to use NAIS deploy.
2. [Enable deployments for your repository](deploy.md#enable-deployments-for-your-repository).
3. [Implement NAIS deploy in your pipeline](deploy.md#performing-the-deployment).

When things break, check the section on [troubleshooting](deploy.md#troubleshooting). For CircleCI/manual you also need to [obtain Github deployment credentials](deploy.md#obtain-github-deployment-credentials).

## Getting started

### Prerequisites

* Create a [nais.yaml](../nais-application/manifest.md) file for any application you want to deploy. A `nais.yaml` file contains an _Application resource_. The application resource provides NAIS with the necessary information to run your application. If starting out for the first time, try the [nais.yaml minimal example](../nais-application/min-example.md).
* The repository containing `nais.yaml` must be on Github.
* Be a maintainer of a [Github team](https://help.github.com/en/articles/about-teams). The team name must be the same as your Kubernetes _team label_, and additionally must have write access to your repository.
* Secure your Github repository by restricting write access to team members. Activating NAIS deploy for your repository will enable anyone with write access to your repository to deploy on behalf of your team. This is a _major security concern_ and should not be overlooked.

### Enable deployments for your repository

NAIS deploy must authorize your Github repository before you can perform deployments. This must be done once per repository. When authorizing, you can pick one or more teams to deploy as. Any permissions these teams have in the NAIS cluster will be granted to the deployment pipeline. In order to do this, you need to have _maintainer_ access rights to your Github team, and _admin_ access to the repository.

When you're ready, go to the [registration portal](https://deployment.prod-sbs.nais.io/auth/login) and follow the instructions.

### Obtain Github deployment credentials \(for CircleCI/manual only\)

The self-service way of obtaining credentials is to [create a personal access token](https://help.github.com/en/articles/creating-a-personal-access-token-for-the-command-line) that your CI pipeline can use to trigger the deployment. The token needs only the `repo_deployment` scope.

If you want to use Github App as credentials you can head on over to [utvikling/Oppsett av Github App](https://github.com/navikt/utvikling/blob/master/Oppsett%20av%20Github%20App.md) for documentation.

### Performing the deployment

At this point, you have:

* met the prerequisites
* enabled deployments
* obtained credentials \(for CircleCI/manual only\)

You are ready to perform a deployment.

#### Using GitHub Actions

The easiest way of deploying your application to NAIS is using GitHub Action, and we have created a workflow that uses the deployment-cli, see [navikt/deployment-cli](https://github.com/navikt/deployment-cli/tree/master/action). This example workflow uses Github Package Registry that \(for now\) requires a personal access token set in repo&gt;Settings&gt;Secrets with the name `GITHUB_ACCESS_TOKEN`. See [create a personal access token](https://help.github.com/en/articles/creating-a-personal-access-token-for-the-command-line). Also remember to set SSO enabled and access to `write:packages`.

Start by creating a folder for your workflows in the root of your applications repository

```text
$ mkdir -p .github/workflows
```

Inside that folder, create a workflow yaml-file. You can use our example as a starting point and adjust the values as needed.

The following example will build and push your Docker image to Github Package Registry, and then deploy that image to `dev-fss`.

This example also expects that your `spec.image` in your `nais.yaml` is set to `{{ image }}:{{ tag }}`. See [example nais.yaml](deploy.md#template-example-naisyaml-for-github-actions) below if you're unsure.

```text
name: Deploy to NAIS
on: push
jobs:
  deploy-to-dev:
    name: Build, push and deploy to dev-fss
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v1
      - name: Build code
        run: <put in your build-script here> # or remove if it's done in your Dockerfile
      - name: Create Docker tag
        env:
          NAME: <app-name>
        run: |
          echo "docker.pkg.github.com"/"$GITHUB_REPOSITORY"/"$NAME" > .docker_image
          echo "$(date "+%Y.%m.%d")-$(git rev-parse --short HEAD)" > .docker_tag
      - name: Build Docker image
        run: |
          docker build -t $(cat .docker_image):$(cat .docker_tag) .
      - name: Login to Github Package Registry
        env:
          DOCKER_USERNAME: x-access-token
          DOCKER_PASSWORD: ${{ secrets.GITHUB_ACCESS_TOKEN }}
        run: |
          echo "$DOCKER_PASSWORD" | docker login --username "$DOCKER_USERNAME" --password-stdin docker.pkg.github.com
      - name: Push Docker image
        run: "docker push $(cat .docker_image):$(cat .docker_tag)"
      - name: deploy to dev-fss
        uses: navikt/deployment-cli/action@0.4.0
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          cluster: dev-fss
          team: <team-name>
          resources: nais/dev-fss.yaml
```

Remember to specify `app-name` and `team-name`!

When these files and folders are commited and pushed, you can see the workflow running under the `Actions` tab of your repository.

**Template example nais.yaml for Github Actions**

```text
apiVersion: "nais.io/v1alpha1"
kind: "Application"
metadata:
  name: <app-name>
  namespace: default
  labels:
    team: <team-name>
spec:
  image: {{ image }}:{{ tag }}
```

**Badge in markdown**

Use the following URL to create a small badge for your workflow/action.

```text
https://github.com/{github_id}/{repository}/workflows/{workflow_name}/badge.svg
```

### Troubleshooting

Your deployment status page can be found on Github, under the repository you are deploying from. The status page will live at the URL `https://github.com/navikt/<YOUR_REPOSITORY>/deployments`. Links to deployment logs are available from this page.

Generally speaking, if the deployment status is anything else than `success`, `queued`, or `pending`, it means that your deployment failed. _Check the logs_ and _compare messages to the table below_ before asking for support.

If everything fails, and you checked your logs, you can ask for help in the [\#nais-deployment](https://nav-it.slack.com/messages/CHEQ22Q94) channel on Slack.

| Message | Action |
| :--- | :--- |
| the repository 'foo/bar' does not have access to deploy as team 'Baz' | Is your team name in _lowercase_ everywhere? |
| Repository _foo/bar_ is not registered | Please read the [registering your repository](deploy.md#registering-your-repository) section. |
| Deployment status `error` | There is an error with your request. The reason should be specified in the error message. |
| Deployment status `failure` | Your application didn't pass its health checks during the 5 minute startup window. It is probably stuck in a crash loop due to mis-configuration. Check your application logs using `kubectl logs <POD>` and event logs using `kubectl describe app <APP>` |
| Deployment is stuck at `queued` | The deployment hasn't been picked up by the worker process. Did you specify a [supported cluster](../#nais-clusters) with `--cluster=<CLUSTER>`? |
| team `foo` does not exist in Azure AD | Your team is not [registered in the team portal](teams.md). |

If for any reason you are unable to use _deployment-cli_, please read the section on [NAIS deploy with cURL](https://github.com/nais/doc/tree/5d1a8d8f05ea1da04ac269bc0dd64441908248ad/deployment/advanced-usage.md#nais-deploy-with-curl).

