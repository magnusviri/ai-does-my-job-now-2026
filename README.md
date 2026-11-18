# AI Does My Job Now presentation files

3 demos showing different levels of Terraform integration with Jamf Pro and how to use an AI agent.

[Presentation slides](AI Does My Job Now.pdf)

## Install

To run all of the demos you need to install all of the following on your computer and also set up a Jamf Pro server to run locally in Docker.

- Install [git](https://git-scm.com/install/mac)
- Install [Docker](https://www.docker.com/products/docker-desktop/)
- Download [Jamformer](https://github.com/Jamf-Concepts/jamformer/releases)
- Install [Terraform](https://developer.hashicorp.com/terraform/install)

Clone the repository and its submodules like this:

```sh
git clone --recurse-submodules https://github.com/magnusviri/ai-does-my-job-now-2026.git
```

If you cloned it without getting the submodules, you can get the submodules with this command.

```sh
git submodule update --init --recursive
```

### Set up Jamf Pro test server

[`docker-jamfpro`](https://github.com/magnusviri/docker-jamfpro) runs a local Jamf Pro
instance for this demo. It requires Docker, a Jamf Pro development license, and the Jamf
Pro `ROOT.war` file; the Jamf software and license are not included in this repository.

1. In the [Jamf Account portal](https://account.jamf.com/products), download
   the Manual Jamf Pro installer and keep its activation code. Unzip the
   download to obtain `ROOT.war`.
2. Choose the Jamf Pro version from the downloaded installer, then configure
   the submodule. The version must match both the directory name and the
   `JAMF_PRO_VERSION` value.

   ```sh
   cd docker-jamfpro
   cp .env_example .env
   # Edit .env: set JAMF_PRO_VERSION and, if needed, JAMF_IMAGE_VERSION,
   # JAMF_PRO_PORT, and the MySQL credentials.
   mkdir -p data/<jamf-pro-version>
   mv /path/to/ROOT.war data/<jamf-pro-version>/ROOT.war
   ```

3. Start the local services:

   ```sh
   docker compose up -d
   ```

   Jamf Pro can take several minutes to initialize. Browse to
   `http://localhost` (or `http://localhost:<JAMF_PRO_PORT>` if you changed
   the port), accept the license agreement, enter the activation code, and
   complete the initial account setup.

To stop the services later, run `docker compose down` from `docker-jamfpro`.

## Demo 1: Isolate Reading with Jamformer

Run Jamformer. It will ask you all the questions it needs answered. It will create a folder named "generated."

```
./jamformer
```

[![Presentation video preview](jamformer.gif)](jamformer.mp4)

Inspect your config files in the "generated" folder to make sure there are no secrets or sensitive or restricted data. When you are sure that the data is safe to use with an AI agent, go ahead and start one.

The first thing you probably want to do is run "/init" to create AGENTS.md or CLAUDE.md (depending on the agent you are using). Because I'm using a brand new Jamf server, there probably isn't a lot there to discuss with the AI. But a mature server would have a lot of stuff for the AI to inspect and discuss.

## Demo 2: Isolate Writing with Terraform

First, let's turn the "generated" directory into a git repo so we can track all the changes the AI agent makes.

```
git init
git add .
git commit -m "1st"
```

Use an AI Agent to change files. In a new Terminal window run your agent. In this case, I'm using codex (I also use my [agent-container](https://github.com/magnusviri/agent-container) to run it in a Docker container).

```
codex
```

This is my first prompt.

```
Add a computer policy. It should run recon once a day on all computers.
```

Check for changes in a different Terminal window.

```
git status
git diff
```

Once you're done inspecting everything and you're happy with it apply it to the server. Jamformer already configured things so you only need to enter the password and username of the Jamf server.

```
terraform apply
```

Now when you go to the local Jamf server you should see the new policy (assuming it didn't create an error in the hcl file).

Then add the changes to git.

```
git add .
git commit -m "new policy"
```

## Demo 3: Running Terraform in GitLab's CI/CD

There is quite a bit of setup for this.

### Start GitLab

Start the local GitLab server from its Compose directory ("gitlab"):

```sh
cd gitlab
docker compose up -d
```

GitLab may take several minutes to become available. Open
[http://localhost:8929](http://localhost:8929) for the web interface.

Set the `root` account password after the container is running:

```sh
docker exec -it gitlab gitlab-rake "gitlab:password:reset[root]"
```

### GitLab Repo

Create a new project (click the + button at the top) and call it "jamf".

Now upload the "generated" directory to this repo.

```
git remote add origin http://localhost:8929/root/jamf.git
git push --set-upstream origin main
```

### Create GitLab Runner

The GitLab Runner does the compute backend work for the GitLab server. This safely separates repo CI/CD code execution from the GitLab server.

- Go to the "jamf" project in GitLab.
- Click [Settings->CI/CD->Runners](http://localhost:8929/root/jamf/-/settings/ci_cd#js-runners-settings).
- Click "Create project runner" button.
- Check "Run untagged jobs" checkbox.
- Click "Create runner" button.
- Copy the token (begins with "glrt-").
- Run the following command (replacing the token with the one you copied).

```
docker compose run --rm gitlab-runner register \
  --non-interactive \
  --url "http://gitlab:8929" \
  --token "glrt-..." \
  --executor "docker" \
  --docker-image "alpine:latest" \
  --description "local-docker-runner"
```

### Test CI/CD pipeline

Go to [Build->Pipeline editor](http://localhost:8929/root/jamf-repo/-/ci/editor?branch_name=main) and create your own then enter this.

```
diagnostics:
  stage: test
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  script:
    - 'echo "Hostname: $(hostname)"'
    - getent hosts localhost
    - cat /etc/os-release
```

Click "Commit changes" when done. Then go
to [Pipelines](http://localhost:8929/root/jamf1/-/pipelines). There should be a job there and it should pass. If it doesn't pass, then debug it until it passes.

### Create CI/CD Environments

If you want to fully automate making changes, it is best to test those changes in test rings. The first ring will be your laptop (which you already have running). The second ring is a development Jamf server (not part of the demo). The third ring is the production Jamf server (not part of the demo).

Terraform likes to save state. So in order to run Terraform on the GitLab server, we have to save it's state somewhere. We can use GitLab's Environment's feature to do this.

Click [Operate->Environments](http://localhost:8929/root/jamf/-/environments). Click "New environment". Set it's name to "dev" and then click "Save". Create a "prod" environment also.

### Set up CI/CD Variables

The variables are the location and credentials for the Jamf server. Storing them here keeps them out of the repo and away from the AI.

Click [Settings->CI/CD->Variables](http://localhost:8929/root/jamf/-/settings/ci_cd#js-cicd-variables-settings). Add these variables and values if you are using user credentials.

	TF_VAR_jamfpro_instance_fqdn = http://localhost
	TF_VAR_jamfpro_basic_auth_username = jamf_username
	TF_VAR_jamfpro_basic_auth_password = jamf_password

TF_VAR_ variables are mapped to input variables in variables.tf.

### Run Terraform with CI/CD 

I've already prepared a .gitlab-ci.yml file. Copy it from the parent directory to the generated directory and add then commit it to the repo.

```
cp ../gitlab-ci.yml .gitlab-ci.yml
git add .
git commit -m "Add terraform CI/CD"
git push
```

To Do

## Cleanup

```
cd docker-jamfpro
docker compose down
cd gitlab
docker compose down
```