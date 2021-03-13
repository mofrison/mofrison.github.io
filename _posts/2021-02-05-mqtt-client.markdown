---
title:  MQTT-client for Android
repository: mqtt-client
preview: /assets/images/posts/2021-02-05-mqtt-client/mqtt.jpg
date:   2021-02-05 10:00:00 +0300
categories: cases
tags: [java, android, mqtt]
lang: En
layout: post
---

Today I will tell you how, while writing my [thesis work]({{site.url}}/ru/projects/mobile-monitoring-system), I implemented MqttClient for an android application. It is not difficult to do this, but there was very little detailed information at that time in my native language, and I decided to fix it.

## What you need to do this:
* Working environment of **Android Studio**
* Smartphone running Android OS
* **MQTT** server (broker)

## Connecting the **Paho Android Service** Library
First of all, we will connect the libraries needed to configure the MQTT client and service in the Android app. We will use the **Paho MQTT** client and the Android service, provided by **Eclipse**. Android uses **Gradle** as a build and dependency management system, so the **Paho Android Service** can be added to the app via Gradle. To enable the latest version, add the following entry to the `build.gradle` file Android project:
```gradle
repositories {
    maven {
        url "https://repo.eclipse.org/content/repositories/paho-snapshots/"
    }
}
```
It contains a link to the Paho release repository in the gradle configuration so that Gradle can find the packaged **Paho Android Service JAR**.

In `/app/build.gradle` apps add an entry that connects the Piano Android Service to configure the app's dependency:
```gradle
dependencies {
    implementation 'androidx.legacy:legacy-support-v4:1.0.0' // Required, for Android version 4. x
    implementation 'org.eclipse.paho:org.eclipse.paho.client.mqttv3:1.1.0'
    implementation ('org.eclipse.paho:org.eclipse.paho.android.service:1.1.1')
}
```
Dependency `androidx.legacy:legacy-support-v4:1.0.0` only necessary if the app is designed for Android version 4.x.

## Configuring the **AndroidManifest** file
Before you start writing code, you need to configure `AndroidManifest.xml` file, in order for the application to get Internet access permissions, access to the network state, and the application can run as a service. Add these lines before the opening tag `<application>`:
```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>
<uses-permission android:name="android.permission.WAKE_LOCK"/>
```

**Paho Android Service** encapsulates the **MQTT** connection and offers an API for doing so. To create a binding to the **Piano Android Service**, the service must be declared in `AndroidManifest.xml`. To do this, you need to add the following entry already inside the `<application>` tag:
```xml
<service android:name="org.eclipse.paho.android.service.MqttService" />
```

## Working with the **MQTT** service
The _paho.client.mqttv3_  library from **Eclipse** allows you to interact with the MQTT server via an instance of the `MqttAndroidClient` class. To create it, you need two required parameters: `Context` and a string with a URL of the form _tcp://192.168.1.11:1883_. An important parameter is `clientId`, which should be unique for each device.
```java
MqttAndroidClient client = new MqttAndroidClient(context, serverURI, clientId);
```

The object of the `MqttAndroidClient` class provides methods that can be used to connect to the **MQTT** server or break the connection, subscribe to one topic or several, as well as unsubscribe from topics, send and receive messages. Some of the provided methods(`connect`, `subscribe`, `unsubscribe`, `disconnect`) return `IMqttToken`. On it, using the `setActionCallback` method, you can hang an object with the implementation of the `IMqttActionListener` interface, which contains only two methods: `onSuccess` и `onFailure`. They are called depending on the result of the operation and will serve as a link between our application and the server.

### Setting up the connection
Using an object of the `MqttConnectOptions` type, you can pass additional options, for example, you can pass the version of **MQTT 3.1** as options:
```java
MqttConnectOptions options = new MqttConnectOptions();
options.setMqttVersion(MqttConnectOptions.MQTT_VERSION_3_1);
```

You can also use the options to pass parameters to the _Last Will And Testament_ (**LWT**) function, which is used in **MQTT** to notify other clients about a client that is unbiased.
```java
String topic = "users/last/will";
byte[] payload = "some payload".getBytes();
options.setWill(topic, payload ,1,false);
```

