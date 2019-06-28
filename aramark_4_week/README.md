# Aramark Platform Team Artifacts

**DRAFT**

Initially, set up central management:

1. Build a [jump box][jumpbox]
2. Use that jump box to deploy a [Management BOSH][mgmt-bosh-config]
3. Use that BOSH to deploy a [Concourse][concourse]

If you need to access secrets managed by Concourse (like Azure
credentials or PCF admin private keys), you can configure your shell to
[target Concourse's CredHub][target_concourse_credhub].

Then, for each PCF foundation, use that Concourse to: 

1. [Install PCF][pcf-pipelines-aramark] (includes [runtime config for Log Analytics][ALA])
2. Configure [orgs and spaces][Orgs-Spaces]
3. [Install HealthWatch][install-healthwatch]
4. [Install the Log Analytics nozzle][install-log-analytics-nozzle] to send *app* logs to ALA

[jumpbox]: https://github.com/pivotalservices/aramark/tree/master/jumpbox/doc/adr
[mgmt-bosh-config]: https://github.com/pivotalservices/aramark/tree/master/mgmt-bosh-config
[concourse]: https://github.com/pivotalservices/aramark/tree/master/concourse
[target_concourse_credhub]: https://github.com/pivotalservices/aramark/blob/master/target_concourse_credhub.sh
[pcf-pipelines-aramark]: https://github.com/pivotalservices/pcf-pipelines-aramark/
[ALA]: https://github.com/pivotalservices/pcf-pipelines-aramark/blob/master/update-runtime-config-ops.yml
[install-healthwatch]: https://github.com/pivotalservices/aramark/tree/master/install-healthwatch
[Orgs-Spaces]: https://github.com/pivotalservices/aramark/tree/master/Orgs-Spaces
[install-log-analytics-nozzle]: https://github.com/pivotalservices/aramark/tree/master/install-log-analytics-nozzle
