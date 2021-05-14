---
title: XNginx升级记录
tags: ["XNginx","jqpeng"]
categories: ["博客","jqpeng"]
date: 2019-08-26 19:30
---
文章作者:jqpeng
原文链接: [XNginx升级记录](https://www.cnblogs.com/xiaoqi/p/xnginx-v-1-1.html)

之前的博文提到过，[XNginx - nginx 集群可视化管理工具](https://www.cnblogs.com/xiaoqi/p/xnginx.html), 开发完成后一直稳定运行，直到前面因为一个站点的proxy站点配置问题，导致需要修改nginx 配置文件模板，因此借此机会对系统做了升级。

## 前端升级到最新版的ng-alain

事实证明，升级是痛苦的，前端项目真是一言难尽，能不动最好不动！

主要的变更是：  
 - 之前的simple-table变成了st  
 - desc也没了，成了sv，  
 - page-header等的action也需要显示的指定

查查文档，前后花了一个多小时，前端的升级真是太快了。。。

## vhost增加default

通常会有类似下面的配置存在，通过default来标示是默认的配置：


      server {
            listen 80 default;
            client_max_body_size 10240M;
          
            location / {
            
            proxy_pass http://proxy234648622.k8s_server;
                proxy_set_header HOST $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                                
          }          
       }


因此，这次给vhost增加了default选项，这样生成配置文件就可以加上default。

![default](https://gitee.com/jadepeng/pic/raw/master/pic/2019/8/26/1566817603353.png)

生成的配置文件：

![配置文件](https://gitee.com/jadepeng/pic/raw/master/pic/2019/8/26/1566817664946.png)

## SSL配置增加导入证书

之前SSL配置需要手动打开证书文件，拷贝文件内容到文本框，这次前端升级，增加了导入按钮，用户选择后直接读取证书文件.

实现很简单，使用`nz-upload`上传文件，通过`nzBeforeUpload`进行拦截，读取文件。


     <div nz-col [nzSpan]="2" *ngIf="dto.enableSSL">
            <nz-upload nzShowUploadList="false" [nzBeforeUpload]="readCertificate"><button nz-icon="upload" nz-button
                nzType="nz-button.default" nzSize="small">导入</button> </nz-upload>
          </div>  


读取可以使用FileReader，记得return false。


      readCertificate = (file: File) => {
        const reader = new FileReader();
        reader.readAsText(file);
        this.dto.sslCertificate.commonName = file.name;
        reader.onload = () => {
          this.dto.sslCertificate.content = reader.result.toString();
        }
        return false;
      }


## 导入已有配置文件

本次升级，在vhosts管理地方，增加了一个导入按钮，可以导入配置信息。

![导入](https://gitee.com/jadepeng/pic/raw/master/pic/2019/8/26/1566818042877.png)

支持的方式是要求将配置文件及其相关资源，打包为zip，上传到系统后台进行解析, 接口代码：


    
    @PostMapping("/importConfig/{groupId}")
        @Timed
        public String uploadConfFile(@RequestParam("file") MultipartFile file, @PathVariable String groupId) {
            if (file.isEmpty()) {
                return "Please select a file to upload";
            }
    
            if (!file.getContentType().equalsIgnoreCase("application/x-zip-compressed")) {
                return "only support.zip";
            }
    
            File upFile = new File(new File(TEMP_FILE_PATH),  System.currentTimeMillis() + file.getOriginalFilename());
            try {
                if(upFile.exists()){
                    upFile.delete();
                }
                file.transferTo(upFile);
            } catch (IllegalStateException | IOException ex) {
                return "upload error！";
            }
    
            try {
                nginxConfigService.parseFromZipFile(upFile, groupId);
            } catch (IOException e) {
                return "upload error！";
            }
            return "success";
        }
    
    


解析代码比较简单，先解压zip，然后找到nginx.conf，再调用上文提到的解析代码解析指令。


     public void parseConfig(String confidDir, String groupId) {
    
            // 查找nginx.conf
            String nginxFile = searchForFile(new File(confidDir), "nginx.conf");
            if (nginxFile.length() == 0) {
                throw new RuntimeException("can't find nginx.conf,please make sure nginx.conf exist !");
            }
    
            List<Directive> directives = NginxConfParser.newBuilder().withConfigurationFile(nginxFile).parse();
            directives.stream().forEach(directive -> {
                if (directive instanceof ProxyDirective) {
                    saveUpStream((ProxyDirective) directive);
                } else if (directive instanceof VirtualHostDirective) {
                    saveVHost((VirtualHostDirective) directive, groupId);
                }
            });
    
        }
    
        public void parseFromZipFile(File file, String groupId) throws IOException {
            String tempDir = Paths.get(file.getPath()).getParent().toString() + File.separator + file.getName() + ".extract";
            UnZipFile.unZipFiles(file, tempDir);
            parseConfig(tempDir, groupId);
        }


## 前后端项目合并到一起

之前前后端独立部署，如果项目足够大尚可，但是这个xnginx相对比较简单，独立部署费时费力，因此本次将前后端合并到一起

合并方法：

- 在backend新建一个webapp目录，将web代码放入
- 将web的相关配置文件拷贝到上层目录


![目录结构](https://gitee.com/jadepeng/pic/raw/master/pic/2019/8/26/1566818538910.png)

然后修改`angular.json`、`tsconfig.json` 等包含路径的地址进行修改


     "xnginx": {
          "projectType": "application",
          "root": "",
          "sourceRoot": "webapp/src",
          "prefix": "app",
          "schematics": {
            "@schematics/angular:component": {
              "styleext": "less"
            }
          },


最后，修改angular.json的build配置，将构建结果保存到'target/classes/static',这样java项目打包时就能将前端资源带入：


      "build": {
              "builder": "@angular-devkit/build-angular:browser",
              "options": {
                "outputPath": "target/classes/static",
                "index": "webapp/src/index.html",
                "main": "webapp/src/main.ts",
                "tsConfig": "tsconfig.app.json",
                "polyfills": "webapp/src/polyfills.ts",
                "assets": [
                  "webapp/src/assets",
                  "webapp/src/favicon.ico"
                ],
                "styles": [
                  "webapp/src/styles.less"
                ],
                "scripts": [
                  "node_modules/@antv/g2/build/g2.js",
                  "node_modules/@antv/data-set/dist/data-set.min.js",
                  "node_modules/@antv/g2-plugin-slider/dist/g2-plugin-slider.min.js",
                  "node_modules/ajv/dist/ajv.bundle.js",
                  "node_modules/qrious/dist/qrious.min.js"
                ]
              },


注意事项：

- 先构建前端，`npm run build`
- 再构建后端 `mvn package -DskipTests`


* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


