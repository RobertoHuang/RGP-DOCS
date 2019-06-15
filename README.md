# 文档服务

> 使用`Docker`+`Gitlab`+`Docsify`搭建文档服务

## Docker环境配置

- 安装`Docker`，Link:[Centos7上安装配置Docker](https://github.com/RobertoHuang/RGP-LEARNING/blob/master/Docker/01.Centos7%E4%B8%8A%E5%AE%89%E8%A3%85%E9%85%8D%E7%BD%AEDocker.md)

- 安装`Docker Compose`，Link:[Docker Compose安装与简单使用](https://github.com/RobertoHuang/RGP-LEARNING/blob/master/Docker/Docker%20Compose%E5%AE%89%E8%A3%85%E4%B8%8E%E7%AE%80%E5%8D%95%E4%BD%BF%E7%94%A8.md)

- 编写`DockerFile.onbuild`文件，并使用该文件构建出基础镜像供后续使用

  ```shell
  FROM node:10-alpine
  
  RUN npm i docsify-cli -g --registry=https://registry.npm.taobao.org
  
  ONBUILD COPY src /srv/docsify/docs
  ONBUILD WORKDIR /srv/docsify
  
  CMD ["/usr/local/bin/docsify", "serve", "docs"]
  ```

  ```shell
  $ docker build -t docsify:onbuild -f Dockerfile.onbuild .
  ```

## 安装Runner

- 添加`Gitlab`官方源

  ```shell
  $ curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.rpm.sh | sudo bash
  ```

- 安装`Runner`

  ```shell
  $ sudo yum install -y gitlab-runner
  ```

  以上安装步骤参考:[https://docs.gitlab.com/runner/install/linux-repository.html](https://docs.gitlab.com/runner/install/linux-repository.html)

- 为`Runner`添加操作`Docker`权限

  ```shell
  $ sudo groupadd docker
  $ sudo gpasswd -a gitlab-runner docker
  $ sudo systemctl restart docker
  $ sudo newgrp - docker
  ```

## Gitlab项目准备

- 安装`Gitlab`环境(略)

- 初始化项目

  - 新建一个`Gitlab`项目

  - 新建完成后进入项目，在左侧菜单中导航进入`Settings > CI / CD`菜单，在右侧面板中点击 `Runner` 选项卡的 `Expand` 展开Runner 详情，如下图

    ![runner-setting](<https://raw.githubusercontent.com/RobertoHuang/RGP-LEARNING/master/Others/images/runner-setting.png>)

    **注意:** 上图红框中的内容`TOKEN`在下一步注册`Runner`会用到

## 将Runner注册到项目中

> 其中用中括号 `[...]` 包括的内容表示需要手动修改的

```shell
$ sudo gitlab-runner register \
  --non-interactive \
  --executor "shell" \
  --url "[Gitlab服务地址 对应上图红框1中内容]" \
  --registration-token "[项目对应的TOKEN, 对应上图红框2中内容]" \
  --description "[描述, 例如: my description]" \
  --tag-list "[标签列表, 多个逗号隔开, 例如: commdoc-runner-tag-1]"
```

`Runner`注册完成之后，刷新 `Settings > CI / CD > Runner` 页面，如下图

![runner-setting-runner.png](<https://raw.githubusercontent.com/RobertoHuang/RGP-LEARNING/master/Others/images/runner-setting-runner.png>)

可以点击上图中的红框所示编辑按钮，对`Runner`进行编辑(修改`description，tags`等..)

**注意:** 注册`Runner`时，`tag`很重要，后续的步骤会用到

## Docsify项目准备

从[https://github.com/RobertoHuang/RGP-DOCS](https://github.com/RobertoHuang/RGP-DOCS)下载基础项目模板

项目结构说明:

```reStructuredText
项目跟路径
 ┣ src
 ┃ ┣ 文档文件
 ┃ ┃  ┣_sidebar.md  # 文档左侧导航
 ┃ ┣ _coverpage.md  # 文档首页导航
 ┃ ┗ index.html     # 文档右上角导航
 ┣ .gitlab-ci.yml
 ┣ docker-compose.yml
 ┣ Dockerfile
 ┗ Dockerfile.onbuild
```

- `.gitlab-ci.yml`

  > `Gitlab`持续集成脚本(提交代码时触发)内容如下，根据实际情况调整
  >
  > 完整语法规范可以参考:[https://docs.gitlab.com/ee/ci/yaml/](https://docs.gitlab.com/ee/ci/yaml/)

  ```yaml
  variables:
  
  stages:
    - deploy
  
  .deploy_template: &deploy_template
    stage: deploy
    script:
      - docker-compose build
      - docker-compose up -d
    after_script:
      - docker images | grep none | awk '{print $3}' | xargs docker rmi -f
    only:
      - master
  
  docs-deploy-1:
    <<: *deploy_template
    tags:
      - tag1
  ```

  **注意:** 需要特别注意修改`tags`对应的内容，如下所示

  ```yaml
  ...
    tags:
      - tag1 # Gitlab-Runner注册到该项目的Tag，根据实际情况调整
  ...
  ```

  其中`tag`的值即为上面步骤【将`Runner`注册到项目中】注册时为`Runner`添加的`tag`

  是指`gitbook-deploy-1`这个任务由拥有`tag1`标签的`Runner`来执行，实际上执行过程就是在`Runner`对应的机器上启动一个`Docker`容器，这个`Docker`容器正是上面`docker-compose.yml`和`Dockerfile`的产物，也即是文档的服务，事实上它是一个`node`服务

- `docker-compose.yml`

  > 用于在`Gitlab-Runner`上启动文档容器的脚本，内容如下

  ```yml
  version: '3'
  
  services:
    common-docsify:                    # docker-compose服务名称，可根据实际情况进行调整
      image: docsify                   # 镜像名称，可根据实际情况进行调整
      build:
        context: .
      container_name: common-docsify   # 容器名称，可根据实际情况进行调整
      restart: always
      ports:
        - 8080:3000
  ```

- `DockerFile`

  > 用于构建进行的脚本，内容固定如下

  ```shell
  FROM docsify:onbuild
  ```

  其中`docsify:onbuild`是在第一步`Docker`环境准备时打的基础镜像，参考`Dockerfile.onbuild`

将从`GitHub`上下载的模板项目导入之前在`Gitlab`中初始化好的项目，将本地文件`Commit & Push`，在`Gitlab UI`左侧菜单点击`CI / CD > Pipelines`查看持续集成的进度及状态，如下

![pipelines.png](<https://raw.githubusercontent.com/RobertoHuang/RGP-LEARNING/master/Others/images/pipelines.png>)

`Pipelines`完成后，访问虚拟机`IP`(8080端口)，查看文档是否生成。**注意:** 首次集成如果失败，直接**Retry** 

## Docsify文档说明

> 项目文档采用Markdown编写，由[Docsify](https://docsify.js.org/#/zh-cn/)实时渲染

- 文档主体

  文档主体在`src`目录下，每个项目单独一个目录。例如:

  消息队列的项目文档放在`src/RGP-MESSAGE`下，图片文件放在`src/RGP-MESSAGE/images`目录下

- 文档目录(侧边栏)

  文档目录在`src/项目目录/_sidebar.md`文件中，每个项目有单独的文档目录

  例如:消息队列的文档目录在`src/RGP-MESSAGE/_sidebar.md`中，内容如下

  ```reStructuredText
  - 产品介绍
    - [背景](RGP-MESSAGE/01.introduction/background.md)
  
  - 产品设计文档
    - [应用信息设计](RGP-MESSAGE/02.design-document/application_information_design.md)
  
  - 用户使用手册
    - [用户指南](RGP-MESSAGE/03.user-guide/user-guide.md)
  
  - 运维手册
    - [监控报警事项](RGP-MESSAGE/04.devops-guide/monitor_alarm.md)
  
  - OKR
    - [OKR 目标](RGP-MESSAGE/05.okr-summary/okr.md)
  ```

## 开发方式

> 改动提交到 `master` 分支后，会触发`CI`，稍等片刻再访问文档地址就可以看到相应的变化

- `Docsify`对`Markdown`语法进行了扩展，如果你想让文档的表现力更好，可以参考 [Docsify Markdown扩展](https://docsify.js.org/#/zh-cn/helpers)

- 标题锚点`ID`的生成规则:例如# ## ### 等标题会生成标题锚点用于跳转，`ID`生成规规则为:

  - 如果标题由数字开头生成的ID前面会添加 `_`下划线
  - 标题中包含空格的会被转成`-`中横杠【例如 `# 1. 背景`生成的`ID`为 `_1.-背景`】
  - 如果不想使用默认的ID生成规则，可以[为标题添加id属性](https://docsify.js.org/#/zh-cn/helpers?id=设置标题的-id-属性)【`### 你好，世界 :id=hello-world`】

- 页面间跳转方式:以上面文档目录为例，在本页面想要跳转到背景的二级标题 `概述` 那么应该写成

  ```text
  [跳转概述](#概述)
  ```

  不同文件间的跳转必须加上完整的项目路径，跳转到具体的标题锚点应该使用渲染后的锚点`ID`，如

  ```reStructuredText
  [跳转应用信息设计](RGP-MESSAGE/02.design-document/application_information_design.md)
  ```

## 文档服务效果图

![文档服务效果图1](https://raw.githubusercontent.com/RobertoHuang/RGP-LEARNING/master/Others/images/%E6%96%87%E6%A1%A3%E6%9C%8D%E5%8A%A1%E6%95%88%E6%9E%9C%E5%9B%BE1.png)

![文档服务效果图2](https://raw.githubusercontent.com/RobertoHuang/RGP-LEARNING/master/Others/images/%E6%96%87%E6%A1%A3%E6%9C%8D%E5%8A%A1%E6%95%88%E6%9E%9C%E5%9B%BE2.png)

## 附录

- 参考文档:
  - [.gitlab-ci.yml语法详解 https://docs.gitlab.com/ee/ci/yaml/](https://docs.gitlab.com/ee/ci/yaml/)
  - [docsify文档生成器 https://docsify.js.org/#/?id=docsify-494](https://docsify.js.org/#/?id=docsify-494)
  - [VuePress静态网站生成器 https://vuepress.vuejs.org/zh/guide/](https://vuepress.vuejs.org/zh/guide/)

- `UML`支持

  `Markdown`扩展支持[PlantUML](http://plantuml.com/zh/index)

  例如代码块:

  ```reStructuredText
  ​```plantuml
  Alice -> Bob: 你好！！
  ​```
  ```

  将会生成时序图:

  ![markdown-plantuml01.png](https://raw.githubusercontent.com/RobertoHuang/RGP-LEARNING/master/Others/images/markdown-plantuml01.png)

  而如下代码块

  ```text
  ​```plantuml
  class Car
    
  Driver - Car : drives >
  Car *- Wheel : have 4 >
  Car -- Person : < owns
  ​```
  ```

  将会生成类图:

  ![markdown-plantuml02.png](https://raw.githubusercontent.com/RobertoHuang/RGP-LEARNING/master/Others/images/markdown-plantuml02.png)