If you need to enter a username and password to connect to the broker, you can add them to the `MqttConnectOptions`object:
```java
options.setUserName(userName);
options.setPassword(password.toCharArray());
```

### Connecting to the server
Before you can interact with the server in any way, you need to create a connection to it.
To do this, we will take a previously created instance of `MqttAndroidClient` and call the`connect` method on it. In this method, you can pass predefined options, a reference to `Context` and `IMqttActionListener` callback.
```java
try {
    client.connect(options, new IMqttActionListener(){
        @Override
        public void OnSuccessMqtt(IMqttToken asyncActionToken) {
            // We are connected
        }
        @Override
        public void OnFailureMqtt(IMqttToken asyncActionToken, Throwable exception) {
            // Something went wrong e.g. connection timeout or firewall problems
        }
});
} catch (MqttException e) {
    e.printStackTrace();
}
```

When calling the connect method, `MqttAndroidClient` asynchronously tries to connect to the **MQTT** broker and return token. This token can be used to register callbacks to receive notifications about an **MQTT** connection or an error.
```java
IMqttToken token = client.connect();
token.setActionCallback(new IMqttActionListener() {
    @Override
    public void OnSuccessMqtt(IMqttToken asyncActionToken) {
        // We are connected
    }
    @Override
    public void OnFailureMqtt(IMqttToken asyncActionToken, Throwable exception) {
        // Something went wrong e.g. connection timeout or firewall problems
    }
});
```

