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

## Creating posts

- name posts always in the following format 'YYYY-MM-DD-post-title.md'
- save new posts in ./_posts/
- save draft posts with format 'draft-title.md' in ./_drafts/ and call `bundle exec jekyll serve --draft` to enter preview mode 
