---
layout:     post

title:      "AWS tutorial: DynamoDB part 2"
subtitle:   "Retrieve items from DynamoDB using Lambda and API Gateway"
author: Jamie Craane
date:       2016-12-12
description: "In this tutorial we create a Lambda function which retrieves this data from the DynamoDB table and expose this functionality over HTTP using API Gateway."
#image: "https://img.zhaohuabing.com/in-post/2018-06-02-istio08/background.jpg"
published: true
tags:
- AWS, DynamoDB

categories: [ AWS ]
URL: "/2016/12/12/aws_dynamo_db/"
---

todo: add part 1 link

[source code](https://github.com/jcraane/LambdaDynamoDBApiGateway)

In the previous tutorial I showed you how to use AWS Lambda and API Gateway to insert items in a DynamoDB table. The example implemented a function which stored the location of the user. In this tutorial we create a Lambda function which retrieves this data from the DynamoDB table and expose this functionality over HTTP using API Gateway.

**Create a DynamoDB Global Secondary Index**

The DeviceLocation DynamoDB table uses the id as partition key. To retrieve the locations for a given device, a global secondary index must be created to query on deviceId.

1. Open the AWS console and navigate to the DynamoDB section.
2. Select the DeviceLocation table and click the Indexes tab.
3. Select Create Index.
4. Use deviceId as the partition key, use 1 for read/write capacity units and click create. See the create_dynamodb_index screenshot.

![img.png](/img/posts/dynamodb-create-index.png)

**Create the implementation**

1. Create a new class named RetrieveLocationRequest with one field named “deviceId” with type String. This class holds the parameters which are used as query input to the  DynamoDB table.
2. Create a new class named RetrieveLocationResponse with a field named “locations” of type List. This class will hold the data returned to the consumer of the Lambda function.
3. Initialize the locations field to: new ArrayList<>() since we are going to add DeviceLocation objects to it.
4. Create a new class named RetrieveLocationFunction which implements the com.amazonaws.services.lambda.runtime.RequestHandler interface.

The RetrieveLocationFunction is the implementation of our Lambda function which queries DynamoDB using the given deviceId. It retrieves all locations for the given device.

Copy the following code and paste this in the handleRequest method of the RetrieveLocationFunction class. Make sure the region used here is the same region as the region of the DynamoDB table!

```java

final AmazonDynamoDBClient client = new AmazonDynamoDBClient(new EnvironmentVariableCredentialsProvider());
client.withRegion(Regions.EU_CENTRAL_1); // specify the region you created the table in.
final DynamoDB dynamoDB = new DynamoDB(client);

System.out.println("input = " + input); // Pure for testing. Do not use System.out in production code

final Table table = dynamoDB.getTable("DeviceLocation");
final Index index = table.getIndex("deviceId-index");
final ItemCollection items = index.query("deviceId", input.getDeviceId());

final RetrieveLocationResponse response = new RetrieveLocationResponse();
for (final Item item : items) {
    final DeviceLocation deviceLocation = new DeviceLocation();
    deviceLocation.setDeviceId(item.getString("deviceId"));
    deviceLocation.setLat(item.getDouble("lat"));
    deviceLocation.setLng(item.getDouble("lng"));
    response.getLocations().add(deviceLocation);
}

return response;
```

**Create the Lambda function**

After the implementation is ready we need to upload the function and configure it in AWS.

1. Run the Gradle “clean build” task to create the zip distribution which holds our code and dependencies.
2. Open the AWS console and navigate to the Lambda section.
3. Select “Create a Lambda function” and select Blank Function.
4. Click Next.
5. As name specify “retrieveLocations” and as runtime select Java 8.
6. Upload the PROJECT/build/distributions/DISTRIBUTION_NAME.zip file
7. Use com.example.persister.RetrieveLocationFunction as the Handler
8. Use the existing lambda_location_persister role create in part 1 of this series.
9. Leave the rest default and click Next.
9. Click Create Function.
10. To test the function click Test and use the following test data (replace DEVICE_ID with the deviceId of a record in the table. Refer to part 1 to insert data in the DeviceLocation table):

```json
{
  "deviceId": “DEVICE_ID”
}
```

**Exposing the functionality using API gateway**

The retrieveLocations function will be exposed using an HTTP GET method using API Gateway.

1. Open the AWS console and navigate to API Gateway.
2. Create a new GET method under the deviceLocation resource.
3. As Integration Type choose Lambda Function and select the region the Lambda Function was created in.
4. Specify retrieveLocations as the name and click Save.

The deviceId is passed in as a query parameter in the URL and must be mapped as input to the Lambda function.

1. Open the Method Request settings and open URL Query String Parameters.
2. Add a query parameter named deviceId.
3. Open the Integration Request settings and open Body Mapping Templates.
4. Choose the option: When there are no templates defined (recommended)
5. Add a mapping template for: application/json
6. Add the following template to map the deviceId URL query parameter to the Lambda input (see screenshot body_mapping).

```json
{
    "deviceId": "$input.params('deviceId')"
}
```
![img.png](/img/posts/dynamodb-body-mapping.png)

7. Click Save.
8. Click Test, add a known deviceId and Click the Test button. You should see the output of the Lambda function.
9. Click Actions -> Deploy API to deploy your api.
10. You function should now be publicly accessible.

Make sure you delete your resources when you are done with the tutorial to prevent unwanted billing.

