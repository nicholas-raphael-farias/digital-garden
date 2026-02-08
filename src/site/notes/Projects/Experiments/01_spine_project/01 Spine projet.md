---
{"dg-publish":true,"permalink":"/projects/experiments/01-spine-project/01-spine-projet/"}
---

This spine project is a first to practice devops steps

### Overview
Simple nest js api 
ci/cd with github actions
infrastructure automation with terraform


## Process

### Release strategy
There are 2 environments, dev prod, on this version im not using docker so the artifact creation process involves s3
- development: on pushes to development branch an artifact is created with the commit hash associated
- production: is attached to a github release version pushed to main, an artifact is created with the tag version associated 
### Terraform 
- environments
- modules

### Configuration steps
- setup aws credentials





## Notes

- Terraform commands