# X509Certificates

## 1. Introduction

The **System.Security.Cryptography.X509Certificates** namespace in .NET is part of the System.Security.Cryptography library

It provides classes that help you work with **X.509** certificates

These are digital certificates that follow the **X.509** standard defined by the ITU (International Telecommunication Union)

**X.509** certificates are used in many security protocols, including **SSL/TLS** for securing web communications

Certificates serve as a form of digital identity for entities (like servers, clients, or users) and can be used for various purposes, 

such as encrypting data, signing code, or establishing secure connections over the internet

Key classes and types in the **System.Security.Cryptography.X509Certificates** namespace include:

**X509Certificate and X509Certificate2**: These classes represent an **X.509** certificate

**X509Certificate2** is an enhanced version that provides more functionality than the **X509Certificate** class

**X509Store**: Represents a store of certificates. This can be thought of as a database on the system that holds certificates

**X509Chain**: Used to build a chain of certificates from a given certificate to a **root certificate authority (CA)**. It helps in verifying the validity of a certificate

**X509CertificateCollection**: A collection that stores multiple X.509 certificates

## 2. Sample 1: Establishing an SSL/TLS Connection

A very common use case for **System.Security.Cryptography.X509Certificates** is to establish an **SSL/TLS** connection to a server in a secure manner

This involves validating the **server's SSL certificate** to ensure it's **trusted**, valid, and matches the server you're connecting to

Here's a simplified C# example application that demonstrates how to make an HTTPS request to a **server** while validating

its **SSL certificate** using the **X509Certificate2** class

This example overrides the default server certificate validation to introduce custom logic, which is generally not recommended

for production unless you have a very specific reason and understand the security implications

```csharp
using System;
using System.Net;
using System.Net.Security;
using System.Security.Cryptography.X509Certificates;

class Program
{
    static void Main(string[] args)
    {
        ServicePointManager.SecurityProtocol = SecurityProtocolType.Tls12;
        // Override the default server certificate validation callback
        ServicePointManager.ServerCertificateValidationCallback = ServerCertificateCustomValidation;

        string url = "https://google.com";
        HttpWebRequest request = (HttpWebRequest)WebRequest.Create(url);
        using (HttpWebResponse response = (HttpWebResponse)request.GetResponse())
        {
            Console.WriteLine($"Response status: {response.StatusCode}");
        }
    }

    // Custom validation function
    private static bool ServerCertificateCustomValidation(object sender, X509Certificate certificate, X509Chain chain, SslPolicyErrors sslPolicyErrors)
    {
        // Here you can examine the server's certificate and chain to apply custom logic
        // For demonstration, we're just printing the server certificate details
        Console.WriteLine($"Server Certificate Issuer: {certificate.Issuer}");
        Console.WriteLine($"Server Certificate Subject: {certificate.Subject}");

        // You could also check certificate properties, like expiration, issuer name, etc.
        // For now, we'll trust any certificate (not recommended for production!)
        return true;
    }
}
```

This example demonstrates making an HTTPS request and printing the server's certificate issuer and subject

The custom validation function always returns true, meaning it trusts any certificate

In a real-world scenario, you would replace this logic with proper certificate validation to ensure security

## 3. Sample 2: Creating a certificate for an IoT device and storing it in a file

The following is a simplified C# example demonstrating how to create a **self-signed X.509** certificate and store it in a file

This example might be particularly useful in a development or testing environment for IoT devices

Note that for production environments, certificates should ideally be issued by a **trusted Certificate Authority (CA)**

This example focuses on creating a self-signed certificate and saving it as a **PFX file** (which can contain both the certificate and the private key) and as a DER-encoded .cer file (certificate only)

```csharp
using System;
using System.IO;
using System.Security.Cryptography;
using System.Security.Cryptography.X509Certificates;

class Program
{
    static void Main(string[] args)
    {
        string certificateName = "IoTDeviceSelfSignedCertificate";

        // Create a self-signed certificate
        using (RSA rsa = RSA.Create(2048)) // You might choose other algorithms or key sizes
        {
            var request = new CertificateRequest($"cn={certificateName}", rsa, HashAlgorithmName.SHA256, RSASignaturePadding.Pkcs1);

            // Certificate valid for 1 year from today
            DateTimeOffset start = DateTimeOffset.UtcNow;
            DateTimeOffset end = start.AddYears(1);

            // Using the same certificate for issuer and subject since it's self-signed
            X509Certificate2 certificate = request.CreateSelfSigned(start, end);

            // Export the certificate with the private key as a PFX file
            string pfxPath = Path.Combine(Environment.CurrentDirectory, $"{certificateName}.pfx");
            File.WriteAllBytes(pfxPath, certificate.Export(X509ContentType.Pfx, "password"));

            // Export the certificate without the private key as a DER encoded .cer file
            string cerPath = Path.Combine(Environment.CurrentDirectory, $"{certificateName}.cer");
            File.WriteAllBytes(cerPath, certificate.Export(X509ContentType.Cert));

            Console.WriteLine($"Certificate (PFX) saved to: {pfxPath}");
            Console.WriteLine($"Certificate (.CER) saved to: {cerPath}");
        }
    }
}
```

