# CST8917 Lab 4

## Setting up the architecture

### Setting up Azure Event Hub

In Azure, go and create an Event Hubs resource and fill in the following prameters:

- **Resource group:** CST8917_Lab_4
- **Name:** event-hub-lab-4
- **Region:** Canada Center
- **Pricing tier:** Basic (for saving cost)

Then create one. After that, select "+ Event Hub" to make an event hub and then fill in the following prameters:

- **Name:** event_hub_lab_4

and then create one. Go to Date Explore section in the event hub you made and this is where you able to test your event hub by sending example events. When sending events, click Send events and click Yello taxi in selecting the dataset. There they give you a json file to test with. If you want to make edits of it, copy the entire json file and patse it in the Custom payload Dataset.

### Setting up Azure Logic App

Create your Azure Logic app and pick the Workflow Service Plan in the standard section. Then fill in the following:

- **Resource group:** CST8917_Lab_4
- **Logic App name:** logic-app-lab-4
- **Region:** Canada East
- **Pricing Plan:** Lowest plan cost

After that create and generate the resoruce. Go to diagram view and start creating the workflow displayed bellow:

### Setting up Azure Function App

Create your Function App and pick the Flex Consumption plan.

- **Resource group:** CST8917_Lab_4
- **Funtion App name:** function-app-lab-4
- **Region:** Canada East
- **Pricing Plan:** Lowest plan cost
- **Runtime stack:** Python
- **Version:** 3.10

After that create and generate the resoruce. 

## Input and output

In order to input to the arcitecture is to goto Date Explore section in the event hub you made and this is where you able to test your event hub by sending example events. When sending events, click Send events and click Yello taxi in selecting the dataset. There they give you a json file to test with. The example of the json file is displayed bellow

### Input

{
    "vendorID": "4",
    "tpepPickupDateTime": 1528119858000,
    "tpepDropoffDateTime": 1528121148000,
    "passengerCount": 2,
    **"tripDistance": 8.36,**
    "puLocationId": "186",
    "doLocationId": "230",
    "startLon": null,
    "startLat": null,
    "endLon": null,
    "endLat": null,
    "rateCodeId": 1,
    "storeAndFwdFlag": "N",
    **"paymentType": 1,**
    "fareAmount": 13.5,
    "extra": 0,
    "mtaTax": 0.5,
    "improvementSurcharge": "0.3",
    "tipAmount": 2.86,
    "tollsAmount": 0,
    "totalAmount": 17.16
}

If you want to make edits of it, copy the entire json file and patse it in the Custom payload Dataset. In this lab the two variables we are going to test with is tripDistance, and paymentType. For testing to get different output. You need to have tripDistance to be above or bellow zero and paymentType to be 1 or 2 to get different results.

### Output

To get the output is to go to team in your account and you will see a new chat added which is called "Workflows". If the value "isInteresting" is falses, then it will send out a "Trip Analyzed - No Issues" card. If the value "isInteresting" is true and the payment type is 2 and the distance bellow 1, then it will send out a "Suspicious Vendor Activity Detected" card. If the value "isInteresting" is true and the payment type is 1 and the distance above 1, then  it will send out an "Interesting Trip Detected".

