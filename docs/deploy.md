# deploy

`rake deploy`, is a short cut to run the full Jekyll command `JEKYLL_ENV=production bundle exec jekyll build`.
Using the production environment causes template to include analytics and additional machine markup in build, which can take considerably longer than `jekyll s`.
