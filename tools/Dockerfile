FROM debian

WORKDIR /github

RUN apt update
RUN apt install -y ruby-full build-essential zlib1g-dev git
RUN echo 'export GEM_HOME="$HOME/gems"' >> ~/.bashrc
RUN echo 'export PATH="$HOME/gems/bin:$PATH"' >> ~/.bashrc
RUN gem install jekyll bundler

ENTRYPOINT bundle install && tools/run.sh