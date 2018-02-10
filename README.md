# automating docker builds from repos you don't own

## the problem

You have dockerized someone else's project (source repo) and want to keep a docker build up-to-date with that project. You would normally use Dockerhub's automated builds, but that is only an option if you own the source repo. So, you need to implement your own automation system that checks the source repo for new versions and when a new version is discovered, builds and pushes a corresponding docker image. A side-effect of this approach is that the README and Dockerfile are not entered/updated on the registry as would be the case for Docker's automated builds. This leads back to the desire to setup a Dockerhub automated build.

## the solution

Setup a GitHub repo with an automated system that updates it when the source repo is updated. Setup the Dockerhub automated build on your GitHub repo.

Source repo releases new version
  -> GitHub repo job discovers new version of source repo and updates a branch of its own repo accordingly
  -> Dockerhub automated build is triggered

## specific implementation

Look at the files and scripts in this repo as a reference for the steps below. If I were setting up a project like this, I would clone this repo as a starting place.

- Create a GitHub repo for your docker project
- Create a Dockerhub automated build on that GitHub repo
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
- Setup a travic CI cron job that clones your GitHub repo and checks out a branch dedicated to automated builds. Use `https://$GITHUB_TOKEN@github.com/$REPO.git` as the remote url. Check that the branch has a tag corresponding to the latest version of the source repo. If not, do your build step if you have one, make a commit, tag it, and push that updated branch to the GitHub repo. This will trigger the docker automated build.
