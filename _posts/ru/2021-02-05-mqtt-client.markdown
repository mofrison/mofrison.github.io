---
title:  "MQTT-клиент для Androud"
repository: mqtt-client
preview: /assets/images/posts/2021-02-05-mqtt-client/mqtt.jpg
date:   2021-02-05 10:00:00 +0300
categories: ru cases
tags: [java, android, mqtt]
lang: Ru
layout: post
---

Сегодня расскажу, как, во время написания [дипломной работы]({{site.url}}/ru/projects/mobile-monitoring-system), мною был реализован MQTT-client для android приложения. Сделать это не сложно, но подробной информации на тот момент на родном для меня языке было крайне мало, и я решил это исправить.

## Что для этого нужно:
* Среда работки **Android Studio**
* Смартфон под управлением ОС **Android**
* **MQTT**-сервер (или брокер)

## Подключение библиотеки **Paho Android Service**
Первым делом подключим библиотеки, необходимые для настройки клиента и сервиса MQTT в приложении Android. Мы будем использовать клиент **Paho MQTT** и Android-сервис, предоставляемый **Eclipse**. Android использует **Gradle** в качестве системы сборки и управления зависимостями, поэтому служба **Paho Android Service** может быть добавлена в приложение через Gradle. Чтобы подключить последнюю версию, необходимо добавить следующю запись в файл `build.gradle` Android-проекта:
```gradle
repositories {
    maven {
        url "https://repo.eclipse.org/content/repositories/paho-snapshots/"
    }
}
```
Она содержит ссылку на репозиторий выпуска Paho в конфигурацию gradle, чтобы Gradle мог найти упакованный **Paho Android Service JAR**.

В `/app/build.gradle` приложения добавим запись, которая подключает службу Paho Android Service для настройки зависимости приложения:
```gradle
dependencies {
    implementation 'androidx.legacy:legacy-support-v4:1.0.0' // Требуется, для Android версии 4.x
    implementation 'org.eclipse.paho:org.eclipse.paho.client.mqttv3:1.1.0'
    implementation ('org.eclipse.paho:org.eclipse.paho.android.service:1.1.1')
}
```
Зависимость `androidx.legacy:legacy-support-v4:1.0.0` необходимо только в том случае, если приложение расчитано на версии Android 4.x.

## Настройка файла AndroidManifest
Прежде чем начать писать код, необходимо настроить `AndroidManifest.xml` файл, чтобы приложение получило разрешения на доступ в Интернет, доступ к состоянию сети и приложение могло работать как сервис. Добавим эти строки перед открывающим тегом `<application>`.
```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>
<uses-permission android:name="android.permission.WAKE_LOCK"/>
```

Paho Android Service инкапсулирует соединение MQTT и предлагает API для этого. Чтобы создать привязку к Paho Android Service, сервис должен быть объявлен в `AndroidManifest.xml`. Для этого уже внутри тега `<application>` нужно добавить следующую запись:
```xml
<service android:name="org.eclipse.paho.android.service.MqttService" />
```

## Работа с MQTT-сервисом 
Библиотека _paho.client.mqttv3_ от **Eclipse** даёт возможность взаимоедействия с MQTT-сервером через экземпляр класса `MqttAndroidClient`. Для его создания требуются два обязательных параметра: `Context` и строка с URI-адресом вида _tcp://192.168.1.11:1883_. Важным параметром является `clientId`, который должен быть уникальным для каждого устройства.
```java
MqttAndroidClient client = new MqttAndroidClient(context, serverURI, clientId);
```

Объект класса `MqttAndroidClient` предоставляет методы, с помощью которых можно подключится к **MQTT**-серверу или разорвать соединение, подписаться на одну тему(topic) или несколько, а так же отписываться от тем, отправлять и получать сообщения. Некоторые из предоставляемых методов(`connect`, `subscribe`, `unsubscribe`, `disconnect`) возвращают `IMqttToken`. На него, с помощью метода `setActionCallback` можно повесить объект с реализацией интерфейса `IMqttActionListener`, который содержит всего два метода: `onSuccess` и `onFailure`. Они вызываются в зависимости от результата операции и будут служить связующим звеном нашего приложения и сервера.

### Настройка подключения
С помощью объекта типа `MqttConnectOptions` можно передать дополнительные опции. Например в качестве опций можно передать версию MQTT 3.1:
```java
MqttConnectOptions options = new MqttConnectOptions();
options.setMqttVersion(MqttConnectOptions.MQTT_VERSION_3_1);
```

