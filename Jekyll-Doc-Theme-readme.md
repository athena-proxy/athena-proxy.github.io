# Jekyll Doc Theme

Go to [the website](https://aksakalli.github.io/jekyll-doc-theme/) for detailed information and demo.

## Running locally

You need Ruby and gem before starting, then:

```bash
# install bundler
gem install bundler

# clone the project
git clone https://github.com/aksakalli/jekyll-doc-theme.git
cd jekyll-doc-theme

# install dependencies
bundle install

# run jekyll with dependencies
bundle exec jekyll serve
```

### Theme Assets

As of the move to support [Github Pages](https://pages.github.com/) a number of files have been relocated to the `/asset` folder.
- css/
- fonts/
- img/
- js/
- 404.html
- allposts.html
- search.json

## Docker

Alternatively, you can deploy it using the multi-stage [Dockerfile](Dockerfile)
that serves files from Nginx for better performance in production.

Build the image for your site's `JEKYLL_BASEURL`:

```
docker build --build-arg JEKYLL_BASEURL="/your-base/url" -t jekyll-doc-theme .
```

(or leave it empty for root: `JEKYLL_BASEURL=""`) and serve it:

```
docker run -p 8080:80 jekyll-doc-theme
```

## Github Pages

The theme is also available to [Github Pages](https://pages.github.com/) by making use of the [Remote Theme](https://github.com/benbalter/jekyll-remote-theme) plugin:

**Gemfile**
```
# If you want to use GitHub Pages, remove the "gem "jekyll"" above and
# uncomment the line below. To upgrade, run `bundle update github-pages`.
gem "github-pages", group: :jekyll_plugins
```

**_config.yml**
```
# Configure the remote_theme plugin with the gh-pages branch
# or the specific tag
remote_theme: aksakalli/jekyll-doc-theme@gh-pages   
```

### Theme Assets

Files from your project will override any theme file with the same name.  For example, the most comment use case for this, would be to modify your sites theme or colors.   To do this, the following steps should be taken:

1) Copy the contents of the `aksakalli/jekyll-doc-theme/asset/css/main.scss` to your own project (maintaining folder structure)
2) Modify the variables you wish to use prior to the import statements, for example:

```
// Bootstrap variable overrides
$grid-gutter-width: 30px !default;
$container-desktop: (900px + $grid-gutter-width) !default;
$container-large-desktop: (900px + $grid-gutter-width) !default;

@import // Original import statement
  {% if site.bootwatch %}
    "bootswatch/{{site.bootwatch | downcase}}/variables",
  {% endif %}

  "bootstrap",

  {% if site.bootwatch %}
    "bootswatch/{{site.bootwatch | downcase}}/bootswatch",
  {% endif %}

  "syntax-highlighting",
  "typeahead",
  "jekyll-doc-theme"
;

// More custom overrides.
```

3) Import or override any other theme styles after the standard imports

## License

Released under [the MIT license](LICENSE).
