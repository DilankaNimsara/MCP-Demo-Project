# MCP-Demo-Project

 Kafka Producer (Go)

```go
package main

import (
	"fmt"
	"os"
	"time"

	"github.com/confluentinc/confluent-kafka-go/kafka"
)

func main() {
	producer, err := kafka.NewProducer(&kafka.ConfigMap{
		"bootstrap.servers": "localhost:9092",
	})
	if err != nil {
		panic(err)
	}
	defer producer.Close()

	topic := "demo-topic"

	// Delivery report handler
	go func() {
		for e := range producer.Events() {
			switch ev := e.(type) {
			case *kafka.Message:
				if ev.TopicPartition.Error != nil {
					fmt.Printf("Delivery failed: %v\n", ev.TopicPartition)
				} else {
					fmt.Printf("Delivered message to %v\n", ev.TopicPartition)
				}
			}
		}
	}()

	// Produce a few messages
	for i := 0; i < 5; i++ {
		msg := fmt.Sprintf("Hello Kafka! Message #%d", i)
		err := producer.Produce(&kafka.Message{
			TopicPartition: kafka.TopicPartition{Topic: &topic, Partition: kafka.PartitionAny},
			Value:          []byte(msg),
		}, nil)

		if err != nil {
			fmt.Printf("Failed to produce message: %v\n", err)
		}
		time.Sleep(time.Second)
	}

	// Wait for messages to be delivered
	producer.Flush(5000)
}

```

Kafka Consumer (Go)

```go
package main

import (
	"fmt"
	"os"
	"os/signal"
	"syscall"

	"github.com/confluentinc/confluent-kafka-go/kafka"
)

func main() {
	consumer, err := kafka.NewConsumer(&kafka.ConfigMap{
		"bootstrap.servers": "localhost:9092",
		"group.id":          "demo-group",
		"auto.offset.reset": "earliest",
	})

	if err != nil {
		panic(err)
	}
	defer consumer.Close()

	topic := "demo-topic"
	err = consumer.SubscribeTopics([]string{topic}, nil)
	if err != nil {
		panic(err)
	}

	fmt.Println("Kafka consumer started. Waiting for messages...")

	sigchan := make(chan os.Signal, 1)
	signal.Notify(sigchan, syscall.SIGINT, syscall.SIGTERM)

runLoop:
	for {
		select {
		case sig := <-sigchan:
			fmt.Printf("Caught signal %v: terminating\n", sig)
			break runLoop
		default:
			msg, err := consumer.ReadMessage(-1)
			if err == nil {
				fmt.Printf("Received message: %s on %s\n", string(msg.Value), msg.TopicPartition)
			} else {
				fmt.Printf("Consumer error: %v\n", err)
			}
		}
	}
}


```
