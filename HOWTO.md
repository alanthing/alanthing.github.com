# Pre-reqs

```bash
brew install direnv
# Then, configure shell profile to use direnv hook in PROMPT_COMMAND

# cd into this directory and the .envrc file should load rbenv without additional edits to the shell profile
brew install rbenv rbenv-gemset

rbenv install -v 2.3.5
```

# Verify

```
$ rbenv version
2.3.5 (set by ./.ruby-version)

$ rbenv gemset active
octopress global
```

# Octopress gems

```bash
gem install bundler

rbenv rehash

bundle install -j4
```

# Octopress commands

```bash
bundle exec rake generate

bundle exec rake preview

bundle exec rake deploy
```
