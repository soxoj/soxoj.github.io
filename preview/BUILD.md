# Building the site: how to run it, and where the time goes

Measured on 2026-11-14 with the `amirpourmand/al-folio:v0.12.1` image, 202 files
in the output. These numbers go stale if the plugin list in `_config.yml`
changes, the image is bumped, or the contents of `assets/css` change.

## Building locally

```bash
docker run --rm -v "$PWD":/srv/jekyll -e JEKYLL_ENV=development \
  amirpourmand/al-folio:v0.12.1 \
  sh -c 'gem install rexml -v 3.3.9 --no-document && bundle exec jekyll build'
```

Then serve it: `python3 -m http.server 8080 --directory _site`.

Three traps in that one command, each of which cost a full build to find:

- **`gem install rexml -v 3.3.9` is required.** The image is older than
  `Gemfile.lock`, and without it bundler dies with
  `Could not find rexml-3.3.9 in locally installed gems`.
- **Do not set `BUNDLE_PATH`.** The gems live inside the image at their own
  path. Setting `BUNDLE_PATH=/gems` without mounting a volume there makes
  bundler lose sight of every gem and print a list of eighty missing ones.
- **Pass `JEKYLL_ENV=development` explicitly.** The image already sets
  `JEKYLL_ENV=production` (`docker run --rm <image> env | grep JEKYLL`), so a
  build you think is "not production" still is unless you say otherwise. This
  is an easy way to record a measurement that is simply wrong.

## Where the time goes

Phases as reported by `jekyll build --profile`:

| build                                    | RENDER | WRITE       | total     |
| ---------------------------------------- | ------ | ----------- | --------- |
| production, same as CI                   | 2.0 s  | **208.4 s** | 211 s     |
| production, `compress_javascript: false` | 2.2 s  | 187.6 s     | 209 s     |
| production, `compress_css: false`        | 2.1 s  | **20.0 s**  | 41 s      |
| development                              | 2.2 s  | 0.1 s       | 21 s      |
| development, warm cache                  | —      | —           | **2.7 s** |

**Almost all of the time is CSS minification** inside `jekyll-minifier`: about
188 seconds out of 211. JS compression costs another ~20 seconds. Rendering
every template on the site takes two seconds, and writing the files takes
fractions of one.

The culprit is the site's **own `main.css`** (585 KB, compiled from `_sass`),
not the vendored libraries: excluding the already-minified `mdb.min.css` and
`bootstrap.min.css` changes nothing at all.

And the work is wasted. `_config.yml` sets `sass: style: compressed`, so Jekyll
already emits compressed CSS — 585 KB on 26 lines. Running `cssminify2` over it
takes 186 seconds and saves 0.0%: input 585 KB, output 585 KB. Adding
`assets/css/main.css` to the minifier's `exclude` list brings a production build
down from 211 to 41 seconds without changing a single byte of the output.

### Checked and found innocent

So nobody walks this circle a second time:

- **the cache** — a cold build and one with a warm `.jekyll-cache` both take the
  same 210 seconds in production;
- **imagemagick** — disabling it through a config override changes nothing;
- **the network** — the theme does fetch RSS at build time (`external_sources`
  in `_config.yml`), but the feed answers in 0.27 s from inside the container;
- **the docker bind mount** — copying the whole repository into the container
  takes 0 seconds, and building entirely on the container's own filesystem takes
  the same 230 s;
- **`assets/plotly/demo.html`** — the largest file in the output at 4 MB, but
  excluding it gains nothing;
- **the pre-minified libraries** — excluding `assets/css/*.min.css` and
  `assets/css/jupyter-*.css` gains nothing either; WRITE stays at 208 s.

The last two were working hypotheses of mine, and both were wrong. The only
thing that moves the needle is to stop minifying CSS — either for `main.css`
specifically, or by building in development.

## Production builds differently

`.github/workflows/deploy.yml` builds on `ubuntu-latest` through
`ruby/setup-ruby`, not in this image. Two consequences:

- **imagemagick does not run in CI.** The `jekyll-imagemagick` plugin is enabled
  in `_config.yml` and does generate webp locally in docker, but no such files
  exist in production: `soxoj.com/assets/img/1-480.webp` and
  `template_error-480.webp` both return 404, and only the hand-made
  `prof_pic_anim-*.webp` are served. **So any image inserted through
  `figure.liquid` gets a `srcset` pointing at files that do not exist.** This is
  why the project thumbnails use a plain `<img>` — see the comment in
  `_includes/projects.liquid`.
- **The `dev` branch is built with `--baseurl /preview`** and published into the
  `preview` subfolder; production does not wipe it (`clean-exclude: preview`).

After the build, CI runs `purgecss` over `_site/**/*.html`, so new classes in
templates survive the sweep as long as they appear in the rendered markup.

## A small landmine: colons in front matter

An unquoted value containing `: ` breaks the YAML silently. The page does not
fail to build — it loses its entire front matter, `permalink` included, and ends
up at the path of its source file. Caught on
`description: Hosted tools I built: …`, which moved the page to
`/_pages/projects/`. Check before building:

```bash
python3 -c "import yaml,pathlib,sys
for f in sys.argv[1:]:
    yaml.safe_load(pathlib.Path(f).read_text().split('---',2)[1])
    print(f,'OK')" _pages/*.md _projects/*.md
```
