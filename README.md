


```
ytt -f bundle/config -f bundle/schema.yaml -f example/values.yaml
```


```
VERSION=0.0.1
kbld -f bundle/config --imgpkg-lock-output bundle/.imgpkg/images.yml
imgpkg push -b ghcr.io/making/rbac-mgmt-bundle:${VERSION} -f bundle
```

```

```