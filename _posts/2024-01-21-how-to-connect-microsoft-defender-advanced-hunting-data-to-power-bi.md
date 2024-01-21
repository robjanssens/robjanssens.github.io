---
layout: post
title:  "How to connect Microsoft Defender Advanced Hunting data to Power BI"
---

# How to connect Microsoft Defender Advanced Hunting data to Power BI

Imagine the tedious task of manually exporting data from [Microsoft Defender for Endpoint](https://www.microsoft.com/en-us/security/business/endpoint-security/microsoft-defender-endpoint) to Excel, only to spend more work creating tables and calculations. This was the reality for a client who initially used a KQL query to filter device data. While this manual process was functional, it was far from efficient. Enter Power BI, a tool that seemed perfect for automating this task.

However, there was a lack of current and comprehensive documentation on integrating Microsoft Defender data into Power BI:

- The [official documentation on Microsoft](https://learn.microsoft.com/en-us/microsoft-365/security/defender-endpoint/api/api-power-bi?view=o365-worldwide) provided to be rather invaluable. It uses a different API endpoint than the one mentioned in the official Microsoft Advanced Hunting API reference. Additionally, it recommended using a standard user account for credentials. While not incorrect, data engineering best practices typically favor **service principals** or **managed identities (for Azure resources)**.
- Further Google Search results yielded no clear, up-to-date tutorial on transferring data from Microsoft Defender to Power BI.

As such, this gap in accessible, practical guidance inspired me to create this article. I aim to demystify the process of fetching data from Microsoft Defender and utilizing it effectively in Power BI to generate insightful reports. Let’s dive into this step-by-step tutorial.

## Prerequisites

To follow along this tutorial, you will need the following:

- A **Microsoft 365 account**: To access Microsoft Defender, Power BI, and Microsoft Entra (part of Azure)
- **Power BI Desktop**: The primary tool for retrieving data. We will use Power Query within Power BI, leveraging the M language.
- **Service principal**: The key component for API access. You will need to set up a Service Principal with the API permission labeled **AdvancedHuntingRead.All** (part of Microsoft Threat Protection). Additionally, you will have to create a service principal secret. Remember to securely store the application (Client) ID and the secret value, as they will be crucial later on.

## Procedure

### Acquiring a Bearer token for API access

Firstly, we will need a Bearer token. This token is crucial as it authenticates the service principal with the Advanced Hunting API and carries the necessary API permissions.

**Action steps**

Start with Power BI Desktop and go to `Get data > Blank Query`. Then, navigate to `View > Advanced Editor`. You will be presented with a basic code template. 

```
    let
        Source = ""
    in
        Source

```

**Set up API constants:** We need to define some constants for the API data request. This includes the **Advanced Hunting API base url** and your **KQL query.**

```
    let
        // API variables
        AdvancedHuntingAPI = "<https://api.security.microsoft.com/api/advancedhunting/run>",
        Query = "<...INSERT YOUR KQL Query HERE...>",
    in
        Query

```

**Request the Bearer token:** Afterwards, we will make an HTTP Post Request to the Microsoft Identity Platform. This involves using the `Web.Contents()` function, including the Content-Type header, and providing a specially formatted request body with the following details:

- grant_type = client_credentials
- resource = [https://api.security.microsoft.com](https://api.security.microsoft.com/)
- client_id = <...your service principal Client id...>
- client_secret = <... your service principal secret value...>

Remember to format the request body correctly. It should be a single string with key-value pairs separated by an ampersand (&) and passed through the `Text.ToBinary()` function.

```
BearerJSON = Web.Contents(
    "<https://login.microsoft.com/><YOUR_TENANT_ID>/oauth2/token",
    [
        Headers=[#"Content-Type"="application/x-www-form-urlencoded"],
        Content=Text.ToBinary(
            "grant_type=client_credentials&resource=https://api.security.microsoft.com&client_id=...&client_secret=..."
        )
    ]
),

```

**Extract the Bearer token:** Once you receive a response, parse the JSON object to retrieve the ‘access_token’ field. This is your Bearer token as a JWT, which will be used for subsequent API requests.

```
BearerToken = Record.Field(Json.Document(BearerJSON), "access_token"),
```

### Fetching data data from Microsoft Defender API

With the Bearer token in hand, we can proceed to fetch data through the API.

**Action steps**

**Make the API Request:** Construct an HTTP POST Request to the API. Include headers for the `Content-Type` and the Bearer token under the `Authorization` key. Also, incorporate your predefined KQL query constant in the HTTP body as a formatted JSON object (`Json.FromValue`).

Tip: Add the word “Bearer “ (note the space after Bearer) before your Bearer token in the Authorization header. This is a necessary format for the API request.

```
DataFetch = Web.Contents(
    AdvancedHuntingAPI,
    [
        Headers = [
            #"Content-Type"="application/json",
            Authorization="Bearer " & BearerToken
        ],
        Content = Json.FromValue([Query=Query])
    ]

)

```

**Parse the received data:** Once you have the JSON response, use `Json.Document` to parse it and extract the data from the `Results` field. 

```
ParsedData = Json.Document(DataFetch),
Results = ParsedData[Results]
```

Finally, the data, now in a list of records, is ready to be converted into a table format for your use.

### Full code

Below, you will find the full code utilized for retrieving data from the Microsoft Defender Advanced Hunting API through Power BI. To improve readability, I have modified the original script by introducing the variable `Bearer_REQ_BODY`, designed to encapsulate the string required for the HTTP Post request’s body when acquiring the Bearer token.

```
    let
        // API variables
        AdvancedHuntingAPI = "<https://api.security.microsoft.com/api/advancedhunting/run>",
        Query = "<...INSERT YOUR KQL Query HERE...>",

        // Bearer Request
        Bearer_REQ_BODY = "grant_type=client_credentials&resource=https://api.security.microsoft.com&client_id=...&client_secret=...",
        BearerJSON = Web.Contents(
            "<https://login.microsoft.com/><YOUR_TENANT_ID>/oauth2/token",
            [
                Headers=[#"Content-Type"="application/x-www-form-urlencoded"],
                Content=Text.ToBinary(Bearer_REQ_BODY)
            ]
        ),
        BearerToken = Record.Field(Json.Document(BearerJSON), "access_token"),

        // API Fetching
        DataFetch = Web.Contents(
            AdvancedHuntingAPI,
            [
                Headers = [
                    #"Content-Type"="application/json",
                    Authorization="Bearer " & BearerToken
                ],
                Content = Json.FromValue([Query=Query])
            ]
        ),
        ParsedData = Json.Document(DataFetch),
        Results = ParsedData[Results]
    in
        Results

```
