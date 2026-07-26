
#### Fetching

- Server sends **large chunks of data** (*fetch size* By *fetch size*)
- Driver reads big buffers
- Network is used **efficiently**

>  one truck carrying 1000 boxes

---

#### Streaming

- Driver processes rows as they arrive (**possible pauses waiting for the client to consume data**)
- Internal buffers are **smaller**
- More frequent reads from the network (**sends rows continuously over the socket**)

> many small deliveries instead of one big shipment