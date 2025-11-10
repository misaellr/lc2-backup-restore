# Naming Conventions

## Namespaces
- `<project>-dev`, `<project>-uat`, `<project>-prd`

## Labels (required on workloads/services)
- `app.kubernetes.io/name`
- `app.kubernetes.io/instance`
- `cockpit.liferay.io/project`
- `cockpit.liferay.io/env`
- optional: `cockpit.liferay.io/app`

## Annotations
- `service.cockpit.liferay.io/allow=true` for approved LoadBalancer services
- `external-dns.alpha.kubernetes.io/hostname=<host>` for ExternalDNS

## Hostnames
- dev: `app-dev.<project>.<root-domain>`
- uat: `app-uat.<project>.<root-domain>`
- prd: `app.<project>.<root-domain>`
