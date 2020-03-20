notes.md
# Notes 

## Run your Jekyll site locally:

```shell
bundle exec jekyll serve
```

- to add a new subpage 
    - add folder <subpage>
    - in this subpage folder, create an index.html document
    - add link to subpage in _includes/nav.html if desired
    - add a condition in _layouts/home.html if desired

## Add new plugins

- install plugin
```shell
bundle exec gem install <plugin>
```
- add `<plugin>` to plugins section of ./_config.yml
- add `gem '<plugin>'` to ./Gemfile
- `bundle install`

## Creating posts

- name posts always in the following format 'YYYY-MM-DD-post-title.md'
- save new posts in ./_posts/ 
- save draft posts with format 'draft-title.md' in ./_drafts/ and call `bundle exec jekyll serve --draft` to enter preview mode 

## Creating Rmd posts

- put R markdown file in _posts/ with kebab-case formatted name '%Y-%m-%d-*'
- execute `r source('R/build_one.R')
- see https://bookdown.org/yihui/blogdown/jekyll.html

## Code Highlighting

- use Pygment (see https://help.github.com/en/articles/using-syntax-highlighting-on-github-pages, http://pygments.org/, and https://github.com/stephencelis/ghi/issues/221, and https://sachingpta.gitlab.io/_posts/jekyll-pygments-rouge.html):
    1. run `sudo pip install Pygments` (Python installation)
    2. run `sudo gem install pygments.rb` (Ruby installation)
    3. set `pygments: true` in './_config.yml'
    4. add `gem 'pygments.rb'` to './Gemfile'
    5. run `bundle install`
    5. run `bundle exec jekyll serve`

- change color scheme
    - see https://github.com/richleland/pygments-css for available style sheets (and preview: http://richleland.github.io/pygments-css/)
    - do
        1. download CSS style from https://github.com/richleland/pygments-css
        2. save CSS as .scss file to './_sass'
        3. add `@import "foo";` to './assets/css/main.css' 