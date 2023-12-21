# Common Proxy Configuration Langauge

## Introduction
The Common Proxy Configuration Language allows for describing the configuration of the external entities connected to an AAI proxy, typically based on SAML or OIDC, independently from the proxy product used to implement the proxy.

Using CPCL one may describe how incoming and outgoing identifiers and credentials should be mapped between two entities connected to the proxy, to allow for a consistent flow. CPCL uses a YAML based format to describe flows on a per flow basis.

## Main configuation
The general setup of the consists of 3 section: in, processor and out

### IN
This section describes the configuration parameters of the entity or entities connected to the proxy which are the source of incoming credentials. Depending on the protocol used to provide the credentials, a protocol specific section is used to describe the configuration of the remote party. 
CPCL provides support for using one and only one incoming protocol per configuration. If none or multiple incoming protocol configurations are described, the configuration MUST be rejected as faulty and an error MUST must be thrown.
*TODO: define common errors?*

#### SAML IDP
To describe the configuration of a SAML IdP delivering attributes and identifiers, as "saml_idp" section MAY be defined.
This section describs the required and optional parameters for the "saml_idp" configuration. If a required parameter is empty or missing, the configuration MUST be rejected as faulty and an error MUST must be thrown. 
To establish a secure connection between the saml_idp entity and the proxy it may be required to validate the entity metadata using a trusted third party. This is out of scope for CPCL. 

| Parameter | Status | Supported Values | Description |
| --- | --- | --- | --- |
| name | optional | free text | The name of the entity described in the configuration. The implementation MAY decide to use a name for the entity if so provided in the entity metadata. If no name is provided in the entity metadata, the implementation SHOULD use the name provided in this parameter |
| description | optional | free text | A description of the entity described in the configuration. |
| entityid | required | URI | entityid of the entity described in the configuration. |
|metadata_url| required | URL | The URL of the SAML metadata containing either an entity descriptor or an entities descriptor containing metadata for the entity with the entityid configured above.| 

#### Non normative example

```
---
in: # This side of the proxy recieves the credentials from the resource configured below
  saml_idp: # A configuration for a SAML IdP
     name: "SURF bv"
     description: “Connect to the SURF Identity provider”
     entityid: "http://login.surf.nl/adfs/services/trust"
     metadata_url: "https://metadata.surfconext.nl/idps-metadata.xml"
processor:
    ...
out:
    ...
```

#### SAML IDPs
To describe the configuration of multiple SAML IdPs delivering attributes and identifiers, a "saml_idps" section MAY be defined.
This section describs the required and optional parameters for the "saml_idps" configuration. If a required parameter is empty or missing, the configuration MUST be rejected as faulty and an error MUST must be thrown.
For proper user flow in this scenario, it is assumed the proxy implements a discovery interface to allow the user to select the correct SAML identity provider from the set of available identity providers.  
The entiteis descriptor provided as part of this configuration MAY include none IdP entity types. The proxy MUST ignore such entities.
To establish a secure connection between the saml_idps and the proxy it may be required to validate the entities metadata using a trusted third party. This is out of scope for CPCL. 

| Parameter | Status | Supported Values | Description |
| --- | --- | --- | --- |
| name | optional | free text | The name of the (group of) entities described in the configuration. |
| description | optional | free text | A description of the (group of) entities described in the configuration. |
|metadata_url| required | URL | The URL of the SAML metadata containing an entities descriptor containing at least one SAML IdP.| 

#### Non normative example

```
---
in: # This side of the proxy recieves the credentials from the resource configured below
  saml_idps: # A configuration for a group of SAML IdPs
     name: "SURFconext Federatie"
     description: “All IdPs from SURFconext Federatie”
     metadata_url: "https://metadata.surfconext.nl/idps-metadata.xml"
processor:
    ...
out:
    ...
```

#### OIDC OP
To describe the configuration of an OpenID Connect OP delivering claims and identifiers, an "oidc_op" section MAY be defined.
This section describs the required and optional parameters for the "oidc_op" configuration. If a required parameter is empty or missing, the configuration MUST be rejected as faulty and an error MUST must be thrown. 
To establish a connection between the oidc_op entity and the proxy it may be required to register the proxy as a client to the OIDC OP. This is out of scope for CPCL. 

