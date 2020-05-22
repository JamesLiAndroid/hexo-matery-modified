---
title: 一阶段前端构建和CI/CD流程优化
top: false
cover: false
toc: true
mathjax: true
date: 2020-05-19 17:05:45
password:
summary:
tags: 前端打包
categories:
---

# 一阶段前端构建和CI/CD流程优化

## 前言

深受前端构建时间过长的荼毒，导致前端构建极端浪费时间的情况，当发布一次前端项目时，都要承受等待20多分钟的情况，这是完全不能接受的。所以非常需要进行前端构建的优化，于是首先针对构建流程进行优化，其次针对前端打包的流程进行优化。

## 原始构建方式分析

### 1. 构建配置

整体的构建是基于jenkins，通过nvm和npm插件进行构建的，然后去打包，最终构建docker镜像，去k8s中进行部署。整个配置图片如图：

![](构建配置图片.png)

### 2. 构建时间分析

然后进行构建的时间分析，以某次构建为案例进行构建分析，构建整体日志如下：

```
13:50:42 Started by user admin
13:50:42 Running as SYSTEM
13:50:42 Building in workspace /var/jenkins_home/workspace/test-frontend-k8s
13:50:42 using credential lisongyang-Test-CICD
13:50:42  > git rev-parse --is-inside-work-tree # timeout=10
13:50:42 Fetching changes from the remote Git repository
13:50:42  > git config remote.origin.url http://192.168.238.159:8181/test/frontend/test1.0-ui.git # timeout=10
13:50:42 Fetching upstream changes from http://192.168.238.159:8181/test/frontend/test1.0-ui.git
13:50:42  > git --version # timeout=10
13:50:42 using GIT_ASKPASS to set credentials 
13:50:42  > git fetch --tags --progress -- http://192.168.238.159:8181/test/frontend/test1.0-ui.git +refs/heads/*:refs/remotes/origin/* # timeout=10
13:50:43  > git rev-parse refs/remotes/origin/release^{commit} # timeout=10
13:50:43  > git rev-parse refs/remotes/origin/origin/release^{commit} # timeout=10
13:50:43 Checking out Revision 3a454a94861b96a5668cfe5606242f2f3955c0b0 (refs/remotes/origin/release)
13:50:43  > git config core.sparsecheckout # timeout=10
13:50:43  > git checkout -f 3a454a94861b96a5668cfe5606242f2f3955c0b0 # timeout=10
13:50:45 Commit message: "Merge branch 'dev' into release"
13:50:45  > git rev-list --no-walk 3a454a94861b96a5668cfe5606242f2f3955c0b0 # timeout=10
13:50:45 [test-frontend-k8s] $ bash -c "test -f /var/jenkins_home/nvm/nvm.sh"
13:50:45 NVM is already installed
13:50:45 
13:50:45 [test-frontend-k8s] $ bash -c "export > env.txt"
13:50:45 [test-frontend-k8s] $ bash -c "NVM_DIR=/var/jenkins_home/nvm && source $NVM_DIR/nvm.sh --no-use && NVM_NODEJS_ORG_MIRROR=http://nodejs.org/dist nvm install 10.12.0 && nvm use 10.12.0 && export > env.txt"
13:50:49 v10.12.0 is already installed.
13:50:53 Now using node v10.12.0 (npm v6.4.1)
13:50:55 Now using node v10.12.0 (npm v6.4.1)
13:50:55 [test-frontend-k8s] $ /bin/sh -xe /tmp/jenkins5873642462170513819.sh
13:50:55 + npm --registry https://registry.npm.taobao.org install -g yarn
13:51:06 /var/jenkins_home/nvm/versions/node/v10.12.0/bin/yarn -> /var/jenkins_home/nvm/versions/node/v10.12.0/lib/node_modules/yarn/bin/yarn.js
13:51:06 /var/jenkins_home/nvm/versions/node/v10.12.0/bin/yarnpkg -> /var/jenkins_home/nvm/versions/node/v10.12.0/lib/node_modules/yarn/bin/yarn.js
13:51:06 + yarn@1.22.4
13:51:06 updated 1 package in 5.6s
13:51:06 + yarn --registry https://registry.npm.taobao.org install
13:51:06 yarn install v1.22.4
13:51:07 [1/4] Resolving packages...
13:51:08 success Already up-to-date.
13:51:08 $ opencollective-postinstall
13:51:09 [96m[1mThank you for using vue-antd-pro![96m[1m
13:51:09 [0m[96mIf you rely on this package, please consider supporting our open collective:[22m[39m
13:51:09 > [94mhttps://opencollective.com/ant-design-pro-vue/donate[0m
13:51:09 
13:51:09 Done in 2.33s.
13:51:09 + yarn run build
13:51:09 yarn run v1.22.4
13:51:09 $ vue-cli-service build
13:51:22 
13:51:22 -  Building for production...
13:51:40 (node:23259) DeprecationWarning: Tapable.plugin is deprecated. Use new API on `.hooks` instead
13:53:11  WARNING  Compiled with 3 warnings1:53:11 PM
13:53:11 
13:53:11  warning  in ./src/views/operationManagementCenter/README.md
13:53:11 
13:53:11 Module parse failed: Assigning to rvalue (2:0)
13:53:11 You may need an appropriate loader to handle this file type, currently no loaders are configured to process this file. See https://webpack.js.org/concepts#loaders
13:53:11 
13:53:11  @ ./src/views lazy ^\.\/.*$ namespace object
13:53:11  @ ./src/router/generator-routers.js
13:53:11  @ ./src/store/modules/async-router.js
13:53:11  @ ./src/store/index.js
13:53:11  @ ./src/main.js
13:53:11  @ multi ./src/main.js
13:53:11 
13:53:11  warning  
13:53:11 
13:53:11 asset size limit: The following asset(s) exceed the recommended size limit (244 KiB).
13:53:11 This can impact web performance.
13:53:11 Assets: 
13:53:11   css/app.e64713f9.css (478 KiB)
13:53:11   js/chunk-7b215be7.2f6c4877.js (480 KiB)
13:53:11   css/chunk-vendors.6a5a4912.css (745 KiB)
13:53:11   js/chunk-vendors.896630e5.js (5.02 MiB)
13:53:11   lib/ace/ace.js (363 KiB)
13:53:11   lib/ace/mode-php.js (470 KiB)
13:53:11   lib/ace/mode-php_laravel_blade.js (474 KiB)
13:53:11   lib/ace/worker-coffee.js (324 KiB)
13:53:11   lib/ace/worker-xquery.js (1.56 MiB)
13:53:11 
13:53:11  warning  
13:53:11 
13:53:11 entrypoint size limit: The following entrypoint(s) combined asset size exceeds the recommended limit (244 KiB). This can impact web performance.
13:53:11 Entrypoints:
13:53:11   app (6.45 MiB)
13:53:11       css/chunk-vendors.6a5a4912.css
13:53:11       js/chunk-vendors.896630e5.js
13:53:11       css/app.e64713f9.css
13:53:11       js/app.245e3f1d.js
13:53:11 
13:53:11 
13:53:13   File                                      Size             Gzipped
13:53:13 
13:53:13   dist/lib/ace/mode-xquery.js               225.60 KiB       34.91 KiB
13:53:13   dist/lib/ace/worker-html.js               211.77 KiB       48.72 KiB
13:53:13   dist/lib/ace/worker-javascript.js         160.14 KiB       48.28 KiB
13:53:13   dist/lib/ace/worker-css.js                137.18 KiB       35.86 KiB
13:53:13   dist/lib/ace/mode-slim.js                 104.49 KiB       29.90 KiB
13:53:13   dist/lib/ace/keybinding-vim.js            96.70 KiB        29.52 KiB
13:53:13   dist/lib/ace/mode-html_elixir.js          76.08 KiB        20.79 KiB
13:53:13   dist/lib/ace/worker-php.js                75.62 KiB        21.10 KiB
13:53:13   dist/lib/ace/mode-markdown.js             70.67 KiB        20.78 KiB
13:53:13   dist/lib/ace/mode-html_ruby.js            69.95 KiB        20.69 KiB
13:53:13   dist/lib/ace/mode-ejs.js                  69.63 KiB        21.00 KiB
13:53:13   dist/lib/ace/mode-soy_template.js         67.59 KiB        19.25 KiB
13:53:13   dist/lib/ace/mode-luapage.js              67.53 KiB        20.13 KiB
13:53:13   dist/lib/ace/mode-csound_document.js      66.97 KiB        19.52 KiB
13:53:13   dist/lib/ace/mode-razor.js                66.18 KiB        19.67 KiB
13:53:13   dist/lib/ace/mode-rhtml.js                64.23 KiB        18.94 KiB
13:53:13   dist/lib/ace/mode-velocity.js             63.92 KiB        18.79 KiB
13:53:13   dist/lib/ace/mode-twig.js                 62.84 KiB        18.72 KiB
13:53:13   dist/lib/ace/mode-smarty.js               62.67 KiB        18.65 KiB
13:53:13   dist/lib/ace/mode-autohotkey.js           62.47 KiB        16.37 KiB
13:53:13   dist/lib/ace/mode-coldfusion.js           61.76 KiB        18.35 KiB
13:53:13   dist/lib/ace/mode-handlebars.js           61.12 KiB        18.09 KiB
13:53:13   dist/lib/ace/mode-django.js               60.58 KiB        18.03 KiB
13:53:13   dist/lib/ace/mode-curly.js                60.28 KiB        17.92 KiB
13:53:13   dist/lib/ace/mode-html.js                 59.29 KiB        17.77 KiB
13:53:13   dist/lib/ace/mode-pgsql.js                55.09 KiB        17.29 KiB
13:53:13   dist/lib/ace/mode-objectivec.js           53.76 KiB        18.01 KiB
13:53:13   dist/lib/ace/worker-xml.js                53.60 KiB        16.32 KiB
13:53:13   dist/lib/ace/mode-jade.js                 52.03 KiB        14.93 KiB
13:53:13   dist/js/chunk-ffcd749a.9a38b6de.js        50.88 KiB        12.69 KiB
13:53:13   dist/lib/ace/worker-lua.js                45.52 KiB        13.64 KiB
13:53:13   dist/lib/ace/mode-mask.js                 41.59 KiB        13.21 KiB
13:53:13   dist/lib/ace/mode-haml.js                 38.86 KiB        12.16 KiB
13:53:13   dist/lib/ace/mode-jsp.js                  36.57 KiB        11.55 KiB
13:53:13   dist/lib/ace/mode-csound_orchestra.js     36.44 KiB        10.57 KiB
13:53:13   dist/lib/ace/snippets/lsl.js              34.91 KiB        6.97 KiB
13:53:13   dist/lib/ace/ext-language_tools.js        34.74 KiB        10.97 KiB
13:53:13   dist/lib/ace/mode-powershell.js           32.25 KiB        8.61 KiB
13:53:13   dist/lib/ace/mode-ftl.js                  32.18 KiB        10.29 KiB
13:53:13   dist/lib/ace/mode-liquid.js               31.85 KiB        10.09 KiB
13:53:13   dist/lib/ace/mode-svg.js                  31.67 KiB        9.15 KiB
13:53:13   dist/lib/ace/worker-json.js               31.64 KiB        9.40 KiB
13:53:13   dist/lib/ace/mode-erlang.js               29.34 KiB        4.71 KiB
13:53:13   dist/lib/ace/theme-ambiance.js            27.14 KiB        18.98 KiB
13:53:13   dist/lib/ace/mode-lsl.js                  26.31 KiB        8.82 KiB
13:53:13   dist/js/chunk-2394da78.9aaff4ed.js        26.02 KiB        4.22 KiB
13:53:13   dist/js/chunk-4e4be866.01b60c14.js        25.66 KiB        6.04 KiB
13:53:13   dist/js/chunk-23759318.77953657.js        25.47 KiB        6.13 KiB
13:53:13   dist/lib/ace/mode-mel.js                  24.67 KiB        10.43 KiB
13:53:13   dist/js/chunk-eb8257d2.c62af659.js        24.09 KiB        5.64 KiB
13:53:13   dist/lib/ace/keybinding-emacs.js          23.70 KiB        6.74 KiB
13:53:13   dist/js/chunk-682634f1.271074ae.js        23.52 KiB        5.89 KiB
13:53:13   dist/lib/ace/mode-scala.js                22.53 KiB        7.69 KiB
13:53:13   dist/lib/ace/mode-groovy.js               22.35 KiB        7.53 KiB
13:53:13   dist/lib/ace/mode-less.js                 22.31 KiB        7.27 KiB
13:53:13   dist/lib/ace/mode-java.js                 22.14 KiB        7.48 KiB
13:53:13   dist/js/chunk-78733bee.c6949174.js        21.72 KiB        5.00 KiB
13:53:13   dist/js/chunk-0201a64a.019142e7.js        21.38 KiB        5.42 KiB
13:53:13   dist/lib/ace/mode-sjs.js                  21.22 KiB        6.88 KiB
13:53:13   dist/lib/ace/ext-emmet.js                 21.16 KiB        7.13 KiB
13:53:13   dist/lib/ace/snippets/ruby.js             21.10 KiB        5.84 KiB
13:53:13   dist/js/chunk-213aeccc.602fc2e3.js        20.50 KiB        5.22 KiB
13:53:13   dist/js/chunk-ec42ef58.de24853f.js        20.44 KiB        5.00 KiB
13:53:13   dist/lib/ace/mode-matlab.js               20.40 KiB        8.81 KiB
13:53:13   dist/lib/ace/mode-actionscript.js         20.39 KiB        7.92 KiB
13:53:13   dist/lib/ace/mode-css.js                  20.32 KiB        6.88 KiB
13:53:13   dist/lib/ace/mode-gobstones.js            20.30 KiB        6.84 KiB
13:53:13   dist/lib/ace/mode-wollok.js               20.27 KiB        6.83 KiB
13:53:13   dist/lib/ace/mode-tsx.js                  20.12 KiB        6.70 KiB
13:53:13   dist/lib/ace/mode-typescript.js           19.83 KiB        6.64 KiB
13:53:13   dist/lib/ace/snippets/css.js              19.51 KiB        4.00 KiB
13:53:13   dist/js/chunk-60867039.42fd60bd.js        19.41 KiB        5.11 KiB
13:53:13   dist/js/chunk-05bc1686.2fe54f4d.js        19.07 KiB        5.20 KiB
13:53:13   dist/lib/ace/mode-ocaml.js                15.54 KiB        5.54 KiB
13:53:13   dist/js/chunk-e625997c.e374ecd3.js        14.70 KiB        2.72 KiB
13:53:13   dist/lib/ace/mode-scss.js                 14.57 KiB        5.29 KiB
13:53:13   dist/lib/ace/mode-stylus.js               14.39 KiB        5.02 KiB
13:53:13   dist/lib/ace/mode-dart.js                 14.37 KiB        4.67 KiB
13:53:13   dist/lib/ace/ext-settings_menu.js         14.35 KiB        5.76 KiB
13:53:13   dist/js/chunk-25ed122e.3e2e562c.js        13.98 KiB        3.65 KiB
13:53:13   dist/lib/ace/mode-apache_conf.js          13.91 KiB        4.40 KiB
13:53:13   dist/lib/ace/ext-options.js               13.79 KiB        5.61 KiB
13:53:13   dist/js/chunk-f3f382b8.14ae366f.js        13.66 KiB        3.52 KiB
13:53:13   dist/js/chunk-cf91679c.d4e659a9.js        13.54 KiB        3.22 KiB
13:53:13   dist/lib/ace/mode-nix.js                  13.33 KiB        4.35 KiB
13:53:13   dist/js/chunk-4a1e5c3b.0b038e96.js        13.30 KiB        3.88 KiB
13:53:13   dist/lib/ace/mode-glsl.js                 13.21 KiB        4.54 KiB
13:53:13   dist/js/chunk-9a54878e.9774f6d5.js        13.20 KiB        3.46 KiB
13:53:13   dist/js/chunk-721b8175.890e4a39.js        13.13 KiB        2.43 KiB
13:53:13   dist/js/user.9301b70a.js                  12.87 KiB        3.86 KiB
13:53:13   dist/lib/ace/mode-protobuf.js             12.76 KiB        4.25 KiB
13:53:13   dist/lib/ace/mode-red.js                  12.62 KiB        4.67 KiB
13:53:13   dist/js/chunk-2fadf79e.b2035703.js        12.39 KiB        3.13 KiB
13:53:13   dist/js/chunk-07d9aff0.7e3a95ec.js        12.37 KiB        3.24 KiB
13:53:13   dist/js/chunk-49b7fd8f.b8456323.js        12.34 KiB        3.14 KiB
13:53:13   dist/js/chunk-945a6b30.d0020035.js        12.20 KiB        3.12 KiB
13:53:13   dist/js/chunk-319fd138.eb33bfee.js        12.11 KiB        3.10 KiB
13:53:13   dist/js/chunk-6d233d0a.80c38f77.js        12.07 KiB        3.38 KiB
13:53:13   dist/js/chunk-061a5fa2.3dc97429.js        12.03 KiB        2.53 KiB
13:53:13   dist/lib/ace/mode-kotlin.js               11.97 KiB        2.83 KiB
13:53:13   dist/js/chunk-4a00d4af.d3fab71e.js        11.93 KiB        3.22 KiB
13:53:13   dist/lib/ace/mode-xml.js                  11.89 KiB        3.32 KiB
13:53:13   dist/js/chunk-112948ec.81b3d619.js        11.73 KiB        2.65 KiB
13:53:13   dist/lib/ace/ext-searchbox.js             11.64 KiB        3.49 KiB
13:53:13   dist/lib/ace/mode-sass.js                 11.62 KiB        4.34 KiB
13:53:13   dist/js/chunk-158513f2.c63e7bf4.js        11.61 KiB        2.96 KiB
13:53:13   dist/js/chunk-574e585c.cd2faf98.js        11.55 KiB        2.91 KiB
13:53:13   dist/lib/ace/mode-haskell.js              11.53 KiB        3.75 KiB
13:53:13   dist/js/chunk-cbe1287c.6a5b89c0.js        11.36 KiB        3.17 KiB
13:53:13   dist/lib/ace/mode-drools.js               11.24 KiB        3.12 KiB
13:53:13   dist/js/chunk-7cb23b21.0baab3b1.js        11.20 KiB        4.19 KiB
13:53:13   dist/lib/ace/mode-c_cpp.js                11.11 KiB        3.96 KiB
13:53:13   dist/js/chunk-37598709.b71d6923.js        10.94 KiB        2.90 KiB
13:53:13   dist/js/chunk-1afc103e.b838c85d.js        10.94 KiB        2.90 KiB
13:53:13   dist/js/chunk-30c32723.7fc373ff.js        10.94 KiB        2.90 KiB
13:53:13   dist/js/chunk-e83ecf78.ddc98804.js        10.94 KiB        2.90 KiB
13:53:13   dist/js/chunk-3a8c83d5.d7fcb4d9.js        10.93 KiB        2.89 KiB
13:53:13   dist/lib/ace/mode-praat.js                10.47 KiB        3.84 KiB
13:53:13   dist/js/chunk-1d9fe70c.87f23114.js        10.43 KiB        3.25 KiB
13:53:13   dist/js/chunk-f16e9740.8f01b904.js        10.34 KiB        3.13 KiB
13:53:13   dist/lib/ace/mode-nsis.js                 10.31 KiB        3.91 KiB
13:53:13   dist/js/chunk-5f261a47.17b02d91.js        10.30 KiB        3.10 KiB
13:53:13   dist/js/chunk-7ea2dba3.4184708e.js        10.20 KiB        3.05 KiB
13:53:13   dist/js/chunk-e7434404.e5c2f054.js        10.20 KiB        3.05 KiB
13:53:13   dist/js/chunk-421e36d3.5b8c8b28.js        10.20 KiB        3.04 KiB
13:53:13   dist/lib/ace/mode-ruby.js                 10.17 KiB        3.81 KiB
13:53:13   dist/js/chunk-0fa7c848.341d5fff.js        10.08 KiB        2.97 KiB
13:53:13   dist/js/chunk-2a999ff8.b86dff40.js        10.08 KiB        3.20 KiB
13:53:13   dist/js/chunk-fa671de4.8e841360.js        9.76 KiB         2.23 KiB
13:53:13   dist/js/chunk-1476d79a.46892e12.js        9.42 KiB         2.92 KiB
13:53:13   dist/js/chunk-47294af0.399c5861.js        9.41 KiB         2.91 KiB
13:53:13   dist/js/chunk-7793b159.61431453.js        9.24 KiB         2.86 KiB
13:53:13   dist/js/chunk-608e2fe3.0ae4f9e3.js        9.16 KiB         2.12 KiB
13:53:13   dist/lib/ace/ext-textarea.js              9.11 KiB         3.60 KiB
13:53:13   dist/js/chunk-7775a5f9.00e80283.js        9.08 KiB         2.02 KiB
13:53:13   dist/lib/ace/mode-d.js                    9.05 KiB         3.22 KiB
13:53:13   dist/lib/ace/mode-assembly_x86.js         9.01 KiB         3.39 KiB
13:53:13   dist/lib/ace/mode-csharp.js               8.94 KiB         2.78 KiB
13:53:13   dist/lib/ace/mode-asl.js                  8.86 KiB         3.42 KiB
13:53:13   dist/js/chunk-12effcec.12ef237e.js        8.80 KiB         1.95 KiB
13:53:13   dist/js/chunk-d114fb66.b31b93ab.js        8.58 KiB         2.56 KiB
13:53:13   dist/lib/ace/mode-fortran.js              8.55 KiB         3.59 KiB
13:53:13   dist/js/chunk-2337c4ca.c69650d6.js        8.53 KiB         2.41 KiB
13:53:13   dist/lib/ace/mode-prolog.js               8.49 KiB         2.50 KiB
13:53:13   dist/lib/ace/mode-dockerfile.js           8.35 KiB         2.97 KiB
13:53:13   dist/lib/ace/mode-asciidoc.js             8.28 KiB         2.57 KiB
13:53:13   dist/lib/ace/mode-clojure.js              8.15 KiB         3.32 KiB
13:53:13   dist/lib/ace/mode-redshift.js             8.12 KiB         2.77 KiB
13:53:13   dist/js/chunk-1a25084c.61b9a82e.js        8.10 KiB         2.43 KiB
13:53:13   dist/lib/ace/mode-sparql.js               7.99 KiB         2.55 KiB
13:53:13   dist/js/chunk-bba7e5be.cefce3ce.js        7.81 KiB         2.76 KiB
13:53:13   dist/lib/ace/mode-dot.js                  7.70 KiB         2.75 KiB
13:53:13   dist/lib/ace/mode-julia.js                7.68 KiB         2.45 KiB
13:53:13   dist/lib/ace/mode-csound_score.js         7.61 KiB         1.91 KiB
13:53:13   dist/js/chunk-e26d2c80.a37f666c.js        7.58 KiB         2.92 KiB
13:53:13   dist/lib/ace/mode-coffee.js               7.56 KiB         2.76 KiB
13:53:13   dist/lib/ace/mode-perl.js                 7.52 KiB         2.91 KiB
13:53:13   dist/lib/ace/mode-lua.js                  7.43 KiB         2.86 KiB
13:53:13   dist/lib/ace/mode-sh.js                   7.34 KiB         2.71 KiB
13:53:13   dist/lib/ace/mode-puppet.js               7.28 KiB         2.59 KiB
13:53:13   dist/lib/ace/mode-forth.js                7.15 KiB         2.46 KiB
13:53:13   dist/js/chunk-1273618c.52bf12a5.js        7.14 KiB         2.32 KiB
13:53:13   dist/lib/ace/snippets/php.js              7.11 KiB         2.07 KiB
13:53:13   dist/lib/ace/mode-jsx.js                  7.11 KiB         2.55 KiB
13:53:13   dist/lib/ace/mode-golang.js               7.06 KiB         2.53 KiB
13:53:13   dist/lib/ace/mode-swift.js                7.06 KiB         2.63 KiB
13:53:13   dist/lib/ace/mode-terraform.js            6.97 KiB         2.36 KiB
13:53:13   dist/js/chunk-14e07ffc.6aa6124e.js        6.89 KiB         1.95 KiB
13:53:13   dist/lib/ace/mode-mushcode.js             6.87 KiB         3.04 KiB
13:53:13   dist/lib/ace/mode-rust.js                 6.71 KiB         2.61 KiB
13:53:13   dist/lib/ace/mode-makefile.js             6.71 KiB         2.28 KiB
13:53:13   dist/lib/ace/mode-haxe.js                 6.69 KiB         2.38 KiB
13:53:13   dist/lib/ace/mode-mysql.js                6.68 KiB         2.84 KiB
13:53:13   dist/lib/ace/mode-scad.js                 6.67 KiB         2.23 KiB
13:53:13   dist/lib/ace/theme-iplastic.js            6.65 KiB         3.85 KiB
13:53:13   dist/lib/ace/mode-pig.js                  6.48 KiB         2.53 KiB
13:53:13   dist/lib/ace/mode-bro.js                  6.38 KiB         2.27 KiB
13:53:13   dist/lib/ace/mode-tcl.js                  6.29 KiB         1.95 KiB
13:53:13   dist/lib/ace/mode-hjson.js                6.15 KiB         1.97 KiB
13:53:13   dist/lib/ace/mode-jssm.js                 6.08 KiB         1.72 KiB
13:53:13   dist/lib/ace/mode-abap.js                 5.99 KiB         2.60 KiB
13:53:13   dist/js/chunk-0b27757e.087b2317.js        5.97 KiB         2.30 KiB
13:53:13   dist/lib/ace/mode-jack.js                 5.94 KiB         2.06 KiB
13:53:13   dist/lib/ace/mode-logiql.js               5.89 KiB         2.05 KiB
13:53:13   dist/lib/ace/mode-io.js                   5.85 KiB         2.11 KiB
13:53:13   dist/js/chunk-effdf402.389432cd.js        5.81 KiB         2.21 KiB
13:53:13   dist/lib/ace/snippets/perl.js             5.71 KiB         2.12 KiB
13:53:13   dist/js/chunk-30ec3699.c9d46488.js        5.67 KiB         2.02 KiB
13:53:13   dist/lib/ace/mode-applescript.js          5.60 KiB         2.29 KiB
13:53:13   dist/lib/ace/mode-yaml.js                 4.56 KiB         1.62 KiB
13:53:13   dist/lib/ace/snippets/java.js             4.54 KiB         1.59 KiB
13:53:13   dist/lib/ace/snippets/edifact.js          4.54 KiB         1.59 KiB
13:53:13   dist/lib/ace/mode-c9search.js             4.30 KiB         1.59 KiB
13:53:13   dist/lib/ace/ext-modelist.js              4.26 KiB         2.17 KiB
13:53:13   dist/js/chunk-34986174.5704663c.js        4.25 KiB         1.46 KiB
13:53:13   dist/lib/ace/snippets/django.js           4.23 KiB         1.38 KiB
13:53:13   dist/js/chunk-6923b696.3386014d.js        4.11 KiB         1.47 KiB
13:53:13   dist/lib/ace/snippets/javascript.js       4.08 KiB         1.52 KiB
13:53:13   dist/lib/ace/ext-keybinding_menu.js       4.07 KiB         1.59 KiB
13:53:13   dist/js/chunk-2d0c0afd.48291ccc.js        4.00 KiB         1.63 KiB
13:53:13   dist/lib/ace/ext-beautify.js              3.97 KiB         1.70 KiB
13:53:13   dist/lib/ace/ext-elastic_tabstops_lite    3.94 KiB         1.49 KiB
13:53:13   .js
13:53:13   dist/lib/ace/mode-snippets.js             3.92 KiB         1.41 KiB
13:53:13   dist/lib/ace/snippets/python.js           3.91 KiB         1.35 KiB
13:53:13   dist/lib/ace/mode-scheme.js               3.88 KiB         1.52 KiB
13:53:13   dist/lib/ace/snippets/tex.js              3.87 KiB         1.30 KiB
13:53:13   dist/lib/ace/ext-static_highlight.js      3.84 KiB         1.64 KiB
13:53:13   dist/js/chunk-771e4ee6.ea048885.js        3.84 KiB         1.36 KiB
13:53:13   dist/lib/ace/snippets/erlang.js           3.81 KiB         1.29 KiB
13:53:13   dist/lib/ace/ext-split.js                 3.78 KiB         1.23 KiB
13:53:13   dist/lib/ace/theme-tomorrow_night_brig    3.75 KiB         1.04 KiB
13:53:13   ht.js
13:53:13   dist/lib/ace/mode-graphqlschema.js        3.68 KiB         1.36 KiB
13:53:13   dist/js/chunk-cad5ac02.66c4e86a.js        3.57 KiB         1.47 KiB
13:53:13   dist/lib/ace/theme-tomorrow_night_eigh    3.48 KiB         0.94 KiB
13:53:13   ties.js
13:53:13   dist/lib/ace/theme-dreamweaver.js         3.39 KiB         1.03 KiB
13:53:13   dist/lib/ace/theme-katzenmilch.js         3.38 KiB         1.03 KiB
13:53:13   dist/lib/ace/snippets/vala.js             3.38 KiB         1.04 KiB
13:53:13   dist/lib/ace/theme-tomorrow_night_blue    3.29 KiB         0.95 KiB
13:53:13   .js
13:53:13   dist/lib/ace/mode-rst.js                  3.27 KiB         1.05 KiB
13:53:13   dist/lib/ace/mode-cirru.js                3.24 KiB         1.01 KiB
13:53:13   dist/lib/ace/snippets/actionscript.js     3.23 KiB         1.26 KiB
13:53:13   dist/lib/ace/theme-terminal.js            3.17 KiB         0.97 KiB
13:53:13   dist/lib/ace/mode-verilog.js              3.15 KiB         1.32 KiB
13:53:13   dist/lib/ace/theme-sqlserver.js           3.15 KiB         1.05 KiB
13:53:13   dist/lib/ace/theme-chaos.js               3.09 KiB         1.00 KiB
13:53:13   dist/lib/ace/mode-eiffel.js               3.09 KiB         1.18 KiB
13:53:13   dist/lib/ace/mode-mixal.js                3.09 KiB         1.08 KiB
13:53:13   dist/lib/ace/theme-tomorrow_night.js      3.08 KiB         0.95 KiB
13:53:13   dist/lib/ace/theme-crimson_editor.js      3.07 KiB         0.98 KiB
13:53:13   dist/lib/ace/theme-mono_industrial.js     3.05 KiB         0.96 KiB
13:53:13   dist/js/chunk-1d68162b.43d062f2.js        3.05 KiB         1.21 KiB
13:53:13   dist/lib/ace/mode-edifact.js              3.03 KiB         1.29 KiB
13:53:13   dist/lib/ace/snippets/jsp.js              3.03 KiB         0.93 KiB
13:53:13   dist/lib/ace/ext-whitespace.js            3.02 KiB         1.36 KiB
13:53:13   dist/lib/ace/mode-tex.js                  3.00 KiB         0.96 KiB
13:53:13   dist/lib/ace/theme-chrome.js              2.97 KiB         1.03 KiB
13:53:13   dist/lib/ace/mode-ini.js                  2.96 KiB         1.08 KiB
13:53:13   dist/lib/ace/snippets/c_cpp.js            2.92 KiB         0.99 KiB
13:53:13   dist/lib/ace/theme-pastel_on_dark.js      2.90 KiB         0.97 KiB
13:53:13   dist/lib/ace/snippets/r.js                2.89 KiB         0.85 KiB
13:53:13   dist/lib/ace/theme-textmate.js            2.87 KiB         1.03 KiB
13:53:13   dist/lib/ace/theme-dracula.js             2.84 KiB         0.95 KiB
13:53:13   dist/lib/ace/theme-tomorrow.js            2.82 KiB         0.92 KiB
13:53:13   dist/lib/ace/theme-twilight.js            2.76 KiB         0.96 KiB
13:53:13   dist/lib/ace/theme-merbivore_soft.js      2.72 KiB         0.90 KiB
13:53:13   dist/lib/ace/theme-clouds_midnight.js     2.70 KiB         0.89 KiB
13:53:13   dist/lib/ace/mode-diff.js                 2.66 KiB         0.96 KiB
13:53:13   dist/lib/ace/theme-gob.js                 2.64 KiB         0.98 KiB
13:53:13   dist/lib/ace/mode-gherkin.js              2.64 KiB         0.98 KiB
13:53:13   dist/lib/ace/theme-monokai.js             2.64 KiB         0.92 KiB
13:53:13   dist/lib/ace/theme-solarized_light.js     2.63 KiB         0.88 KiB
13:53:13   dist/lib/ace/theme-cobalt.js              2.62 KiB         0.94 KiB
13:53:13   dist/lib/ace/theme-solarized_dark.js      2.59 KiB         0.89 KiB
13:53:13   dist/lib/ace/mode-cobol.js                2.57 KiB         1.29 KiB
13:53:13   dist/lib/ace/theme-kr_theme.js            2.56 KiB         0.93 KiB
13:53:13   dist/lib/ace/mode-ada.js                  2.56 KiB         1.17 KiB
13:53:13   dist/lib/ace/mode-haskell_cabal.js        2.55 KiB         0.92 KiB
13:53:13   dist/lib/ace/theme-idle_fingers.js        2.52 KiB         0.89 KiB
13:53:13   dist/lib/ace/theme-merbivore.js           2.51 KiB         0.87 KiB
13:53:13   dist/lib/ace/theme-dawn.js                2.51 KiB         0.92 KiB
13:53:13   dist/lib/ace/snippets/coffee.js           2.50 KiB         0.79 KiB
13:53:13   dist/lib/ace/mode-space.js                2.49 KiB         0.85 KiB
13:53:13   dist/lib/ace/theme-vibrant_ink.js         2.48 KiB         0.86 KiB
13:53:13   dist/lib/ace/mode-toml.js                 2.46 KiB         0.96 KiB
13:53:13   dist/lib/ace/theme-github.js              2.45 KiB         0.89 KiB
13:53:13   dist/lib/ace/snippets/sqlserver.js        2.42 KiB         0.97 KiB
13:53:13   dist/lib/ace/theme-eclipse.js             2.40 KiB         0.87 KiB
13:53:13   dist/lib/ace/theme-clouds.js              2.35 KiB         0.85 KiB
13:53:13   dist/lib/ace/mode-vhdl.js                 2.35 KiB         1.07 KiB
13:53:13   dist/lib/ace/mode-textile.js              2.35 KiB         0.85 KiB
13:53:13   dist/lib/ace/ext-rtl.js                   2.35 KiB         0.94 KiB
13:53:13   dist/lib/ace/theme-kuroir.js              2.33 KiB         0.83 KiB
13:53:13   dist/lib/ace/snippets/sh.js               2.32 KiB         0.93 KiB
13:53:13   dist/lib/ace/snippets/clojure.js          2.32 KiB         0.80 KiB
13:53:13   dist/lib/ace/snippets/haskell.js          2.25 KiB         0.91 KiB
13:53:13   dist/lib/ace/snippets/markdown.js         2.25 KiB         0.80 KiB
13:53:13   dist/lib/ace/mode-lisp.js                 2.23 KiB         0.91 KiB
13:53:13   dist/lib/ace/theme-xcode.js               2.19 KiB         0.82 KiB
13:53:13   dist/lib/ace/mode-sql.js                  2.11 KiB         0.97 KiB
13:53:13   dist/js/chunk-11ebe073.a0e42fd1.js        2.07 KiB         0.91 KiB
13:53:13   dist/lib/ace/snippets/xquery.js           2.00 KiB         0.66 KiB
13:53:13   dist/lib/ace/snippets/jsoniq.js           2.00 KiB         0.66 KiB
13:53:13   dist/lib/ace/snippets/tcl.js              1.97 KiB         0.73 KiB
13:53:13   dist/lib/ace/theme-gruvbox.js             1.95 KiB         0.78 KiB
13:53:13   dist/js/chunk-2d0c1f36.8b3907fe.js        1.83 KiB         0.90 KiB
13:53:13   dist/lib/ace/mode-gcode.js                1.74 KiB         0.74 KiB
13:53:13   dist/lib/ace/ext-themelist.js             1.68 KiB         0.71 KiB
13:53:13   dist/lib/ace/mode-lucene.js               1.66 KiB         0.63 KiB
13:53:13   dist/lib/ace/snippets/dart.js             1.61 KiB         0.61 KiB
13:53:13   dist/lib/ace/ext-spellcheck.js            1.60 KiB         0.69 KiB
13:53:13   dist/lib/ace/snippets/wollok.js           1.56 KiB         0.64 KiB
13:53:13   dist/lib/ace/mode-csp.js                  1.53 KiB         0.69 KiB
13:53:13   dist/lib/ace/snippets/io.js               1.51 KiB         0.59 KiB
13:53:13   dist/lib/ace/mode-properties.js           1.39 KiB         0.51 KiB
13:53:13   dist/lib/ace/snippets/csound_orchestra    1.38 KiB         0.46 KiB
13:53:13   .js
13:53:13   dist/lib/ace/ext-statusbar.js             1.25 KiB         0.60 KiB
13:53:13   dist/lib/ace/snippets/abc.js              1.25 KiB         0.50 KiB
13:53:13   dist/lib/ace/snippets/sql.js              1.24 KiB         0.48 KiB
13:53:13   dist/js/chunk-2d0b6ecc.d84b63d8.js        1.24 KiB         0.70 KiB
13:53:13   dist/js/chunk-2d0dcff1.8f257698.js        1.20 KiB         0.69 KiB
13:53:13   dist/lib/ace/mode-gitignore.js            1.20 KiB         0.46 KiB
13:53:13   dist/lib/ace/ext-linking.js               1.19 KiB         0.48 KiB
13:53:13   dist/js/chunk-fb0f7f1a.8a629e9f.js        1.14 KiB         0.58 KiB
13:53:13   dist/js/chunk-2d0aa1b9.11d1012d.js        1.07 KiB         0.72 KiB
13:53:13   dist/lib/ace/snippets/graphqlschema.js    0.98 KiB         0.37 KiB
13:53:13   dist/lib/ace/snippets/velocity.js         0.96 KiB         0.42 KiB
13:53:13   dist/lib/ace/snippets/gobstones.js        0.92 KiB         0.39 KiB
13:53:13   dist/js/chunk-2d207786.1df04e6d.js        0.91 KiB         0.61 KiB
13:53:13   dist/js/chunk-2d230473.2d79c76f.js        0.86 KiB         0.41 KiB
13:53:13   dist/lib/ace/snippets/diff.js             0.86 KiB         0.47 KiB
13:53:13   dist/lib/ace/snippets/textile.js          0.85 KiB         0.42 KiB
13:53:13   dist/lib/ace/mode-plain_text.js           0.82 KiB         0.36 KiB
13:53:13   dist/lib/ace/snippets/lua.js              0.82 KiB         0.36 KiB
13:53:13   dist/lib/ace/snippets/csound_score.js     0.47 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/assembly_x86.js     0.47 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/soy_template.js     0.47 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/html_elixir.js      0.46 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/applescript.js      0.46 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/apache_conf.js      0.46 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/plain_text.js       0.46 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/powershell.js       0.46 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/dockerfile.js       0.46 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/properties.js       0.46 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/autohotkey.js       0.46 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/typescript.js       0.46 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/livescript.js       0.46 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/objectivec.js       0.46 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/handlebars.js       0.46 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/coldfusion.js       0.46 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/gitignore.js        0.46 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/batchfile.js        0.46 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/html_ruby.js        0.46 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/terraform.js        0.46 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/protobuf.js         0.45 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/mushcode.js         0.45 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/vbscript.js         0.45 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/asciidoc.js         0.45 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/c9search.js         0.45 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/redshift.js         0.45 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/verilog.js          0.45 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/luapage.js          0.45 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/fortran.js          0.45 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/gherkin.js          0.45 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/kotlin.js           0.45 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/sparql.js           0.45 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/elixir.js           0.45 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/turtle.js           0.45 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/smarty.js           0.45 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/stylus.js           0.45 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/prolog.js           0.45 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/scheme.js           0.45 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/pascal.js           0.45 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/golang.js           0.45 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/eiffel.js           0.45 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/puppet.js           0.45 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/liquid.js           0.45 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/matlab.js           0.45 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/lucene.js           0.45 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/logiql.js           0.45 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/fsharp.js           0.45 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/groovy.js           0.45 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/csharp.js           0.45 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/hjson.js            0.45 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/jssm.js             0.45 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/praat.js            0.45 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/mysql.js            0.45 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/nsis.js             0.45 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/latex.js            0.45 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/ocaml.js            0.45 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/julia.js            0.45 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/pgsql.js            0.45 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/curly.js            0.45 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/scala.js            0.45 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/mixal.js            0.45 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/cobol.js            0.45 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/cirru.js            0.45 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/gcode.js            0.45 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/forth.js            0.45 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/swift.js            0.45 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/rhtml.js            0.45 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/space.js            0.45 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/bro.js              0.44 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/csp.js              0.44 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/mask.js             0.44 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/text.js             0.44 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/abap.js             0.44 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/twig.js             0.44 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/lisp.js             0.44 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/yaml.js             0.44 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/rust.js             0.44 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/sass.js             0.44 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/scad.js             0.44 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/less.js             0.44 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/json.js             0.44 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/scss.js             0.44 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/slim.js             0.44 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/glsl.js             0.44 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/vhdl.js             0.44 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/haxe.js             0.44 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/jade.js             0.44 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/toml.js             0.44 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/jack.js             0.44 KiB         0.21 KiB
13:53:13   dist/lib/ace/snippets/rdoc.js             0.44 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/red.js              0.44 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/jsx.js              0.44 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/xml.js              0.44 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/sjs.js              0.44 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/ftl.js              0.44 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/dot.js              0.44 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/ini.js              0.44 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/nix.js              0.44 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/mel.js              0.44 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/elm.js              0.44 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/tsx.js              0.44 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/pig.js              0.44 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/asl.js              0.44 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/svg.js              0.44 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/ada.js              0.44 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/ejs.js              0.44 KiB         0.20 KiB
13:53:13   dist/lib/ace/snippets/d.js                0.43 KiB         0.20 KiB
13:53:13   dist/js/chunk-2d0da6a8.f46faf41.js        0.37 KiB         0.29 KiB
13:53:13   dist/js/chunk-2d2253ae.81f78e55.js        0.37 KiB         0.28 KiB
13:53:13   dist/js/fail.9d1b914c.js                  0.36 KiB         0.28 KiB
13:53:13   dist/js/chunk-2d0e95df.acbc094b.js        0.33 KiB         0.26 KiB
13:53:13   dist/lib/ace/ext-error_marker.js          0.33 KiB         0.15 KiB
13:53:13   dist/lib/ace/mode-text.js                 0.32 KiB         0.15 KiB
13:53:13   dist/js/lang-ja-JP.facbd014.js            0.21 KiB         0.21 KiB
13:53:13   dist/js/chunk-2d0dd63f.eaf7c999.js        0.21 KiB         0.18 KiB
13:53:13   dist/js/lang-fr-FR.7bb8cf92.js            0.20 KiB         0.17 KiB
13:53:13   dist/js/lang-en-GB.f0851053.js            0.19 KiB         0.17 KiB
13:53:13   dist/js/chunk-2d20977c.861ea0f1.js        0.17 KiB         0.15 KiB
13:53:13   dist/css/chunk-vendors.6a5a4912.css       745.29 KiB       96.21 KiB
13:53:13   dist/css/app.e64713f9.css                 477.93 KiB       98.08 KiB
13:53:13   dist/css/chunk-7cb23b21.af37fb1c.css      4.75 KiB         2.61 KiB
13:53:13   dist/css/user.c31e0589.css                4.18 KiB         2.46 KiB
13:53:13   dist/css/chunk-6923b696.173cf026.css      3.70 KiB         2.32 KiB
13:53:13   dist/css/chunk-2394da78.0e79d50a.css      2.48 KiB         0.38 KiB
13:53:13   dist/css/chunk-7b215be7.5846bcc1.css      1.78 KiB         0.65 KiB
13:53:13   dist/loading/loading.css                  1.62 KiB         0.54 KiB
13:53:13   dist/css/chunk-1a25084c.7cf4d9c4.css      0.55 KiB         0.29 KiB
13:53:13   dist/css/chunk-ffcd749a.7cf4d9c4.css      0.55 KiB         0.29 KiB
13:53:13   dist/css/chunk-bba7e5be.60ba599d.css      0.48 KiB         0.25 KiB
13:53:13   dist/css/chunk-421e36d3.aced983b.css      0.43 KiB         0.25 KiB
13:53:13   dist/css/chunk-e7434404.45b80a38.css      0.43 KiB         0.25 KiB
13:53:13   dist/css/chunk-0fa7c848.8c8de4cc.css      0.43 KiB         0.25 KiB
13:53:13   dist/css/chunk-f16e9740.c7849a1f.css      0.43 KiB         0.25 KiB
13:53:13   dist/css/chunk-5f261a47.fcc48993.css      0.43 KiB         0.25 KiB
13:53:13   dist/css/chunk-7ea2dba3.f471e889.css      0.43 KiB         0.25 KiB
13:53:13   dist/css/chunk-11ebe073.20670f24.css      0.34 KiB         0.20 KiB
13:53:13   dist/css/chunk-4e4be866.a9dfd643.css      0.32 KiB         0.22 KiB
13:53:13   dist/loading/option2/loading.css          0.30 KiB         0.18 KiB
13:53:13   dist/css/chunk-60867039.1d961b54.css      0.29 KiB         0.21 KiB
13:53:13   dist/css/chunk-213aeccc.99dfac86.css      0.26 KiB         0.20 KiB
13:53:13   dist/css/chunk-34e01bd8.5a4006cf.css      0.23 KiB         0.19 KiB
13:53:13   dist/css/chunk-671f39ca.8642fc4a.css      0.23 KiB         0.19 KiB
13:53:13   dist/css/chunk-7e84b310.aace4747.css      0.20 KiB         0.16 KiB
13:53:13   dist/css/chunk-12effcec.aace4747.css      0.20 KiB         0.16 KiB
13:53:13   dist/css/chunk-f3f382b8.1e7a6bf2.css      0.19 KiB         0.16 KiB
13:53:13   dist/css/chunk-e26d2c80.e52e2e15.css      0.19 KiB         0.15 KiB
13:53:13   dist/css/chunk-cad5ac02.5eedec8d.css      0.18 KiB         0.14 KiB
13:53:13   dist/css/chunk-6d233d0a.44bf8e75.css      0.16 KiB         0.14 KiB
13:53:13   dist/css/chunk-7775a5f9.28de7f11.css      0.16 KiB         0.14 KiB
13:53:13   dist/css/chunk-574e585c.28de7f11.css      0.16 KiB         0.14 KiB
13:53:13   dist/css/chunk-0201a64a.71adeaf3.css      0.11 KiB         0.11 KiB
13:53:13   dist/css/chunk-23759318.71adeaf3.css      0.11 KiB         0.11 KiB
13:53:13   dist/css/chunk-4a1e5c3b.9190ca97.css      0.10 KiB         0.10 KiB
13:53:13   dist/css/chunk-2337c4ca.e315ce2b.css      0.09 KiB         0.11 KiB
13:53:13   dist/css/chunk-14e07ffc.e315ce2b.css      0.09 KiB         0.11 KiB
13:53:13   dist/css/chunk-3942c521.e804fcd7.css      0.08 KiB         0.09 KiB
13:53:13   dist/css/chunk-eb8257d2.159c0bfd.css      0.07 KiB         0.08 KiB
13:53:13   dist/css/chunk-1273618c.be742cbe.css      0.06 KiB         0.07 KiB
13:53:13   dist/css/chunk-5f1dd326.eff5faa3.css      0.05 KiB         0.07 KiB
13:53:13   dist/css/chunk-66809a5c.be5a7a56.css      0.05 KiB         0.07 KiB
13:53:13   dist/css/chunk-319fd138.20827ec7.css      0.05 KiB         0.07 KiB
13:53:13   dist/css/chunk-0b27757e.9d60fcac.css      0.05 KiB         0.07 KiB
13:53:13   dist/css/chunk-30ec3699.a920eb84.css      0.05 KiB         0.07 KiB
13:53:13   dist/css/chunk-158513f2.a07d0447.css      0.05 KiB         0.07 KiB
13:53:13   dist/css/chunk-2fadf79e.da3da5ec.css      0.05 KiB         0.07 KiB
13:53:13   dist/css/chunk-c3b33c8e.094e5867.css      0.05 KiB         0.07 KiB
13:53:13   dist/css/chunk-05bc1686.74338155.css      0.05 KiB         0.07 KiB
13:53:13   dist/css/chunk-7793b159.6295b42e.css      0.05 KiB         0.07 KiB
13:53:13   dist/css/chunk-78733bee.c5b0b76f.css      0.04 KiB         0.06 KiB
13:53:13   dist/css/chunk-41b7bf83.c5b0b76f.css      0.04 KiB         0.06 KiB
13:53:13   dist/css/chunk-682634f1.250c35f1.css      0.04 KiB         0.06 KiB
13:53:13   dist/css/chunk-2a999ff8.250c35f1.css      0.04 KiB         0.06 KiB
13:53:13   dist/css/chunk-fb0f7f1a.6f26ea36.css      0.04 KiB         0.06 KiB
13:53:13 
13:53:13   Images and other types of assets omitted.
13:53:13 
13:53:13  DONE  Build complete. The dist directory is ready to be deployed.
13:53:13  INFO  Check out deployment instructions at https://cli.vuejs.org/guide/deployment.html
13:53:13       
13:53:13 Done in 124.33s.
14:03:30 [Docker] INFO: Step 1/3 : FROM 192.168.238.159:5000/base_frontend:0.0.1
14:03:30 [Docker] INFO: 
14:03:30 
14:03:32 [Docker] INFO:  ---> c283c15686cf
14:03:32 
14:03:32 [Docker] INFO: Step 2/3 : COPY dist/ /var/www
14:03:32 [Docker] INFO: 
14:03:32 
14:07:10 [Docker] INFO:  ---> Using cache
14:07:10 
14:07:10 [Docker] INFO:  ---> ed96447e99da
14:07:10 
14:07:10 [Docker] INFO: Step 3/3 : ENTRYPOINT ["nginx","-g","daemon off;"]
14:07:10 [Docker] INFO: 
14:07:10 
14:07:44 [Docker] INFO:  ---> Running in 4447e3f52087
14:07:44 
14:08:32 [Docker] INFO:  ---> e9ff4657b42a
14:08:32 
14:12:30 [Docker] INFO: Successfully built e9ff4657b42a
14:12:30 
14:12:31 [Docker] INFO: Successfully tagged 192.168.238.159:5000/test-frontend-k8s:21
14:12:31 
14:12:31 [Docker] INFO: Build image id:e9ff4657b42a
14:12:31 [Docker] INFO: Pushing image 192.168.238.159:5000/test-frontend-k8s:21
14:12:39 [Docker] INFO: Done pushing image 192.168.238.159:5000/test-frontend-k8s:21
14:12:39 [Docker] INFO: Removed image 192.168.238.159:5000/test-frontend-k8s:21
14:12:39 Starting Kubernetes deployment
14:12:45 Loading configuration: /var/jenkins_home/workspace/test-frontend-k8s/frontend-opt-k8s.yaml
14:12:46 Applied V1Service: class V1Service {
14:12:46     apiVersion: v1
14:12:46     kind: Service
14:12:46     metadata: class V1ObjectMeta {
14:12:46         annotations: null
14:12:46         clusterName: null
14:12:46         creationTimestamp: 2020-04-02T06:12:55.000Z
14:12:46         deletionGracePeriodSeconds: null
14:12:46         deletionTimestamp: null
14:12:46         finalizers: null
14:12:46         generateName: null
14:12:46         generation: null
14:12:46         initializers: null
14:12:46         labels: {app=test-frontend}
14:12:46         managedFields: null
14:12:46         name: test-frontend
14:12:46         namespace: test-all-service
14:12:46         ownerReferences: null
14:12:46         resourceVersion: 26990888
14:12:46         selfLink: /api/v1/namespaces/test-all-service/services/test-frontend
14:12:46         uid: 41f878bf-c4f1-491b-997b-e52a1038ad5c
14:12:46     }
14:12:46     spec: class V1ServiceSpec {
14:12:46         clusterIP: 10.43.135.170
14:12:46         externalIPs: null
14:12:46         externalName: null
14:12:46         externalTrafficPolicy: Cluster
14:12:46         healthCheckNodePort: null
14:12:46         loadBalancerIP: null
14:12:46         loadBalancerSourceRanges: null
14:12:46         ports: [class V1ServicePort {
14:12:46             name: tcp
14:12:46             nodePort: 31080
14:12:46             port: 80
14:12:46             protocol: TCP
14:12:46             targetPort: 80
14:12:46         }]
14:12:46         publishNotReadyAddresses: null
14:12:46         selector: {app=test-frontend}
14:12:46         sessionAffinity: None
14:12:46         sessionAffinityConfig: null
14:12:46         type: NodePort
14:12:46     }
14:12:46     status: class V1ServiceStatus {
14:12:46         loadBalancer: class V1LoadBalancerStatus {
14:12:46             ingress: null
14:12:46         }
14:12:46     }
14:12:46 }
14:12:47 Applied V1Deployment: class V1Deployment {
14:12:47     apiVersion: apps/v1
14:12:47     kind: Deployment
14:12:47     metadata: class V1ObjectMeta {
14:12:47         annotations: null
14:12:47         clusterName: null
14:12:47         creationTimestamp: 2020-04-26T10:08:53.000Z
14:12:47         deletionGracePeriodSeconds: null
14:12:47         deletionTimestamp: null
14:12:47         finalizers: null
14:12:47         generateName: null
14:12:47         generation: 7
14:12:47         initializers: null
14:12:47         labels: null
14:12:47         managedFields: null
14:12:47         name: test-frontend
14:12:47         namespace: test-all-service
14:12:47         ownerReferences: null
14:12:47         resourceVersion: 26990889
14:12:47         selfLink: /apis/apps/v1/namespaces/test-all-service/deployments/test-frontend
14:12:47         uid: 25aa0a0d-3b88-4a00-ac4f-df731440d21c
14:12:47     }
14:12:47     spec: class V1DeploymentSpec {
14:12:47         minReadySeconds: 10
14:12:47         paused: null
14:12:47         progressDeadlineSeconds: 600
14:12:47         replicas: 1
14:12:47         revisionHistoryLimit: 10
14:12:47         selector: class V1LabelSelector {
14:12:47             matchExpressions: null
14:12:47             matchLabels: {app=test-frontend}
14:12:47         }
14:12:47         strategy: class V1DeploymentStrategy {
14:12:47             rollingUpdate: class V1RollingUpdateDeployment {
14:12:47                 maxSurge: 1
14:12:47                 maxUnavailable: 0
14:12:47             }
14:12:47             type: RollingUpdate
14:12:47         }
14:12:47         template: class V1PodTemplateSpec {
14:12:47             metadata: class V1ObjectMeta {
14:12:47                 annotations: null
14:12:47                 clusterName: null
14:12:47                 creationTimestamp: null
14:12:47                 deletionGracePeriodSeconds: null
14:12:47                 deletionTimestamp: null
14:12:47                 finalizers: null
14:12:47                 generateName: null
14:12:47                 generation: null
14:12:47                 initializers: null
14:12:47                 labels: {app=test-frontend}
14:12:47                 managedFields: null
14:12:47                 name: null
14:12:47                 namespace: null
14:12:47                 ownerReferences: null
14:12:47                 resourceVersion: null
14:12:47                 selfLink: null
14:12:47                 uid: null
14:12:47             }
14:12:47             spec: class V1PodSpec {
14:12:47                 activeDeadlineSeconds: null
14:12:47                 affinity: class V1Affinity {
14:12:47                     nodeAffinity: null
14:12:47                     podAffinity: null
14:12:47                     podAntiAffinity: class V1PodAntiAffinity {
14:12:47                         preferredDuringSchedulingIgnoredDuringExecution: [class V1WeightedPodAffinityTerm {
14:12:47                             podAffinityTerm: class V1PodAffinityTerm {
14:12:47                                 labelSelector: class V1LabelSelector {
14:12:47                                     matchExpressions: [class V1LabelSelectorRequirement {
14:12:47                                         key: app
14:12:47                                         operator: In
14:12:47                                         values: [app-test-frontend]
14:12:47                                     }]
14:12:47                                     matchLabels: null
14:12:47                                 }
14:12:47                                 namespaces: null
14:12:47                                 topologyKey: kubernetes.io/hostname
14:12:47                             }
14:12:47                             weight: 1
14:12:47                         }]
14:12:47                         requiredDuringSchedulingIgnoredDuringExecution: null
14:12:47                     }
14:12:47                 }
14:12:47                 automountServiceAccountToken: null
14:12:47                 containers: [class V1Container {
14:12:47                     args: null
14:12:47                     command: null
14:12:47                     env: [class V1EnvVar {
14:12:47                         name: GATEWAY_HOST
14:12:47                         value: 192.168.238.241:31333
14:12:47                         valueFrom: null
14:12:47                     }]
14:12:47                     envFrom: null
14:12:47                     image: 192.168.238.159:5000/test-frontend-k8s:21
14:12:47                     imagePullPolicy: Always
14:12:47                     lifecycle: null
14:12:47                     livenessProbe: null
14:12:47                     name: test-frontend
14:12:47                     ports: [class V1ContainerPort {
14:12:47                         containerPort: 80
14:12:47                         hostIP: null
14:12:47                         hostPort: null
14:12:47                         name: null
14:12:47                         protocol: TCP
14:12:47                     }]
14:12:47                     readinessProbe: null
14:12:47                     resources: class V1ResourceRequirements {
14:12:47                         limits: {memory=Quantity{number=1610612736, format=DECIMAL_SI}}
14:12:47                         requests: {memory=Quantity{number=1073741824, format=BINARY_SI}}
14:12:47                     }
14:12:47                     securityContext: null
14:12:47                     stdin: null
14:12:47                     stdinOnce: null
14:12:47                     terminationMessagePath: /dev/termination-log
14:12:47                     terminationMessagePolicy: File
14:12:47                     tty: null
14:12:47                     volumeDevices: null
14:12:47                     volumeMounts: null
14:12:47                     workingDir: null
14:12:47                 }]
14:12:47                 dnsConfig: null
14:12:47                 dnsPolicy: ClusterFirst
14:12:47                 enableServiceLinks: null
14:12:47                 hostAliases: null
14:12:47                 hostIPC: null
14:12:47                 hostNetwork: null
14:12:47                 hostPID: null
14:12:47                 hostname: null
14:12:47                 imagePullSecrets: null
14:12:47                 initContainers: null
14:12:47                 nodeName: null
14:12:47                 nodeSelector: null
14:12:47                 preemptionPolicy: null
14:12:47                 priority: null
14:12:47                 priorityClassName: null
14:12:47                 readinessGates: null
14:12:47                 restartPolicy: Always
14:12:47                 runtimeClassName: null
14:12:47                 schedulerName: default-scheduler
14:12:47                 securityContext: class V1PodSecurityContext {
14:12:47                     fsGroup: null
14:12:47                     runAsGroup: null
14:12:47                     runAsNonRoot: null
14:12:47                     runAsUser: null
14:12:47                     seLinuxOptions: null
14:12:47                     supplementalGroups: null
14:12:47                     sysctls: null
14:12:47                     windowsOptions: null
14:12:47                 }
14:12:47                 serviceAccount: null
14:12:47                 serviceAccountName: null
14:12:47                 shareProcessNamespace: null
14:12:47                 subdomain: null
14:12:47                 terminationGracePeriodSeconds: 30
14:12:47                 tolerations: null
14:12:47                 volumes: null
14:12:47             }
14:12:47         }
14:12:47     }
14:12:47     status: class V1DeploymentStatus {
14:12:47         availableReplicas: 1
14:12:47         collisionCount: null
14:12:47         conditions: [class V1DeploymentCondition {
14:12:47             lastTransitionTime: 2020-04-26T10:09:24.000Z
14:12:47             lastUpdateTime: 2020-04-26T10:09:24.000Z
14:12:47             message: Deployment has minimum availability.
14:12:47             reason: MinimumReplicasAvailable
14:12:47             status: True
14:12:47             type: Available
14:12:47         }, class V1DeploymentCondition {
14:12:47             lastTransitionTime: 2020-04-26T10:09:05.000Z
14:12:47             lastUpdateTime: 2020-05-07T01:33:25.000Z
14:12:47             message: ReplicaSet "test-frontend-5c458c5fbf" has successfully progressed.
14:12:47             reason: NewReplicaSetAvailable
14:12:47             status: True
14:12:47             type: Progressing
14:12:47         }]
14:12:47         observedGeneration: 6
14:12:47         readyReplicas: 1
14:12:47         replicas: 1
14:12:47         unavailableReplicas: null
14:12:47         updatedReplicas: 1
14:12:47     }
14:12:47 }
14:12:48 Finished Kubernetes deployment
14:12:49 Finished: SUCCESS

```

