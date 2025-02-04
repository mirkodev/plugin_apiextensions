# Plugin API Extensions
API Extensions plugin framework allows writing standalone extensions while taking care for:
- execution flow
- client permissions
- extension settings
- error handling
- logging
- contexts

In addition it comes with ready to go extensions for most common needs: \
\+ [API Extensions table](./docs/api-extensions-table.md)

## Compatibility
This cartridge supports compatibility mode of 18.10 and hight.
(to work on older implementations code is written in old ES5 style and not using ES6 features)


## Setup
- Import metadata
- Add to cartridge path
- Configure ApiExtension (group) custom preferences (see bellow)
- Create OCAPI / SCAPI clients and setup permissions (don't forget to enable also SCAPI hooks feature switch)


## Configuration
Configuration is JSON using following format:
```
{
    <clientId>: {
        <ExtensionName>: {
            enabled: bool
            allowed: bool
            settings: object
        }
    }
}
```
There is also a shorthand format:
```
{
    <ExtensionName>: true
}
```
which equals to
```js
    <ExtensionName>: {
        enabled: true,
        allowed: true,
        settings: {}
    }
```

You can also have default config will be common base for all clients.  
Client specific config will be merged on top of base / default config (if exists).  
Default config key is `_DEFAULT_`

Example config:
```json
{
    "_DEFAULT_": {
        "product": {
            "OcapiMasterPrices": true
        }
    },

    "00000000-0000-0000-0000-000000000000": {
        "product_search": {
            "OcapiPrices": true,
            "CustomAttributes": {
                "enabled": true,
                "allowed": true,
                "settings": {
                    "attributes": ["isNew", "isSale", "productBadge"]
                }
            }
        }
    }
}
```


## Usage

### Control and settings
OCAPI and (now) SCAPI allow passing custom http parameters.  
Framework allows clients _(if [allowed] in the site pref - see above)_ to:
- enable endpoint extensions
- set extension settings

#### Enable endpoint extensions
Pass csv of endpoint extensions to be enabled via ``extensions`` parameter.  
Example:
```
/dw/shop/v22_4/product_search?q=1G010159&c_extensions=OcapiPrices,CustomAttributes
```

#### Set extension settings
Pass settings as JSON using ``c_{ExtensionName}`` http parameter.  
Example:
```
...&c_extensions=CustomAttributes&CustomAttributes={"attributes": ["isNew", "isSale"]}
```
_(i: for OCAPI you can omit "c_" prefix)_

## Monitoring 
OOTB SFCC hooks do not have any logging.  
If there is an error / unhandled exception the error is NOT returned nether logged. 

The framework adds advanced logging, that can be used to track:
- errors
- execution flow
- debug information

All logs are written in a dedicated file:  
Prefix: `api-extensions`  
Group: `api-extensions`  
_(You might want to enable the debug level during extension development)_

Errors and warnings are logged also in the common error and warning files.




## Extension Development
Framework uses conventions that allow easy transformation of hook handlers to extensions.  
To work extensions just need to be placed in the correct location:
```
scripts/hooks/apis/{endpoint}/extensions/{ExtName}.js
```  


### Endpoints
Endpoints should be registered to related hooks  
See [`hooks.json`](./cartridges/plugin_apiextensions/hooks.json) and [`product_search.js`](./cartridges/plugin_apiextensions/cartridge/scripts/hooks/apis/product_search.js) endpoint for example.

NOTE: Endpoint naming convention:  
Endpoints follow the system hook naming.  
For example Product Search endpoint is `product_search`: 
```
dw.ocapi.shop.product_search.modifyGETRespons
```
(i) All hook endpoints can be seen here:  
https://developer.salesforce.com/docs/commerce/commerce-api/guide/hook-method-details.html


### Simple extension
Example of simple extension is [ProductPageUrl](cartridges/plugin_apiextensions/cartridge/scripts/hooks/apis/product/extensions/ProductPageUrl.js).\
It extends the product resource/endpoint, adding the (SFRA) PDP URL:
```
scripts/hooks/apis/product/extensions/ProductPageUrl.js
```

### Advanced extension

#### Custom Context Providers
The problem that framework resolves:
Sometimes you need an extension working on on diffrent than the default context.
Typical example are extensions of product_search endpoint, where you usually need to add some data to the search hits rather than modify / work on the searchResult (which is the context you get by the platform):  
```
product_search.modifyGETResponse(searchResult)
```
In this case if you have 5 separate extensions that add extend the search hit with some data then each extension will have to loop trough the result and re-load the related api Product in order to get additional data. This is not only inconvinient but could be also a performance issue.

The framework resolves this by allowing you to create and bind your extension to a Custom Context Provider.\
For example:
 - native hook context: `product_search.modifyGETResponse(searchResult)`
 - custom hook context: `product_search.modifyGETResponse(resSearchHit, apiProduct)`

##### Creation of Custom Context
To create custom context provider:
 - create a js module in `/{endpoint}/contexts/` folder
 - implement js iterator interface and pass expected context (arguments).

For details see the implementation of the exiting [SearchHitContext](cartridges/plugin_apiextensions/cartridge/scripts/hooks/apis/product_search/contexts/SearchHitContext.js):
```
scripts/hooks/apis/product/contexts/SearchHitContext.js
```

##### Binding to Custom Context
There are two ways you can bind your extension to a Custom Context Provider:

a) (declarative) via `contextProvider` property  
Simply add `contextProvider` property to your hook handler function:
```js
// file: scripts/hooks/apis/product/extensions/ProductPageUrl.js
exports.modifyGETResponse = function (resHit, apiProduct) { /*...*/ }
exports.modifyGETResponse.contextProvider = 'SearchHitContext';
```

b) (imperative) via `ApiExtension` object  
For complex cases / advanced usage ApiExtension helper class could add clarity and allows multiple handlers attached on same hook with different contexts:
```js
// file: scripts/hooks/apis/product/extensions/CustomAttributes.js
apiExtension.add('modifyGETResponse', function (searchResult) { /*...*/ }
apiExtension.add('modifyGETResponse', 'SearchHitContext', function (resHit, apiProduct) { /*...*/ }
```

#### Extension Settings
Extensions allow global or per request configuration 
(via custom site prefs or request param - see above).  

Extension runtime (current) settings can be read:

a) using static method:
```js
// example: scripts/hooks/apis/product/extensions/ProductPageUrl.js
ApiExtension.GetRuntimeSettings(module)
```

b) from a property of `ApiExtenion` instance:
```js
// example: scripts/hooks/apis/product/extensions/CustomAttributes.js
apiExtension.runtimeSettings
```