| Parameter | Status | Supported Values | Description |
| --- | --- | --- | --- |
| name | optional | free text | The name of the entity described in the configuration. The implementation MAY decide to use a name for the entity if so provided in the entity metadata. If no name is provided in the entity metadata, the implementation SHOULD use the name provided in this parameter |
| description | optional | free text | A description of the entity described in the configuration. |
|discovery_url| required | URL | The URL of the discovery interface of the OP of the '.well-known' directory, including the the '.well-known' path.| 

#### Non normative example

```
---
in: # This side of the proxy recieves the credentials from the resource configured below
  oidc_op: # A configuration for an OIDC OP
     name: "SURFconext Test"
     description: “Connect to the SURFconext test OP”
     discovery_url: "https://connect.test.surfconext.nl/.well-known/openid-configuration"
processor:
    ...
out:
    ...
```

### Rules
The rules section of the configuration describes how the proxy should internally process the conversion of identifiers or credentials from the IN side to the OUT side. 

As a general formatting rule in the rules section, an incoming credential is prefixed with "in.", while an outgoing credential is prefixed with "out.". Both prefixes MUST NOT be part of credentials exiting the proxy. 

Multiple rules may operate on the same in or outgoing credential.

Rules may be nested to combine multiple operations on a credentials. In case nesting is detected, the proxy MUST prcess the rules starting at the innermost rule.

To support CPCL, the proxy MUST implement the following processing rules. All values provided to the rule operators are 

| Rule | IN | OUT | Description | 
| --- | --- | ---| --- |
| pass | in.foo | out.bar | The value of the one or more incoming credentials is copied into the outgoing credential| 
| pass | in.foo in.bar | out.foobar | place either incoming credential value in outgoing credential value. If first credential is matched, others MUST be ignored.|
| passe | in.foo in.bar | out.foobar | Place incoming credential values in into outgoing credential value as an enumeration.|
| passo | in.foo in.bar| out.foobar | Place incoming credential value in preference of order into outgoing credential value.|
| concat | in.foo in.bar 'stringvalue' | out.foobar | Concatenate 1 or more incoming credentials using an optional seperator. of the value provided is not prefixed with either 'in.' or 'out.' a literal string value is assumed |
| split | in.foobar seperator | out.foo out.bar | Split an incoming credentials using the first occurance of the seperator|
| triml | in.foobar len | out.obar | Trim the incoming credential startign fromthe leftmost character for length 'len'|
| trimr | in.foobar len | out.foob | Trim the incoming credential startign fromthe rightmost character for length 'len'|
| regexp | in.foobar regexp | out.raboof | Perform a regular experession 'regexp' on the incoming credential |


ToDo: add examples
```
rules:

    pass # place incoming credential value in outgoing credential value
        in.foo
        out.bar

    pass # 
        in.foo
        in.bar
        out.foobar

    passe # place incoming credential values in into outgoing credential value as an enumeration.
        in.foo
        in.bar
        out.foobar

    passo # place incoming credential value in preference of order into outgoing credential value. 
        in.foo
        in.bar
        out.foobar

    concat # Concatenate 1 or more incoming credentials using an optional seperator. of the value provided is not prefixed with either 'in.' or 'out.' a literal string value is assumed 
        in.foo 
        in.bar 
        "stringvalue"
        out.foobar
        
    split # Split an incoming credentials using the first occurance of the seperator 
        in.foobar
        seperator
        out.foo
        out.bar

    triml # Trim the incoming credential startign fromthe leftmost character for length 'len'
        in.foobar
        len
        out.obar

    trimr # Trim the incoming credential startign fromthe rightmost character for length 'len'
        in.foobar
        len
        out.foob

    regexp # perform a regular experession on the incoming credential
        in.foobar
        regexp
        out.raboof
``` 

### OUT
This section describes the configuration parameters of the entity or entities connected to the proxy which are the source of incoming credentials. Depending on the protocol used to provide the credentials, a protocol specific section is used to describe the configuration of the remote party. 
CPCL provides support for using one and only one incoming protocol per configuration. If none or multiple incoming protocol configurations are described, the configuration MUST be rejected as faulty and an error MUST must be thrown.
*TODO: define common errors?*

