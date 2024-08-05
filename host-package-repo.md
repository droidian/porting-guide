# Hosting a Personal Droidian Package Repository on GitHub/GitLab Pages

This guide explains how to set up and maintain a personal Droidian package repository using GitHub Pages or GitLab Pages.

## Steps for GitHub Pages

1. Create a new GitHub repository.

2. Add the workflow action file to `.github/workflows/main.yml`. You can find the content of this file [here](#github-action-yml)

>Note: The instructions assume you are using the `main` branch. If you are using a different branch, make sure to update the branch name in the GitHub Action file accordingly.

3. Add GPG-related secrets to your repository:
   - Go to your repository's Settings > Secrets and variables > Actions
   - Add new repository secrets:
     - `GPG_KEY_ID`
     - `GPG_PASSPHRASE`
     - `GPG_PRIVATE_KEY`

4. After the first run, a new branch `gh-pages` will be created
   - Go to your repository's Settings > Pages
   - Under "Build and deployment", select `Source` as `Deploy from a branch` and set the Branch to `gh-pages`

## Steps for GitLab Pages (Alternative)

1. Create a new GitLab repository.

2. Add the `.gitlab-ci.yml` file to the root of your repository. You can find the content of this file [here](#gitlab-ci-yml).

>Note: The instructions assume you are using the `main` branch. If you are using a different branch, make sure to update the branch name in the GitLab CI configuration file accordingly.

3. Add GPG-related variables to your repository:
   - Go to your repository's Settings > CI/CD > Variables
   - Add new variables:
     - `GPG_KEY_ID`
     - `GPG_PASSPHRASE`
     - `GPG_PRIVATE_KEY` (use base64 encoded GPG private key)

>Note: 
>1. Select visibilty as `Masked` while creating variables.
>2. GitLab doesn't support whitespace in their variables, so use a base64 encoded GPG private key for GitLab.

## Usage

To add new packages:

1. Place `.deb` files in the `packages` directory of the repo.
2. Commit and push to the `main` branch.
3. The CI/CD pipeline will automatically update the repository and deploy to GitHub/GitLab Pages.

Your personal Droidian package repository will be available at:
- GitHub: `https://<username>.github.io/<repository-name>`
- GitLab: `https://<username>.gitlab.io/<repository-name>`

## GitHub Action YML

```yaml
name: Update Packages Repository

on:
  workflow_dispatch:
  push:
    branches:
      - main
    paths:
      - 'packages/*.deb'

jobs:
  update-repo:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4

      - name: Import GPG key and setup gpg
        env:
          GPG_PRIVATE_KEY: ${{ secrets.GPG_PRIVATE_KEY }}
          GPG_PASSPHRASE: ${{ secrets.GPG_PASSPHRASE }}
        run: |
          echo "$GPG_PRIVATE_KEY" | gpg --import --batch
          echo "allow-preset-passphrase" > ~/.gnupg/gpg-agent.conf
          echo "pinentry-mode loopback" >> ~/.gnupg/gpg.conf
          echo "no-tty" >> ~/.gnupg/gpg.conf
          gpg-connect-agent reloadagent /bye

      - name: Generate Packages and Release files
        env:
          GPG_TTY: $(tty)
          GNUPGHOME: /home/runner/.gnupg
          GPG_KEY_ID: ${{ secrets.GPG_KEY_ID }}
          GPG_PASSPHRASE: ${{ secrets.GPG_PASSPHRASE }}
          USER: ${{github.repository_owner}}
          REPO: ${{github.event.repository.name}}
        run: |
          mkdir -p public/dists/stable/main/binary-arm64
          dpkg-scanpackages --arch arm64 packages > public/dists/stable/main/binary-arm64/Packages
          gzip -k -f public/dists/stable/main/binary-arm64/Packages
          
          cd public/dists/stable
          
          cat << EOF > Release
          Origin: https://$USER.github.io/$REPO
          Label: Droidian PKG
          Suite: stable
          Codename: stable
          Version: 1.0
          Architectures: arm64
          Components: main
          Description: Packages 
          EOF
          
          apt-ftparchive release . >> Release
          
          echo "$GPG_PASSPHRASE" | gpg --batch --yes --pinentry-mode loopback --passphrase-fd 0 --default-key $GPG_KEY_ID --clearsign -o InRelease Release
          echo "$GPG_PASSPHRASE" | gpg --batch --yes --pinentry-mode loopback --passphrase-fd 0 --default-key $GPG_KEY_ID -abs -o Release.gpg Release

      - name: Move packages directory to public
        run: mv packages public/packages

      - name: Deploy to GitHub Pages
        uses: peaceiris/actions-gh-pages@v4
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public
          force_orphan: true
```

## GitLab CI YML

```yaml
stages:
  - build
  - deploy

update-repo:
  stage: build
  image: ubuntu:latest
  rules:
    - if: $CI_PIPELINE_SOURCE == "push" && $CI_COMMIT_BRANCH == "main"
    - if: $CI_PIPELINE_SOURCE == "web"
  script:
    - apt-get update && apt-get install -y gnupg dpkg-dev apt-utils gzip
    - echo "$GPG_PRIVATE_KEY" | base64 -d | gpg --import --batch
    - echo "allow-preset-passphrase" > ~/.gnupg/gpg-agent.conf
    - echo "pinentry-mode loopback" >> ~/.gnupg/gpg.conf
    - echo "no-tty" >> ~/.gnupg/gpg.conf
    - gpg-connect-agent reloadagent /bye
    - mkdir -p public/dists/stable/main/binary-arm64
    - dpkg-scanpackages --arch arm64 packages > public/dists/stable/main/binary-arm64/Packages
    - gzip -k -f public/dists/stable/main/binary-arm64/Packages
    - cd public/dists/stable
    - |
      cat << EOF > Release
      Origin: https://$CI_PROJECT_NAMESPACE.gitlab.io/$CI_PROJECT_NAME
      Label: Droidian PKG
      Suite: stable
      Codename: stable
      Version: 1.0
      Architectures: arm64
      Components: main
      Description: Droidian Packages 
      EOF
    - apt-ftparchive release . >> Release
    - echo "$GPG_PASSPHRASE" | gpg --batch --yes --pinentry-mode loopback --passphrase-fd 0 --default-key $GPG_KEY_ID --clearsign -o InRelease Release
    - echo "$GPG_PASSPHRASE" | gpg --batch --yes --pinentry-mode loopback --passphrase-fd 0 --default-key $GPG_KEY_ID -abs -o Release.gpg Release
    - cd ../../../
    - mv packages public/packages
  artifacts:
    paths:
      - public

pages:
  stage: deploy
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
  dependencies:
    - update-repo
  script:
    - echo "Deploying to GitLab Pages..."
  artifacts:
    paths:
      - public
```
