# Wangdoc 新增仓库的流程

## Git 初始化

新建一个文档仓库，命名为`xxx-tutorial`。其中，`xxx`为该仓库的主题，比如`javascript-tutorial`。

将这个仓库进行 Git 初始化。

```bash
$ git init
```

然后，新建`.gitignore`，写入下面的内容。

```bash
node_modules/
dist/
package-lock.json
```

## npm 模块初始化

第一步，新建`package.json`。

```bash
$ npm init -y
```

第二步，修改`package.json`。

`package.json`的`license`字段可以改成创意共享许可证（署名-相同方式共享）。

```javascript
"license": "CC-BY-SA-4.0",
```

`author`字段改成“Ruan Yifeng”。

`scripts`字段下插入如下脚本。

```javascript
"scripts": {
    "build": "loppo --site \"[xxx] 教程\" --id [xxx] --theme wangdoc",
    "build-and-commit": "npm run build && npm run commit",
    "commit": "gh-pages --dist dist --dest dist/[xxx] --branch master --repo git@github.com:wangdoc/website.git",
    "chapter": "loppo chapter",
    "server": "loppo server"
},
```

注意，要把脚本里面的`[xxx]`替换掉，共有三处。

第三步，安装依赖。需要以下三个库。

  - loppo
  - loppo-theme-wangdoc
  - gh-pages

```bash
$ npm install --save loppo@latest loppo-theme-wangdoc@latest gh-pages
```

为了确保安装的是最新版本，可以打开`package.json`，将`loppo`和`loppo-theme-wangdoc`的版本改成`latest`，然后执行一下`npm update`。

```bash
$ npm update
```

第四步，运行`npm run chapter`，生成目录文件`chapters.yml`。

```bash
$ npm run chapter
```

编辑`chapters.yml`文件，使得目录编排正确。

然后，运行`npm run build`，生成`loppo.yml`文件。

第五步，本地运行`npm run server`，看看构建是否正确。

第六步，写入代码仓库。

```bash
$ git add -A
$ git commit -m "feat: first commit"
```

## GitHub 仓库

在 GitHub 的 wangdoc 团队下新建仓库，然后推送本地仓库。

## GitHub Actions

第一步，新建子目录`.github/workflows`。

```bash
$ mkdir -p .github/workflows
```

第三步，检查仓库设置的`secrets`里面，有没有`WANGDOC_BOT_TOKEN`这一项。如果没有，需要为 wangdoc-bot 生成一个 TOKEN 并添加在`secrets`里面。

第三步，新建配置文件`wangdoc.yml`。

```bash
$ vi .github/workflows/wangdoc.yml
```

文件内容如下。

```yaml
name: [XXX] tutorial CI
on:
  push:
    branches:
      - main

jobs:
  page-generator:
    name: Generating pages
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          persist-credentials: false
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 'latest'
      - name: Install dependencies
        run: npm install
      - name: Build pages
        run: npm run build
      - name: Deploy to website
        uses: JamesIves/github-pages-deploy-action@v4
        with:
          git-config-name: wangdoc-bot
          git-config-email: yifeng.ruan@gmail.com
          repository-name: wangdoc/website
          token: ${{ secrets.WANGDOC_BOT_TOKEN }}
          branch: master # The branch the action should deploy to.
          folder: dist # The folder the action should deploy.
          target-folder: dist/[xxx]
          clean: true # Automatically remove deleted files from the deploy branch
          commit-message: update from [xxx] tutorial
```

注意：

（1）将上面的`[XXX]`改成当前库，共有三处。

（2）上面的部署`token`，只有公开库才能拿到，免费账户的私有库不能获取这个 token。

如果`JamesIves/github-pages-deploy-action`使用 3.7.1 的老版，原始设置如下。

