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

After that create and generate the resoruce. Open up the Azure function and paset the function that is shown in Azure Function logic section.

## Azure Logic App steps

The Azure Logic App workflow begins with the "When events are available in Event Hub" trigger, which activates whenever a new event is published to the configured Event Hub resource.

If the action is triggered, the workflow calls the Azure Function, passing the JSON payload from the Event Hub to process the trip data. The function returns an array of trips with insights, which are then iterated over using a For each loop. 

Within this loop, a condition checks whether the current trip is marked as interesting by evaluating item()?['isInteresting']. If the condition is false, a post card is sent indicating that the trip was analyzed but has no issues. 

If the condition is true, another condition checks whether the trip is suspicious by evaluating contains(items('For_each')?['insights'], 'SuspiciousVendorActivity'). If this condition is true, a post card is sent stating that the trip is suspicious; otherwise, a post card is sent to indicate that the trip is interesting but not suspicious.

## Azure Function logic

The Function app we have in this lab is displayed bellow:

import azure.functions as func
import logging
import json

app = func.FunctionApp(http_auth_level=func.AuthLevel.ANONYMOUS)

@app.route(route="")
def analyze_trip(req: func.HttpRequest) -> func.HttpResponse:
    try:
        input_data = req.get_json()
        trips = input_data if isinstance(input_data, list) else [input_data]

        results = []

        for record in trips:
            trip = record.get("ContentData", {})  # ✅ Extract inner trip data

            vendor = trip.get("vendorID")
            distance = float(trip.get("tripDistance", 0))
            passenger_count = int(trip.get("passengerCount", 0))
            payment = str(trip.get("paymentType"))  # Cast to string to match logic

            insights = []

            if distance > 10:
                insights.append("LongTrip")
            if passenger_count > 4:
                insights.append("GroupRide")
            if payment == "2":
                insights.append("CashPayment")
            if payment == "2" and distance < 1:
                insights.append("SuspiciousVendorActivity")

            results.append({
                "vendorID": vendor,
                "tripDistance": distance,
                "passengerCount": passenger_count,
                "paymentType": payment,
                "insights": insights,
                "isInteresting": bool(insights),
                "summary": f"{len(insights)} flags: {', '.join(insights)}" if insights else "Trip normal"
            })

        return func.HttpResponse(
            body=json.dumps(results),
            status_code=200,
            mimetype="application/json"
        )

    except Exception as e:
        logging.error(f"Error processing trip data: {e}")
        return func.HttpResponse(f"Error: {str(e)}", status_code=400)

This Azure Function is an HTTP-triggered function that processes trip data and returns insights for each trip in JSON format. The function follows a request-response pattern: it accepts JSON input via an HTTP request, analyzes the trip data, and responds with a structured result.

- Lines 10–13: The function retrieves the request body as JSON object. If the input is a single trip, it wraps it into a list to ensure consistent iteration over trip records.

- Lines 16–21: For each trip record, it extracts the trip details from the "ContentData" field. It then retrieves specific attributes such as vendorID, tripDistance, passengerCount, and paymentType.

- Lines 23–32: Based on the trip attributes, the function adds relevant "insight" tags (e.g., "LongTrip", "GroupRide", "CashPayment", "SuspiciousVendorActivity") to the insights list.

- Lines 34–42: A result object is created for each trip, containing the trip data, insights, a boolean flag isInteresting (true if any insights are detected), and a summary message.

- Lines 44–48: After processing all trips, the function returns an HTTP response (func.HttpResponse) containing the results list in JSON format with a 200 status code.

## Input and output

In order to input to the arcitecture is to goto Date Explore section in the event hub you made and this is where you able to test your event hub by sending example events. When sending events, click Send events and click Yellow taxi in selecting the dataset. There they give you a json file to test with. The example of the json file is displayed bellow

### Input

{
    "vendorID": "4",
    "tpepPickupDateTime": 1528119858000,
    "tpepDropoffDateTime": 1528121148000,
    "passengerCount": 2,
    **"tripDistance": 0.87,**
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

To get the output is to go to team in your account and you will see a new chat added which is called "Workflows". If the value "isInteresting" is falses, then it will send out a "Trip Analyzed - No Issues" card. If the value "isInteresting" is true and the payment type is 2 and the distance bellow 1, then it will send out a "Suspicious Vendor Activity Detected" card. If the value "isInteresting" is true and the payment type is 1 and the distance above 1, then  it will send out an "Interesting Trip Detected". From the example input above, it will return a "Trip Analyzed - No Issues" card like this

https://youtu.be/bkZFVowx--A