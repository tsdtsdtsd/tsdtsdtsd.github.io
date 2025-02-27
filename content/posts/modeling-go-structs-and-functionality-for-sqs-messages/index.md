+++
date = '2025-02-27T17:58:28+01:00'
modified = '2025-02-27T17:58:28+01:00'
title = 'Modelling Go Structs and Functionality for SQS Messages'
description = 'Recently we built a background processing service, which uses AWS SQS as its queue. I needed to model our data and the SQS functionality in Go, and I’d like to share my experience and final solution.'

categories = ['Development']
tags = ['go', 'aws', 'sqs']
+++

Recently we built a background processing service, which uses AWS SQS as its queue. I needed to model our data and the SQS functionality in Go, and I’d like to share my experience and final solution.

In this post I want to cover following imaginary scenario:
- a system needs the ability to execute a large amount of webhooks asynchronously in the background
- there are many targets for the webhooks
- payloads should be pretty generic in the first iteration
- we decide to create a service which processes jobs from a SQS queue and executes those webhooks

*All code mentioned in this post is just exemplary and could certainly be improved a lot. Please don't treat this post as "the answer", but rather as a starting point and collection of ideas.*

## Data

SQS messages are pretty simple: put some data in, get some data out. Simple, but also restrictive:

> A message can include only XML, JSON, and unformatted text. The following Unicode characters are allowed. For more information, see the W3C specification for characters.
> 
> `#x9` | `#xA` | `#xD` | `#x20` to `#xD7FF` | `#xE000` to `#xFFFD` | `#x10000` to `#x10FFFF`
> 
> Amazon SQS does not throw an exception or completely reject the message if it contains invalid characters. Instead, it replaces those invalid characters with `U+FFFD` before storing the message in the queue, as long as the message body contains at least one valid character.
> 
> <cite>Source: [API Reference](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_SendMessage.html)</cite>

Let's create a simple struct for our data and prepare it for JSON:

```go
package message

type WebhookJob struct {
    TargetID int    `json:"target-id"`
    Payload  []byte `json:"payload"`
}
```

This struct represents the data towards our application and we have a definition of how the data should be (un)marshaled towards SQS. But it also only represents a message body.

## Functionality

AWS SQS provides the option to attach attributes to a message. Such attributes are a nice way to differentiate between *data* and *metadata*. A good example could be a tracing ID for an open tracing integration.

So let's use message attributes as an example functionality and create a new entity for attributes with basic functionality:

```go
package message

import (
    "github.com/aws/aws-sdk-go-v2/aws"
    "github.com/aws/aws-sdk-go-v2/service/sqs/types"
)

type Attributes struct {
    data map[string]types.MessageAttributeValue
}

// SetStringAttribute sets given key to value 
func (attr *Attributes) SetStringAttribute(key string, value string) {
    if attr.data == nil {
		attr.data = make(map[string]types.MessageAttributeValue)
	}

    attr.data[key] = types.MessageAttributeValue{
		DataType:    aws.String("String"),
		StringValue: aws.String(value),
	}
}

// GetStringAttribute returns the value at given key.
// If the key does not exist, an empty string is returned instead.
func (attr *Attributes) GetStringAttribute(key string) string {
    if value, exists := attr.data[key]; exists && value.StringValue != nil {
        return *value.StringValue
    }
    return ""
}

// SetAttributes overwrites attributes with given set
func (attr *Attributes) SetAttributes(data map[string]types.MessageAttributeValue) {
    attr.data = data
}

// GetAttributes returns the complete map of attributes
func (attr *Attributes) GetAttributes() map[string]types.MessageAttributeValue {
    return attr.data
}
```

Great, `Attributes` encapsulates functionality around message attributes and can now be embedded into `WebhookJob`:

```go
type WebhookJob struct {
    Attributes `json:"-"` // embed Attributes & drop field in JSON

    TargetID int   `json:"target-id"`
    Payload []byte `json:"payload"`
}
```
With `json:"-"` we also make sure that `Attributes` will not be marshaled.  
Using the things we created up until now is straightforward:

```go
msg := &message.WebhookJob {
    TargetID: targetID,
    Payload: payload,
}

msg.SetStringAttribute("some-id", "abc123")
```

Surely we could have put all functionality straight into `WebhookJob`, but using this approach of embedding promotes code reuse across different message types that we may have.

## Receiving Messages

Fetching from SQS via [`Client.ReceiveMessage`](https://pkg.go.dev/github.com/aws/aws-sdk-go-v2/service/sqs@v1.37.15#Client.ReceiveMessage) gives us [`*ReceiveMessageOutput`](https://pkg.go.dev/github.com/aws/aws-sdk-go-v2/service/sqs@v1.37.15#ReceiveMessageOutput), which contains a list of messages `[]types.Message`. It's always a slice as we have the option to receive up to 10 messages from the queue.

A [`Message`](https://pkg.go.dev/github.com/aws/aws-sdk-go-v2/service/sqs@v1.37.15/types#Message) contains the messages `Body` and `MessageAttributes`. The quick solution for us is to create a constructor for our `WebhookJob` type, so we can easily ingest the raw message from SQS:

```go
func NewWebhookJobFromRawMessage(rawMsg types.Message) (*WebhookJob, error) {
    msg := new(WebhookJob)
	err := json.Unmarshal(StringPtrToByteSlice(rawMsg.Body), msg)
	if err != nil {
		return msg, err
	}

	msg.SetAttributes(rawMsg.MessageAttributes)
	return msg, nil
}
```

## Sending Messages

When sending messages with the SDK, we need to pass a [`*SendMessageInput`](https://pkg.go.dev/github.com/aws/aws-sdk-go-v2/service/sqs@v1.37.15#SendMessageInput) to the [`Client.SendMessage`](https://pkg.go.dev/github.com/aws/aws-sdk-go-v2/service/sqs@v1.37.15#Client.SendMessage) method, where the message body is stored as `*string`.  
What can be helpful here is to implement `fmt.Stringer` on `WebhookJob` to easily retrieve marshaled JSON, like this:

```go
// String returns the message body as marshaled JSON
// Implements the fmt.Stringer interface.
func (msg *WebhookJob) String() string {
    data, err := json.Marshal(msg)
	if err != nil {
		slog.Error("json.Marshal failed on WebhookJob", "error", err)
		return ""
	}

	return string(data)
}
```

Now we can quickly build the necessary `*SendMessageInput`:

```go
input := &sqs.SendMessageInput{
    QueueUrl:          aws.String(myQueueURL),
    MessageBody:       aws.String(msg.String()),
    MessageAttributes: msg.GetAttributes(),
}
```

## What's Next?

- there are a lot more things we could add, like passing the messages `ReceiptHandle` for later use
- depending on how you process your messages, you may need to add thread safety with mutexes
- sending [messages in batches](https://pkg.go.dev/github.com/aws/aws-sdk-go-v2/service/sqs@v1.37.15#Client.SendMessageBatch) needs different input types
- utilizing generics for different kinds of messages
- utilizing *options pattern* for nicer constructor functions
- error handling, logging, default return values