分析上述日志，可以得到以下信息：

1. yarn run build这个过程大约在2分钟到3分钟之间。
2. 在完成上述构建后，到进入docker镜像构建的时候，存在一个很长的时间，10分钟左右，该位置时间节点原因未知。
3. 在Dockerfile文件中，执行COPY命令时，发现持续时间很长，在4分钟左右。
4. 完成镜像打包到推送到镜像仓库中，这个位置耗时较长，在4分钟左右。

根据上述的信息，可以预见的是目前构建中存在如下问题：

1. 前端构建时生成文件过多，导致COPY时速度缓慢，这一点需要优化。
2. 需要查看打包后生成的前端镜像是否过大。
3. 完成构建到docker镜像构建过程时，耗时长的问题。

## 构建改造

决定全部走docker镜像构建，剥离当前jenkins内的镜像对于构建的影响，主要是nvm插件的使用。使用两段式构建。基础镜像不做变更，使用nodejs的镜像，构建全部前端代码，然后使用基础镜像构建前端项目。编写Dockerfile，如下：

```Dockerfile

# 第一层面构建，打包前端代码
#### 1. 指定node镜像版本
FROM node:10.12.0 AS builder
# 2. 指定编译的工作空间
WORKDIR /home/node/app
# 3. 安装打包需要的yarn工具
RUN npm --registry https://registry.npm.taobao.org install -g yarn
# 4. 添加package.json
COPY package.json /home/node/app/
# 5. 安装依赖，如果package.json未变更，将会沿用之前的镜像缓存
RUN yarn --registry https://registry.npm.taobao.org install
# 6. 添加剩余代码到工作空间
COPY . /home/node/app
# 7. 编译代码
RUN yarn run build

# 第二层面构建
#### 1.拉取自定义镜像名称
FROM 192.168.238.159:5000/base_frontend:0.0.1

# 2.将打包后的代码复制到运行位置
COPY --from=builder /home/node/app/dist /var/www

# 3.启动nginx
ENTRYPOINT ["nginx","-g","daemon off;"]

```

