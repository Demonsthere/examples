# Description

This is an example scenario which demonstrates how Kyma can receive Cloud Events from an external solution. For enabling developers, this scenario is targeted for a **local Kyma installation**.

The external solution provides events which is mocked in this example scenario.

* Sending of events is mocked by using a HTTP Client.

You can use any other real solutions or mocks that you want to try.

The runtime flow involves following steps:

* External solution sends an `order.created` event.
* The event is accepted by Kyma.
* Event triggers the deployed lambda.
* Lambda prints the CE context attributes

# Set up

## Start Kyma locally

* Refer these [instructions](https://github.com/kyma-project/kyma/blob/master/docs/kyma/docs/030-inst-local-installation-from-release.md)

## Create Kyma namespace

* Throughout this workshop we use `workshop` namespace so please create it first.
* If you desire to use a different namespace then change each `workshop` here with your desired namespace name.

## Create Application

* Under `Integration`, create Application.
* Name it `sample-external-solution`.

## Establish secure connection between external solution and Kyma

We will set up the secure connectivity between the mock external solution and the Kyma application `sample-external-solution`.

Here we will use the [connector service](https://github.com/kyma-project/kyma/blob/master/docs/application-connector/docs/010-architecture-connector-service.md) to secure the communication.

* Clone this [repository](https://github.com/janmedrek/one-click-integration-script).

* Copy the token
  * Navigate to the `Integration --> Applications --> sample-external-solution`
  * `Connect Application`
  * Copy the token to the clipboard
  
* Use the one-click-generation [helper script](https://github.com/janmedrek/one-click-integration-script) to generate the certificate

  ```bash
  ./one-click-integration.sh -u <paste token here>
  ```

  This will generate a `generated.pem` file. This contains the signed certificate and key that we will use for subsequent communications.

  > **NOTE** The token is short-lived. So either hurry up or regenrate the token

  ```bash
  export NODE_PORT=$(kubectl get svc -n kyma-system application-connector-ingress-nginx-ingress-controller -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
  ```

## Mock External solution

* Calls from external solution to kyma
  * Use any http client e.g curl, httpie, postman etc.

## Register Events

* Register events via the metadata api using [register-events.json](./register-events.json)

    ```bash
    # Using httpie
    http POST https://gateway.kyma.local/sample-external-solution/v1/metadata/services --cert=generated.pem --verify=no < register-events.json
    # Using curl
    curl -X POST -H "Content-Type: application/json" -d @./register-events.json https://gateway.kyma.local/sample-external-solution/v1/metadata/services --cert generated.pem -k
    ```

* Verify
  * Navigate to Administration --> Applications --> sample-external-solution
  * You should see the registered events.

## Bind Application

* Navigate to Administration --> Applications --> sample-external-solution
* Create a binding with the Kyma namespace `workshop`.

## Add events and APIs to the Kyma Namespace

* Navigate to Service Catalog for Kyma namespace `workshop`

* You should see the registered events available.
* Click on the details
* Add to your namespace
* Check under `Service Instances` if it appears

## Create Lambda

* Create the lambda definition in `lambda ui` using [lambda.js](./lambda.js)
  * Add label `app:<name-of-the-lambda>`
* Select function trigger as event trigger, then choose `order.created`

# Runtime

## Publish the event

```go
package main

import (
    "context"
    "crypto/tls"
    "crypto/x509"
    "io/ioutil"
    "log"
    "net/http"
    "time"

    cloudevents "github.com/cloudevents/sdk-go"
    "github.com/google/uuid"
)

func main() {
    event := cloudevents.NewEvent()
    event.Context = cloudevents.EventContextV03{}.AsV03()
    event.SetID(uuid.New().String())
    event.SetType("order.created")
    event.SetSource("external-application")
    event.SetTime(time.Now())
    event.SetExtension("eventtypeversion", "v1")
    event.SetDataContentType("application/json")
    event.SetData(23)

    // Add path to client certificate and key
    cert, err := tls.LoadX509KeyPair("generated.crt", "generated.key")
    if err != nil {
        log.Fatalln("Unable to load cert", err)
    }
    clientCACert, err := ioutil.ReadFile("generated.crt")
    if err != nil {
        log.Fatal("Unable to open cert", err)
    }

    clientCertPool := x509.NewCertPool()
    clientCertPool.AppendCertsFromPEM(clientCACert)

    tlsConfig := &tls.Config{
        Certificates:       []tls.Certificate{cert},
        RootCAs:            clientCertPool,
        InsecureSkipVerify: true,
    }

    tlsConfig.BuildNameToCertificate()

    client := &http.Client{
        Transport: &http.Transport{TLSClientConfig: tlsConfig},
    }
    t, err := cloudevents.NewHTTPTransport(
        cloudevents.WithTarget("https://gateway.kyma.local/sample-external-solution/v2/events"),
        cloudevents.WithStructuredEncoding())

    t.Client = client
    c, err := cloudevents.NewClient(t)
    if err != nil {
        panic("unable to create cloudevent client: " + err.Error())
    }
    _, err = c.Send(context.Background(), event)
    if err != nil {
        panic("failed to send cloudevent: " + err.Error())
    }
}
```

# Verification

* **Lambda was triggered**
  * Inspect the pod logs for lambda in `workshop` namespace

# Tracing

* Traces are available under <https://jaeger.kyma.local/>
