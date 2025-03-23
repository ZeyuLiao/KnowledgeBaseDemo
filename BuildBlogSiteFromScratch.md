---
date: '2025-03-04T22:42:08-08:00'
title: 'Deploy Hugo Blog From Scratch'
---


## Getting Started

- Install [Hugo](https://gohugo.io/installation/)

- Install [Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)

- Verify that you have installed Hugo **v0.128.0** or later.
```
hugo version
```

- Create a new hugo site.
```
hugo new site MyFreshWebsite --format yaml
```

- Go to `MyFreshWebsite`
```
cd ./MyFreshWebsite
```

- Installing/Updating PaperMod
```
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
git submodule update --init --recursive # needed when you reclone your repo (submodules may not get cloned automatically)
```

- Render hugo pages
```
hugo server -D
```

- Set up default yaml config:
```
default_test_num: 539
```