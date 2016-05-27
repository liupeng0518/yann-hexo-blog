FROM jprjr/lighttpd
#添加静态页面
ADD / /srv/http/
#启动服务器
ENTRYPOINT ["lighttpd", "-D", "-f", "/etc/lighttpd/lighttpd.conf"]
