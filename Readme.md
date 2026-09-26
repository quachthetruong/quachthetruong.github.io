prod: hugo
local: hugo server

Enable the pre-push hook after cloning:

```sh
git config core.hooksPath .githooks
```

The hook runs `hugo` before each push and blocks the push if the build fails.
Generated changes in `docs/` still need to be committed to be included in a push.

hugo-PaperMod theme currently having a problem with hugo version 0.146,
i need to install raw code https://github.com/msfjarvis/hugo-PaperMod/commit/5e7d4f3dd28cca7d4fe0f9fff8fb525bede7bec0
