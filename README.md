# azure-pipelines-templates
Opinionated templates for azure pipelines builds.

This repo is to be used as central repository for an organizations
azure pipelines templates, so that all libs in an organization
can reuse the same core build template, without having to update
each lib independently when changes to build process are made.

# Supported Build Types

1. Python package builds - single python version (python_package_main.ynl)
2. Golang module builds (go_module_main.yml).

# Getting started

1. Fork this repo into your organization. Azure devops requires that remote templates referenced by a pipeline must be part of the same organization.

2. Setup an [Azure Devops](https://azure.microsoft.com/en-us/services/devops/) account 
    and [Azure Pipelines](https://azure.microsoft.com/en-us/services/devops/pipelines/) 
    project.

3. Connect a pipeline to your github repo.

# Repo Requirements - General

1. **Accessible via ssh**: This template uses ssh to handle git commands. 
    [Directions for enabling ssh access](https://help.github.com/en/github/authenticating-to-github/connecting-to-github-with-ssh)

2. **Dev Branch**: Builds are triggered from the ``dev`` branch. Changes are merged into master at the end of a successful build.

# Related Repos

There are a few other repos related to these templates.

- [azure-pipelines-scripts](https://github.com/illuscio-dev/azure-pipelines-scripts): 
  Helper scripts for some pipeline steps. Can be forked or left as-is if no changes 
  are desired. If forked, the ``BUILD_SCRIPTS_REPO`` variable in ``variables.yml`` 
  should be changed to the fork.

- [islelib-py](https://github.com/illuscio-dev/islelib-py): python package template 
  with all tooling configs preset.

- [isleservice-py](https://github.com/illuscio-dev/isleservice-py): python service
  template with  all tooling configs preset.
  
- [isleconsumer-py](https://github.com/illuscio-dev/isleservice-py): python consumer 
  service template with  all tooling configs preset.
    
- [islelib-go](https://github.com/illuscio-dev/islelib-go): golang module template with
  all tooling configs preset.


# Repo Yaml Templates

The ``/repo_yaml_templates`` folder contains template pipeline definitions to be placed 
in your repo's and references these core templates.

# General Build Pipeline

Builds are performed in a single Linux job, in order to reduce build time. All pipelines
follow the same general order

1. **Lint:** Run linters to check code style.

2. **Test:** Run tests.

3. **Check test Coverage:** By default, 85% code coverage is required.

4. **Version up:** The pipeline automatically chooses the next available patch version 
    to be used for the target major, minor version in setup.cfg

5. **Build Docs:** static html docs are built to ``./docs/``. There are
   options for publishing to an S3 bucket, github pages, or both. This build
   is added to the merge for master.
    
6. **Build software:** Creates target build type for the software (go binary / python 
    package / docker container) if needed.
    
7. **Upload build:** (to pypi / dockerhub, etc.)

8. **Git tag version:** Version tag added to git, (ex: ``v1.1.13``)

9. **Force merge master:** The source code is tagged with the version
    and force merged into master, favoring the dev branch in code conflict.
    
10. **Master and tags pushed to git:** New master branch and build tags are pushed to 
    git.
    
# Pull Request Checks.

The pipeline is run on any PR request made to ``dev`` by default. Build steps (stages 
5-11 above) are not executed during PR validation.

# Documentation Publication Options.

**Push docs to S3 Bucket:**  Whenever a new commit is made to the ``master`` branch, 
documentation is built as part of the pipeline and can be pushed to an `Amazon S3`_ 
bucket for hosting. The documentation will be pushed to two locations:

   * s3://{docs_bucket}/{repo_name}/latest
   * s3://{docs_bucket}/{repo_name}/v{version}

S3 can be configured to handle user authorization through `Cloudfront`_ and `Cognito`_
using [this lambda edge template](https://console.aws.amazon.com/lambda/home?region=us-east-1#/create/app?applicationId=arn:aws:serverlessrepo:us-east-1:520945424137:applications/cloudfront-authorization-at-edge),
following [this tutorial](https://aws.amazon.com/blogs/networking-and-content-delivery/authorizationedge-using-cookies-protect-your-amazon-cloudfront-content-from-being-downloaded-by-unauthenticated-users/)

There are very few cross-platform affordable static website hosting services that allow
easy identity protection for private docs, and have found that this solutions works
well for us.

**Publish to Github Pages:** Since docs are always copied and committed to /docs/ on the
master branch, its easy to configure documentation publication through 
[github pages](https://pages.github.com/). **NOTE: GITHUB PAGES WILL ALWAYS BE PUBLIC, 
EVEN ON A PRIVATE REPO**


# Azure Pipelines Variable Groups.

These templates rely on access to logins or credentials for other services, and accesses
them via [pipeline variables]. The following variables are required for these templates
(depending on the type of build.) They are broken into suggested groups.

- **GIT_CREDENTIALS:** Used for adding tags and updating master.

    - GIT_SSH_PUBLIC_KEY: Public key for git ssh access.
    - GIT_SSH_PASSPHRASE: Passphrase for ssh key.
    - GIT_KNOWN_HOSTS_ENTRY: Known hosts entry created with public and private keys.
    - GIT_USERNAME: User to use with git commits.
    - GIT_EMAIL: Email to use with git commits. 
    
    For more information on how to create ssh keys, [see here](https://help.github.com/en/github/authenticating-to-github/connecting-to-github-with-ssh). 
    For more information on how these values are installed in a pipeline, 
    [see here](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/utility/install-ssh-key?view=azure-devops).
    
- **OPEN_SOURCE_TWINE_CREDENTIALS:** Used for uploading public packages to pypi.org
    
    - PYPIORG_USER: username for pypi.org
    - PYPIORG_PASSWORD: password for pypi.org.
    
- **CONTAINER_REGISTRY_CREDENTIALS:** Used for uploading service images.

    - CONTAINER_REGISTRY_URL: URL for pushing  / pulling from container registry.
    - CONTAINER_REGISTRY_ID: ID to sign into container registry.
    - CONTAINER_REGISTRY_PASSWORD: Password to sign into container registry
    
- **AWS_DOCS_BUCKET_CREDENTIALS:** Used for uploading docs to S3 bucket.

    - AWS_ACCESS_KEY_ID: Access key id for account to upload docs with.
    - AWS_SECRET_ACCESS_KEY: Secret Access key for account to upload docs with.
    - DOCS_S3_BUCKET: S3 bucket name to upload docs to.
    - DOCS_CLOUDFORMATION_DISTRO_ID: Cloudfront distro to invalidate indexes for when
        altering "latest" docs.
        
NOTE: These groups will need to be added to every build pipeline individually. The 
configuration pane to add groups to a pipeline is currently a little hard to find.

In the overview page for a pipeline go to: ``Edit -> Hamburger Menu -> Triggers``. This
will bring you to a general settings page. At the top choose ``Varaibles -> 
Variable Groups -> Link variable group`` to give a pipeline access to a variable group.

To make a new group, look at the left hand pane and go to ``Pipelines -> Library -> 
Variable groups``.


# Azure Pipelines Secure Files

The following secure files are required for pipelines to function properly:

- git_ssh_key.private: SSH private key for accessing git.

To upload this file, look at the left hand pane and go to ``Pipelines -> Library -> 
Secure files``. 


# Python Packages Builds

The template library for using this build pipelines can be 
[found here](https://github.com/illuscio-dev/islelib-py). The following tools are used 
during python package builds:

- **Root Template**: python_package_main.yml

- **Dependency Installation:** Handled via [pip]. 

- **Linters:** Linting is done via [Flake8](https://flake8.pycqa.org/en/latest/), 
    [Black](https://github.com/psf/black), and [mypy](http://mypy-lang.org/) for 
    static-type analysis.
    
- **Tests:** Handled via [pytest](https://docs.pytest.org/en/latest/).

- **Docs:** Handled via [sphinx](https://www.sphinx-doc.org/en/master/).

- **Builds:** Handled via [setuptols](https://setuptools.readthedocs.io/en/latest/).

- **Package Uploads:** Handled via [twine](https://pypi.org/project/twine/).

# Go Module Builds

The template library for using this build pipelines can be 
[found here](https://github.com/illuscio-dev/islelib-go). The following 
tools are used during python package builds:

- **Root Template**: go_module_main.yml

- **Dependency Installation:** Handled via 
    [go get](https://golang.org/pkg/cmd/go/internal/get/). Git is configured to use ssh 
    instead of http to enable private package fetching. 

- **Linters:** Linting is done via [Revive](https://github.com/mgechev/revive).
    
- **Tests:** Handled via [go test](https://golang.org/pkg/testing/).

- **Docs:** Handled via [sphinx](https://www.sphinx-doc.org/en/master/) for quickstarts 
    and guides + a cli tool to generate API Documentation via 
    [godoc](https://godoc.org/golang.org/x/tools/cmd/godoc).

- **Builds:** Go modules do not need to be built.

- **Package Uploads:** Handled via merge into master and version tag.


# Docker Image Builds:

- **Root Template**: docker_image_main.yml

- **Image Builds:** Handled via docker commandline + dockerfile.

- **Image Uploads:** Handled via docker commandline.


# Python REST and consumer Service Builds:

The template library for using this build pipelines can be 
[found here for REST](https://github.com/illuscio-dev/isleservice-py) and 
[here for Consumer](https://github.com/illuscio-dev/isleconsumer-py) services. The 
following tools are used during python service builds:

- **Root Template:** python_service_main.yml

- **Dependencies, linting testing, and docs:** are identical to python package builds.

- **Image Builds and Uploads:** Handled via docker commandline + dockerfile.

