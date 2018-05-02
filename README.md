## run jekyll server
bundle exec jekyll serve

## publish
bundle exec jekyll build

## copy nginx
/bin/rm -rf /opt/programs/nginx_1.14.0/html/* && /bin/cp -rf /root/myblog/_site/* /opt/programs/nginx_1.14.0/html/

## upload github.com
git add .
git commit -m "update xxx"
git push
