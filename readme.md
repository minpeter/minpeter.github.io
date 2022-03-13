## Method 2

You can use as submodule with

```
git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod --depth=1
git submodule update --init --recursive # needed when you reclone your repo (submodules may not get cloned automatically)
```

**Note**: You may use --branch v5.0 to end of above command if you want to stick to specific release.

Updating theme :

```
git submodule update --remote --merge
```
