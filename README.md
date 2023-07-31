# Helm charts from github pages.
started out by following this guide:
https://www.opcito.com/blogs/creating-helm-repository-using-github-pages

To package a chart use `Helm package chart-name`

Make first index with `Helm repo index --url <github_repository_path> .`

Following indexes can be appended to with `helm repo index --url <github_repository_path> --merge index.yaml .`

Be aware <github_repository_path> should be the URL to the github pages page in this example
[https://simonstiil.github.io/helm](https://simonstiil.github.io/helm)

This was then packaged into a Jenkinsfile to automate the whole process.

From main branch a release is triggered using `#release [chart-name] [chart-version]`