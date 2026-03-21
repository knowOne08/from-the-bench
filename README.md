# from-the-bench

Engineering notes from building a distributed DAQ for rocket engines.

## Setup

This is a Jekyll blog hosted on GitHub Pages.

### To deploy:
1. Push this repo to GitHub
2. Go to Settings → Pages → Source: Deploy from branch (`main`, `/ (root)`)
3. Your site will be live at `https://knowone08.github.io/from-the-bench/`

### To run locally:
```bash
gem install bundler
bundle install
bundle exec jekyll serve
```

## Structure

```
_posts/           → Blog posts (YYYY-MM-DD-title.md)
assets/images/    → Diagrams and figures
about.md          → About page
_config.yml       → Site config
```

## Blog series

### Ethernet → TSN arc
1. Why Ethernet isn't doing it for me
2. TSN and the emergence of determinism *(upcoming)*
3. Making a case for TSN *(upcoming)*
4. The search for a minimum feature set *(upcoming)*
