# Why Every Distributed System Needs an Idempotency Key

Imagine you click the **"Pay Now"** button and your internet connection drops. Unsure whether the payment succeeded, you click the button again.

What should happen?

Ideally, the payment should be processed only once. Without proper safeguards, however, the system may create two payment transactions, resulting in duplicate charges.

![1.duplicate-transaction.png](https://github.com/PiyushMittl/Others/blob/main/idempotent-key/images/1.duplicate-transaction.png)

This is where **idempotency keys** come in.

An idempotency key is a unique identifier associated with a request. When the server receives a request, it stores the key along with the result of the operation. If the same request arrives again with the same key, the server doesn't process it a second time. Instead, it returns the original response.

Consider the following payment request:

```json
{
  "name": "Piyush",
  "bank": "HDFC",
  "amount": 100,
  "idempotencyKey": "o2nsP2"
}
```

### First Request

The client sends the request above.

The server:

1. Checks whether `o2nsP2` already exists.
2. Doesn't find it.
3. Transfers ₹100 to the specified account.
4. Stores the idempotency key and response.

| Idempotency Key | Transaction Id | Status  |
| --------------- | -------------- | ------- |
| o2nsP2          | 123           | Success |

The server then returns:

```json
{
  "transactionId": "123",
  "status": "Success"
}
```

![2.failed-request.png](https://github.com/PiyushMittl/Others/blob/main/idempotent-key/images/2.failed-request.png)

### Retry Request

Suppose the client never receives the response because of a network timeout. The user clicks **"Pay Now"** again.

The application sends the exact same request:

```json
{
  "name": "Piyush",
  "bank": "HDFC",
  "amount": 100,
  "idempotencyKey": "o2nsP2"
}
```

This time, the server finds `o2nsP2` in its store and realizes the payment has already been processed.

Instead of transferring another ₹100, it simply returns the previously stored response:

```json
{
  "transactionId": "123",
  "status": "Success"
}
```
![3.retried-request.png](https://github.com/PiyushMittl/Others/blob/main/idempotent-key/images/3.retried-request.png)

As a result:

* Only one payment is processed.
* The client can safely retry.
* Duplicate transactions are avoided.

This simple mechanism solves many real-world problems:

* Duplicate payment transactions
* Multiple order creations
* Repeated resource provisioning
* Retry storms caused by transient failures

![4.complete-idempotency.png](https://github.com/PiyushMittl/Others/blob/main/idempotent-key/images/4.complete-idempotency.png)

Idempotency becomes especially important in distributed systems because failures are normal. Networks are unreliable, requests can time out, and clients often retry operations automatically.

A common misconception is that retries improve reliability on their own. In reality, retries without idempotency can make systems less reliable by creating duplicate operations.

Many large-scale systems, including payment gateways and cloud platforms such as Azure Resource Manager (ARM), rely heavily on idempotent operations to ensure requests can be retried safely.

The next time you design an API that creates or modifies data, ask yourself:

**"What happens if the same request is received twice?"**

If the answer is "something bad," it's time to add an idempotency key.
