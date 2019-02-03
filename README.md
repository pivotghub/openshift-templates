# Invenio OpenShift Templates

This repository contains the templates for a generic Invenio application (a
Dockerfile) and OpenShift build configuration and deployment of each
infrastructure component. As recommended by the OpenShift service, GitLab will
be used to trigger builds of Docker images and store them in GitLab Registry.

For the sake of clarity of this documentation, examples will be given referring
to:

* `zenodogroup`, for the GitLab group (e.g.
    <https://gitlab.cern.ch/zenodogroup>)
* `zenodo-broker-openshift`, for the GitLab project (e.g.
    <https://gitlab.cern.ch/zenodogroup/zenodo-broker-openshift>)
* `zenodo-broker` for the GitHub repository containing the app source code
    (e.g. <https://github.com/zenodo/zenodo-broker>)
* `zenodobrokerimage` for the name of Docker image built for the application
    (visible at [the GitLab project's Registry, i.e.
    <https://gitlab.cern.ch/zenodogroup/zenodo-broker-openshift/container_registry>)

## Repository content

This repository contains:

* A `Dockerfile` image, which contains the commands to build and run your
    Invenio app `zenodo-broker`.
* `.gitlab-cy.yml`, which is the Continuos Integration configuration to build
    the Dockerfile and push it to the GitLab Registry, and even deploy it in
    some cases.
* A set of `scripts` to trigger the image build and deploy an image to the
    OpenShift infrastructure.
* `application.yaml`, `configuration.yaml`, `image_stream.yaml`,
    `services.yaml`, which are the templates for the OpenShift infrastructure.

## Bootstrap

### Create OpenShift projects

Request your OpenShift environments. Go to https://cern.ch/webservices and
create new official websites as PaaS instances (`https://openshift.cern.ch`):

1. Create a new OpenShift project to have a shared ImageStreamTag repository:
    `zenodo-broker-tags.web.cern.ch`.
2. Create 3 environments, `dev`, `qa` and `prod`:
    * DEV: `zenodo-broker-dev.web.cern.ch`
    * QA: `zenodo-broker-qa.web.cern.ch`
    * PROD: `zenodo-broker.web.cern.ch`

Only the last one will probably be accessible from outside CERN.

You are free to create your set of environments. Note that the documentation
below will refer to these 3 envs.

### Login on OpenShift

Install the OpenShift [`oc`
client](https://docs.openshift.org/latest/cli_reference/get_started_cli.html)

For all interactions with OpenShift, login and select the right project:

```console
$ oc login https://openshift.cern.ch
Authentication required for https://openshift.cern.ch:443 (CERN)
Username: username
Password:
Login successful.

$ oc project zenodo-broker-tags
```

### Create OpenShift service accounts

GitLab needs to be able to communicate with OpenShift. The recommended way is
to create a Service Account with limited privileges and use its access token to
import built Docker images and deploy.

There is a KB article that describes the procedure:
<https://cern.service-now.com/service-portal/article.do?n=KB0004553>. Please
refer to it if you encounter difficulties while running the commands below.

> Note
>
> Keep track of these tokens, you will have to add them to GitLab secrets.

Service account for `tags`:

```console
$ oc project zenodo-broker-tags

# create a service account called `gitlabci-deployer`
$ oc create serviceaccount gitlabci-deployer
serviceaccount "gitlabci-deployer" created

# add `registy-editor` permission to the new account
$ oc policy add-role-to-user registry-editor -z gitlabci-deployer

# We will reffer to the output of this command as `tags-token`
$ oc serviceaccounts get-token gitlabci-deployer
<...tags-token...>
```

Service account for `dev`:

```console
$ oc project zenodo-broker-dev

$ oc create serviceaccount gitlabci-deployer
serviceaccount "gitlabci-deployer" created

$ oc policy add-role-to-user basic-user -z gitlabci-deployer

# We will reffer to the output of this command as `dev-token`
$ oc serviceaccounts get-token gitlabci-deployer
<...dev-token...>
```

Service account for `qa`:

```console
$ oc project zenodo-broker-dev

$ oc create serviceaccount gitlabci-deployer
serviceaccount "gitlabci-deployer" created

$ oc policy add-role-to-user basic-user -z gitlabci-deployer

# We will reffer to the output of this command as `qa-token`
$ oc serviceaccounts get-token gitlabci-deployer
<...qa-token...>
```

For the `prod` environment, you probably don't want to setup automatic
deployments.

### Share OpenShift project

Allow other projects to pull images from `zenodo-broker-tags`, to be able to
share the ImageStream with `dev`, `qa` and `prod`.

```console
$ oc project zenodo-broker-tags
$ oc policy add-role-to-group system:image-puller system:serviceaccounts
```

If you want to restrict this sharing to your projects only, refer to the
[official documentation](https://docs.openshift.org/latest/dev_guide/managing_images.html#allowing-pods-to-reference-images-across-projects).

### Fork this template

First of all, ensure you have access to (or create) the `zenodogroup` GitLab
group. Then:

1. Fork this repo to your group. From now on, you will work in your fork.
2. Change the name and the url of your forked repo. In your repo GitLab page,
    go to `Settings` -> `General` -> `Advanced Settings` -> click on `Expand`
    button and edit `Project name` and `Path` in the `Rename repository`
    section. From now on, we will call it
    [`zenodo-broker-openshift`](https://gitlab.cern.ch/zenodogroup/zenodo-broker-openshift.git).
3. Add some of the secrets/variables. In your repo GitLab page, go to
    `Settings` -> `CI / CD` -> `Secret variables` -> click on `Expand` button
    and create the following variables:
    * `OPENSHIFT_PROJECT_TAGS_NAME`: insert `zenodo-broker-tags`.
    * `OPENSHIFT_PROJECT_TAGS_TOKEN`: insert the token `<...tags-token...>`.
    * `OPENSHIFT_PROJECT_DEV_TOKEN`: insert the token `<...dev-token...>`.
    * `OPENSHIFT_PROJECT_QA_TOKEN`: insert the token `<...qa-token...>`.
4. Generate a GitLab CI trigger token. In your repo GitLab page, go to
    `Settings` -> `CI / CD` -> `Pipeline triggers` -> click on `Expand` button,
    input a `Description` (for example, `Trigger to create Docker image for
    Zenodo Broker`) and edit click the button `Add trigger`. The generated
    token will be used later (don't worry abuot storing it, the creator of the
    token can see it even after leaving the page).

### First configuration

Clone this GitLab repository
<https://gitlab.cern.ch/zenodogroup/zenodo-broker-openshift.git> locally to
your machine. Values to be changed are tagged by the keyword `changeme` to
easily find them. In details:

* ./Dockerfile:
  * edit the `FROM` if needed. If, for example, you need to use `XRootD`, there
    is already an available image for that. The application Dockerfile should
    use one of the base images defined and published by [Invenio
    Base](https://gitlab.cern.ch/invenio/base/container_registry).
  * set the `uwsgi` application name.
  * set the git repo url containing the app code, which will be fetched when
    building your image. (e.g. <https://github.com/zenodo/zenodo-broker.git>)
* ./scripts/build.sh:
  * set the `project id` in the `GITLAB_PIPELINE_TRIGGER_URL` var: you can find
    the `project id` in `Settings` -> `General`.
  * set the `APPLICATION_IMAGE_NAME` to `zenodobrokerimage`: this will be the
    application Docker image name.
* ./scripts/deploy.sh:
  * set the `APPLICATION_IMAGE_NAME` to `zenodobrokerimage`.
  * set the `OPENSHIFT_PROJECT_TAGS_NAME` to `zenodo-broker-tags`.
  * set the value for all `OPENSHIFT_PROJECT_NAME`: by default, it contains 3
    OpenShift environments, `dev`, `qa` and `prod`, but you can edit this
    mapping and modify it according to your configuration.

### Create OpenShift deployment

The first time you create `zenodo-broker-tags`, you need to setup the
ImageStream:

```console
$ oc login https://openshift.cern.ch
$ oc project zenodo-broker-tags
$ oc process -f image_stream.yaml \
      --param APPLICATION_IMAGE_NAME='zenodobrokerimage' | oc create -f -
```

Then, for **each** environment, you will have to follow these steps.

Login and select the right project:

```console
$ oc login https://openshift.cern.ch
$ oc project zenodo-broker-dev
```

Create all the needed secrets. Replace username and password in each one.

#### 1. Secrets

Database password:

```console
$ POSTGRESQL_PASSWORD=$(openssl rand -hex 8)
$ POSTGRESQL_USER=invenio
$ POSTGRESQL_HOST=db
$ POSTGRESQL_PORT=5432
$ POSTGRESQL_DATABASE=invenio
$ oc create secret generic \
  --from-literal="POSTGRESQL_PASSWORD=$POSTGRESQL_PASSWORD" \
  --from-literal="SQLALCHEMY_DB_URI=postgresql+psycopg2://$POSTGRESQL_USER:$POSTGRESQL_PASSWORD@$POSTGRESQL_HOST:$POSTGRESQL_PORT/$POSTGRESQL_DATABASE" \
  db-password
secret "db-password" created
```

RabbitMQ password:

```console
$ RABBITMQ_DEFAULT_PASS=$(openssl rand -hex 8)
$ oc create secret generic \
  --from-literal="RABBITMQ_DEFAULT_PASS=$RABBITMQ_DEFAULT_PASS" \
  --from-literal="CELERY_BROKER_URL=amqp://guest:$RABBITMQ_DEFAULT_PASS@mq:5672/" \
  mq-password
secret "mq-password" created
```

Elasticsearch basic auth:

```console
$ ELASTICSEARCH_PASSWORD=$(openssl rand -hex 8)
$ ELASTICSEARCH_USER=username
$ oc create secret generic \
  --from-literal="ELASTICSEARCH_PASSWORD=$ELASTICSEARCH_PASSWORD" \
  --from-literal="ELASTICSEARCH_USER=$ELASTICSEARCH_USER" \
  elasticsearch-password
```

#### 2. Create and start Invenio services

```console
$ oc process -f services.yaml | oc create -f -
```

#### 3. Init Invenio configuration

Copy `configuration.yaml` to `configuration-dev.yaml` and edit its
configuration parameters as needed. Then:

```console
$ oc process -f configuration-dev.yaml | oc create -f -
```

#### 4. Create Invenio application

For `APPLICATION_IMAGE_NAME`, set the name of the Docker image
`zenodobrokerimage`. Note that, since the image has not been built yet, the
application will not be deployed.

```console
$ oc process -f application.yaml \
  --param APPLICATION_IMAGE_NAME='zenodobrokerimage' \
  --param APPLICATION_IMAGE_TAG='dev' \
  --param TAGS_PROJECT='zenodo-broker-tags' | oc create -f -
```

## Deployment lifecycle

You think, you code, you refactor, you test, you test, you test... and now you
want to deploy!

Prepare a release: a release could be simply a specific commit id, or a tag, or
maybe you want to deploy a PR. For this example, let's use the already pushed
release tag `v1.0.3`.

Ensure that `requirements.txt` file exists at the root of your code for the
commit `v1.0.3`. It must contain **all** Python dependencies, including `.` for
the `setup.py`. This is because the Docker image will **only** run `pip install
-r requirements.txt`.

Now, in the console, change directory to the `scripts` folder of your
`zenodo-broker-openshift` repo and login on OpenShift.

```console
$ cd myworkspace/gitlab/zenodo-broker-openshift/scripts
$ oc login https://openshift.cern.ch
$ oc project zenodo-broker-dev
```

### Build

Start a build of your application code on the specified release.

```console
$ build.sh -t v1.0.3
```

This step will trigger a GitLab-CI pipeline which will:

1. build the Invenio Docker file image. It clones the tag `v1.0.3` of
    <https://github.com/zenodo/zenodo-broker.git>
2. push the built image to the GitLab registry:
    <https://gitlab-registry.cern.ch/zenodogroup/zenodo-broker-openshift/zenodobrokerimage:v1.0.3>
3. import the new built image metadata to the `zenodo-broker-tags` OpenShift
    project. With this step, the image will be available to all
    `zenodo-broker-*` OpenShift projects and it will be ready to be deployed.

#### Automation

You can setup automatic building, for example at the end of a Travis build,
sending a POST request to GitLab-CI. Use the `curl` request that you can find
at the end of the script `build.sh` (or call the `build.sh` file).

### Deploy

```console
$ deploy.sh dev v1.0.3
```

This step will:

1. for the `dev` project, it will update the OpenShift ImageStreamTag called
    `dev` to point to the image `v1.0.3`. Something like a symlink.
2. deploy the `dev` tag in the `dev` environment.

#### Automation

When sending the POST request to the GitLab-CI to build a new image, you can
pass an extra parameter `deploy=dev`.

## Customize OpenShift deployment configuration

If you need to customize any of the OpenShift deployment configuration, it is
recommended to edit the templates yaml files and then replace any existing
configuration.

For example:

```console
$ oc process -f services.yaml | oc replace -f -
```

## Run Invenio commands

### Debug session

### OpenShift jobs

## Misc

### Mount EOS

If you need to use EOS in your Invenio instance, you have to change the
OpenShift templates. See [this KB
article](https://cern.service-now.com/service-portal/article.do?n=KB0005259) on
how to set it up.

### Custom domain name

TODO

### Custom certificate

TODO
