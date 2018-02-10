# automating docker builds from repos you don't own

## the problem

You have dockerized someone else's project (source repo) and want to keep a docker build up-to-date with that project. You would normally use DockerHub's automated builds, but that is only an option if you own the source repo. So, you need to implement your own automation system that checks the source repo for new versions and when a new version is discovered, builds and pushes a corresponding docker image. A side-effect of this approach is that the README and Dockerfile are not entered/updated on the registry as would be the case for Docker's automated builds. This leads back to the desire to setup a DockerHub automated build.

## the solution

Setup a GitHub repo with an automated system that updates it when the source repo is updated. Setup the DockerHub automated build on your GitHub repo.

- Source repo releases new version
- GitHub repo job discovers new version of source repo and updates a branch of its own repo accordingly
- DockerHub automated build is triggered

## specific implementation

Look at the files and scripts in this repo as a reference for the steps below. If I were setting up a project like this, I would clone this repo as a starting place.

- Create a GitHub repo for your docker project
- Create a DockerHub automated build on that GitHub repo
  ```
  Customize Autobuild Tags

  Push Type  Name  Dockerfile Location  Docker Tag

  Tag        /.*/  /                    (Same as tag)
  Tag        /.*/  /                    latest
  ```
- Enable the GitHub repo in Travis CI
- add the GitHub as a remote to your local git repo
- Generate a [personal access token for GitHub](https://github.com/settings/tokens) with the "public_repo" permission. I name the token "CI-{REPO_NAME_HERE}".
- Encrypt the token into .travis.yml
  ```sh
  travis encrypt --add --no-interactive GITHUB_TOKEN={YOUR_TOKEN_HERE}
  ```
  ```sh
  # if you don't have travis in your environment and want to use docker instead:
  docker run -it \
    -v $PWD:/project \
    --workdir /project \
    ruby \
    /bin/sh -c \
      'gem install travis && travis encrypt --add --no-interactive GITHUB_TOKEN={YOUR_TOKEN_HERE}'
  ```
- Create a script that gets the latest release of the source project
- Setup a travis CI build and cron job that checks the source repo for new versions and makes a corresponding build and tag for your GitHub repo's `dist` branch, thus triggering the DockerHub automated build.
  - Use `https://$GITHUB_TOKEN@github.com/$REPO.git` as the remote url.
  - Ensure you only run CI on the master branch. You probably need `.travis.yml` in the dist branch in order to ignore it.
  - Don't forget to handle both regular CI builds (on push to master) and cron job builds. Here are some options:
    - Be proper and never re-use a tag. Perhaps you would add a unique build number to each tag. This offers maximum certainty that you'll always pull the exact same image from the registry when using the same docker image tag.
    - Go all in reusing tags, unconditionally pushing to `dist` on every regular CI build and cron job.
    - Only push to `dist` in cron jobs when the source repo has a new version, but on regular CI builds, always push to `dist`, even if it means forcefully reusing a tag. This is the middle ground between the previous options. You stay up-to-date with the source repo, but also get an immediate update to reflect any changes you made to your repo. The tag in your GitHub repo and DockerHub repo represents the current version of the source repo and the current state of your master branch. Just keep in mind that changing your master branch may produce a Docker image in your repo tagged with the same tag as an older image that has now been replaced. A consumer of that image could be implicitly using a different image on a future pull of that tag and encounter perplexing issues.
