# flatgraph docs

Companion repository with documentation sources for [flatgraph](https://github.com/joernio/flatgraph), served on https://flatgraph.joern.io

## Serve locally
```
hugo server --disableFastRender
```
Prerequisite: hugo, in doubt with the same version that's configured in [gh-pages.yaml](.github/workflows/gh-pages.yaml). Other versions might work, or not... 

## A few words on hugo
The minimum version is defined by the relearn theme, and newer hugo versions might have backwards incompatible API changes. 

If you upgrade the hugo version, you typically need to also upgrade the relearn theme, i.e. 
```bash
cd ./themes/relearn
git pull
```

