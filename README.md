## Self Notes:
- `hugo server -D` will show a localhost live preview
- `hugo new post/my-first-post.md` (note that usually this would be `posts/my-first-post` but the even theme I'm using uses post)
- https://gohugo.io/getting-started/quick-start/
- https://github.com/olOwOlo/hugo-theme-even (guessing this will break one day because it doesn't seem to be maintained)

### Submodules
```
bash
# if i remember before cloning
git clone --recurse-submodules -j8 git@github.com:m-richards/m-richards.github.io.git
cd bar
#  For already cloned repos,
git clone git@github.com:m-richards/m-richards.github.io.git
cd bar
git submodule update --init --recursive
```

### Linting
```
bash
pip install mdformat mdformat-frontmatter
mdformat content --wrap 120
```
Note the frontmatter formatter only detects yaml, so unfortunately, that keeps me using yaml and not toml for the frontmatter