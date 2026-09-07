source "https://rubygems.org"

# Jekyll directly rather than the github-pages meta-gem: nginx serves the built
# output, so pinning GitHub's whole build environment bought nothing while
# dragging in nokogiri and friends, whose native extensions are what exhaust
# memory during deploys.
#
# Held at Jekyll 3.x, the major github-pages was pinning. Moving to Jekyll 4 is
# a separate change with real template implications.
gem "jekyll", "~> 3.9"

# The plugins _config.yml declares.
gem "jekyll-gist"
gem "jekyll-paginate"
gem "jekyll-seo-tag"
gem "jekyll-redirect-from"

# Jekyll 3 defaults to kramdown's GFM input, and the GFM parser has lived in its
# own gem since kramdown 2.0. github-pages supplied it implicitly.
gem "kramdown-parser-gfm"

# Removed from Ruby's stdlib in 3.0; only needed for `jekyll serve` locally.
gem "webrick", "~> 1.9"
