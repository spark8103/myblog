## run jekyll server
bundle exec jekyll serve

## publish
bundle exec jekyll build

## copy nginx
/bin/rm -rf /opt/programs/nginx_1.12.2/html/* && /bin/cp -rf /root/myblog/_site/* /opt/programs/nginx_1.12.2/html/
