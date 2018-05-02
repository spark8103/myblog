## install
```bash
yum remove ruby ruby-devel
cd /root/software
wget https://cache.ruby-lang.org/pub/ruby/2.5/ruby-2.5.1.tar.gz
tar xvfvz ruby-2.5.1.tar.gz && cd ruby-2.5.1
./configure
make
make install

gem update --system
gem update

gem install jekyll bundler

cd /root/myblog
bundle install
```
