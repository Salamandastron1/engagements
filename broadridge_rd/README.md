**README**

This repository has the install and upgrade concourse pipelines for PAS.


## To upgrade PAS

1. Bump the version of PAS in the download pipeline at
   `pipelines/download/download-product-configs/pas.yml`
2. Trigger the `fetch-pas` job in the `download-to-s3` pipeline
3. Bump the version of PAS in the Sandbox pipeline at `sbx/pas/version.yml`
4. Trigger the `upload-and-stage-pas` job in the `opsman-pas` pipeline
5. Trigger the `configure-pas` job in the `opsman-pas` pipeline

Most of the following is problem and environment specific. Please refer to Pivotal's platform automation docs for a guide to pipelines.

- [Platform Automation Documentation](http://docs.pivotal.io/platform-automation/v3.0/index.html)

# Changes made for Sandbox in Ops Manager
1. Flipped all post install errands to default
    - all errands = `default`
1. In PAS tile 
    - `Metric Registrar` = enabled
1. 
1. Then run apply changes

## P-Automator and Setting Explicit Proxies

The current version of Pivotal's tooling on the Platform automation docker image (P-Automator) requires a special work around when using proxies. Of note is this code:

```shell
    cat > /usr/local/sbin/aws <<EOF
    #!/bin/sh

    export http_proxy='PROXY.IP.ADDRESS:PORT/
    export https_proxy=PROXY.IP.ADDRESS:PORT/
    export no_proxy=0,1,2,3,4,5,6,7,8,9,opsman.pas-sbx.bfsaws.net,pcf.pivotal.pcfdev.bfsaws.net,opsman.pks-sbx.bfsaws.net,opsman.pas-dev.bfsaws.net,opsman.pas-uat.bfsaws.net
    /usr/local/bin/aws "\$@"
    EOF
    chmod +x /usr/local/sbin/aws
```

The code above is included in the entire Ops Manager install task. TLDR; it will set proxy variables explicity for P-Automator to use. Below is the entire task including that code.


```shell
run:
  path: bash
  args:
  - "-c"
  - |
    cat /var/version && echo ""
    p-automator -v
    set -eux

    vars_files_args=("")
    for vf in ${VARS_FILES}
    do
      vars_files_args+=("--vars-file ${vf}")
    done

    generated_state_path="generated-state/$(basename "$STATE_FILE")"
    cp "state/$STATE_FILE" "$generated_state_path"

    cat > /usr/local/sbin/aws <<EOF
    #!/bin/sh

    export http_proxy='PROXY.IP.ADDRESS:PORT/
    export https_proxy=PROXY.IP.ADDRESS:PORT/
    export no_proxy=0,1,2,3,4,5,6,7,8,9,opsman.pas-sbx.bfsaws.net,pcf.pivotal.pcfdev.bfsaws.net,opsman.pks-sbx.bfsaws.net,opsman.pas-dev.bfsaws.net,opsman.pas-uat.bfsaws.net
    /usr/local/bin/aws "\$@"
    EOF
    chmod +x /usr/local/sbin/aws

    # ${vars_files_args[@] needs to be globbed to split properly (SC2068)
    # INSTALLATION_FILE needs to be globbed (SC2086)
    # shellcheck disable=SC2068,SC2086
    p-automator upgrade-opsman \
    --config config/"${OPSMAN_CONFIG_FILE}" \
    --env-file env/"${ENV_FILE}" \
    --image-file "$(find image/*.{yml,ova,raw} | head -n1)"  \
    --state-file "$generated_state_path" \
    --installation installation/$INSTALLATION_FILE \
    ${vars_files_args[@]}
```

## Some Handy Hacks

### To add secrets to CredHUb, first configure the `credhub` CLI to point to the Control Plane CredHub:

```shell
export CREDHUB_CA_CERT="$(cat /tmp/your-control-plane-environment-cert)"
export CREDHUB_CLIENT='credhub_admin_client'
export CREDHUB_SERVER='https://10.146.132.6:8844'
#This command protects your secrets
read -s -p 'Concourse CredHub secret:' CREDHUB_SECRET; echo; export CREDHUB_SECRET
```

### Adding CredHub secrets to allow Platform Automation to talk to CredHub:

```shell
credhub set --type value --value "$CREDHUB_SERVER" --name /concourse/uat/credhub-server

credhub set --type value --value "$CREDHUB_CLIENT" --name /concourse/uat/credhub-client

credhub set --type value --value "$CREDHUB_SECRET" --name /concourse/uat/credhub-secret
```

### Adding PAS secrets for each foundation:

```shell
credhub set --type value --value 'ops.manager.url.string.io' --name /concourse/uat/opsman_url

credhub set --type value --value 'admin' --name /concourse/uat/opsman_username
read -s -p 'Ops Manager password:' OM_PASSWORD; echo

credhub set --type value --value "$OM_PASSWORD" --name /concourse/uat/opsman_password

credhub set --type value --value "$OM_PASSWORD" --name /concourse/uat/opsman_decryption_passphrase
```

### To copy credentials from one CredHub path to another with a loop.

- requires manual input of secret

```shell
for cred_name in $(credhub find | grep '/concourse/main/s3' | cut -f 3 -d ' '); do credhub get --name "$cred_name"; credhub set --type value --name "${cred_name/main/sbx}"; done
```
### Looking for AWS S3 resources w/ a loop
- can give a large output, recommended for smaller batches of buckets
```
for i in $(aws --profile cp-uat s3 ls | awk '{print $3}'); 
    do echo $i; aws --profile cp-uat s3 ls $i; 
done | less
```

To get some additional debug info out of the `aws` command when using it through `p-automator`, we experimented with the following `/usr/local/sbin/aws` shim:

```shell
#!/bin/sh

export http_proxy=http://10.26.27.235:3128/
export https_proxy=http://10.26.27.235:3128/
export no_proxy=0,1,2,3,4,5,6,7,8,9,opsman.pas-sbx.bfsaws.net,pcf.pivotal.pcfdev.bfsaws.net,opsman.pks-sbx.bfsaws.net,opsman.pas-dev.bfsaws.net,opsman.pas-uat.bfsaws.net

/usr/local/bin/aws "$@" >/scratch/aws.out
AWS_EXIT=$?
cat /scratch/aws.out
exit $AWS_EXIT
```


If for reasons, you need to disable Ops Manager's IaaS verifiers:

```shell
foundation=sbx
for verifier in \
    'IaasConfigurationVerifier' \
    'AvailabilityZonesVerifier' \
    'NetworksExistenceVerifier'
do
    om --env env/$foundation/env/env.yml curl \
        -p "/api/v0/staged/director/verifiers/install_time/$verifier" \
        -x PUT \
        -d '{ "enabled": false }' \
        -H 'Content-Type: application/json'
done
```

### Setting a fly script

```shell
#!/bin/sh

PIPELINE='opsman-pas'

FOUNDATION="${1:-sbx}"
TEAM="$FOUNDATION"
TARGET="$FOUNDATION"

if [ "$FOUNDATION" = 'sbx' ]; then
    CONCOURSE_URL='https://10.47.21.6/'
elif [ "$FOUNDATION" = 'uat' ]; then
    CONCOURSE_URL='https://10.146.132.6'
else
    echo "Unknown foundation $FOUNDATION, exiting" >&2
    exit 1
fi

fly_target_exists() {
    local target="$1"

    fly targets | grep "^$target "
}

fly_logged_into_target() {
    local target="$1"

    fly --target  "$target" status
}

if ! fly_target_exists "$TARGET"; then
    fly --target "$TARGET" login \
        --concourse-url "$CONCOURSE_URL" \
        --team-name "$TEAM" \
        --insecure \
        --open-browser
fi

if ! fly_logged_into_target "$TARGET"; then
    fly --target "$TARGET" login \
        --concourse-url "$CONCOURSE_URL" \
        --team-name "$TEAM" \
        --insecure \
        --open-browser
fi

if ! fly --target "$TARGET" teams | grep "$TEAM"; then
    fly --target "$TARGET" set-team \
        --team-name "$TEAM" \
        --local-user admin
fi

fly --target "$TARGET" set-pipeline \
    --pipeline "$PIPELINE" \
    --config ./"$PIPELINE.yml" \
    --var foundation="$FOUNDATION"
```

### Setting Environment Variables for OM cli and Credhub Cli

- In general saving this to an `.envrc` file is preferred for reproducibility. 
```shell
if [ -z "$OM_TARGET" ]; then
    export OM_TARGET=pcf.pivotal.pcfdev.bfsaws.net
fi
if [ -z "$OM_USERNAME" ]; then
    export OM_USERNAME='admin'
fi
if [ -z "$OM_SKIP_SSL_VALIDATION" ]; then
    export OM_SKIP_SSL_VALIDATION=true
fi
if [ -z "$OM_PRIVATE_KEY_PATH" ]; then
    export OM_PRIVATE_KEY_PATH="$HOME/.ssh/id_rsa-pas-sbx"
fi
if [ -z "$OM_PASSWORD" ]; then
    read -s -p "$OM_USERNAME@$OM_TARGET password:" OM_PASSWORD; echo; export OM_PASSWORD
fi

export CREDHUB_SERVER='https://10.47.21.6:8844'
export CREDHUB_CLIENT='credhub_admin_client'
export CREDHUB_CA_CERT="$(cat /tmp/cp-sbx-ca-cert )"
if [ -z "$CREDHUB_SECRET" ]; then
    read -s -p 'Concourse CredHub secret:' CREDHUB_SECRET; echo; export CREDHUB_SECRET
fi
```