This code does the following:

- Generates a new **RSA key pair** to be used for the certificate

- Creates a **self-signed** certificate request with the specified subject name, using SHA256 for signing

- Sets the certificate **validity** for 1 year

- Exports the certificate with the private key in a **PFX file**, protected by a password. This file format is commonly used to store both the certificate and its private key securely

- Also exports the certificate without the private key in a DER-encoded **.cer** file, which can be used in situations where only the public part of the certificate is needed

Please ensure you replace "**password**" with a strong, securely generated password in a real application, especially if you're going to use the certificate in a production environment

Also, for production use, consider obtaining certificates from a trusted CA to ensure the security and trustworthiness of your IoT ecosystem

## 4. Sample 3: Integrating X.509 certificates for securing MQTT communication

In the following example we are going to use the **HiveMQ broker**: https://www.hivemq.com/mqtt/public-mqtt-broker/

![image](https://github.com/luiscoco/X509Certificates_sample1/assets/32194879/a1eb42d3-d8a8-4d05-a772-aaabdb3d16fd)

For a quick start, I'll use **HiveMQ's public MQTT** broker for this example, which doesn't require any setup:

**Host**: broker.hivemq.com

**TCP Port**: 1883

**Websocket Port**:	8000

**TLS TCP Port**: 8883

**TLS Websocket Port**: 8884

**No authentication** is needed for this public broker, and it's great for testing

However, remember that it's **public**, so don't send sensitive information through it

In this sample we are going to use **uPLibrary.Networking.M2Mqtt** library for **MQTT** communications with **X.509**

certificate-based security involves configuring the MQTT client to use **SSL/TLS** for encrypted connections

Below is an example that demonstrates how to connect to an **MQTT** broker using **TLS** with an **X.509** certificate for authentication

This example is particularly useful for **IoT** scenarios where secure communication is crucial

First, ensure you have the **M2Mqtt** library in your project. You can add it via NuGet:

```
Install-Package M2Mqtt
```

Here is a sample C# application that creates an **MQTT** client, which uses an **X.509** certificate for a secure connection:

```csharp
using System;
using uPLibrary.Networking.M2Mqtt;
using uPLibrary.Networking.M2Mqtt.Messages;
using System.Security.Cryptography.X509Certificates;

class Program
{
    static void Main(string[] args)
    {
        // Broker URL (use your MQTT broker's hostname or IP address)
        string mqttBrokerUrl = "your_mqtt_broker_url";

        // Path to the CA certificate file (if needed, for broker certificate validation)
        string caCertPath = @"path_to_ca_certificate.cer";

        // Path to the client certificate file (PFX file containing the certificate and private key)
        string clientCertPath = @"path_to_client_certificate.pfx";
        string clientCertPassword = "your_client_certificate_password"; // Password for the PFX file

        // Load the client certificate
        X509Certificate clientCert = new X509Certificate2(clientCertPath, clientCertPassword);

        // Load the CA certificate if your MQTT broker uses a self-signed certificate
        // Comment out the next line if your broker's certificate is signed by a well-known CA already trusted by the system
        X509Certificate caCert = X509Certificate.CreateFromCertFile(caCertPath);

        // Create the MQTT client instance
        MqttClient client = new MqttClient(mqttBrokerUrl, MqttSettings.MQTT_BROKER_DEFAULT_SSL_PORT, true, caCert, clientCert, MqttSslProtocols.TLSv1_2);

        // Register to message received
        client.MqttMsgPublishReceived += Client_MqttMsgPublishReceived;

        // Connect to the broker
        string clientId = Guid.NewGuid().ToString();
        client.Connect(clientId);

        // Subscribe to a topic
        string topic = "your/subscription/topic";
        client.Subscribe(new string[] { topic }, new byte[] { MqttMsgBase.QOS_LEVEL_EXACTLY_ONCE });

        Console.WriteLine("MQTT Client connected and subscribed to topic. Press any key to exit...");
        Console.ReadKey();

        // Clean up
        client.Disconnect();
    }

    static void Client_MqttMsgPublishReceived(object sender, MqttMsgPublishEventArgs e)
    {
        // Handle message received
        string receivedMessage = System.Text.Encoding.UTF8.GetString(e.Message);
        Console.WriteLine($"Message received on topic {e.Topic}: {receivedMessage}");
    }
}
```

Replace placeholders like **your_mqtt_broker_url**, **path_to_ca_certificate.cer**, **path_to_client_certificate.pfx**, 

and **your_client_certificate_password** with your actual **MQTT** broker's details and certificate paths

The **CA certificate** is needed if the broker uses a **self-signed** certificate or a certificate not trusted by the system's certificate store

If your broker's certificate is issued by a **well-known CA**, you might not need to load a CA certificate explicitly

This code sets up an **MQTT client** to connect securely to an MQTT broker using **TLS version 1.2**, subscribes to a topic,

and prints out messages received on that topic

It registers an event handler for processing received messages and uses both CA and client certificates for the **SSL/TLS** connection setup
