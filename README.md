# ObsInt Processing Local Deployment

This repository contains files necessary to run containerized version of services 
maintained by Observability Intelligence Processing Team locally. The purpose is to provide 
an alternative to full deployment of these services in ephemeral environment and 
to make future development of these service easier.

The repository contains two directories, `external` and `internal`, both containing configuration 
files needed to run services for external data pipeline or internal data pipeline respectively.

## Prerequisites

For running the containers using this repository, you need to have:

- Docker
- Docker Compose
- Git

Both internal and external data pipeline need Ingress image.
In order to make it available locally, run:

```
git clone https://github.com/RedHatInsights/insights-ingress-go.git
cd insights-ingress-go
docker build . -t ingress:latest
```

## Deployment

In order to run services of one of the pipelines, navigate to one of the directories
and run `docker compose up -d`. You can alter the compose YAML and replace the `image`
field to substitute the service image(s) with the one containing your changes.

The rest of the YAML files in the directories are configurations needed for Python services
based on [ccx-messaging](https://github.com/RedHatInsights/insights-ccx-messaging) module.
