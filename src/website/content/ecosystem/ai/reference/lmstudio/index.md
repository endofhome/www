---
category: Reference
type: ecosystem
ecosystem: http4k AI
title: "LMStudio"
description: Feature overview of the http4k AI LMStudio modules
aliases:
    - /ecosystem/http4k/reference/lmstudio/
---
### Installation

```kotlin
dependencies {

    {{< http4k_bom >}}

    // for the low-level LMStudio API client
    implementation("org.http4k:http4k-connect-ai-lmstudio")

    // for the FakeLMStudio server
    implementation("org.http4k:http4k-connect-ai-lmstudio-fake")
}
```

The http4k-ai LmStudio integration provides:

- Low-level API Client
- FakeLmStudio server which can be used as testing harness for either API Client

## Low-level API Client

The LmStudio connector provides the following Actions:

* GetModels
* ChatCompletion
* CreateEmbeddings

New actions can be created easily using the same transport.

The client APIs utilise the LmStudio API Key (Bearer Auth). There is no reflection used anywhere in the library, so
this is perfect for deploying to a Serverless function.

### Example usage

```kotlin
const val USE_REAL_CLIENT = false

fun main() {
    // we can connect to the real service or the fake (drop in replacement)
    val http: HttpHandler = if (USE_REAL_CLIENT) JavaHttpClient() else FakeLmStudio()

    // create a client
    val client = LmStudio.Http(http.debug())

    // all operations return a Result monad of the API type
    val result: Result<Models, RemoteFailure> = client
        .getModels()

    println(result)
}
```

Other examples can be
found [here](https://github.com/http4k/http4k-connect/tree/master/ai/lmstudio/fake/src/examples/kotlin).

## Fake LmStudio Server

The Fake LmStudio provides the below actions and can be spun up as a server, meaning it is perfect for using in test
environments without using up valuable request tokens!

* GetModels
* ChatCompletion

### Generation of responses

By default, a random LoremIpsum generator creates chat completion responses for the Fake. This behaviour can be
overridden to generate custom response formats (eg. structured responses) if required. To do so, create instances of
the `ChatCompletionGenerator` interface and return as appropriate.

### Default Fake port: 58438

To start:

```kotlin
FakeLmStudio().start()
```
