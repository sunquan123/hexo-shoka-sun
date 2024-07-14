# hexo-shoka-sun

我的博客2.0

部署到服务器/opt/hexo-sun/目录下，Nginx代理静态页面。nginx配置：

```nginx
server {
    listen 8848;
    #填写证书绑定的域名
    server_name www.sunq.site;
    root /opt/hexo-sun;
    #将所有HTTP请求通过rewrite指令重定向到HTTPS。
    location  / {
            index          index.html;
        }

    }
```
