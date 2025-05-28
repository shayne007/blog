# Blog Project
This is a blog project use hexo
# Preparation
```bash
# first clone the repository
git clone https://github.com/shayne007/blog
cd blog

# before running install the packages
npm install hexo-cli -g
npm install

```
# Run locally
```bash
# this command will compile the md files to js and html file
hexo clean && hexo generate
# run locally and then visit localhost:4000
hexo server
```
# Publish to github
```bash
# we publish the generate files to https://shayne007.github.io/ configured in _config.yml
hexo deploy
```

# Create a new post
```bash
hexo new "My New Post"
```