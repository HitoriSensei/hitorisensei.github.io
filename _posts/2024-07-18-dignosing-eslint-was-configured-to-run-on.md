---
layout: post
title:  "ESLint with TypeScript and symlinked directories don't play well together"
date:   2024-07-18 12:00:00 -0400
cover-img: https://i.ibb.co/48DZ4Yx/vividwarthog-diagonal-chain-icon-with-3-big-loops-red-exclamati-97821cab-f060-4410-9e06-02d2a09f441a.png
categories: 
  - programming
---

I recently ran into an issue where ESLint was throwing a mysterious error when I tried to run it on a project.


# The issue

The error was:

```

Parsing error: ESLint was configured to run on `/Users/piortbosak/Projekty/blahblahblabh/somefile.ts` using `parserOptions.project`: <tsconfigRootDir>/tsconfig.eslint.json
However, that TSConfig does not include this file. Either:
- Change ESLint's list of included files to not include this file
- Change that TSConfig to include this file
- Create a new TSConfig that includes this file and include it in your parserOptions.project
```

# TL;DR

Check if your project root isn't located in a symlinked directory. Symlinked directories can cause issues with TypeScript and ESLint.

# Investigating the issue

The project was indeed using TypeScript.

* I've checked the `tsconfig.eslint.json` file and it was correctly configured to include the file that was causing the error, so nothing was wrong there.

* I've also checked the `eslint` configuration and it was also correctly configured to use the `tsconfig.eslint.json` file and no exclude any files.

Having exhausted all the obvious options, I started to dig deeper.

# The solution

After some time, I've noticed that the project was located in a symlinked directory:

```
/Users/piortbosak/Projekty -> /Volumes/External/Projekty
```

As this was the only thing that was different from other projects, I've decided to move the project to a non-symlinked directory and voila! The error was gone.

Thesis: It seems that either TypeScript or ESLint resolve the symlinked file to the original path and when inclusion/exclusion checks are performed, they are done on the original path, not the symlinked one.

The topic is interesting and I'll try to dig deeper into it in the future.

