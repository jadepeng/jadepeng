---
title: 通过 Drone Rest API 获取构建记录日志
tags: ["drone","jqpeng"]
categories: ["博客","jqpeng"]
date: 2019-12-18 16:17
---
文章作者:jqpeng
原文链接: [通过 Drone Rest API 获取构建记录日志](https://www.cnblogs.com/xiaoqi/p/drone-logs-api.html)

* * *

Drone是一款CICD工具，提供rest API，简单介绍下如何使用API 获取构建日志。

## 获取token

登录进入drone，点头像，在菜单里选择token

![enter description here](https://gitee.com/jadepeng/pic/raw/master/pic/2019/12/18/1576655214876.png)

复制token即可

## API 介绍

Drone的api分为几大类

- Builds 构建
- Cron 定时任务
- Repos 仓库
- Secrets
- User 用户
- Users


调用举例：

![enter description here](https://gitee.com/jadepeng/pic/raw/master/pic/2019/12/18/1576655383949.png)

## Build API

### 构建列表（Build List）

获取仓库的最新构建：


    GET /api/repos/{owner}/{repo}/builds
    
    curl -i http://drone.YOUR_HOST.cn/api/repos/jqpeng/springboot-rest-demo/builds -H "Authorization: Bearer TOKEN"
    


响应正文示例：


    [
      {
          "id": 100207,
          "repo_id": 296163,
          "number": 42,
          "status": "success",
          "event": "pull_request",
          "action": "sync",
          "link": "https://github.com/octoat/hello-world/compare/e3320539a4c0...9fc1ad6ebf12",
          "message": "updated README",
          "before": "e3320539a4c03ccfda992641646deb67d8bf98f3",
          "after": "9fc1ad6ebf12462f3f9773003e26b4c6f54a772e",
          "ref": "refs/heads/master",
          "source_repo": "spaceghost/hello-world",
          "source": "develop",
          "target": "master",
          "author_login": "octocat",
          "author_name": "The Octocat",
          "author_email": "octocat@github.com",
          "author_avatar": "http://www.gravatar.com/avatar/7194e8d48fa1d2b689f99443b767316c",
          "sender": "bradrydzewski",
          "started": 1564085874,
          "finished": 1564086343,
          "created": 1564085874,
          "updated": 1564085874,
          "version": 3
      }
    ]


### 构建详情

通过该接口获取构建详情，返回构建状态等信息，{build} 为上面列表里的number，既构建序号。


    GET /api/repos/{owner}/{repo}/builds/{build}


Example Response Body:


    {
        "id": 39862,
        "number": 20,
        "parent": 0,
        "event": "push",
        "status": "success",
        "error": "",
        "enqueued_at": 1576636849,
        "created_at": 1576636849,
        "started_at": 1576636850,
        "finished_at": 1576639053,
        "deploy_to": "",
        "commit": "7729006bfe11933da6c564101acaf8c7f78c5f62",
        "branch": "master",
        "ref": "refs/heads/master",
        "refspec": "",
        "remote": "",
        "title": "",
        "message": "通过update更新\n",
        "timestamp": 0,
        "sender": "",
        "author": "jqpeng",
        "author_avatar": "https://www.gravatar.com/avatar/4ab53b564545f18efc4079c30a2d35cf.jpg?s=128",
        "link_url": "",
        "signed": false,
        "verified": true,
        "reviewed_by": "",
        "reviewed_at": 0,
        "procs": [
            {
                "id": 247912,
                "build_id": 39862,
                "pid": 1,
                "ppid": 0,
                "pgid": 1,
                "name": "",
                "state": "success",
                "exit_code": 0,
                "start_time": 1576636850,
                "end_time": 1576639053,
                "machine": "21e73ce43038",
                "children": [
                    {
                        "id": 247913,
                        "build_id": 39862,
                        "pid": 2,
                        "ppid": 1,
                        "pgid": 2,
                        "name": "clone",
                        "state": "success",
                        "exit_code": 0,
                        "start_time": 1576636853,
                        "end_time": 1576636933,
                        "machine": "21e73ce43038"
    },
                    {
                        "id": 247914,
                        "build_id": 39862,
                        "pid": 3,
                        "ppid": 1,
                        "pgid": 3,
                        "name": "build",
                        "state": "success",
                        "exit_code": 0,
                        "start_time": 1576636933,
                        "end_time": 1576636998,
                        "machine": "21e73ce43038"
    }
    ]
    }
    ]
    }


procs 是构建的步骤，记住pid，获取构建日志有用

### 构建日志

获取构建日志，需要传入{log} 和 {pid}, log是上面的{build}，{pid}是上一步返回的pid


    GET /api/repos/{owner}/{repo}/logs/{log}/{pid}
    


响应正文示例：


    [
      {
        "proc": "clone",
        "pos": 0,
        "out": "+ git init\n"
      },
      {
        "proc": "clone",
        "pos": 1,
        "out": "Initialized empty Git repository in /drone/src/github.com/octocat/hello-world/.git/\n"
      },
      {
        "proc": "clone",
        "pos": 2,
        "out": "+ git remote add origin https://github.com/octocat/hello-world.git\n"
      },
      {
        "proc": "clone",
        "pos": 3,
        "out": "+ git fetch --no-tags origin +refs/heads/master:\n"
      },
      {
        "proc": "clone",
        "pos": 4,
        "out": "From https://github.com/octocat/hello-world\n"
      },
      {
        "proc": "clone",
        "pos": 5,
        "out": " * branch            master     -> FETCH_HEAD\n"
      },
      {
        "proc": "clone",
        "pos": 6,
        "out": " * [new branch]      master     -> origin/master\n"
      },
      {
        "proc": "clone",
        "pos": 7,
        "out": "+ git reset --hard -q 62126a02ffea3dabd7789e5c5407553490973665\n"
      },
      {
        "proc": "clone",
        "pos": 8,
        "out": "+ git submodule update --init --recursive\n"
      }
    ]


* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


