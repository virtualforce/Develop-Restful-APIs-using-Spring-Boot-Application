# Develop-Restful-APIs-using-Spring-Boot-Application
Here is a complete guide how Spring Boot Application interacts with MongoDB, Amazon Web S3, SNS Services and MailChimp API.

## Getting Started
This guide only relates to J2EE web application developed with “Spring Boot”. In database section, we will choose mongoDB over MySQL due to its flexibility & scalability and it will also described here in detail how mongoDB & Amazon Web Services work in conjunction to provide services through Restful APIs, there are five key entities which will be explained here:

1.	Spring Boot Application
2.	MongoDB
3.	Amazon Web Service S3
4.	Amazon Web Service SNS
5.	MailChimp (Email API) Integration

### Prerequisites
Its recommended to setup these four things on your environment to get full taste of this guide:

1. IDE: IntelliJ IDEA (Prefereabley)
2. JDK (used 8.0)
3. MongoDB (used 3.4)
4. Setup Amazon Web Services S3 &amp; SNS

### 1. Spring Boot Application
Spring Boot makes it easier to develop Spring-powered gradle applications & services, there is no XML required for configurations. Application developed with Spring Boot called Spring Boot application, in our case, we have to create Restful Services using Spring Boot Application

### 2. MongoDB
It’s actually open source flexible & scalable database which store JSON like documents data and uses dynamic schemas, meaning that you can create records without first defining the structure, such as the fields or the types of their values and complex information can be stored easily. Here you can see terminology & concept in each system

| MySQL        | MongoDB           | 
| ------------- |:-------------:| 
| Table      | Collection | 
| Row      | Document      |  
| Column | Field      |  
| Joins | Embedded documents, linking      |

#### Installation & Setup on Windows 64-bit
At first, we need to do installation then will do the setup to make it work properly.
For installation you need to follow these three steps:

1.	Determine the right build for your system
2.	Download the latest version
3.	Installation

I used version **3.4**.  
Now we are going to setup MongoDB, it requires two steps:

1.	MongoDB requires a data directory to store all data. MongoDB’s default data directory path is the absolute path **\data\db** on the drive from which you start MongoDB, such as if you install MongoDB in C drive then **C:\data\db** should be the path.
2.	 you need to add installation path **"C:\Program Files\MongoDB\Server\3.4\bin\"** in Environment Path Variable to access its utilities anywhere through command prompt.


#### Start/Stop MongoDB:
Now setup has completed, to **start** MongoDB,
Run **mongod.exe** command from Command Prompt.

Later, to **stop** MongoDB, 
Press **Control+C** in the terminal where the mongod instance is running.

### 3. Amazon Web Service S3
Amazon S3 has a simple web services interface that you can use to store and retrieve any amount of data, at any time, from anywhere on the web. In our case, we are using it to upload/retrieve images.

#### Integration to Spring Boot Application
Amazon S3 provides web service interface to access its features. It requires two keys, ACCESS KEY, SECRET KEY for authentication, such as:
```
AWSCredentials awsCreds = new BasicAWSCredentials(
         "ACCESS_KEY",
         "SECRET_KEY");

You can also specify region to restrict user’s access, for uploading image
Here is the code snippet:
byte[] decoded = Base64.getDecoder().decode(base64ImageString);
InputStream inputStream = new ByteArrayInputStream(decoded);

ObjectMetadata metadata = new ObjectMetadata();
metadata.setContentLength(decoded.length);

s3.putObject(new PutObjectRequest(S3_BUCKET, s3Key + "/" + fileName, inputStream, metadata));
```

#### Expected Problems:
While accessing Amazon S3 features you can face these kind of problems according to my experience:

-	Credentials are not valid
-	Access incorrect Region which cause exception
-	Accessing path should be the same to the path given while uploading

### 4. Amazon Web Service SNS
Amazon Simple Notification Service (SNS) is a simple, fully-managed "push" messaging service that allows users to push texts, alerts or notifications.
To get started with Amazon SNS, developers first have to create a "topic" which is an access point that allows subscribers, or "clients", who are interested in receiving notifications about a specific topic, to subscribe or request notifications, subscribers can choose how notifications will be delivered, even selecting the specific endpoint and protocol.

