---
title: 教程 —— 如何在自己的应用集成superset
tags: ["superset","BI","python","jqpeng"]
categories: ["博客","jqpeng"]
date: 2019-09-17 20:01
---
文章作者:jqpeng
原文链接: [教程 —— 如何在自己的应用集成superset](https://www.cnblogs.com/xiaoqi/p/intergate-superset.html)

Superset 是apache的一个孵化项目，定位为一款现代的，准商用BI系统

## superset


> Apache Superset  (incubating) is a modern, enterprise-ready business intelligence web application


Superset 是apache的一个孵化项目，定位为一款现代的，准商用BI系统。

Superset（Caravel）是由Airbnb（知名在线房屋短租公司）开源的数据分析与可视化平台（曾用名Caravel、Panoramix），该工具主要特点是可自助分析、自定义仪表盘、分析结果可视化（导出）、用户/角色权限控制，还集成了一个SQL编辑器，可以进行SQL编辑查询等。

通过superset，可以制作漂亮的统计图表。

### 预览

![_images/bank_dash.png](https://superset.incubator.apache.org/_images/bank_dash.png)

* * *

![_images/explore.png](https://superset.incubator.apache.org/_images/explore.png)

* * *

![_images/sqllab.png](https://superset.incubator.apache.org/_images/sqllab.png)

* * *

![_images/deckgl_dash.png](https://superset.incubator.apache.org/_images/deckgl_dash.png)

## superset安装

我们这里直接使用docker


    git clone https://github.com/apache/incubator-superset/
    cd incubator-superset/contrib/docker
    # prefix with SUPERSET_LOAD_EXAMPLES=yes to load examples:
    docker-compose run --rm superset ./docker-init.sh
    # you can run this command everytime you need to start superset now:
    docker-compose up


等构建完成后，访问 [http://localhost:8088](http://localhost:8088) 即可。

想要在自己的应用集成，首先要解决认证

## superset 认证分析

superset基于flask-appbuilder开发，security基于flask\_appbuilder.security，翻阅其代码，

找到入口： `superset/__init__.py`:


    custom_sm = app.config.get('CUSTOM_SECURITY_MANAGER') or SupersetSecurityManager
    if not issubclass(custom_sm, SupersetSecurityManager):
        raise Exception(
            """Your CUSTOM_SECURITY_MANAGER must now extend SupersetSecurityManager,
             not FAB's security manager.
             See [4565] in UPDATING.md""")
    
    appbuilder = AppBuilder(
        app,
        db.session,
        base_template='superset/base.html',
        indexview=MyIndexView,
        security_manager_class=custom_sm,
        update_perms=get_update_perms_flag(),
    )
    
    security_manager = appbuilder.sm


默认使用`SupersetSecurityManager`,继承自`SecurityManager`：


    class SupersetSecurityManager(SecurityManager):
    
        def get_schema_perm(self, database, schema):
            if schema:
                return '[{}].[{}]'.format(database, schema)
    
        def can_access(self, permission_name, view_name):
            """Protecting from has_access failing from missing perms/view"""
            user = g.user
            if user.is_anonymous:
                return self.is_item_public(permission_name, view_name)
            return self._has_view_access(user, permission_name, view_name)		...


我们再来看SecurityManager及父类，发现，登录是通过auth\_view来控制的，默认是AUTH\_DB，也就是AuthDBView。


    
    """ Override if you want your own Authentication LDAP view """
          authdbview = AuthDBView      if self.auth_type == AUTH_DB:
                self.user_view = self.userdbmodelview
                self.auth_view = self.authdbview()
     @property
        def get_url_for_login(self):
            return url_for('%s.%s' % (self.sm.auth_view.endpoint, 'login'))


再来看authdbview：


    class AuthDBView(AuthView):
        login_template = 'appbuilder/general/security/login_db.html'
    
        @expose('/login/', methods=['GET', 'POST'])
        def login(self):
            if g.user is not None and g.user.is_authenticated:
                return redirect(self.appbuilder.get_url_for_index)
            form = LoginForm_db()
            if form.validate_on_submit():
                user = self.appbuilder.sm.auth_user_db(form.username.data, form.password.data)
                if not user:
                    flash(as_unicode(self.invalid_login_message), 'warning')
                    return redirect(self.appbuilder.get_url_for_login)
                login_user(user, remember=False)
                return redirect(self.appbuilder.get_url_for_index)
            return self.render_template(self.login_template,
                                   title=self.title,
                                   form=form,
                                   appbuilder=self.appbuilder)


对外提供'/login/'接口，读取HTTP POST里的用户名，密码，然后调用auth\_user\_db验证，验证通过调用login\_user生成认证信息。

因此，我们可以自定义AuthDBView，改为从我们自己的应用认证即可。

## 使用jwt来验证superset

自定义CustomAuthDBView，继承自AuthDBView，登录时可以通过cookie或者url参数传入jwt token，然后验证通过的话，自动登录，。


    import jwt
    import json
    class CustomAuthDBView(AuthDBView):
        login_template = 'appbuilder/general/security/login_db.html'
    
        @expose('/login/', methods=['GET', 'POST'])
        def login(self):
            token = request.args.get('token')
            if not token:
                token = request.cookies.get('access_token')
            if token is not None:
                jwt_payload = jwt.decode(token,'secret',algorithms=['RS256'])
                user_name = jwt_payload.get("user_name")
                user = self.appbuilder.sm.find_user(username=user_name)
                if not user:
                   role_admin = self.appbuilder.sm.find_role('Admin')
                   user = self.appbuilder.sm.add_user(user_name, user_name, 'aimind', user_name + "@aimind.com", role_admin, password = "aimind" + user_name)
                if user:
                    login_user(user, remember=False)
                    redirect_url = request.args.get('redirect')
                    if not redirect_url:
                        redirect_url = self.appbuilder.get_url_for_index
                    return redirect(redirect_url)
                else:
                    return super(CustomAuthDBView,self).login()
            else:
                flash('Unable to auto login', 'warning')
                return super(CustomAuthDBView,self).login()
    


如果用户不存在，通过self.appbuilder.sm.add\_user自动添加用户。

然后再引入这个CustomAuthDBView，


    class CustomSecurityManager(SupersetSecurityManager):
        authdbview = CustomAuthDBView


最后，再引入这个CustomSecurityManager,在superset\_config.py 里增加：


    from aimind_security import CustomSecurityManager
    CUSTOM_SECURITY_MANAGER = CustomSecurityManager


## 在应用里集成superset

集成就简单了，访问，'SUPER\_SET\_URL/login/?token=jwt\_token' 即可，可以通过iframe无缝集成。

* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


