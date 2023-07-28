# Helm charts from github pages.
followed this guide:
https://www.opcito.com/blogs/creating-helm-repository-using-github-pages

To package a chart use `Helm package chart-name`

Make first index with `Helm repo index --url <github_repository_path> .`

Following indexes can be appended to with `helm repo index --url <github_repository_path> --merge index.yaml .`

Be aware <github_repository_path> should be the URL to the github pages page in this example
`https://simonstiil.github.io/helm`

TOTO: Use Releases:
https://docs.github.com/en/rest/releases/releases?apiVersion=2022-11-28#get-a-release-by-tag-name
https://docs.github.com/en/rest/releases/assets?apiVersion=2022-11-28#upload-a-release-asset
