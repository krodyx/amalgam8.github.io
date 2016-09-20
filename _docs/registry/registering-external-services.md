---
layout: page
title: File-based Catalog
permalink: /docs/registry/file-based-catalog/
category: Service Registry
order: 2
---

# Registering External Services with File-based Catalog <a id="filecatalog"></a>

Amalgam8 is typically used to control communication between internal application microservices. However, routing rules may also be injected for external cloud- and Web-based services, 
such social network sites, hosted database services that are part of the cloud platform, etc.
This extends Amalgam8's benefits, such as resiliency testing, to include the full set of services used by the application. 
To enable specifying routing rules on external services, they must first be visible via the Amalgam8 registry, by use of a static (file based) catalog type.

## File-based Catalog

The File-based catalog extension allows the registry to pull static service information from a file, in addition to supporting standard registration via REST API.
This catalog is useful when you have services that do not run the Amalgam8 protocol (e.g., a database service), and allows:

- discovery of these services using the standard Amalgam8 discovery REST API.
- specifying routing rules (such as fault injection for resiliency testing) on external services

To enable this catalog type you have to provide the `--fs_catalog` command line flag and set it to the directory of the catalog configuration files (one file per namespace).
The name of the configuration file should be `<namespace>.conf`, and it contains a single JSON object holding an array of instances.
Each instance is formatted according to the registration body defined in the [API Swagger Documentation](https://amalgam8.io/api/registry).
Service instances registered using this method are persistent and do not need to periodically heartbeat the registry in order to stay valid.

_Note: If you run multiple registry containers (e.g., for high-availability), you (probably) want to put the config files in a shared volume and to configure that volume for all the registry containers._

### Configuration File Example

The below shows a sample configuration file adding two services: the first (`db`) with a single endpoint, and the second (`ext-service`) having two endpoints. 

```json
{
    "instances": [
        {
            "service_name": "db",
            "endpoint": {
                "type" : "http",
                "value": "www.xdb-service.url"
             },
             "tags"  : [ "v1", "us-central" ]
        },
        {
            "service_name": "ext-service",
            "endpoint": {
                "type" : "tcp",
                "value": "192.168.1.2:8080"
             }
        },
        {
            "service_name": "ext-service",
            "endpoint": {
                "type" : "tcp",
                "value": "192.168.1.2:8443"
             },
             "tags"  : [ "secure" ]
        }        
    ]
}
```

## Example Usage

The following code snippets show how to register and access an external database using Amalgam8.

### Registring an External Service

The instance endpoint information is added in file called `default.conf` (the file name determines the registry's namespace used, in this case the external database instance is registered in the default namespace).

The sample file shows the service credentials added as metadata to the instance. This is done for illustrative purpose only, and credentials may be provided to the calling application directly. 

```json
{
   "instances": [
       {
          "service_name": "cloudant",
          "endpoint" : {
             "type" : "https",
             "value" : "71a0c41e-6ea4-4abc-813a-5a08ad8bke3f.cloudant.com:443"
          },
          "tags" : [ "dal05" ],
          "metadata" : {
             "credentials": {
                "username": "71a0c41e-6ea4-4abc-813a-5a08ad8bke3f",
                "password": "5b0d3f2db7401917f9ba3714ba07245f213abba3ecb96632796487f0e553435c"
             }
          }
       }
    ]
}
```

The file is stored in a location accessible to the Amalgam8 registry, to allow the registry to integrate the data on startup.
For example, the following command will load catalogs from `/amalgam8/fs_catalog/` (note that we specify the path to the _directory_ containing the file, not the full file path).

`docker run amalgam8/a8-registry:latest --fs_catalog=/amalgam8/fs_catalog/`
 
Once the registry is running, we can confirm the external service is registered successfully, and a call to 

```bash
curl amalgam8-registry:8080/api/v1/instances
```

results in:

```json
{
  "instances": [
    {
      "id": "5aff5a8eefc853d1",
      "service_name": "cloudant",
      "endpoint": {
        "type": "https",
        "value": "71a0c41e-6ea4-4abc-813a-5a08ad8bke3f.cloudant.com:443"
      },
      "status": "UP",
      "metadata": {
        "credentials": {
           "username": "71a0c41e-6ea4-4abc-813a-5a08ad8bke3f",
           "password": "5b0d3f2db7401917f9ba3714ba07245f213abba3ecb96632796487f0e553435c"
        }
      },
      "last_heartbeat": "0001-01-01T00:00:00Z",
      "tags": [
        "dal05",
        "filesystem"
      ]
    }
  ]
}
```

### Accessing the External Service

Once the external service instance is registered in the Amalgam8 registry, you may specify routing rules to apply on the communication path.
However, in order for the rules to be applied, communication must flow through the sidecar's proxy. The client application must call the local service URL instead of the original URL.

The recomended way (e.g., [Twelve-Factor Apps](https://12factor.net/)) is to externalize all service URL's as configuration, allowing different service endpoints to be used in development, testing and production. 
Thus, the original URL `https://71a0c41e-6ea4-4abc-813a-5a08ad8bke3f.cloudant.com:443` should be replaced in the application configuration with `http://localhost:6379/cloudant`, matching the service name specified for the instance.
The application should still provide all required headers (e.g., authorization) when contacting the sidecar. These will be passed to the upstream service unmodified.

```bash
curl -H "Authorization: basic NWIwZDNmMmRiNzQwMTkxN2Y5YmEzNzE0YmEwNzI0NWYyMTNlYmVmM2VjYjk2NjMyNzk2NDg3ZjBlNTUzNDM1Ywo=" http://localhost:6379/cloudant/<database>/<document_id>/
```