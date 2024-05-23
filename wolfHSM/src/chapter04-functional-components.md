# Library Design / wolfHSM Internals

## Generic Component Architecture

To support easily porting wolfHSM to different hardware platforms and build
environments, each component of wolfHSM is designed to have a common initialization,
configuration, and context storage architecture to allow compile-time, link-
time, and/or run-time selection of functional components. Hardware specifics
are abstracted from the logical operations by associating callback functions
with untyped context structures, referenced as a `void*`.

### Example component initialization

The prototypical compile-time static instance configuration and initialization
sequence of a wolfHSM component is:

```c
#include "wolfhsm/component.h"        /* wolfHSM abstract API reference for a component */
#include "port/vendor/mycomponent.h"  /* Platform specific definitions of configuration
                                       * and context structures, as well as declarations of
                                       * callback functions */ 

/* Provide the lookup table for function callbacks for mycomponent. Note the type
is the abstract type provided in wolfhsm/component.h */
whComponentCb my_cb[1] = {MY_COMPONENT_CB};

/* Fixed configuration data.  Note that pertinent data is copied out of the structure 
 * during init() */
const myComponentConfig my_config = {
    .my_number = 3,
    .my_string = "This is a string",
}

/* Static allocation of the dynamic state of the myComponent. */
myComponentContext my_context[1] = {0};

/* Initialization of the component using platform-specific callbacks */
const whComponentConfig comp_config[1] = {
        .cb = my_cb,
        .context = my_context,
        .config = my_config
    };
whComponentContext comp_context[1] = {0};
int rc = wh_Component_Init(comp_context, comp_config);

rc = wh_Component_DoSomething(comp_context, 1, 2, 3);
rc = wh_Component_CleanUp(comp_context);
```

## Communications

The communication layer of wolfHSM is designed to provide reliable, bidirectional, and packet-based communication between clients and servers. This layer abstracts the underlying transport mechanisms, allowing for flexibility and modularity. A key aspect of wolfHSM communcation is split request and response functions for both client and server, enabling synchronous polling of message reception or asynchronous handling based on interrupt/event support.

### Key Components

- Client/Server APIs: Main interface for communicating between client and server. These are the APIs that are directly used by user applications.
- Comms layer: Defines the format and structure of messages exchanged between clients and servers, and provides an abstract interface to the underlying transport layer implementation, providing a consistend interface for sending and receiving messages.
- Transport Layer: Concrete implementations of the underlying transport. Defines how data is actualy transported between client and server.

### Client/Server APIs:

High level client and server APIs (defined in [wh_client.h](wolfhsm/wh_client.h) and [wh_server.h](wolfhsm/wh_server.h)) are the primary interface for communication. These functions abstract the low level communciations details from the caller, providing a simple split transaction interface for logical operations.

For example, using the client API to send an echo request to the server:

```c
/* send the echo request */
wh_Client_EchoRequest(&clientCtx, sendLen, &sendBuffer));

/* optionally do stuff */

/* poll for the server response */
while (WH_ERROR_NOTREADY == wh_Client_EchoResponse(client, &recv_len, recv_buffer));
```

And using the server API to receive the echo request from the client and send the response
```c
/* pol
```

### Comms Layer

The comms layer encapsulates the messaging structure and control logic to send and rececive data from lower level transports. The comms layer is directly invoked by the higher level client and server APIs. The comms layer provides comm client and comm server abstractions that hold communication state and provide the abstract interface functions to interact with lower level transports. The comms layer API consists of send and receive functions for requests and responses, where the requests and responses pertain to messages rather than high level operations.

Each client is only allowed a single outstanding request to the server at a time.
The server will process a single request at a time to ensure client isolation.

#### Messages

Messages comprise a header with a variable length payload.  The header indicates
the sequence id, and type of a request or response.  The header also provides 
additional fields to provide auxiliary flags or session information. 

```c
/* wolfhsm/wh_message.h */

typedef struct {
    uint16_t magic;
    uint16_t kind;
    uint16_t seq;
    uint16_t size;
} whCommHeader;
```

