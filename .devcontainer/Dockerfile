FROM ruby:3.3.0

RUN gem update --system

RUN gem install jekyll bundler

COPY Gemfile .

RUN bundle install