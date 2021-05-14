---
title: drone的pipeline原理与代码分析
tags: ["drone","pipeline","jenkins","jqpeng"]
categories: ["博客","jqpeng"]
date: 2019-01-22 17:11
---
文章作者:jqpeng
原文链接: [drone的pipeline原理与代码分析](https://www.cnblogs.com/xiaoqi/p/drone-pipeline.html)

最近的一个项目，需要实现一个工作任务流（task pipeline），基于之前CICD的经验，jenkins pipeline和drone的pipeline进入候选。

drone是基于go的cicd解决方案，github上有1.6万+star，本文简单对比了其和jenkins的区别，重点介绍了drone的pipeline原理，并简单分析了代码。

## jenkins 与 drone


| 对比项 | jenkins | drone |
| --- | --- | --- |
| pipeline定义 | 编写jenkinsfile | 编写流程yml |
| 运行方式 | 在一个pod里运行 | 每一步骤起对应的container，通过挂载volume实现数据共享 |
| 运行环境 | 物理机或者容器环境，包括K8S | docker容器环境 |
| 开发语言 | java | golang |


drone pipeline好处是相对更轻量级，yml定义也相对简洁清晰，按照功能来划分容器，可以方便的实现task的复用，而jenkins则是完全打包到一个镜像，会造成单个镜像体积过大，比如jenkins的单个镜像超过2G。

drone的pipeline，是基于https://github.com/cncd/pipeline 实现的，这里简单分析下其原理。

## 编译和执行 drone pipeline

要了解一个程序的原理，先从输入输出讲起。

先安装：


    go get -u github.com/cncd/pipeline
    go install github.com/cncd/pipeline/pipec


然后测试


    cd $GOPATH/github.com/cncd/pipeline/samples/sample_1
    # ll
    total 28
    drwxr-xr-x  2 root root 4096 Jan 22 11:44 ./
    drwxr-xr-x 13 root root 4096 Jan 22 11:02 ../
    -rw-r--r--  1 root root  549 Jan 22 11:02 .env
    -rw-r--r--  1 root root 6804 Jan 22 16:30 pipeline.json
    -rw-r--r--  1 root root  229 Jan 22 11:02 pipeline.yml
    -rw-r--r--  1 root root  138 Jan 22 11:02 README.md
    
    


- pipeline.yml 定义文件
- pipeline.json 编译后的配置文件
- .env 环境变量


先来查看`pipeline.yml` 定义


    workspace:
      base: /go
      path: src/github.com/drone/envsubst
    
    clone:
      git:
        image: plugins/git
        depth: 50
    
    pipeline:
      build:
        image: golang:1.7
        commands:
          - go get -t ./...
          - go build
          - go test -v


上面的yml定义了：

- 工作目录workspace
- 初始化工作，git clone仓库,仓库地址在.env里定义
- 然后是定义pipeline，
    - pipeline下面是step数组，这里只有一个build
    - 使用golang:1.7镜像
    - 构建命令在commands数组里定义


通过`pipec compile`compile配置文件：


    # pipec compile
    Successfully compiled pipeline.yml to pipeline.json


查看编译后的`pipeline.json`


    {
      "pipeline": [
        {
          "name": "pipeline_clone_0",
          "alias": "git",
          "steps": [
            {
              "name": "pipeline_clone_0",
              "alias": "git",
              "image": "plugins/git:latest",
              "working_dir": "/go/src/github.com/drone/envsubst",
              "environment": {
                "CI": "drone",
                "CI_BUILD_CREATED": "1486119586",
                "CI_BUILD_EVENT": "push",
                "CI_BUILD_NUMBER": "6",
                "CI_BUILD_STARTED": "1486119585",
                "CI_COMMIT_AUTHOR": "bradrydzewski",
                "CI_COMMIT_AUTHOR_NAME": "bradrydzewski",
                "CI_COMMIT_BRANCH": "master",
                "CI_COMMIT_MESSAGE": "added a few more test cases for escaping behavior",
                "CI_COMMIT_REF": "refs/heads/master",
                "CI_COMMIT_SHA": "d0876d3176965f9552a611cbd56e24a9264355e6",
                "CI_REMOTE_URL": "https://github.com/drone/envsubst.git",
                "CI_REPO": "drone/envsubst",
                "CI_REPO_LINK": "https://github.com/drone/envsubst",
                "CI_REPO_NAME": "drone/envsubst",
                "CI_REPO_REMOTE": "https://github.com/drone/envsubst.git",
                "CI_SYSTEM": "pipec",
                "CI_SYSTEM_ARCH": "linux/amd64",
                "CI_SYSTEM_LINK": "https://github.com/cncd/pipec",
                "CI_SYSTEM_NAME": "pipec",
                "CI_WORKSPACE": "/go/src/github.com/drone/envsubst",
                "DRONE": "true",
                "DRONE_ARCH": "linux/amd64",
                "DRONE_BRANCH": "master",
                "DRONE_BUILD_CREATED": "1486119586",
                "DRONE_BUILD_EVENT": "push",
                "DRONE_BUILD_LINK": "https://github.com/cncd/pipec/drone/envsubst/6",
                "DRONE_BUILD_NUMBER": "6",
                "DRONE_BUILD_STARTED": "1486119585",
                "DRONE_COMMIT": "d0876d3176965f9552a611cbd56e24a9264355e6",
                "DRONE_COMMIT_AUTHOR": "bradrydzewski",
                "DRONE_COMMIT_BRANCH": "master",
                "DRONE_COMMIT_MESSAGE": "added a few more test cases for escaping behavior",
                "DRONE_COMMIT_REF": "refs/heads/master",
                "DRONE_COMMIT_SHA": "d0876d3176965f9552a611cbd56e24a9264355e6",
                "DRONE_JOB_STARTED": "1486119585",
                "DRONE_REMOTE_URL": "https://github.com/drone/envsubst.git",
                "DRONE_REPO": "drone/envsubst",
                "DRONE_REPO_LINK": "https://github.com/drone/envsubst",
                "DRONE_REPO_NAME": "envsubst",
                "DRONE_REPO_OWNER": "drone",
                "DRONE_REPO_SCM": "git",
                "DRONE_WORKSPACE": "/go/src/github.com/drone/envsubst",
                "PLUGIN_DEPTH": "50"
              },
              "volumes": [
                "pipeline_default:/go"
              ],
              "networks": [
                {
                  "name": "pipeline_default",
                  "aliases": [
                    "git"
                  ]
                }
              ],
              "on_success": true,
              "auth_config": {}
            }
          ]
        },
        {
          "name": "pipeline_stage_0",
          "alias": "build",
          "steps": [
            {
              "name": "pipeline_step_0",
              "alias": "build",
              "image": "golang:1.7",
              "working_dir": "/go/src/github.com/drone/envsubst",
              "environment": {
                "CI": "drone",
                "CI_BUILD_CREATED": "1486119586",
                "CI_BUILD_EVENT": "push",
                "CI_BUILD_NUMBER": "6",
                "CI_BUILD_STARTED": "1486119585",
                "CI_COMMIT_AUTHOR": "bradrydzewski",
                "CI_COMMIT_AUTHOR_NAME": "bradrydzewski",
                "CI_COMMIT_BRANCH": "master",
                "CI_COMMIT_MESSAGE": "added a few more test cases for escaping behavior",
                "CI_COMMIT_REF": "refs/heads/master",
                "CI_COMMIT_SHA": "d0876d3176965f9552a611cbd56e24a9264355e6",
                "CI_REMOTE_URL": "https://github.com/drone/envsubst.git",
                "CI_REPO": "drone/envsubst",
                "CI_REPO_LINK": "https://github.com/drone/envsubst",
                "CI_REPO_NAME": "drone/envsubst",
                "CI_REPO_REMOTE": "https://github.com/drone/envsubst.git",
                "CI_SCRIPT": "CmlmIFsgLW4gIiRDSV9ORVRSQ19NQUNISU5FIiBdOyB0aGVuCmNhdCA8PEVPRiA+ICRIT01FLy5uZXRyYwptYWNoaW5lICRDSV9ORVRSQ19NQUNISU5FCmxvZ2luICRDSV9ORVRSQ19VU0VSTkFNRQpwYXNzd29yZCAkQ0lfTkVUUkNfUEFTU1dPUkQKRU9GCmNobW9kIDA2MDAgJEhPTUUvLm5ldHJjCmZpCnVuc2V0IENJX05FVFJDX1VTRVJOQU1FCnVuc2V0IENJX05FVFJDX1BBU1NXT1JECnVuc2V0IENJX1NDUklQVAp1bnNldCBEUk9ORV9ORVRSQ19VU0VSTkFNRQp1bnNldCBEUk9ORV9ORVRSQ19QQVNTV09SRAoKZWNobyArICJnbyBnZXQgLXQgLi8uLi4iCmdvIGdldCAtdCAuLy4uLgoKZWNobyArICJnbyBidWlsZCIKZ28gYnVpbGQKCmVjaG8gKyAiZ28gdGVzdCAtdiIKZ28gdGVzdCAtdgoK",
                "CI_SYSTEM": "pipec",
                "CI_SYSTEM_ARCH": "linux/amd64",
                "CI_SYSTEM_LINK": "https://github.com/cncd/pipec",
                "CI_SYSTEM_NAME": "pipec",
                "CI_WORKSPACE": "/go/src/github.com/drone/envsubst",
                "DRONE": "true",
                "DRONE_ARCH": "linux/amd64",
                "DRONE_BRANCH": "master",
                "DRONE_BUILD_CREATED": "1486119586",
                "DRONE_BUILD_EVENT": "push",
                "DRONE_BUILD_LINK": "https://github.com/cncd/pipec/drone/envsubst/6",
                "DRONE_BUILD_NUMBER": "6",
                "DRONE_BUILD_STARTED": "1486119585",
                "DRONE_COMMIT": "d0876d3176965f9552a611cbd56e24a9264355e6",
                "DRONE_COMMIT_AUTHOR": "bradrydzewski",
                "DRONE_COMMIT_BRANCH": "master",
                "DRONE_COMMIT_MESSAGE": "added a few more test cases for escaping behavior",
                "DRONE_COMMIT_REF": "refs/heads/master",
                "DRONE_COMMIT_SHA": "d0876d3176965f9552a611cbd56e24a9264355e6",
                "DRONE_JOB_STARTED": "1486119585",
                "DRONE_REMOTE_URL": "https://github.com/drone/envsubst.git",
                "DRONE_REPO": "drone/envsubst",
                "DRONE_REPO_LINK": "https://github.com/drone/envsubst",
                "DRONE_REPO_NAME": "envsubst",
                "DRONE_REPO_OWNER": "drone",
                "DRONE_REPO_SCM": "git",
                "DRONE_WORKSPACE": "/go/src/github.com/drone/envsubst",
                "HOME": "/root",
                "SHELL": "/bin/sh"
              },
              "entrypoint": [
                "/bin/sh",
                "-c"
              ],
              "command": [
                "echo $CI_SCRIPT | base64 -d | /bin/sh -e"
              ],
              "volumes": [
                "pipeline_default:/go"
              ],
              "networks": [
                {
                  "name": "pipeline_default",
                  "aliases": [
                    "build"
                  ]
                }
              ],
              "on_success": true,
              "auth_config": {}
            }
          ]
        }
      ],
      "networks": [
        {
          "name": "pipeline_default",
          "driver": "bridge"
        }
      ],
      "volumes": [
        {
          "name": "pipeline_default",
          "driver": "local"
        }
      ],
      "secrets": null
    }


简单分析结构：

- pipeline 定义了执行的stage，每个stage有一个或者多个step
- networks、volumes、secrets 分别定义网络、存储和secrets
    - 通过network，实现container互通
    - 通过volumes实现数据共享


最后执行，通过`pipec exec`：


    # pipec exec
    proc "pipeline_clone_0" started
    + git init
    Initialized empty Git repository in /go/src/github.com/drone/envsubst/.git/
    + git remote add origin https://github.com/drone/envsubst.git
    + git fetch --no-tags --depth=50 origin +refs/heads/master:
    From https://github.com/drone/envsubst
     * branch            master     -> FETCH_HEAD
     * [new branch]      master     -> origin/master
    + git reset --hard -q d0876d3176965f9552a611cbd56e24a9264355e6
    + git submodule update --init --recursive
    proc "pipeline_clone_0" exited with status 0
    proc "pipeline_step_0" started
    + go get -t ./...
    + go build
    + go test -v
    === RUN   TestExpand
    --- PASS: TestExpand (0.00s)
    === RUN   TestFuzz
    --- PASS: TestFuzz (0.01s)
    === RUN   Test_len
    --- PASS: Test_len (0.00s)
    === RUN   Test_lower
    --- PASS: Test_lower (0.00s)
    === RUN   Test_lowerFirst
    --- PASS: Test_lowerFirst (0.00s)
    === RUN   Test_upper
    --- PASS: Test_upper (0.00s)
    === RUN   Test_upperFirst
    --- PASS: Test_upperFirst (0.00s)
    === RUN   Test_default
    --- PASS: Test_default (0.00s)
    PASS
    ok  	github.com/drone/envsubst	0.009s
    proc "pipeline_step_0" exited with status 0


## pipeline 原理分析

### 编译过程

可以形象的理解为 .env+pipeline.yml --&gt; pipeline.json

编译过程不复杂，主要是解析pipeline.yml为`Config`:


    Config struct {	Cache     libcompose.Stringorslice	Platform  string	Branches  Constraint	Workspace Workspace	Clone     Containers	Pipeline  Containers	Services  Containers	Networks  Networks	Volumes   Volumes	Labels    libcompose.SliceorMap}


然后转换为json对应的config：


    Config struct {	Stages   []*Stage   `json:"pipeline"` // pipeline stages	Networks []*Network `json:"networks"` // network definitions	Volumes  []*Volume  `json:"volumes"`  // volume definitions	Secrets  []*Secret  `json:"secrets"`  // secret definitions}


该部分主要代码在pipeline/frontend里

### 执行过程

我们主要关注执行过程，主要代码在pipeline/backend里。

首先是读取配置文件为backend.Config


    config, err := pipeline.Parse(reader)if err != nil {	return err}


然后创建执行环境，目前的代码仅docker可用，k8s是空代码。


    var engine backend.Engineif c.Bool("kubernetes") {	engine = kubernetes.New(		c.String("kubernetes-namepsace"),		c.String("kubernetes-endpoint"),		c.String("kubernetes-token"),	)} else {	engine, err = docker.NewEnv()	if err != nil {		return err	}}


接着开始执行


    	ctx, cancel := context.WithTimeout(context.Background(), c.Duration("timeout"))defer cancel()ctx = interrupt.WithContext(ctx)
    return pipeline.New(config,	pipeline.WithContext(ctx),	pipeline.WithLogger(defaultLogger),	pipeline.WithTracer(defaultTracer),	pipeline.WithEngine(engine),).Run()


其中pipeline.NEW创建了`Runtime`对象;


    type Runtime struct {err     error  // 错误信息spec    *backend.Config  // 配置信息engine  backend.Engine  // docker enginestarted int64 // 开始时间
    ctx    context.Contexttracer Tracerlogger Logger
    }


其中Engine，操作容器的interface，目前仅docker可用。


    // Engine defines a container orchestration backend and is used
    // to create and manage container resources.
    type Engine interface {// Setup the pipeline environment.Setup(context.Context, *Config) error// Start the pipeline step.Exec(context.Context, *Step) error// Kill the pipeline step.Kill(context.Context, *Step) error// Wait for the pipeline step to complete and returns// the completion results.Wait(context.Context, *Step) (*State, error)// Tail the pipeline step logs.Tail(context.Context, *Step) (io.ReadCloser, error)// Destroy the pipeline environment.Destroy(context.Context, *Config) error
    }


关注Run：


    // Run starts the runtime and waits for it to complete.
    func (r *Runtime) Run() error {
        // 延迟函数，用于销毁docker envdefer func() {	r.engine.Destroy(r.ctx, r.spec)}()// 初始化docker enginer.started = time.Now().Unix()if err := r.engine.Setup(r.ctx, r.spec); err != nil {	return err}
       
       // 依次运行stagefor _, stage := range r.spec.Stages {	select {	case <-r.ctx.Done():		return ErrCancel    // 执行	case err := <-r.execAll(stage.Steps):		if err != nil {			r.err = err		}	}}
    return r.err
    }


重点在于使用errgroup.Group通过协程方式运行step：


    
    // 执行所有steps
    func (r *Runtime) execAll(procs []*backend.Step) <-chan error {var g errgroup.Groupdone := make(chan error)
        // 遍历执行stepfor _, proc := range procs {   // 协程 exec	proc := proc	g.Go(func() error {		return r.exec(proc)	})}
    go func() {	done <- g.Wait()	close(done)}()return done
    }
    
    // 执行单个step
    func (r *Runtime) exec(proc *backend.Step) error {switch {case r.err != nil && proc.OnFailure == false:	return nilcase r.err == nil && proc.OnSuccess == false:	return nil}// trace日志if r.tracer != nil {	state := new(State)	state.Pipeline.Time = r.started	state.Pipeline.Error = r.err	state.Pipeline.Step = proc	state.Process = new(backend.State) // empty	if err := r.tracer.Trace(state); err == ErrSkip {		return nil	} else if err != nil {		return err	}}
       
       // docker engine执行if err := r.engine.Exec(r.ctx, proc); err != nil {	return err}
       
       // 记录日志信息if r.logger != nil {	rc, err := r.engine.Tail(r.ctx, proc)	if err != nil {		return err	}
    	go func() {		r.logger.Log(proc, multipart.New(rc))		rc.Close()	}()}
    if proc.Detached {	return nil}
    
       // 等待docker engine执行完成wait, err := r.engine.Wait(r.ctx, proc)if err != nil {	return err}
    if r.tracer != nil {	state := new(State)	state.Pipeline.Time = r.started	state.Pipeline.Error = r.err	state.Pipeline.Step = proc	state.Process = wait	if err := r.tracer.Trace(state); err != nil {		return err	}}
       if wait.OOMKilled {	return &OomError{		Name: proc.Name,		Code: wait.ExitCode,	}} else if wait.ExitCode != 0 {	return &ExitError{		Name: proc.Name,		Code: wait.ExitCode,	}}return nil
    }


* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


