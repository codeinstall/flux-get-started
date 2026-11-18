```
"uri-js": "npm:uri-js-replace@1.0.2",
"picomatch": "npm:@ryotsu/picomatch@4.0.7-a"
```

```
npm view uri-js-replace@1.0.2 version --registry=https://nexus.nonprod.dwpcloud.uk/repository/npm/
npm view @ryotsu/picomatch@4.0.7-a version --registry=https://nexus.nonprod.dwpcloud.uk/repository/npm/


npm install --package-lock-only --ignore-scripts
npm ci --ignore-scripts
npm ls uri-js picomatch --all
npm run build
npm run lint
npm test
```
