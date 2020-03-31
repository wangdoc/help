# Wangdoc 新增仓库的流程

## Git 初始化

新建的文档仓库，应该命名为`xxx-tutorial`。其中，`xxx`为该仓库的主题，比如`javascript-tutorial`。

将这个仓库进行 Git 初始化，然后新建`.gitignore`，写入下面的内容。

```javascript
node_modules/
dist/
package-lock.json
```

## npm 模块初始化

第一步，新建`package.json`。

```bash
$ npm init -y
```

`package.json`的`license`字段可以改成创意共享许可证（署名-相同方式共享）。

```javascript
"license": "CC-BY-SA-4.0",
```

第二步，安装依赖。需要以下四个库。

  - loppo
  - loppo-theme-wangdoc
  - gh-pages
  - husky

```bash
$ npm install --save loppo@latest loppo-theme-wangdoc@latest gh-pages husky
```

为了确保安装的是最新版本，可以打开`package.json`，将`loppo`和`loppo-theme-wangdoc`的版本改成`latest`，然后执行一下`npm update`。

```bash
$ npm update
```

第三步，打开`package.json`，在`scripts`字段下插入如下脚本。

```javascript
"scripts": {
    "build": "loppo --site \"[xxx] 教程\" --id [xxx] --theme wangdoc",
    "build-and-commit": "npm run build && npm run commit",
    "commit": "gh-pages --dist dist --dest dist/[xxx] --branch master --repo git@github.com:wangdoc/website.git",
    "chapter": "loppo chapter",
    "server": "loppo server"
},
"husky": {
  "hooks": {
    "pre-push": "npm update"
  }
},
```

注意，要把脚本里面的`[xxx]`替换掉。

最后，本地运行`npm run server`，看看构建是否正确。

## 持续构建

第一步，到 [Travis CI 的官网](https://travis-ci.org/organizations/wangdoc/repositories)，关联该仓库，开启自动构建。

注意，要到设置里面，把 Pull Request 触发自动构建的选项关掉。

第二步，在项目根目录下，新建`.travis.yml`，写入以下内容。

```yml
language: node_js
node_js:
- '10'

branches:
  only:
  - master

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

第四步，在项目根目录下，添加私钥`wangdoc-deploy-rsa`。

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

## Disqus 讨论区

第一步，到 Disqus 新建讨论区，讨论区的 slug 为`wangdoc-[id]`。

第二步，设好 Disqus 以后，在`loppo.yml`里面设定如下设置。

```yaml
hasComments: true|false
isTutorial: true|false
```

