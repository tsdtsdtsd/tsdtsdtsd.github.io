+++
date = '2025-02-24T21:30:14+01:00'
modified = '2025-02-24T21:30:14+01:00'
title = 'Modeling Go Structs and Functionality for SQS Messages'
draft = true

categories = ['Development']
tags = ['go', 'aws', 'sqs']
+++

Recently we built a background processing service, which uses AWS SQS as its queue. I needed to model our data as well as SQS functionality around them in Go and wanted to share my experience and with what I came up with in the end.

In this post I want to cover following imaginary scenario:
- a system needs the ability to execute a large amount of webhooks asynchronously in the background
- there are many targets for the webhooks
- payloads should be pretty generic in the first iteration
- we decide to create a service which processes jobs from a SQS queue and executes those webhooks

### The Data

SQS messages are pretty simple: put some data in, get some data out. Simple, but also restrictive:

> A message can include only XML, JSON, and unformatted text. The following Unicode characters are allowed. For more information, see the W3C specification for characters.
> 
> `#x9` | `#xA` | `#xD` | `#x20` to `#xD7FF` | `#xE000` to `#xFFFD` | `#x10000` to `#x10FFFF`
> 
> Amazon SQS does not throw an exception or completely reject the message if it contains invalid characters. Instead, it replaces those invalid characters with `U+FFFD` before storing the message in the queue, as long as the message body contains at least one valid character.
> 
> <cite>source: [API Reference](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_SendMessage.html)</cite>

Let's create a simple struct for our data and prepare it for JSON:

```go
package message

type WebhookJob struct {
    TargetID int `json:"target-id"`
    Payload  []byte `json:"payload"`
}
```

This struct represents the data towards our application and we have a definition of how the data should be (un)marshaled towards SQS. But it also only represents a message body. In this example we want to add message attributes as additional data.

AWS SQS provides the option to attach attributes to a message, such attributes are a nice way to differentiate between *data* and *metadata*. A good example could be a tracing ID for an open tracing integration.

So let's create a new entity for attributes and add basic functionality:

```go
type Attributes struct {

}

func (attr *Atributes) SetStringAttribute() {

}

func (attr *Atributes) GetStringAttribute() string {

}
```

Great, `Attributes` encapsulates functionality around message attributes and can now be embedded into `WebhookJob`:

```go
type WebhookJob struct {
    Attributes `json:"-"` // Embed Attributes, drop field in JSON

    TargetID int `json:"target-id"`
    Payload []byte `json:"payload"`
}
```

Using the things we created up until now is straight forward:

```go
job := &WebhookJob {
    TargetID: targetID,
    Payload: payload,
}

job.SetStringAttribute("some-id", []byte("abc123"))
```

```go
func NewWebhookJob(targetID int, payload []byte) *WebhookJob {

    job := &WebhookJob{
        TargetID: targetID,
        Payload: payload,
    }

    return job
}
```

All code mentioned above is just exemplary and could certainly be improved, like making `Attributes` thread safe. Please don't treat this post as "the answer", but rather as a starting point and collection of ideas. 