```yaml
      - name: Deploy to website
        uses: JamesIves/github-pages-deploy-action@3.7.1
        with:
          GIT_CONFIG_NAME: wangdoc-bot
          GIT_CONFIG_EMAIL: yifeng.ruan@gmail.com
          REPOSITORY_NAME: wangdoc/website
          ACCESS_TOKEN: ${{ secrets.WANGDOC_BOT_TOKEN }}
          BASE_BRANCH: master
          BRANCH: master # The branch the action should deploy to.
          FOLDER: dist # The folder the action should deploy.
          TARGET_FOLDER: dist/[xxx]
          CLEAN: true # Automatically remove deleted files from the deploy branch
          COMMIT_MESSAGE: update from [xxx] tutorial
```

## Travis-CI 构建

注意，如果使用 GitHub Actions，则不需要设置 Travis-CI。

第一步，到 [Travis CI 的官网](https://travis-ci.com/organizations/wangdoc/repositories)，关联该仓库，开启自动构建。

注意，要到设置里面，把 Pull Request 触发自动构建的选项关掉。

第二步，在项目根目录下，新建`.travis.yml`，写入以下内容。

```yml
language: node_js
node_js:
- 'node'

branches:
  only:
  - master

install:
- npm ci
# keep the npm cache around to speed up installs
cache:
  directories:
  - "$HOME/.npm"

script: bash ./deploy.sh
env:
  global:
  - ENCRYPTION_LABEL: [xxxxxx]
  - COMMIT_AUTHOR_EMAIL: yifeng.ruan@gmail.com
```

注意，上面代码中的`[xxxxxx]`要等到第四步进行替换。

第三步，项目根目录下，新建`deploy.sh`，写入以下内容。

```bash
#!/bin/bash
set -e # Exit with nonzero exit code if anything fails

# Get the deploy key by using Travis's stored variables to decrypt deploy_key.enc
ENCRYPTED_KEY_VAR="encrypted_${ENCRYPTION_LABEL}_key"
ENCRYPTED_IV_VAR="encrypted_${ENCRYPTION_LABEL}_iv"
ENCRYPTED_KEY=${!ENCRYPTED_KEY_VAR}
ENCRYPTED_IV=${!ENCRYPTED_IV_VAR}
openssl aes-256-cbc -K $ENCRYPTED_KEY -iv $ENCRYPTED_IV -in wangdoc-deploy-rsa.enc -out wangdoc-deploy-rsa -d
chmod 600 wangdoc-deploy-rsa
eval `ssh-agent -s`
ssh-add wangdoc-deploy-rsa

# Now that we're all set up, we can push.
# git push $SSH_REPO $TARGET_BRANCH
npm run build-and-commit
```

第四步（待验证，可以跳过这步，直接执行下一个命令），在项目根目录下，添加私钥`wangdoc-deploy-rsa`。

```bash
$ cp ~/.ssh/wangdoc-deploy-rsa .
```

然后，使用`travis`命令加密`wangdoc-deploy-rsa`。如果没有安装 travis ci 的命令行客户端，可以参考官方的[安装文档](https://github.com/travis-ci/travis.rb#installation)`sudo gem install travis`。

```bash
$ travis encrypt-file ~/.ssh/wangdoc-deploy-rsa
```

屏幕上会显示加密编号，将这个编号写入`.travis.yml`。可以参考这个[范例文件](https://github.com/wangdoc/javascript-tutorial/blob/master/.travis.yml)。

第五步，重要！此时一定要删掉私钥`wangdoc-deploy-rsa`，只留下`wangdoc-deploy-rsa.enc`。

```bash
$ rm wangdoc-deploy-rsa
```

## loppo.yml

在 loppo.yml 里面加入文档仓库所在的分支。

```yaml
branch: main
```

如果不加 branch 字段，默认值为`master`。

## Disqus 讨论区

第一步，到 Disqus 新建讨论区，讨论区的 slug （shortname）为`wangdoc-[id]`（需要点击链接“customize your url”）。

第二步，设好 Disqus 以后，在`loppo.yml`里面设定如下设置。

```yaml
hasComments: true|false
isTutorial: true|false
```

