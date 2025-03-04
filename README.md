# JRPERIN BLOG

Instalar jekyll

<https://jekyllrb.com/docs/installation/ubuntu/>

``` shell
sudo apt install -y ruby-full build-essential zlib1g-dev
echo '# Install Ruby Gems to ~/gems' >> ~/.bashrc
echo 'export GEM_HOME="$HOME/gems"' >> ~/.bashrc
echo 'export PATH="$HOME/gems/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
gem install jekyll bundler
```

Especificar as dependencias no arquivo Gemfile na raiz do projeto:

``` shell
# na ultima instalacao 04/02/2025 nao precisou mais
# source 'https://rubygems.org'
# gem 'nokogiri'
# gem 'rack', '~> 2.0.1'
# gem 'rspec'

```

Instalar todas as dependencias:

``` shell
# $ bundle install
$ bundle update sass
$ git add Gemfile Gemfile.lock
```

Executar site localmente:
``` shell
bundle exec jekyll serve

# se der erro no comando acima:
bundler exec jekyll build && bash -c 'cd _site && python -m http.server 300
```