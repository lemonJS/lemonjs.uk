# lemonJS Blog

### Requirements
- Hugo (`brew install hugo`)

### Running the dev server
```
$ hugo server -D
```

### Deploying
```
$ hugo -D
$ aws s3 sync ./public s3://lemonjs.uk --delete
$ aws cloudfront create-invalidation --distribution-id EFFM08EAF103G --paths "/*"
```
