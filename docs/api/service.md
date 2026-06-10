
# Service

<!-- Auto-generated — do not edit. Regenerate with: doxygen Doxyfile && python3 generate_api_reference.py -->


### `agnocast::Service<ServiceT>`

Service server for zero-copy Agnocast service communication.

**Example:**

```cpp
using SrvT = example_interfaces::srv::AddTwoInts;
using Request = SrvT::Request;
using Response = SrvT::Response;

auto service = agnocast::create_service<SrvT>(
  this, "add_two_ints",
  [this](const agnocast::ipc_shared_ptr<Request> & req,
         const agnocast::ipc_shared_ptr<Response> & res) {
    res->sum = req->a + req->b;
  });
```


---

#### `send_response()`

```cpp
void Service::send_response(agnocast::ipc_shared_ptr<typename ServiceT::Request> &&request, agnocast::ipc_shared_ptr<typename ServiceT::Response> &&response)
```

Sends a response to the client that initiated the service call. This function is expected to be used in deferred response callbacks. response must be the object returned by borrow_loaned_response(). The entire borrow_loaned_response() -> populate -> send_response() sequence must run on the same thread (typically in a single callback).

| Template Parameter | Description |
|-----------|-------------|
| `ServiceT` | ROS service type. |
| **Parameter** | **Description** |
| `request` | The request that initiated the service call. |
| `response` | The response to send. Must be acquired by calling borrow_loaned_response(). |


---

#### `borrow_loaned_response()`

```cpp
agnocast::ipc_shared_ptr<typename ServiceT::Response> Service::borrow_loaned_response(agnocast::ipc_shared_ptr<typename ServiceT::Request> &request)
```

Allocate a service response message in shared memory. This function is expected to be used in deferred response callbacks. This function does not consume request. In deferred callbacks, keep request and pass it to send_response() after populating the returned response.

| Template Parameter | Description |
|-----------|-------------|
| `ServiceT` | ROS service type. |
| **Parameter** | **Description** |
| `request` | The request that initiated the service call. |
| | |
| **Returns** | Owned pointer to the response message in shared memory. |