Messages are used to encapsulate the request data necessary for the server to 
execute the desired function and for the response to provide the results of the
function execution back to the client.  Message types are grouped based on the 
component that is performing the function and uniquely identify which of the
enumerated functions is being performed.  To ensure compatibility (endianness,
and version), messages include a Magic field which has known values used to 
indicate what operations are necessary to demarshall data passed within the 
payload for native processing.  Each functional component has a "remote" 
implementation that converts between native values and the "on-the-wire" message
formats.  The servers ensures the response format matches the request format.

In addition to passing data contents within messages, certain message types also
support passing shared or mapped memory pointers, especially for performance-
critical operations where the server component may be able to directly access
the data in a DMA fashion.  To avoid integer pointer size (IPS) and size_t
differences, all pointers and sizes should be sent as uint64_t when
possible.

Messages are encoded in the "on-the-wire" format using the Magic field of the 
header indicating the specified endianness of structure members as well as the
version of the communications header (currently 0x01).  Server components that 
process request messages translate the provided values into native format, 
perform the task, and then reencode the result into the format of the request.
Client response handling is not required to process messages that do not match
the request format. Encoded messages assume the same size and layout as the
native structure, with the endianness specified by the Magic field.

```c
uint16_t req_magic = wh_COMM_MAGIC_NATIVE;
uint16_t req_type = 123;
uint16_t request_id;
char* req_data = "RequestData";
rc = wh_CommClient_SendRequest(context, req_magic, req_type, &request_id, 
                    sizeof(req_data), req_data);
/* Do other work */

uint16_t resp_magic, resp_type, resp_id, resp_size;
char response_data[20];
while((rc = wh_CommClient_RecvResponse(context,&resp_magic, &resp_type, &resp_id,
                    &resp_size, resp_data)) == WH_ERROR_NOTREADY) {
        /* Do other work or yield */
}
```

Note that transport errors passed into the message layer are expected to be fatal and the
client/server should Cleanup any context as a result.

### Transports

Transports provide intact packets (byte sequences) of variable size (up to a
maximum MTU), to the messaging layer for the library to process as a request or 
response. Transports implement the abstract interface defined by `whTransportCb`
and are invoked directly by the commClient/commServer when needing to send and 
receive data.

Custom transport modules that implement the `whTransportClientCb` interface
can be registered with the server and client and then are automatically used
via the standard server and client request/response functions.

```c
 /* Example transport module */
```

## Non Volatile Memory

The wolfHSM server uses a layered configuration approach to handle Non-Volatile Memory (NVM) operations. The configuration is divided into generic and specific parts, allowing for flexibility and customization.

1. **Generic Configuration (wh_nvm.h):** This header file defines the generic interface for NVM operations. It includes function pointers for NVM operations like `nvm_Read`, `nvm_Write`, `nvm_Erase`, and `nvm_Init`. These function pointers are part of the `whNvmConfig` structure, which is used to bind an actual NVM implementation to the abstract NVM interface.

2. **Specific Configuration (wh_nvm_flash.c, wh_nvm_flash.h):** These files provide a specific implementation of the NVM interface for flash memory. The functions defined here adhere to the function signatures defined in the generic interface, allowing them to be used as the actual implementation for the NVM operations.

The `whServerContext` structure includes a `whNvmConfig` member. This is used to bind the NVM operations to the server context, allowing the server to perform NVM operations using the configured NVM interface.

When initializing the server object to use NVM, the tests would typically follow these steps:

1. Allocate and initialize a `whNvmConfig` structure, providing the specific NVM operations (e.g., from `wh_nvm_flash.c`).
2. Allocate and initialize a `whServerConfig` structure, and set its `nvmConfig` member to the `whNvmConfig` structure initialized in step 1.
3. Allocate a `whServerContext` structure.
4. Initialize the server with the `whServerConfig` structure by calling `wh_Server_Init()`.

This allows the server to use the configured NVM operations, and these operations can be easily swapped out by providing a different `whNvmConfig` structure.

## Cryptographic Operations

## AUTOSAR SHE 

