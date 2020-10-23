---
description: 'This is a message type, which doesn''t require response back.'
---

# JustSaying

Example:

```javascript
const network = require('ocore/network.js');
const eventBus = require('ocore/event_bus.js');
const ws = network.getInboundDeviceWebSocket(device_address);

// function parameters: websocket, subject, body
network.sendJustsaying(ws, 'custom', 'some data');

eventBus.on('custom_justsaying', function(ws, content) {
    console.log(content);
};
```

Following is a list of `justsaying` type JSON messages that are sent over the network:

### **Send version information**

```javascript
{
    type: 'justsaying',
    content: {
        subject: 'version', 
        body: {
            protocol_version: protocol_version, 
            alt: alt, 
            library: name, 
            library_version: version, 
            program: program, 
            program_version: program_version
        }
    }
}
```

### **Send free joints**

```javascript
{
    type: 'justsaying',
    content: {
        subject: 'free_joints_end', 
        body: null
    }
}
```

### **Send a private transaction**

```javascript
{
    type: 'justsaying',
    content: {
        subject: 'private_payment', 
        body: privateElement
    }
}
```

### **Share your node WebSocket URL to accept incoming connections**

```javascript
{
    type: 'justsaying',
    content: {
        subject: 'my_url', 
        body: my_url
    }
}
```

### **Ask to verify your WebSocket URL**

```javascript
{
    type: 'justsaying',
    content: {
        subject: 'want_echo', 
        body: random_echo_string
    }
}
```

### **Verify your WebSocket URL with echo message**

```javascript
{
    type: 'justsaying',
    content: {
        subject: 'your_echo', 
        body: echo_string
    }
}
```

### **Log in to Hub**

```javascript
{
    type: 'justsaying',
    content: {
        subject: 'hub/login', 
        body: {
            challenge: challenge,
            pubkey: pubkey,
            signature: signature
        }
    }
}
```

### **Get new messages**

```javascript
{
    type: 'justsaying',
    content: {
        subject: 'hub/refresh', 
        body: null
    }
}
```

### Remove handled message

```javascript
{
    type: 'justsaying',
    content: {
        subject: 'hub/delete', 
        body: message_hash
    }
}
```

### **Send pairing message**

```javascript
{
    type: 'justsaying',
    content: {
        subject: 'hub/challenge', 
        body: challenge
    }
}
```

### **Send message to device**

```javascript
{
    type: 'justsaying',
    content: {
        subject: 'hub/message', 
        body: {
            message_hash: message_hash,
            message: message
        }
    }
}
```

### Ask more messages

```javascript
{
    type: 'justsaying',
    content: {
        subject: 'hub/message_box_status', 
        body: 'has_more'
    }
}
```

### **Light wallet transaction update**

```javascript
{
    type: 'justsaying',
    content: {
        subject: 'light/have_updates', 
        body: null
    }
}
```

### **Add light wallet to monitor address**

```javascript
{
    type: 'justsaying',
    content: {
        subject: 'light/new_address_to_watch', 
        body: address
    }
}
```

### **Send bug report**

```javascript
{
    type: 'justsaying',
    content: {
        subject: 'bugreport', 
        body: {
            message: message,
            exception: exception
        }
    }
}
```

### Push project number \(only accepted from hub\)

```javascript
{
    type: 'justsaying',
    content: {
        subject: 'hub/push_project_number', 
        body: {
            projectNumber: projectNumber
        }
    }
}
```

### New version is available \(only accepted from hub\)

```javascript
{
    type: 'justsaying',
    content: {
        subject: 'new_version', 
        body: {
            version: version
        }
    }
}
```

### **Exchange rates \(only accepted from hub\)**

```javascript
{
    type: 'justsaying',
    content: {
        subject: 'exchange_rates', 
        body: exchangeRates
    }
}
```

### Ask to update \(only accepted from hub\)

```javascript
{
    type: 'justsaying',
    content: {
        subject: 'upgrade_required', 
        body: null
    }
}
```

### Custom JustSaying

You can add your own communication protocol on top of the Obyte one. See event [there](../list-of-events.md#event-for-custom-justsaying).

{% code title="" %}
```javascript
{
    type: 'justsaying',
    content: {
        tag: tag,
        subject: 'custom',
        body: body
    }
}
```
{% endcode %}

