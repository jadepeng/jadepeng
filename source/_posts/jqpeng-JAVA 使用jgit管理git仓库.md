---
title: JAVA 使用jgit管理git仓库
tags: ["jgit","jqpeng"]
categories: ["博客","jqpeng"]
date: 2019-12-20 12:08
---
文章作者:jqpeng
原文链接: [JAVA 使用jgit管理git仓库](https://www.cnblogs.com/xiaoqi/p/jgit.html)

最近设计基于gitops新的CICD方案,需要通过java读写git仓库，这里简单记录下。

JGit是一款pure java的软件包，可以读写git仓库，下面介绍基本使用。

## 引入jgit

maven引入：


            <!-- https://mvnrepository.com/artifact/org.eclipse.jgit/org.eclipse.jgit -->
            <dependency>
                <groupId>org.eclipse.jgit</groupId>
                <artifactId>org.eclipse.jgit</artifactId>
                <version>5.6.0.201912101111-r</version>
            </dependency>


jgit 有一个Git类，可以用来执行常规的git操作

## 凭证管理

通过`CredentialsProvider`管理凭证，常用的是`UsernamePasswordCredentialsProvider`

通过下面代码初始化：


    public static CredentialsProvider createCredential(String userName, String password) {
            return new UsernamePasswordCredentialsProvider(userName, password);
        }


## clone远程仓库

git 命令：


    git clone {repoUrl}


通过Git.cloneRepository 来clone远程仓库，如果需要凭证，则需要指定`credentialsProvider`


     public static Git fromCloneRepository(String repoUrl, String cloneDir, CredentialsProvider provider) throws GitAPIException {
            Git git = Git.cloneRepository()
                .setCredentialsProvider(provider)
                .setURI(repoUrl)
                .setDirectory(new File(cloneDir)).call();
            return git;
        }


## commit

git 命令：


    git commit -a -m '{msg}'


commit比较简单，对应commit方法, 注意需要先add


        public static void commit(Git git, String message, CredentialsProvider provider) throws GitAPIException {
            git.add().addFilepattern(".").call();
            git.commit()
                .setMessage(message)
                .call();
        }


## push

git 命令：


    git push origin branchName


push直接调用push即可, 需要指定`credentialsProvider`


       public static void push(Git git, CredentialsProvider provider) throws GitAPIException, IOException {
            push(git,null,provider);
        }
    
        public static void push(Git git, String branch, CredentialsProvider provider) throws GitAPIException, IOException {
            if (branch == null) {
                branch = git.getRepository().getBranch();
            }
            git.push()
                .setCredentialsProvider(provider)
                .setRemote("origin").setRefSpecs(new RefSpec(branch)).call();
        }


## 读取已有仓库

如果git已经clone了，想直接读取，怎么办？


     public static Repository getRepositoryFromDir(String dir) throws IOException {
            return new FileRepositoryBuilder()
                .setGitDir(Paths.get(dir, ".git").toFile())
                .build();
        }


## 读取仓库日志

可以通过RevWalk读取仓库日志。

- revWalk.parseCommit 可读取一条commit
- 遍历revWalk，可读取所有日志



     public static List<String> getLogs(Repository repository) throws IOException {
            return getLogsSinceCommit(repository, null, null);
        }
    
        public static List<String> getLogsSinceCommit(Repository repository, String commit) throws IOException {
            return getLogsSinceCommit(repository, null, commit);
        }
    
        public static List<String> getLogsSinceCommit(Repository repository, String branch, String commit) throws IOException {
            if (branch == null) {
                branch = repository.getBranch();
            }
            Ref head = repository.findRef("refs/heads/" + branch);
            List<String> commits = new ArrayList<>();
            if (head != null) {
                try (RevWalk revWalk = new RevWalk(repository)) {
                    revWalk.markStart(revWalk.parseCommit(head.getObjectId()));
                    for (RevCommit revCommit : revWalk) {
                        if (revCommit.getId().getName().equals(commit)) {
                            break;
                        }
                        commits.add(revCommit.getFullMessage());
                        System.out.println("\nCommit-Message: " + revCommit.getFullMessage());
                    }
                    revWalk.dispose();
                }
            }
    
            return commits;
        }
    


## 测试

我们来先clone仓库，然后修改，最后push


     String yaml = "dependencies:\n" +
                "- name: springboot-rest-demo\n" +
                "  version: 0.0.5\n" +
                "  repository: http://hub.hubHOST.com/chartrepo/ainote\n" +
                "  alias: demo\n" +
                "- name: exposecontroller\n" +
                "  version: 2.3.82\n" +
                "  repository: http://chartmuseum.jenkins-x.io\n" +
                "  alias: cleanup\n";
            CredentialsProvider provider = createCredential("USR_NAME", "PASSWORD");
    
            String cloneDir = "/tmp/test";
    
            Git git = fromCloneRepository("http://gitlab.GITHOST.cn/blog/env-test.git", cloneDir, provider);
    
            // 修改文件
    
            FileUtils.writeStringToFile(Paths.get(cloneDir, "env", "requirements.yaml").toFile(), yaml, "utf-8");      // 提交
            commit(git, "deploy(app): deploy  springboot-rest-demo:0.0.5 to env test", provider);
            // push 到远程仓库
            push(git, "master", provider);
    
            git.clean().call();
            git.close();
    
            FileUtils.deleteDirectory(new File(cloneDir));


读取已有仓库的日志：


            Repository repository = getRepositoryFromDir("GIT_DIR");
            List<String> logs = getLogs(repository);
            System.out.println(logs);
    
            RevCommit head = getLastCommit(repository);
            System.out.println(head.getFullMessage());


## 小结

本文讲述了如何通过jgit完成常规的git操作。

* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


