### MTP CORE ACTIVATION

Overview

The purpose of this document is to provide a solution for China team to support MTP Core activation by activation code. The system should not be fully operational and usable before the activation procedure is performed. The UI of the activation as well as the activation procedure should be independent from the MTP Core system and we should allow different backend/UI implementation(s) to be packaged in the final installer.

### Proposed Solution

The activation functionality will rely on the licensing framework and specifically to its licensed product service. Since the licensing framework can enforce checks only on licensed features such as particular REST API services, installation of plugins and etc., but does not have a concept of enforcing licensed product for the whole system and also it does not have an redirect capability when the system is accessed by web browser and the activation is needed, because of that the enforcement of the activation is going to be performed by a new web filter that is going to rely on the licensing products to check if the license is valid and active.
1.	Obtain the licensing product

The web filter will obtain the licensed product via licensedproducts service, based on its productName and productVersion which values obtain from GenericRestConfiguration and its name and apiVersion properties (currently the productName set there is – MTP Core). If we can find such licensed product we are going to use this entity otherwise we are going to perform a query with the same productName but with apiVersion defaulted to 1.0.0. This is needed so that if we want to specify a generic license that is not impacted by system upgraded, e.g. once the system is activated and then upgraded the system should stay active. If the web filter can’t find the licensedproduct entity then the system will assume that the activation is not needed and it will not perform the activation checks from that time on.
2.	Check the license product validity

We are going to enhance the licenseproduct schema by adding a computed property called valid of type Boolean so that the licensing plugin can perform additional checks to determine if the licensing product contains valid information. For example it can check if the entitelmentId is valid based on the machine MAC address for example. Once the activation filter get the licensed product it will check if the property valid exists and if it does then it will throw MtpCoreRuntimeException if the value is false with the error code mtp.core.error.activation.license.not.valid . 
3.	Check if the activation license is activated

The web filter will then check if the licensed product is activated by comparing the availableCount and licenseCount properties. When the availableCount is less than licenseCount then the licensed product is activated.
4.	Enforce the activation check

If the licensed product is activated then the filter will cache that and on subsequent requests this filter will not perform that check anymore. If the licensed product is not activated and the HTTP request is not performed against the path /api/rest/v1 then the filter will do a redirect to /features/< adapterlicensedproducts_service_owner>/index.html. It is expected that the activation feature that provided the activation screen will own the adapterlicensedproducts. (This can be enhanced if separation of the activation backend service and UI is required). The redirect won’t be performed if the request path starts with /features/< adapterlicensedproducts_service_owner>/ - this will allow the browser to successfully download any UI artifacts that composes the activation UI. The activation UI is a standalone application and have nothing to do with the UI Console. If the requested path /api/rest/v1 and the method is GET then the filter will allow that request so that we can always check which is the version of the product and exposed services. When the product is not yet active the /api/rest/v1/adapterlicenseproduct/** will be made public (not username/password checks) and the web filter will not block those requests. This is needed so that the activation UI can interact with the backend in order to implement the activation processed. All other REST API services will be blocked by the filter during the product inactive state.

### Activation plugin

This plugin will be responsible to provide the activation UI and backend service. This plugin needs to be UI feature plugin that defines adapterlicensedfeatures and adapterlicensedproducts  services. Those are defined by the licensing framework as an extension services to the licensing framework. Those services are going to be called by the licensing framework when we want to get the licensing product and activate it. This plugin will be also responsible to defined the activation licensing product and one way how to do it is by using the data loader and adding the following example product licensing entity
```
[
  {
    "activationId": "",
    "availableCount" : 1,
    "entitlementId": "D8-FC-93-54-8F-6C",
    "productName": "MTP Core",
    "productVersion": "1.0.0",
    "licenseCount": 0
  }
]
```
The activationId will be initially empty and it is provided here because the licensing schema for licensedproduct specified that as required field. The entitlementId in this case will represent the activation challenge key and it will be visible on the activation UI screen. Since initially we do not want the license to be active we are specifying the licenseCount as 0 – meaning that no license for that licensed product is used, and availableCount as 1 – meaning that we can only have one activation. 

In order for the plugin to hook itself during the licensing product validation and activation the extended licensed schema could look like the following
```
{
  "id": "http://avocent.com/schemas/rdum/activation/extended-licensed-product-service-schema-v1",
  "$schema": "http://avocent.com/schemas/core-schema-v1",
  "parent": "http://avocent.com/schemas/licensing/licensed-product-service-schema-v1",
  "includes": [
    "http://avocent.com/schemas/rdum/activation/extended-licensed-product-schema-v1"
  ],
  "type": "object",
  "definitionType": "service",
  "modelConfiguration": {
    "modelType": "ExtendedLicensedProduct"
  },
  "serviceConfiguration": {
    "serviceType": "synchronous",
    "serviceExposed": true,
    "tenantAware": false,
    "path": "adapterlicensedproducts",
    "repositoryConfiguration": {
      "repositoryType": "storage",
      "collection": "adapterlicensedproducts"
    },
    "methodConfiguration": {
      "collectionPostEnabled": false,
      "itemPutEnabled": false,
      "itemPatchEnabled": false,
      "itemDeleteEnabled": false
    },
    "handlers": {
      "description": "",
      "onAfterQuery": [
        {
          "sequence": 1,
          "type": "java",
          "code": "licenseValidationHandler"
        }
      ]
    },
    "actions": [
      {
        "name": "adapteractivate",
        "description": "Resource action. Activates a licensed product so its features could be accessed in the system. For the activation the activationId and count is necessary.",
        "onExecuteAction": [
          {
            "sequence": 1,
            "type": "java",
            "code": "activateOneProductAction"
          }
        ]
      }
    ]
  }
}
```
The activateOneProductAction and licenseValidationHandler  are Spring beans provided by the activation plugin itself. The activation UI that should be provided by this plugin as well should grab the licensed product that represents the activation (for RDUM this is a single licensed product so that it can request /api/rest/v1/adapterlicensedproducts/ and treat that as a single element collection) and then display on the UI the entitlementId. The UI then needs to have an action (like a button) that will invoke the activate action 

POST /api/rest/v1/adapterlicensedproducts/{{_id}}/activate
```
{
	'activationId' : '{{activationIdValue}}'
} 
```
Where the _id variable should be substituted with the activation licenseproduct and the activationIdValue need to be substituted with whatever value the user have provided. If the result code of that operation is HTTP 201 then the activation UI should redirect the browser to / path so that the UI console application will be loaded (this time the license is active and the web filter should not perform the automatic redirection to the activation UI)
