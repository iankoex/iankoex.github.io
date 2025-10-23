# Northendlab Jekyll

![northendlab](https://demo.themefisher.com/thumbnails/northendlab-jekyll.png)

[live demo](https://demo.themefisher.com/northendlab-jekyll/)

## Setup

1. Ensure Ruby 3.4+ is installed (via Homebrew):
   ```bash
   brew install ruby
   export PATH="/opt/homebrew/opt/ruby/bin:$PATH"
   ```

2. Install dependencies:
   ```bash
   bundle install --path vendor/bundle
   ```

3. Build the site:
   ```bash
   bundle exec jekyll build
   ```

4. Serve locally:
   ```bash
   bundle exec jekyll serve
   ```

The site will be available at http://127.0.0.1:4000/
