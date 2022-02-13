# blog.amosti.net

This repo is not yet in use, but will be my main site when moving from Ghost.

Build:
```sh
docker run --rm -it \
  -v $(pwd):/src \
  klakegg/hugo:latest
```

Run server:

```sh
docker run --rm -it \
  -v $(pwd):/src \
  -p 1313:1313 \
  klakegg/hugo:latest \
  server
```

or with the `hugo` CLI:

```sh
hugo server --theme=paper
```

