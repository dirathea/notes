Tiddlywiki has capability to automatically save tiddler to git service, but the auto save output is a single html.

It will be much more useful if its output match with the node version, that could be diff easily.

Luckily, we have a way to do it via `tiddly --load` and `tiddly --savewikifolder` command

# Setup

Simply configure a GitHub action that detects the output of tiddly index.html changes, and do the unpack

```
name: Build and Deploy

on:
  push:
    branches:
      - main


jobs:
  unpack-tiddlers:
    runs-on: ubuntu-latest
    name: Unpack Tiddlers
    steps:
      - name: Checkout latest version
        uses: actions/checkout@v3
        with:
          token: ${{ secrets.PERSONAL_TOKEN }}

      - name: Setup Node
        uses: actions/setup-node@v3
        with:
          node-version: 16

      - name: Get Changed Files
        id: changed-files
        uses: tj-actions/changed-files@v23
        with:
          files: |
            wiki/index.html
        
      - name: Unpack tiddler
        if: steps.changed-files.outputs.any_changed == 'true'
        run: |
          npm install -g tiddlywiki
          rm -rf data
          tiddlywiki --load wiki/index.html --savewikifolder data

          git config --global user.name '${{ github.actor }}'
          git config --global user.email '${{ github.actor }}@users.noreply.github.com'
          git add data

          git commit -m "Update tiddlywiki"
          git push origin main
```

As you can see, there is a checks if the changes comes from the index.html (in my case under `wiki/index.html`). This condition prevent us to trapped into an infinite loop.