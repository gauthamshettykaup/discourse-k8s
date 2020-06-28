# bitnami-discourse-k8s
Now using Helm 3.

This repo contains a Helm chart for standing up a Discourse instance in Kubernetes based on the bitnami images.

# Pre-requisites
Helm client 3.0.2

# Installing
## Setting up HTTPS
It is HIGHLY recommended that you enable HTTPS for Discourse. It requires authenticating to post and users may post sensitive data.
Follow the below steps:

### Request certificate
Request a certificate from your certificate authority, a public one, or Let's Encrypt

### Split and Clean Certificate
You need to split the certificate into two parts:
1. unencrypted privateKey.
2. certificate with Root Chain. Order must be: server cert, intermediate, (intermediate,) root. Cert chain must have no metadata listed... 

In MacOS, to extract a privateKey from a PEM file:
    pemFile="/path/to/file.pem"
    keyFile="/path/to/file.key"
    openssl pkey -in "$pemFile" -out "$keyFile"

In MacOS, to extract the cert, intermediate, and root from a PEM file:
    certChain="/path/to/newFile.pem"
    openssl crl2pkcs7 -nocrl -certfile "$pemFile" | openssl pkcs7 -print_certs -outform PEM -out "$certChain"`

Remove the metadata... just open the file and delete the excess junk about subject, issuer, etc.

### Add Certificate to Rancher/Kubernetes
(your project) > Resources > Secrets > Certificates
Add Certificate:
* Name: (give it a helpful name with no spaces)
* Scope: "Available to a single namespace:" (choose the namespace where discourse is installed)
* Private Key: paste in the contents of $keyFile
* CA Certificate: paste in the contents of $certChain

## Create/Update answers file
copy values.yaml, review the comments, and update the various fields.

## Install the Helm chart
    namespace="macforums-pre-prod"
    name="macforums-pp"
    answersFile="answers-dbaas-pp.yaml"
    helm install -n "$namespace" -f "$answersFile" "$name" .

## Recommended Plugins
- ldap = https://github.com/jonmbake/discourse-ldap-auth.git

## Manually installing Plugins
Connect to the discourse container:

    kubectl -n "$namespace" exec -it "discourse-0" -- /bin/bash

Install the plugin

    cd /opt/bitnami/discourse
    RAILS_ENV=production bundle exec rake plugin:install repo=$plugin
    RAILS_ENV=production bundle exec rake assets:precompile

Based on info at https://github.com/bitnami/bitnami-docker-discourse/issues/28


# Deleting a Helm instance
If you've been playing with the Helm chart, you may need to erase the existing Helm chart and volumes.

    helm del --namespace "$namespace" "$name"

In some cases, you may want to setup Discourse completely clean, if so:
Then go into Rancher and delete the volumes for that helm chart.

# Items to do
- [x] Configure Discourse to connect to DBaaS postgresql instance
- [x] Determine how to add plug-ins to Discourse (and document)
- [x] Add LDAP plug-in
- [x] Async tasks don't work. Sidekiq needs access to the same volume as Discourse... (taken care without having same volume)
- [ ] Determine better way to add plug-ins so we don't have to do it every time.
- [ ] Enable HTTPS for Discourse (use macforums-pp.intel.com as the url)
- [ ] Backup MacForums postgresql database, and determine how to import it into DBaaS.
- [ ] Check if deploying bitnami discourse upgrades MacForums properly

# Miscellaneous
## Using TLS

## Initializing Helm
Mac:
    brew install helm
Windows:
    

Windows Proxy Environment Variables

    set HTTP_PROXY=http://proxy-dmz.intel.com:911
    set HTTPS_PROXY=http://proxy-dmz.intel.com:912
    set NO_PROXY=localhost,intel.com,192.168.0.0/16,172.16.0.0/12,127.0.0.0/8,10.0.0.0/8

    
## Creating Admin account under discourse using console

    cd /opt/bitnami/discourse
    RAILS_ENV=production bundle exec rails c

    u = User.create!(username: "gauthamshetty", email: "gauthamx.shetty@intel.com", password: "root_bitnami1")
    u.admin = true
    u.approved = true
    u.activate
    u.save!
    u = User.find_by_username("gauthamshetty")
    EmailToken.confirm(u.email_tokens.first.token)
    
## Discourse logs

    /opt/bitnami/discourse/logs# tail production.log 
    
## Enable local logins

    SiteSetting.enable_local_logins = true  
    => Requires App restart