Так же через опции можно передать параметры функции _Last Will And Testament_ (**LWT**), которая используется в MQTT для уведомления других клиентов о беспристрастно отключенном клиенте.
```java
String topic = "users/last/will";
byte[] payload = "some payload".getBytes();
options.setWill(topic, payload ,1,false);
```

Если для подключения к брокеру требуется ввести имя пользователя и пароль, их можно добавить к экземпляру `MqttConnectOptions`:
```java
options.setUserName(userName);
options.setPassword(password.toCharArray());
```

### Подключение к серверу
Прежде чем осуществлять какое-либо взаимодействие с сервером, к нему нужно создать подключение. 
Для этого мы возьмём ранее созданный экземпляр `MqttAndroidClient` и вызовем на нём метод `connect`. В этот метод можно передать заранее определенные опции, ссылку на `Context` и обратный вызов `IMqttActionListener`.
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

При вызове метода сonnect, `MqttAndroidClient` асинхронно попытается подключиться к брокеру MQTT и вернуть token. Этот token можно использовать для регистрации обратных вызовов, чтобы получать уведомления при подключении MQTT-соединения или возникновении ошибки.
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

### Подписаться на topic
Чтобы получать сообщения от брокера или внешнего устройства необходимо подписаться на соответсвующий **topic**. Для этого служит метод `MqttAndroidClient.subscribe`. Метод принимает полное имя topic'a и **[QoS](https://ru.wikipedia.org/wiki/QoS)** в качестве параметров, и также возвращает **IMqttToken**. Token можно использовать для отслеживания успешной или неудачной подписки.
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
Метод так же может принимать объект типа `IMqttMessageListener` с функцией `messageArrived`,но на момент написания этого поста, он не работал.
Я сообщил об ошибке возможно сейчас её уже исправили. 

### Отписаться от topic'a
Обратным методу `subscribe` является метод `unsubscribe`.
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

### Публикация сообщений
**MqttAndroidClient** позволяет публиковать сообщения с помощью метода `publish(topic, MqttMessage)`. MqttAndroidClient не ставит в очередь все сообщения.
Если клиент не подключен, метод выдаст ошибку при попытке отправить сообщение.
После первого подключения, когда соединение MQTT установлено успешно, вызывается метод `ImqttActionListener.onSuccess`.
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
Для отправки сохраняемого сообщения **MQTT** используется атрибут `MqttMessage.retained`. Он может быть установлен в **true** через `MqttMessage.setRetained(true)`.

### Получение сообщений
Чтобы получать и обрабатывать сообщения задействуем `MqttCallback`, котрый можно повесить на экземпляр `MqttAndroidClient` ещё до подключения:
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
Когда клиент получит от сервера сообщение, будет вызван метод `messageArrived`. В качестве параметров у него будет topic и `MqttMessage`, который инкапсулирует сообщение. Реализовав это метод мы сможем обрабатывать все сообщения полученные клиентом.

### Отключение от сервера
Для отключения клиента необходимо вызвать метод `disconnect` объекта `MqttAndroidClient`. В приведенном ниже примере сохраняется токен, возвращенный методом `disconnect`, и добавляется `IMqttActionListener` для получения уведомления об успешном отключении клиента.
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

## Мой MQTTClient
В своём [примере](https://github.com/{{ site.github.owner_name }}/{{ page.repository }}) я решил выделить все методы для работы с MQTT-клиентом в отдельный класс `MQTTClient`, он хранит в себе сылку на `MqttAndroidClient` определяет интерфейс `EventHandler`, который позволяет обрабатывать сообщения полученные от сервера. 
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
Далее идёт реализация методов `Connect`, `Subscribe`, `Unsubscribe`, `Publish`, `SetCallback`, `Disconnect`, которую мы рассматривали выше.

В классе `MainActivity` остаётся только реализовать интерфейс `MQTTClient.EventHandler`, создать экземляр `MQTTClient`, "повесить" на него `MqttEventHandler`, и вызвать на этом клиенте метод `Connect`.
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
Для корректного завершения работы сервиса, при создании клиента, в качестве одного из параметров, я предал `ApplicationContext`, а в определение метода `onDestroy` добавил вызов `Unsubscribe` и `Disconnect`.

Вызов методов `Subscribe` и `Publish` я добавил в метод `OnConnectedMqtt`, чтобы подписаться на topic и отправить сообщение сразу после подключения.

Спасибо за внимание, надеюсь вы нашли здесь что то полезное!