{
    "definition": {
        "$schema": "https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#",
        "contentVersion": "1.0.0.0",
        "triggers": {
            "When_events_are_available_in_Event_Hub": {
                "type": "ApiConnection",
                "inputs": {
                    "host": {
                        "connection": {
                            "name": "@parameters('$connections')['eventhubs']['connectionId']"
                        }
                    },
                    "method": "get",
                    "path": "/@{encodeURIComponent('yellotaxidrivereventhub')}/events/batch/head",
                    "queries": {
                        "contentType": "application/json",
                        "consumerGroupName": "$Default",
                        "maximumEventsCount": 10
                    }
                },
                "recurrence": {
                    "interval": 1,
                    "frequency": "Minute"
                }
            }
        },
        "actions": {
            "taxifunctionapp-analyze_trip": {
                "type": "Function",
                "inputs": {
                    "body": "@triggerBody()",
                    "function": {
                        "id": "/subscriptions/3b36c431-a92e-4ac8-927b-17a0f4b30054/resourceGroups/cst8917-lab4/providers/Microsoft.Web/sites/taxifunctionapp/functions/analyze_trip"
                    }
                },
                "runAfter": {}
            },
            "For_each": {
                "type": "Foreach",
                "foreach": "@body('taxifunctionapp-analyze_trip')",
                "actions": {
                    "Is_intresting": {
                        "type": "If",
                        "expression": {
                            "and": [
                                {
                                    "equals": [
                                        "@item()?['isInteresting']",
                                        true
                                    ]
                                }
                            ]
                        },
                        "actions": {
                            "is_suspicious": {
                                "type": "If",
                                "expression": {
                                    "and": [
                                        {
                                            "equals": [
                                                "@contains(items('For_each')?['insights'],'SuspiciousVendorActivity')",
                                                true
                                            ]
                                        }
                                    ]
                                },
                                "actions": {
                                    "send_that_the_post_card_is_suspicious": {
                                        "type": "ApiConnection",
                                        "inputs": {
                                            "host": {
                                                "connection": {
                                                    "name": "@parameters('$connections')['teams']['connectionId']"
                                                }
                                            },
                                            "method": "post",
                                            "body": {
                                                "recipient": "balc0022@algonquinlive.com",
                                                "messageBody": "  {\n  \"type\": \"AdaptiveCard\",\n  \"body\": [\n    {\n      \"type\": \"TextBlock\",\n      \"text\": \"⚠️ Suspicious Vendor Activity Detected\",\n      \"weight\": \"Bolder\",\n      \"size\": \"Large\",\n      \"color\": \"Attention\"\n    },\n    {\n      \"type\": \"FactSet\",\n      \"facts\": [\n        { \"title\": \"Vendor\", \"value\": \"@{items('For_each')?['vendorID']}\" },\n        { \"title\": \"Distance (mi)\", \"value\": \"@{items('For_each')?['tripDistance']}\" },\n        { \"title\": \"Passengers\", \"value\": \"@{items('For_each')?['passengerCount']}\" },\n        { \"title\": \"Payment\", \"value\": \"@{items('For_each')?['paymentType']}\" },\n        { \"title\": \"Insights\", \"value\": \"@{join(items('For_each')?['insights'], ', ')}\" }\n      ]\n    }\n  ],\n  \"actions\": [],\n  \"version\": \"1.2\"\n}"
                                            },
                                            "path": "/v1.0/teams/conversation/adaptivecard/poster/Flow bot/location/@{encodeURIComponent('Chat with Flow bot')}"
                                        }
                                    }
                                },
                                "else": {
                                    "actions": {
                                        "send_that_the_post_card_is_interesting": {
                                            "type": "ApiConnection",
                                            "inputs": {
                                                "host": {
                                                    "connection": {
                                                        "name": "@parameters('$connections')['teams']['connectionId']"
                                                    }
                                                },
                                                "method": "post",
                                                "body": {
                                                    "recipient": "balc0022@algonquinlive.com",
                                                    "messageBody": "  {\n  \"type\": \"AdaptiveCard\",\n  \"body\": [\n    {\n      \"type\": \"TextBlock\",\n      \"text\": \"🚨 Interesting Trip Detected\",\n      \"weight\": \"Bolder\",\n      \"size\": \"Large\",\n      \"color\": \"Attention\"\n    },\n    {\n      \"type\": \"FactSet\",\n      \"facts\": [\n        { \"title\": \"Vendor\", \"value\": \"@{items('For_each')?['vendorID']}\" },\n        { \"title\": \"Distance (mi)\", \"value\": \"@{items('For_each')?['tripDistance']}\" },\n        { \"title\": \"Passengers\", \"value\": \"@{items('For_each')?['passengerCount']}\" },\n        { \"title\": \"Payment\", \"value\": \"@{items('For_each')?['paymentType']}\" },\n        { \"title\": \"Insights\", \"value\": \"@{join(items('For_each')?['insights'], ', ')}\" }\n      ]\n    }\n  ],\n  \"actions\": [],\n  \"version\": \"1.2\"\n}"
                                                },
                                                "path": "/v1.0/teams/conversation/adaptivecard/poster/Flow bot/location/@{encodeURIComponent('Chat with Flow bot')}"
                                            }
                                        }
                                    }
                                }
                            }
                        },
                        "else": {
                            "actions": {
                                "send_that_the_post_card_that_it_worked": {
                                    "type": "ApiConnection",
                                    "inputs": {
                                        "host": {
                                            "connection": {
                                                "name": "@parameters('$connections')['teams']['connectionId']"
                                            }
                                        },
                                        "method": "post",
                                        "body": {
                                            "recipient": "balc0022@algonquinlive.com",
                                            "messageBody": "  {\n  \"type\": \"AdaptiveCard\",\n  \"body\": [\n    {\n      \"type\": \"TextBlock\",\n      \"text\": \"✅ Trip Analyzed - No Issues\",\n      \"weight\": \"Bolder\",\n      \"size\": \"Large\",\n      \"color\": \"Good\"\n    },\n    {\n      \"type\": \"FactSet\",\n      \"facts\": [\n        { \"title\": \"Vendor\", \"value\": \"@{items('For_each')?['vendorID']}\"},\n        { \"title\": \"Distance (mi)\", \"value\": \"@{items('For_each')?['tripDistance']}\"},\n        { \"title\": \"Passengers\", \"value\": \"@{items('For_each')?['passengerCount']}\"},\n        { \"title\": \"Payment\", \"value\": \"@{items('For_each')?['paymentType']}\"},\n        { \"title\": \"Summary\", \"value\": \"@{join(items('For_each')?['insights'], ', ')}\"}\n      ]\n    }\n  ],\n  \"actions\": [],\n  \"version\": \"1.2\"\n}"
                                        },
                                        "path": "/v1.0/teams/conversation/adaptivecard/poster/Flow bot/location/@{encodeURIComponent('Chat with Flow bot')}"
                                    }
                                }
                            }
                        }
                    }
                },
                "runAfter": {
                    "taxifunctionapp-analyze_trip": [
                        "Succeeded"
                    ]
                }
            }
        },
        "outputs": {},
        "parameters": {
            "$connections": {
                "type": "Object",
                "defaultValue": {}
            }
        }
    },
    "parameters": {
        "$connections": {
            "type": "Object",
            "value": {
                "eventhubs": {
                    "id": "/subscriptions/3b36c431-a92e-4ac8-927b-17a0f4b30054/providers/Microsoft.Web/locations/canadacentral/managedApis/eventhubs",
                    "connectionId": "/subscriptions/3b36c431-a92e-4ac8-927b-17a0f4b30054/resourceGroups/cst8917-lab4/providers/Microsoft.Web/connections/eventhubs",
                    "connectionName": "eventhubs"
                },
                "teams": {
                    "id": "/subscriptions/3b36c431-a92e-4ac8-927b-17a0f4b30054/providers/Microsoft.Web/locations/canadacentral/managedApis/teams",
                    "connectionId": "/subscriptions/3b36c431-a92e-4ac8-927b-17a0f4b30054/resourceGroups/cst8917-lab4/providers/Microsoft.Web/connections/teams",
                    "connectionName": "teams"
                }
            }
        }
    }
}