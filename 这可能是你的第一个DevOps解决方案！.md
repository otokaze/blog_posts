## 前言
2020年了，不知道还有哪些程序员还是每次发版上线都还人肉的做着打包、测试、上传、部署等一系列机械性的事情。为了打通这些场景使其能够自动化，避免重复性的劳动，DevOps的概念也随之孕育而生。

至于要怎么更好的实现DevOps的概念，按照公司的规模和项目的大小在市场上都已经有很成熟的解决方案了。不过本着生命不息，折腾不止的原则，也作为研究爱好为目的，这里分享一下我是如何不依赖任何第三方平台，通过最低的成本全自建的构建属于自己的DevOps系统的！

我把本次文章所需要用到的资源文件以及一键部署脚本都在Github上进行了开源，fork一下这些都是你的~

github：https://github.com/otokaze/DevOps

## 快速开始
先让我简要说明一下 Devops 系统本身所应该具备的基本行为：

*Coding >> Commit >> Hooks >> Building >> Unit Testing >> Image Pushing >> Deploying*

显然，Devops是多个系统的组成。各个组件进行了分工与合作，流水线式的作业，最终取得了我们想要的结果。因此，我在开源界挑选了几个极低资源占用简单轻便的开源项目来构成我们的系统组成部分。

### Gitea
作为一个私有GIT仓库，Gitea 足够的轻便，它只有一个二进制文件，60MB不到的大小承载了所有的功能。同样也归功于GO的高性能编译器让它在运行时与生俱来所具备超低内存和CPU等资源占用的能力，这也成为我选择它的最佳理由。

##### 部署
```bash
docker-compose -f ./Gitea/docker-compose.yaml up -d
```

### Registry
Registry 作为私有Docker注册中心是目前唯一的选择，这也是Docker的官方开源的私有注册中心解决方案。此外，您也别无选择。 :(

##### 部署
```bash
docker-compose -f ./Registry/docker-compose.yaml up -d
```

### Drone
现在,我们还需要一个 CI/CD 平台能把任务调动起来!

同样也归功于Go语言的实现，Drone不仅非常易于部署，而且更重要的是：它的稳定运行仅占用服务器上不到20MB的内存。爱了爱了~

##### 部署
```bash
docker-compose -f ./Drone/docker-compose.yaml up -d
```

##### Example
此外，为了能让Drone能跟和我们本身的Gitea协调工作，告诉它代码完成提交后需要做什么，所以我们还需要一个YAML描述它的“工作事项”。

这里以我另一个开源的测试查看Docker容器内负载均衡以及ip&hostname信息的小工具来作为Demo。

github：https://github.com/otokaze/yourip
```yaml
---
kind: pipeline
type: docker
name: default

platform:
  os: linux
  arch: amd64

steps:
- name: linter
  image: alpine/git
  commands:
  - change=$(git diff origin/master $DRONE_COMMIT ./CHANGELOG.md 2> /dev/null) && code=0 || code=$?
  - if [ ! $code -eq 0 ]; then echo 'CHANGELOG.md not fount.'; exit $code; fi
  - if [ -z "$change" ]; then echo 'CHANGELOG.md no change.'; exit 1; fi

- name: builder
  image: golang:1.13.8
  environment:
    GOOS: linux
    GOARCH: amd64
    CGO_ENABLED: 0
  commands:
  - go build -o yourip
  - go test

- name: tagger
  image: alpine
  commands:
  - tags=$(grep -E -o  v[0-9]+\.[0-9]+\.[0-9]+ CHANGELOG.md | head -1 | sed s/v/latest,/g)
  - if [ -z $tags ]; then echo 'No version found in CHANGELOG.md'; exit 1; else echo $tags > .tags; fi
  when:
    event:
    - push

- name: pushing
  image: plugins/docker
  settings:
    username:
      from_secret: DOCKER_REGISTRY_USERNAME
    password:
      from_secret: DOCKER_REGISTRY_PASSWORD
    repo: registry.otokaze.cn/yourip
    registry: registry.otokaze.cn
  when:
    event:
    - push

- name: deploying
  image: curlimages/curl
  environment:
    DEPLOY_API: https://swarm.otokaze.cn/api/services/yourip/redeploy
    TOKEN:
      from_secret: DOCKER_SWARMPIT_TOKEN
  commands:
  - code=$(curl -XPOST -s -w %{http_code} "$DEPLOY_API?tag=latest" -H 'Content-Type:application/json' -H "authorization:$TOKEN")
  - if [[ $code == "" || $code -lt 200 ]]; then echo "redeploy failed. HTTP_CODE=${code}"; exit 1; fi
  when:
    event:
    - push

trigger:
  branch:
  - master
  event:
  - pull_request
  - push

...
```
### Swarmpit
最后的最后，我还需要一个容器编排引擎来管理我的服务，实现服务的平滑升级，横向扩容，回滚发布等功能。

关于容器编排引擎，swarm其实并不是一个主流的选择，但只作为我一个小范围内使用的人来说，我不需要k8s那么多复杂的功能和特性，所以 Docker自家的swarm成了我的最佳选择，因为它足够简单，还非常好用。

##### 部署
```bash
docker stack deploy -c ./Swarmpit/docker-compose.yml swarmpit
```

## 不来一发吗？
- Gitea: https://git.otokaze.cn
- Drone: https://drone.otokaze.cn
- Registry: https://registry.otokaze.cn
- Swarmpit: https://swarm.otokaze.cn