#### SAML SP
To describe the configuration of a SAML SP recieving attributes and identifiers, a "saml_sp" section MAY be defined.
This section describs the required and optional parameters for the "saml_sp" configuration. If a required parameter is empty or missing, the configuration MUST be rejected as faulty and an error MUST must be thrown. 
To establish a secure connection between the saml_sp entity and the proxy it may be required to validate the entity metadata using a trusted third party. This is out of scope for CPCL. 

| Parameter | Status | Supported Values | Description |
| --- | --- | --- | --- |
| name | optional | free text | The name of the entity described in the configuration. The implementation MAY decide to use a name for the entity if so provided in the entity metadata. If no name is provided in the entity metadata, the implementation SHOULD use the name provided in this parameter |
| description | optional | free text | A description of the entity described in the configuration. |
| entityid | required | URI | entityid of the entity described in the configuration. |
|metadata_url| required | URL | The URL of the SAML metadata containing either an entity descriptor or an entities descriptor containing metadata for the entity with the entityid configured above.| 

#### Non normative example

```
---
in:
    ...
processor:
    ...
out: # This side of the proxy sends the credentials from the resource configured below
  saml_sp: # A configuration for a SAML SP
     name: "InAcademia Affiliation Validation Service"
     description: “Connect to InAcademia”
     entityid: "https://inacademia.org/metadata/inacademia-simple-validation.xml"
     metadata_url: "https://inacademia.org/metadata/inacademia-simple-validation.xml"
```

#### SAML SPs
To describe the configuration of multiple SAML SPs recieving attributes and identifiers, a "saml_sps" section MAY be defined.
This section describs the required and optional parameters for the "saml_sps" configuration. If a required parameter is empty or missing, the configuration MUST be rejected as faulty and an error MUST must be thrown.
Please note that in this scenario each and every SP will recieve the same set of attributes from the proxy.  
The entities descriptor provided as part of this configuration MAY include none SP entity types. The proxy MUST ignore such entities.
To establish a secure connection between the saml_sps and the proxy it may be required to validate the entities metadata using a trusted third party. This is out of scope for CPCL. 

| Parameter | Status | Supported Values | Description |
| --- | --- | --- | --- |
| name | optional | free text | The name of the (group of) entities described in the configuration. |
| description | optional | free text | A description of the (group of) entities described in the configuration. |
|metadata_url| required | URL | The URL of the SAML metadata containing an entities descriptor containing at least one SAML IdP.| 

#### Non normative example

```
---
in:
    ...
processor:
    ...
out: # This side of the proxy sends the credentials to the resources configured below
  saml_sps: # A configuration for a group of SAML SPs
     name: "FooBar Collaboration Services"
     description: “All SPs from the FooBar Collaboration”
     metadata_url: "https://foobar.example.org/sps.xml"
```

#### OIDC RP
To describe the configuration of an OpenID Connect RP recieving claims and identifiers, an "oidc_rp" section MAY be defined.
This section describs the required and optional parameters for the "oidc_rp" configuration. If a required parameter is empty or missing, the configuration MUST be rejected as faulty and an error MUST must be thrown. 
To establish a connection between the oidc_op entity and the proxy it may be required to register the RP as a client to the proxy. This is out of scope for CPCL. 

| Parameter | Status | Supported Values | Description |
| --- | --- | --- | --- |
| name | optional | free text | The name of the entity described in the configuration. The implementation MAY decide to use a name for the entity if so provided in the entity metadata. If no name is provided in the entity metadata, the implementation SHOULD use the name provided in this parameter |
| description | optional | free text | A description of the entity described in the configuration. |
| client_id| optional | free text | The client id of the RP.| 
| client_secret| optional | free text | The client secret of the RP.|
| redirect_uri| optional | URI | The redirect URI of the RP.| 
| dynamic_registration| optional | boolean | Should dynamic registration be allowed for this RP.| 

#### Non normative example

```
---
in:
    ...
processor:
    ...
out: # This side of the proxy recieves the credentials from the resource configured below
  oidc_op: # A configuration for an OIDC OP
     name: "FooBar RP"
     description: “Connect to the FooBar RP”
     client_id: "ae45e5a4-288b-4733-b1a8-82aa0510853b"
     client_secret: "******************"
     redirect_uri: "https://foobar.example.org/callback.php"
     dynamic_registration: false
```
