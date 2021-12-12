# JRPERIN BLOG

Instalar jekyll

<https://jekyllrb.com/docs/installation/ubuntu/>

``` shell
sudo apt-get install ruby-full build-essential zlib1g-dev
echo '# Install Ruby Gems to ~/gems' >> ~/.bashrc
echo 'export GEM_HOME="$HOME/gems"' >> ~/.bashrc
echo 'export PATH="$HOME/gems/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
gem install jekyll bundler
```

Especificar as dependencias no arquivo Gemfile na raiz do projeto:

``` shell
source 'https://rubygems.org'
gem 'nokogiri'
gem 'rack', '~> 2.0.1'
gem 'rspec'

```

Instalar todas as dependencias:

``` shell
$ bundle install
$ git add Gemfile Gemfile.lock
```

Executar site localmente:
``` shell
bundle exec jekyll serve
```