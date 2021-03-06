# SPID Bundle

This bundle configures Keycloak to use Italian SPID identity provider.  
SPID (Sistema Pubblico di Identità Digitale) is a system used by Public Administrations and provate subject
to authenticate a given user.

**NOTE:** in this first release we are targeting the public [SPID test server](https://demo.spid.gov.it/#/login), however you can easily change the configuration
to add new desired providers.

This PBC let **Entando 7** to be certified as SPID Service Provider

**NOTE:** installing the PBC alone is not sufficient to start the accreditation process: be sure to read the [technical
documentation](https://docs.italia.it/italia/spid/spid-regole-tecniche/it/stabile/index.html) first and then the whole [certification procedure](https://www.spid.gov.it/cos-e-spid/diventa-fornitore-di-servizi/).


## Setup (before use)

**NOTE**: the following step are intended for devOps / sysOps

### 1 - copy the provider JAR into Keycloak

Download the [JAR provider](https://github.com/entando-ps/entando-spid-integration-bundle/raw/master/extra/spid-provider.jar)

Copy the file inside the Keycloak POD with the command

for Keycloak 7.5.1.GA
```shell
kubectl cp ./bundle/extra/spid-provider.jar <KEYCLOACK_POD>:/opt/eap/standalone/deployments -n <NAMESPACE>
```

for Keycloak 15.1.1 community
```shell
kubectl cp ./bundle/extra/spid-provider.jar <KEYCLOAK_POD>:/opt/jboss/keycloak/standalone/deployments -n <NAMESPACE>
```

where:
- KEYCLOAK_POD is the Keycloak pod: the name always starts with **default-sso-in-namespace-deployment**`
- NAMESPACE is the namespace where Entando 7 is installed


### 2 - secrets creation

After importing the bundle from the Entando Hub you have to create these three secrets:  

```shell
kubectl create secret generic c0a4c5e6-sso-url --from-literal=url=<KEYCLOAK_URL> -n <NAMESPACE>
```

```shell
kubectl create secret generic c0a4c5e6-sso-admin-username --from-literal=<USERNAME> -n <NAMESPACE>
```

```shell
kubectl create secret generic c0a4c5e6-sso-admin-password --from-literal=<PASSWORD> -n <NAMESPACE>
```

### 3 - Test and validation

Access the Keycloak management interface -> Identity Provider -> **SPID** -> SPID service provider metadata, a browser page should open with the metadata: copy the URL.  
Open the [test portal](https://demo.spid.gov.it/validator#/metadata-sp-download) and copy the URL of the metadata and press Download.  
From this moment you should be able to log in using the [test credentials](https://demo.spid.gov.it/users).

## Bundle extension

**NOTE:** the following steps are for those developers that want to generate custom releases of the
bundle.

Make sure to have an updated [version of the CLI]() and to have the proper version of `kubectl` (or `oc`) installed.

### 1 - Preliminary operations

The first step is to download locally the bundle so that the devOps / sysOps can configure it.
Let's say that we want to work in a directory named `spid` in a path of choice:

```shell
mkdir spid && cd spid
```
```shell
git clone https://github.com/entando-ps/entando-spid-integration-bundle.git bundle
```

Delete all the git information

```shell
rm -fr ./bundle/.git
```

At this point you should have this directory layout

```shell
.
└── bundle
		├── descriptor.yaml
		├── extra
		│       ├── spid-logo-c-lb.png
		│       └── spid-provider.jar
		├── plugins
		│       └── spid-plugin.yaml
		└── README.md
```

Let's initialize the project with the command:

```shell
ent prj init
```
When asked `Please provide the project name (spid):` just accept the default.


Now we have to move the bundle in the company repository. To do so we use he command:

```shell
ent prj pbs-init
```

Provide all the requested information:

```shell
Please provide the URL of the publication repository: <MY_COMPANY_REPO>
Please provide the git user name: <USER>
Please provide the git email: <MAIL>
[...]
Should I enable the credentials cache for the publication of the frontend? (y/n/q) y
Expiration in seconds? (86400): 
```

The bundle has been pushed to our company repository and now it is ready to be configured.


### 2 - Copy the spid provider JAR into Keycloak

**NOTE:** this step must be performed by a devOps or a sysOps

Locate the pod containing the Keycloak installation: in a standard installation the name of the pod always start with **default-sso-in-namespace-deployment**

Copy the spid-provider.jar into the Keycloak pod with the command

```shell
kubectl cp ./bundle/extra/spid-provider.jar default-sso-in-namespace-deployment-aaabbbccc-dddee:/opt/jboss/keycloak/standalone/deployments -n <NAMESPACE>
```

where: 
 - `default-sso-in-namespace-deployment-aaabbbccc-dddee` is the name of the Keycloak pod
 - <NAMESPACE> is the namespace where Entando 7 is installed

You have to wait a few instants to let Keycloak sense the new provider and install it.  
The result of this operation is to add a new identity
provider, **SPID**, to the list of those already available. This provider will be configured automatically when the bundle is installed.
For this reason installing the bundle without these preliminary step will result in an error.

### 3 - Obtain the bundle ID

The bundle ID is necessary for the configuration operations. Let's get ot with the command
(don't forget the trailing .git!):

```shell
ent prj get-bundle-id --auto --repo=<MY_COMPANY_REPO>
<BUNDLE_ID>
```

The output of this command <BUNDLE_ID> is an alphanumeric string like eg. `ad14e819`

### 4 - Creation of the secrets [(more info)](https://developer.entando.com/next/tutorials/devops/plugin-environment-variables.html)

**NOTE:** this step must be performed by a devOps or a sysOps  
**NOTE:** the creation of the secrets must be done only once and repeated only when the <MY_COMPANY_REPO> changes.

The bundle always expects three secrets to be present: **their names must always start with the bundle ID**.   
If the bundle ID is `ad14e819` then the following names are expected:

| secret name                     | Scope    | description                                                       |
|---------------------------------|----------|-------------------------------------------------------------------|
| **ad14e819**-sso-url            | URL      | public address of the Keycloak instance to configure              |
| **ad14e819sso**-admin-username  | USERNAME | username of the Keycloak account used to perform setup operations |
| **ad14e819**-sso-admin-password | PASSWORD | password                                                          |


**NOTE:** a mismatch will cause an installation error.

The secrets are created with the following commands:

```shell
kubectl create secret generic <BUNDLE_ID>-sso-url --from-literal=url=<KEYCLOAK_URL> -n <NAMESPACE>
```

Create the secret for username and password with the following commands

```shell
kubectl create secret generic <BUNDLE_ID>-sso-admin-username --from-literal=username=<USERNAME> -n <NAMESPACE>
```

```shell
kubectl create secret generic <BUNDLE_ID>-sso-admin-password --from-literal=password=<PASSWORD> -n <NAMESPACE>
```

where:
  - NAMESPACE is the namespace where Entando 7 is installed
  - USERNAME and PASSWORD the values of the Keycloak account
  - BUNDLE_ID is the bundle ID found in the previous step


### 5 - Configure the bundle

In the last step the developer makes sure the plugin deployer file is well configured, typically **making sure that the correct
secrets are referenced**.

With respect to the last example (bundle id `**ad14e819**`) the plugin descriptor in `../spid/plugins/spid-plugin.yaml`
should be similar to the following:

```yaml
descriptorVersion: v4
image: entandopsdh/spid-bundle:0.1.0
dbms: postgresql
healthCheckPath: "/management/health"
roles:
  - "spid-admin"
ingressPath: "/spid"
permissions:
    - clientId: realm-management
      role: manage-users
    - clientId: realm-management
      role: view-users
environmentVariables:
    - name: SPID_CONFIG_ACTIVE
      value: "true"
    - name: KEYCLOACK_HOST
      valueFrom:
          secretKeyRef:
              name: ad14e819-sso-url
              key: url
    - name: KEYCLOACK_USERNAME
      valueFrom:
          secretKeyRef:
              name: ad14e819-sso-admin-username
              key: username
    - name: KEYCLOACK_PASSWORD
      valueFrom:
          secretKeyRef:
              name: ad14e819-sso-admin-password
              key: password
```

### 6 - Publication of the bundle

From the `spid` directory let's run the command:

```shell
ent prj pbs-publish
```

When asked:

`Please provide the bundle version number:` type 0.0.1
`Please provide the bundle version comment (0.0.1  - 2022-06-16T09:50:32+0000):` First release for <MY_COMPANY>

Finally, when the push is complete

```shell
ent prj deploy
```

It's now possible to install the bundle from the App Builder -> Repository -> SPID bundle
