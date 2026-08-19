# Service Bus Demo

An interactive, browser-based walkthrough of core Azure Service Bus concepts.

## Run the app

No build step or package installation is required.

1. Open [service-bus-demo.html](service-bus-demo.html) in a modern web browser.
2. Use the tabs at the top of the page to explore the demonstrations.
3. Keep the browser connected to the internet if you want the imported Google Fonts to load.

For local development, VS Code's **Live Server** extension is a convenient option:

```sh
# Open service-bus-demo.html with Live Server from VS Code.
```

## What the app covers

This demo visually explains how a service bus decouples services, routes messages, and isolates delivery failures. The first three tabs provide the core walkthrough.

### Tab 1 — Why

Compares direct, point-to-point integrations against using a service bus.

- Switch between **Point-to-point** and **With a bus** layouts.
- Add or remove services to see how direct integrations grow rapidly.
- See how a bus reduces the connection model from approximately $N(N-1)/2$ direct connections to $N$ connections.

### Tab 2 — How it moves

Demonstrates the delivery patterns a bus supports.

- **No bus:** a sender calls each interested service directly.
- **Queue:** one eligible consumer receives each message, distributing work across consumers.
- **Topic – broadcast:** every subscription receives its own copy of a message.
- **Topic – filtered:** subscriptions receive only messages matching their configured tag or filter.
- Select a producer service, choose a delivery mode, and send messages to inspect the delivery log.

### Tab 3 — When it fails

Shows how message delivery failures are isolated and handled.

- Mark a consumer service as unhealthy.
- Send a message from **Orders**.
- Healthy consumers receive their copies normally.
- The unhealthy consumer is retried three times, then its message is moved to the dead-letter queue (DLQ).
- Review failed messages in the DLQ, retry them after recovery, or discard them.

## Video walkthrough

https://github.com/user-attachments/assets/3eb9c734-6cc2-4663-adc2-216d85ed4296