### Subscribe to topic
To receive messages from a broker or an external device, you must subscribe to the corresponding **topic**. To do this, use the method `MqttAndroidClient.subscribe`. The method takes the full name of topic and **[QoS](https://ru.wikipedia.org/wiki/QoS)** as parameters, and also returns **IMqttToken**. The Token can be used to track a successful or unsuccessful subscription.
```java
int qos = 1;
IMqttToken subToken = client.subscribe(topic, qos);
subToken.setActionCallback(new IMqttActionListener() {
    @Override
    public void OnSuccessMqtt(IMqttToken asyncActionToken) {
        // Subscription was successful
    }
    @Override
    public void OnFailureMqtt(IMqttToken asyncActionToken, Throwable exception) {
        // The subscription could not be performed, maybe the user was not
        // authorized to subscribe on the specified topic e.g. using wildcards
    }
});
```
The method can also accept an object of the `IMqttMessageListener` type with the `messageArrived` function, but at the time of writing this post, it wasn't working.
I reported the error. It may have been fixed now.

### Unsubscribe from topic
The reverse of the `subscribe` method is the `unsubscribe` method.
```java
IMqttToken unsubToken = client.unsubscribe(topic);
unsubToken.setActionCallback(new IMqttActionListener() {
    @Override
    public void OnSuccessMqtt(IMqttToken asyncActionToken) {
        // The subscription could successfully be removed from the client
    }
    @Override
    public void OnFailureMqtt(IMqttToken asyncActionToken, Throwable exception) {
        // some error occurred, this is very unlikely as even if the client
        // did not had a subscription to the topic the unsubscribe action
        // will be successfully
    }
});
```

### Publishing messages
**MqttAndroidClient** allows you to publish messages using the method `publish(topic, MqttMessage)`. MqttAndroidClientdoes not queue all messages.
If the client is not connected, the method will throw an error when trying to send a message.
After the first connection, when the **MQTT** connection is established successfully, the method is called `ImqttActionListener.onSuccess`.
```java
byte[] encodedPayload = new byte[0];
try {
    encodedPayload = payload.getBytes("UTF-8");
    MqttMessage message = new MqttMessage(encodedPayload);
    //message.setRetained(true); // To send a retained MQTT message
    client.publish(topic, message);
} catch (MqttException | UnsupportedEncodingException e) {
    e.printStackTrace();
}
```
To send a saved **MQTT** message, use the `Mqtt Message.retained` attribute. It can be set to **true** via `MqttMessage.setRetained(true)`.

### Receiving messages
To receive and process messages, use `MqttCallback`, which can be hung on an `MqttAndroidClient` object even before connecting:
```java
client.setCallback(new MqttCallback() {
    @Override
    public void connectionLost(Throwable cause) {
        // Connection to the server is lost
    }
    @Override
    public void messageArrived(String topic, MqttMessage message) {
        // New message received
        Log.i(topic, message.toString())
    }
    @Override
    public void deliveryComplete(IMqttDeliveryToken token) {
        // Message delivered
    }
});
```
When the client receives a message from the server, the `messageArrived` method is called. As parameters, it will have topic and `MqttMessage`, which encapsulates the message. By implementing this method, we will be able to process all messages received by the client.

### Disconnecting from the server
To disable the client, call the `disconnect` method of the `MqttAndroidClient`object. In the example below, the token returned
by the `disconnect` method is saved and an `IMqttActionListener` is added to receive a notification about the successful disconnection of the client.
```java
IMqttToken disconToken = client.disconnect();
disconToken.setActionCallback(new IMqttActionListener() {
    @Override
    public void OnSuccessMqtt(IMqttToken asyncActionToken) {
        // we are now successfully disconnected
    }
    @Override
    public void OnFailureMqtt(IMqttToken asyncActionToken, Throwable exception) {
        // something went wrong, but probably we are disconnected anyway
    }
});
```

## My MQTTClient
In your [example](https://github.com/{{ site.github.owner_name }}/{{ page.repository }}) I decided to separate all the methods for working with the MQTT client into a separate class `MQTTClient`, it stores a reference to `MqttAndroidClient` defines the `EventHandler` interface, which allows you to process messages received from the server.
```java
public class MQTTClient
{
    private final MqttAndroidClient client;

    public interface EventHandler {
        void OnConnectedMqtt(();
        void OnReceivingMqttMessage(String topic, MqttMessage message);
        void OnSuccessMqtt(String error);
        void OnFailureMqtt(String error);
    }

    public MQTTClient (@NonNull final Context context, String serverURI, String clientId) {
        if(clientId == null || clientId.isEmpty()) { clientId = MqttClient.generateClientId(); }
        client = new MqttAndroidClient(context, serverURI, clientId);
    }

    // Connect, Subscribe, Unsubscribe, Publish, SetCallback, Disconnect
}
```
Next comes the implementation of the `Connect`, `Subscribe`, `Unsubscribe`, `Publish`, `SetCallback`, `Disconnect` methods, which we discussed above. 
In the `MainActivity` class, all that remains is to implement the `MqttClient.EventHandler` interface, create an instance of `MqttClient`, "hang" `MqttEventHandler` on it, and call the 'Connect' method on this client.
```java
public class MainActivity extends AppCompatActivity implements MQTTClient.EventHandler {
    TextView dataReceived;
    MQTTClient mqttClient;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        dataReceived =  (TextView) findViewById(R.id.dataReceived);

        mqttClient = new MQTTClient(getApplicationContext(), "tcp://192.168.1.11:1883", null);
        mqttClient.SetMqttEventHandler(this);

        mqttClient.Connect("username", "password", this);
    }

    @Override
    public void onStart(){
        super.onStart();
    }

    @Override
    public void OnConnectedMqtt(() {
        Toast.makeText(this, "Connect!", Toast.LENGTH_SHORT).show();

        mqttClient.Subscribe("mqttTest", 1, this);
        mqttClient.Publish("mqttTest", Build.MANUFACTURER + " " + Build.PRODUCT + " connected", this);
    }

    @Override
    public void OnReceivingMqttMessage(String topic, MqttMessage message) {
        dataReceived.setText(topic + ": " + message.toString());
    }

    @Override
    public void OnSuccessMqtt(String info) {
        Log.i("mqttTest", info);
    }

    @Override
    public void OnFailureMqtt(String error) {
        Log.w("mqttTest", error);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        // Closing the connection when exiting
        mqttClient.Unsubscribe("mqttTest", this);
        mqttClient.Disconnect(this);
    }
}
```
For the correct shutdown of the service, when creating the client, I gave `ApplicationContext` as one of the parameters, and added the `Unsubscribe` and `Disconnect` calls to the `onDestroy`method definition.

I added the `Subscribe` and `Publish` methods to the `OnConnectedMqtt` method to subscribe to topic and send a message immediately after connecting.
Thank you for your attention, I hope you found something useful here!