这样就将上述的外部打包的工作，变成了镜像内工作，同时根据层次划分，将依赖安装和源码构建分开。对一个Docker镜像来说，Dockerfile中的每一个步骤都是一个分层，从第一个步骤开始，只有当其中一个步骤的操作发生了变化，这之后的每一个分层才会重新生成，而之前的分层都会被复用。这样就能利用之前缓存的层级信息，为后续构建加速！

随后，改造整个构建流程，拿掉和nodejs相关的构建步骤，只保留最后docker打包的部分，如下图：

![](新的docker构建配置.png)

最后，进行构建即可。

## 改造后效果

第一次构建非常慢，涉及到拉取各路依赖包信息，长达一个小时到两个小时左右，主要瓶颈点在于依赖的下载安装。下面进行两个测试，第一是通过修改一行代码来测试，第二是通过添加一个新的依赖包测试。

### 1. 修改一行代码的测试

修改一行代码后，其真正触发的是编译前端代码的部分，起始处日志如下：

```
17:29:32 [Docker] INFO: Step 7/10 : RUN yarn run build

```

会重新开始构建前端代码，这样以后到最终构建完成并部署，花费4分钟左右。比起之前的构建提升巨大，至少节省了15分钟以上。将构建流程压缩到了4分钟左右。

### 2. 添加新的依赖包的测试

