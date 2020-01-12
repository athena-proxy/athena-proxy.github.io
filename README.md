# athena-proxy
## how to write docs
under dir: ./_docs
use markdown

## how to write blog
under dir: ./_posts  
at the top of every blog,we should add the code like:
```yaml
---
  layout: post
  title:  "Remote Working Protocol for the Athena Project"
  date:   2020-01-07 16:00:00
  author: dongyan.xu
---
```

## how to publish your modification
```shell
cd athena.github.io
git config user.email <your email in github>
git config user.name <your name in github>
git add .
git commit -m "your modification"
git push
```
wait for a minute.  
Jekyll will build it.  
interview http://athena-proxy.io
