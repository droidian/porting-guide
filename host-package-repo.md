# 在 GitHub/GitLab Pages 上托管个人 Droidian 包仓库

本指南解释了如何使用 GitHub Pages 或 GitLab Pages 设置和维护个人 Droidian 包仓库。

## GitHub Pages 步骤

1. **创建一个新的 GitHub 仓库**。

2. **添加工作流动作文件到 `.github/workflows/main.yml`**。您可以在 [这里](#github-action-yml) 找到此文件的内容。

> 注意：这些说明假定您使用的是 `main` 分支。如果您使用其他分支，请确保在 GitHub Action 文件中相应地更新分支名称。

3. **将 GPG 相关的密钥添加到您的仓库中**：
   - 转到仓库的设置 > 秘密和变量 > Actions
   - 添加新的仓库秘密：
     - `GPG_KEY_ID`
     - `GPG_PASSPHRASE`
     - `GPG_PRIVATE_KEY`

4. **第一次运行后，将创建一个新的分支 `gh-pages`**：
   - 转到仓库的设置 > Pages
   - 在“构建和部署”下，选择 `Source` 为 `Deploy from a branch` 并将分支设置为 `gh-pages`

## GitLab Pages 步骤（替代方案）

1. **创建一个新的 GitLab 仓库**。

2. **将 `.gitlab-ci.yml` 文件添加到仓库的根目录**。您可以在 [这里](#gitlab-ci-yml) 找到此文件的内容。

> 注意：这些说明假定您使用的是 `main` 分支。如果您使用其他分支，请确保在 GitLab CI 配置文件中相应地更新分支名称。

3. **将 GPG 相关的变量添加到您的仓库中**：
   - 转到仓库的设置 > CI/CD > Variables
   - 添加新的变量：
     - `GPG_KEY_ID`
     - `GPG_PASSPHRASE`
     - `GPG_PRIVATE_KEY`（使用 base64 编码的 GPG 私钥）

> 注意：
> 1. 创建变量时选择可见性为 `Masked`。
> 2. GitLab 不支持变量中的空格，因此请使用 base64 编码的 GPG 私钥。

## 使用方法

要添加新包：

1. 将 `.deb` 文件放置在仓库的 `packages` 目录中。
2. 提交并推送到 `main` 分支。
3. CI/CD 管道将自动更新仓库并部署到 GitHub/GitLab Pages。

您的个人 Droidian 包仓库将可通过以下地址访问：
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
