# HP Warranty Integration for ServiceNow
This project walks through integrating HP's Warranty API with the ServiceNow platform. It enables automated retrieval and display of warranty information for HP assets to support asset management workflows.

## Workflow Diagram

<img src="images/Warranty Integration Workflow.jpg" alt="Workflow Diagram" width="75%"/>

## Configuration Steps
1. Create an integration user
    * Add the following roles: 
        * import_scheduler
        * import_x
        * import_x
3. Create a new Connection & Credential Alias for ServiceNow
    * Create a Basic auth credential to interact with the Import Set API
4. Create a new Connection & Credential Alias for HP Warranty
    * Create a Basic auth credential to retrieve an access token
5. Create an Import Set
6. Create actions
7. Create the flow that we will run weekly to collect warranty information

## Actions Explained
Create the following custom actions for use in your Flow Designer workflow. Each action interacts with a different portion of the warranty API.

1. Get Access Token
    * Add a REST step. Use the HP Warranty connection alias created earlier. Send a POST request to https://warranty.api.hp.com/oauth/v1/token with the data below in the body of your request.
        ```javascript
        { "grant_type": "client_credentials" }
        ```
    * Add a script step. Parse the response body to create a Bearer authentication token.

        ```
        Bearer d76e7a8b9b0a47b2a040fc81
        ```
2. Create Payload
    * Add a lookup records step. Query for HP hardware records where warranty expiration is empty.
    * Add a script step. Transform the records from the previous step into a format the warranty API can use. 

        ```javascript
            [
                {
                    "sn": "serial_1", 
                    "pn": "sku_1"
                },
                ...
            ]
        ```
3. Submit Job
    * Add a REST step. Send a POST request to https://warranty.api.hp.com/productwarranty/v2/jobs with the payload from the previous action.
    * Parse the response body.
        ```javascript
            var responseObject = JSON.parse(inputs.response_body);
            var jobid = responseObject.jobId;
            var estimatedTime = responseObject.estimatedTime;

            outputs.jobid = jobid;
            outputs.estimatedtime = estimatedTime;
        ```
    * Add a script step. Calculate the wait duration.
        ```javascript
            var seconds = inputs.seconds;
            // wait at least two minutes, ensures the job is finished running
            var minutes = Math.ceil(seconds / 60) + 1;
            var duration = GlideDuration('0 00:' + minutes + ':00');

            outputs.duration = duration;
        ```
4. Get Job Status
    * Add a REST step. Send a GET request to https://warranty.api.hp.com/productwarranty/v2/jobs/{jobId}
    * Parse the response body.
        ```javascript
            var responseObject = JSON.parse(inputs.response_body);
            var status = responseObject.status;

            outputs.status = status;
        ```
5. Get Job Results
    * Add a REST step. Send a GET request to https://warranty.api.hp.com/productwarranty/v2/jobs/{jobId}/results
    * Parse the response body, and filter the product list for CarePacks.
        ```javascript
            var product_list = JSON.parse(inputs.response_body);

            // for each product in the list, apply the following function
            var products = product_list.map(function (product) {
                // create default properties
                var serial_number = product?.product?.serialNumber || product?.sn;
                var warranty_match = false;
                var warranty_expiration = null;

                if (product.offers && Array.isArray(product.offers)) {
                    // search for a CarePack product
                    let matchingOffer = product.offers.find(offer => offer.offerProductIdentifier === "HA151AC");
                    // if a CarePack is found, update properties
                    if (matchingOffer) {
                        warranty_match = true;
                        warranty_expiration = matchingOffer.serviceObligationLineItemEndDate;
                    }
                }

                // finally, return an object with our properties
                return { serial_number: serial_number, warranty_match: warranty_match, warranty_expiration: warranty_expiration };
            });

            outputs.records = products;
        ```
    * Output data in this format, this can be used by the importMultiple API endpoint:
        ```javascript
            {
                "records":
                    [
                        {
                            "serial_number": "serial_1", 
                            "warranty_expiration": "2025-10-31"
                        },
                        ...
                    ]
            }
        ```
6. Load Job Results
    * Add a REST step. Use the ServiceNow connection alias, send a POST request to now/import/{import_set_table_name}/insertMultiple with the payload from the previous action.
    * The data is pushed through the Import Set as you normally expect.

## Import Set Configuration
The Import Set and data source are established by attaching a json file with the appropriate data format. This data is then mapped to matching fields on the Hardware (alm_hardware) table.

```javascript
{
    "records":
        [
            {
                "serial_number": "serial_1", 
                "warranty_expiration": "2025-10-31"
            },
            ...
        ]
}
```

I opted for the Import Set API endpoint to insert multiple records at once.

API Endpoint:
<br>POST /api/now/import/{stagingTableName}/insertMultiple

You need to create a REST Insert Multiple record at System Web Services > REST > Insert Multiple for this to work.

Check out ServiceNow's documentation located [here](https://github.com/lifeparticle/Markdown-Cheatsheet) for more information.
