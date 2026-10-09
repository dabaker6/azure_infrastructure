# THIS IS ARCHIVED TERRAFORM IS THE SUPPORTED METHOD FOR CREATING THE REQUIRED INFRASTRUCTURE

# Scripts to create Azure resources 

## Resource Group

### resource-group.ps1

Creates an azure resource group.

Requires a config.psd1 file with the following structure
```
@{
    RESOURCE_GROUP = ""
    LOCATION = ""
}
```

## Cosmos DB

### cosmos-db.ps1

Creates cosmos db database and container.

Requires a config.psd1 file with the following structure
```
@{
    ACCOUNT_NAME = ""
    RESOURCE_GROUP = ""
    DATABASE_NAME = ""
    CONTAINER_NAME = ""
    APP_NAME = ""
    RBAC_ROLE_NAME = ""
}
```

### cosmos-db-access.ps1

Authentication is through RBAC control, with the API given readonly access. The same config file is used.

If local debugging is required then the default azure credential can be used for authentication, but the user must be added to RBAC with the same read permission.

For any write access the `Cosmos DB Built-in Data Reader` access level should be used

Data reader role id is `00000000-0000-0000-0000-000000000001`
Date contributor role is `00000000-0000-0000-0000-000000000002`

## VNET

### vnet.ps1

Creation of vnet containing two subnets for webapp and api. The api will then only accept requests from the flask subnet.

Requires a config.psd1 file with the following structure
```
@{
    VNET_NAME = ""
    FLASK_SUBNET_NAME = ""
    API_SUBNET_NAME = ""
    RESOURCE_GROUP = ""
    FLASK_APP_NAME = ""
    API_APP_NAME = ""
}
```

## ACR

### create-acr.ps1

Creates a new azure container registry

### build-push.ps1

Builds, pushes and run a quick task to confirm container health. The script opens a file picker, where the folder containing the docker file is located.

Requires a config.psd1 file with the following structure
```
@{
    ACR_NAME = ""
    RESOURCE_GROUP = ""
    IMAGE_TAG = ""
    IMAGE_NAME = ""
}
```

## Deploy

### deploy-container-app-service

If required creates an Azure App Service, then creates a webapp, enables managed identity for the webapp and grants it AcrPull, then sets the webapp container settings to point to ACR & a specific image.

Requires a config.psd1 file with the following structure

```
@{
    ACR_NAME = ""
    RESOURCE_GROUP = ""
    LOCATION = ""
    IMAGE_TAG = ""
    IMAGE_NAME = ""    
    ASP_NAME = ""
    ASP_SKU = ""
    FLASK_APP_NAME = ""    
}
```

### Notes

Followed for naming conventions:

https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-abbreviations

Important to gunicorn to requirements.txt

## Terraform

ASP, webapp, acr, rg, vnet, api

- Tricky to get scope correct for cosmosdb - was not just db/mydb/colls/mycoll, needed the account name in front
- private endpoint considered for service bus, but cost not worth it.
- lack of docs for ip_restriction?
- separate folders 
- DNS zones - create the main zone, rest will be manual as this is usually an infrequent change
- OIDC for github, need to assign roles. MS helpfully state assign the appropriate role (https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure-openid-connect), whatever the yml has, for webapps need ACR push, pull and website contributor, for container apps need ACR push, pull and website contributor. So:
OIDC MI needs ACR push as github builds then pushes the image to ACR
each servie, webapp, container etc... needs pull to get the image
OICE MI then needs deploy, restart etc... if needed
- ACR tasks build vs github runner, for least privlages github good as only need ACRPush

COntainer apps - appear to have service bus data receiver twice