测试修改package.json中的内容，添加一个dragable控件信息，如下：

```
$ vim package.json
// 添加下面的内容到dependencies节点
.....
"dependencies": {
  "vue-draggable-resizable": "^2.2.0",

  ....
}

```

然后提交，进行构建。但是并没有提升什么构建速度，相反拖累了整个jenkins中对于前端构建的速度，又回到了一个小时的时候。这时候需要进行进一步的优化。

### 3. 其它尝试

#### 3.1 制作私有镜像仓库

制作私有镜像仓库参考自[该链接](https://www.jianshu.com/p/1674a6bc1c12)。使用nexus3作为本地的npm镜像仓库，仓库地址为：http://192.168.238.159:8081/repository/npm-group/ ，用户名为admin，密码为helloworld，账号的邮箱地址为lisongyang@abc.com。

#### 3.2 在本地使用私有镜像仓库

```
// 查看并修改镜像仓库地址
$ npm config get registry
http://registry.npmjs.org

$ npm config set registry http://192.168.238.159:8081/repository/npm-group/

// 登录镜像仓库
// 如果不登录的话，拉取仓库中的依赖信息，会出现401 unauthorized的错误
$ npm login
// 输入上面提到的用户名密码以及邮箱地址
Username: admin
Password:
Email: (this IS public) lisongyang@abc.com
Logged in as leinov on http://192.168.238.159:8081/repository/npm-group/.

```

上述操作完成后，需要删除本地的缓存文件夹node_cache以及项目中的node_modules文件夹，这样做的目的是保证本地拉取远程的依赖包时，从我们自己搭建的nexus3镜像仓库中拉取，也是将配置的外网淘宝镜像的依赖，缓存在我们自己的镜像仓库中。这样后续在拉取依赖时，会从nexus3镜像仓库中拉取，不需要去访问外网了，只有本地没有的依赖信息才会主动去淘宝镜像拉取。

删除完成后，就可以开始构建了，运行npm install，会发现速度慢了很多。当第一次操作完成后，后续操作时会加速构建。

将npm换为yarn工具，操作是类似的。但是yarn工具不能在docker构建时使用，具体参考下面的章节。

#### 3.3 CI/CD过程中使用私有镜像仓库

在nexus3镜像仓库搭建完成后，重新编写前端的dockerfile，如下：

```Dockerfile
# 第一层面构建，打包前端代码
#### 1. 指定node镜像版本
FROM node:10.16.0 AS builder

# 添加日期信息，如果需要更新缓存层，更新该处日期信息即可
ENV REFRESH_DATE 2020-05-18_11:11:11

# 2. 指定编译的工作空间
WORKDIR /home/node/app

# 安装打包需要的yarn工具
# RUN npm --registry https://registry.npm.taobao.org install -g yarn
# 3. 对yarn设置淘宝镜像
RUN npm config set registry http://192.168.238.159:8081/repository/npm-group/ 

# 4. 添加登录nexus3镜像仓库的插件
RUN npm install -g npm-cli-adduser --registry https://registry.npm.taobao.org

# 5. 解决node-sass安装失败的问题
RUN npm config set sass_binary_site https://npm.taobao.org/mirrors/node-sass/

# 6. 添加package.json
COPY package.json /home/node/app/

# 7. 登录并安装依赖，如果package.json未变更，将会沿用之前的镜像缓存
RUN NPM_USER=admin NPM_PASS=helloworld NPM_EMAIL=lisongyang@abc.com NPM_REGISTRY=http://192.168.238.159:8081/repository/npm-group/ npm-cli-adduser 

# 8. 安装依赖信息
RUN npm --loglevel info install 

# 9. 添加剩余代码到工作空间
COPY . /home/node/app

# 10. 编译代码
RUN npm run build

# 第二层面构建
#### 1.拉取自定义镜像名称
FROM 192.168.238.159:5000/base_frontend:0.0.1

# 2.将打包后的代码复制到运行位置
COPY --from=builder /home/node/app/dist /var/www

# 3.启动nginx
ENTRYPOINT ["nginx","-g","daemon off;"]

```

对比之前的dockerfile，这里主要是将yarn用npm进行了代替，根据构建中出现的node-sass依赖错误的问题进行了优化。这样编写之后，整个前端从打包到构建镜像全部在docker中进行了，相应的前端构建项目也是要变更。创建新的分支，为change_ci_cd，然后进行构建测试，构建流程和之前的一致。

这样完成之后，就可以愉快的开启打包流程了。但是这时候会报错，如下：

```
TypeError: Class extends value undefined is not a constructor or null
    at Object.<anonymous> (/Users/glenn/github/minimalistic-devserver/node_modules/mini-css-extract-plugin/dist/index.js:30:47)
    at Module._compile (/Users/glenn/github/minimalistic-devserver/node_modules/v8-compile-cache/v8-compile-cache.js:178:30)
    at Object.Module._extensions..js (module.js:673:10)
    at Module.load (module.js:575:32)
    at tryModuleLoad (module.js:515:12)
    at Function.Module._load (module.js:507:3)
    at Module.require (module.js:606:17)
    at require (/Users/glenn/github/minimalistic-devserver/node_modules/v8-compile-cache/v8-compile-cache.js:159:20)
    at Object.<anonymous> (/Users/glenn/github/minimalistic-devserver/node_modules/mini-css-extract-plugin/dist/cjs.js:3:18)
    at Module._compile (/Users/glenn/github/minimalistic-devserver/node_modules/v8-compile-cache/v8-compile-cache.js:178:30)
    at Object.Module._extensions..js (module.js:673:10)
    at Module.load (module.js:575:32)
    at tryModuleLoad (module.js:515:12)
    at Function.Module._load (module.js:507:3)
    at Module.require (module.js:606:17)
    at require (/Users/glenn/github/minimalistic-devserver/node_modules/v8-compile-cache/v8-compile-cache.js:159:20)
    at Object.<anonymous> (/Users/glenn/github/minimalistic-devserver/webpack.config.js:8:30)
    at Module._compile (/Users/glenn/github/minimalistic-devserver/node_modules/v8-compile-cache/v8-compile-cache.js:178:30)
    at Object.Module._extensions..js (module.js:673:10)
    at Module.load (module.js:575:32)
    at tryModuleLoad (module.js:515:12)
    at Function.Module._load (module.js:507:3)
    at Module.require (module.js:606:17)
    at require (/Users/glenn/github/minimalistic-devserver/node_modules/v8-compile-cache/v8-compile-cache.js:159:20)
    at WEBPACK_OPTIONS (/Users/glenn/github/minimalistic-devserver/node_modules/webpack-cli/bin/convert-argv.js:133:13)
    at requireConfig (/Users/glenn/github/minimalistic-devserver/node_modules/webpack-cli/bin/convert-argv.js:135:6)
    at /Users/glenn/github/minimalistic-devserver/node_modules/webpack-cli/bin/convert-argv.js:142:17
    at Array.forEach (<anonymous>)
    at module.exports (/Users/glenn/github/minimalistic-devserver/node_modules/webpack-cli/bin/convert-argv.js:140:15)
    at yargs.parse (/Users/glenn/github/minimalistic-devserver/node_modules/webpack-cli/bin/webpack.js:234:39)

```

报错的原因是，我们自身的webpack配置，引发了mini-css-extract-plugin插件的错误。我们在package.json中限定webpack高于3.6.0版本，在jenkins中拉取时，拉去了webpack4以上的版本，这是令人奇怪的地方。看package.json中的匹配规则:

```
...
    "webpack": "^3.6.0",
...

```

这里引述两个符号解释：

```
'~'（波浪符号）:他会更新到当前minor version（也就是中间的那位数字）中最新的版本。放到我们的例子中就是："exif-js": "~2.3.0"，这个库会去匹配更新到2.3.x的最新版本，如果出了一个新的版本为2.4.0，则不会自动升级。波浪符号是曾经npm安装时候的默认符号，现在已经变为了插入符号。

'^'（插入符号）: 这个符号就显得非常的灵活了，他将会把当前库的版本更新到当前major version（也就是第一位数字）中最新的版本。放到我们的例子中就是："vue": "^2.2.2", 这个库会去匹配2.x.x中最新的版本，但是他不会自动更新到3.0.0。

作者：太阳的小号
链接：https://www.jianshu.com/p/612eefd8ef40
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

```

正常来说，webpack应该去匹配3.x相关的版本，而不是匹配4.x相关的版本。当webpack版本匹配到4.1.x版本时，导致了mini-css-extract-plugin插件报错了，这时候参考相关链接中“webpack版本升级的问题”，需要强制把webpack版本升到4.1.0才能构建通过。

**注意：**该问题目前无解，只能临时进行尝试替换并在日常开发中测试，目前尚无明确的不良反应。

当webpack版本升级后，除第一次构建外，构建时间会大幅缩减，即使是修改package.json中的文件，也不会出现耗时过长的情况。更广泛的作用是，它加速了本地下载依赖的速度，优化了打包性能。

重新尝试打包的工作，分别使用npm和yarn进行打包。测试后发现yarn工具不会报错，而且能很好的执行构建工作，而npm会出现依赖报错的问题。但是webpack的版本还是会提升到4.1.0以上，不影响其它组件的使用。

在经过最终讨论和尝试后，我们确定统一使用yarn来构建我们的前端项目。然后尽量采用淘宝镜像来构建。yarn比npm在版本管理上更加科学，也更有效，能避免上述提到的依赖版本问题。

## 总结

## 相关链接

* webpack版本升级的问题：https://github.com/webpack-contrib/mini-css-extract-plugin/issues/3
