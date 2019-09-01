# Local serve

Prerequisites:
 - npm/nodejs
 - Install docsify  `npm i docsify-cli -g`

To serve your local changes under [localhost:3000/#/](http://localhost:3000/#/) you will have to:

```shell
docsify serve ./
```

This will use the ./index.html and not the docs/index.html which is used for GitHub so any index.html changes will need to be ported to docs/index.html.

