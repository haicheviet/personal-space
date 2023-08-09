# personal-space

[![Netlify Status](https://api.netlify.com/api/v1/badges/1629d967-8817-43d2-a166-f436d42bc0af/deploy-status)](https://app.netlify.com/sites/exquisite-torte-3aec3c/deploys)

## [Blog site](http://haicheviet.com/)

## Local development

Build Blog Locally:

```bash
hugo server --source=site
```

## Build introduction gif

* Install [vhs](https://github.com/charmbracelet/vhs#installation) and [neofetch](https://github.com/dylanaraps/neofetch)
* Update your neofetch config to file `./scripts/introduction-gif/neofetch.conf`, remember to modify this file to change to absolute path
* Run command `vhs ./scripts/introduction-gif/introduction.tape` to generate introduction.* format
