### Install ruby dependencies
```sh
sudo apt-get install ruby-full build-essential
```

##

Get dependencies
```sh
bundle config set --local path 'vendor/bundle'
bundle install
``

Serve Site
```sh
  bundle exec jekyll serve \
    [l] --livereload \
    [o] --open-url 
```

example:
```sh
bundle exec jekyll serve -l -o
```

Update jekyll config
```sh  
vim _config.yml
```

### Build with github actions
The following workflow will build and deploy the site to github pages on push to main branch. 
```
.github/workflows/pages-deploy.yml
```

### Manually build and deploy to github pages
```sh
export JEKYLL_ENV=production
bundle exec jekyll b -d "_site${{ steps.pages.outputs.base_path }}"
```

### Google search indexing 

update: GemFile add:  
  
  gem 'jekyll-sitemap'

Run: 
  bundle

update: _config.yml update or add: 

  url: "https://example.com" # the base hostname & protocol for your site
  plugins:
    - jekyll-sitemap


This will now generate sitemap.xml in the _sites dir


## Unpublished posts

Add following front matter to the post you want to keep as draft. 

```md
---
published: false
---
```


To test site with unpublished posts, run the following command to serve the site with drafts.
```
bundle exec jekyll serve --unpublished
```