#### Setting up Platform Applications
At first you need to register your apps to required platforms. For **Android**, you need to register app on Firebase & get SENDER_ID to setup client.
While for **iOS**, you need to register app on APNS & get p12 file then create platform applications on Amazon SNS dashboard using these credentials.
Here is the complete example to create Endpoint ARN from device token & subscribe to the created Topic:
```
if (device == null) {
    device = new Device(deviceToken);
    CreatePlatformEndpointRequest endpointRequest = new CreatePlatformEndpointRequest().withToken(deviceToken);

    if (platform.equals("ios")) {
        endpointRequest.setPlatformApplicationArn(APNS_ARN);
    } else if (platform.equals("android")) {
        endpointRequest.setPlatformApplicationArn(GCM_ARN);
    } else {
        throw new RuntimeException("Unknown platform specified.");
    }

    CreatePlatformEndpointResult endpointResult = sns.createPlatformEndpoint(endpointRequest);
    device.setEndpointArn(endpointResult.getEndpointArn());
}
if (device.getSubscriptionArn() == null && device.getEndpointArn() != null) {
    SubscribeResult subscribeResult = sns.subscribe(RL_MESSAGES_TOPIC, "application", device.getEndpointArn());
    device.setSubscriptionArn(subscribeResult.getSubscriptionArn());
}
```
To publish notification to all subscribers, you need to use PublishRequest API like this:
```
PublishRequest request = new PublishRequest();
request.setMessageStructure("json");
request.setTopicArn(DevicesService.RL_MESSAGES_TOPIC);
request.setSubject("Ramadan Legacy");
request.setMessage("message");
sns.publish(request);
```
To publish notification to particular subscriber, add this line to the PublishRequest API:

```
request.setTargetArn("target_arn");
```

#### Expected Problems:
While working with Amazon SNS, you need to care of these things:

-	Credentials are not valid
-	Access incorrect Region which cause exception
-	Validate Endpoint Subscription 
-	EndpointDisabledException

You can programmatically check the state of particular endpoint whether it’s enabled/disabled, if disabled then it can be reset as enabled:

```
Map<String, String> existedEndPoint = syncFromSns(reflectionOwnerDevice.getEndpointArn());

String enabledStatus = existedEndPoint.get("Enabled");

if (!enabledStatus.equalsIgnoreCase("true")) {
    Map<String, String> map = new HashMap<>();
    map.put("Enabled", "true");
    SetEndpointAttributesRequest setEndpointAttributesRequest = new SetEndpointAttributesRequest()
            .withEndpointArn(reflectionOwnerDevice.getEndpointArn()).withAttributes(map);
    sns.setEndpointAttributes(setEndpointAttributesRequest);
}

if (!token.equalsIgnoreCase(reflectionOwnerDevice.getDeviceToken())) {
    //Update device token
    reflectionOwnerDevice.setDeviceToken(token);
    mongoTemplate.save(reflectionOwnerDevice);
}



private Map<String, String> syncFromSns(String endPoint) {
    GetEndpointAttributesRequest request = new GetEndpointAttributesRequest().withEndpointArn(endPoint);

    try {
        GetEndpointAttributesResult result = sns.getEndpointAttributes(request);
        Map<String, String> map = new HashMap<>();
        map.put("Enabled", result.getAttributes().get("Enabled"));
        map.put("Token", result.getAttributes().get("Token"));
        return map;
    } catch (NotFoundException exception) {
        exception.printStackTrace();
    }

    return null;
}


```
#### Deployment
We use putty as SSH client for linux server & WinSCP to access its directories for uploading/removing JAR file, below are the steps to upload JAR file:

-	Build JAR file from IDE (IntelliJ IDEA)
-	Open WinSCP & upload JAR file
-	Open Putty & login to server
-	Run JAR file (java –jar example.jar) in background
-	Solution deployed

#### Linux Console Commands
| Command        | Description           |
| ------------- |:-------------:|
|	“screen” 			|run process in background|
|	Ctrl+C or “kill -9 “process_id””	|stop process|
|	Ctrl+A,D 			|go back after running process|
|	“px aux \| grep” 		|get all running processes|
|	“screen -x” 			|return back to last session|
|	“screen -ls” 			|get all screens|
|	“screen –r “screen_id”” 	|re-attach particular screen|
|	“htop” 			|open colorful interface of all running processes|
|	“top” 			|open simple interface of all running processes|

### 5. MailChimp (Email API) Integration
MailChimp integration has done in the context to manage subscriber data, you can subscribe and unsubscribe individuals and sync metadata with your systems, you’ll need to create a list to save subscriber data in your MailChimp account if you don’t have one already, and find the List ID such as (9e67587f52).

It provides Restful APIs interface to access its services like (subscribe, unsubscribe User).  
We need to define these four attributes in **application.properties**:

-	List ID
-	Base URL
-	User ID
-	App ID/Key

In Spring Boot Application, we need to initialize “RestTemplate” in Application class for serving Restful API calls. For further information you can check [here](https://github.com/ajainy/webhook)



