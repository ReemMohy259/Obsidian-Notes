The SQL Standard defines both the **resultset** and the **cursor descriptor** through the following properties:

- **Scrollability** (the direction in which the result set can be iterated)

- **Sensitivity** (when should data be fetched)

- **Updatability** (available for cursors, it allows the client to modify records while traversing the result set)

- **Holdability** (the result set scope in regard to a transaction lifecycle)


|     **Property Name**      |                                                               **Description**                                                                |
| :------------------------: | :------------------------------------------------------------------------------------------------------------------------------------------: |
|    `TYPE_FORWARD_ONLY`     |           The result set can only be iterated from the ==first to the last== element. This is the default **scrollability** value            |
| `TYPE_SCROLL_INSENSITIVE`  |                     The result set takes a loading time **snapshot** which can be iterated both **forward and backward**                     |
|  `TYPE_SCROLL_SENSITIVE`   |                      The result set is fetched **on demand** while being iterated ==without any direction restriction==                      |
|     `CONCUR_READ_ONLY`     | The result set is just a ==static data projection== which does not allow row-level manipulation. This is the default **changeability** value |
|     `CONCUR_UPDATABLE`     |                        The cursor position can be used to **update or delete records**, or **even insert a new one**                         |
| `CLOSE_CURSORS_AT_COMMIT`  |                                      The result set is **closed** when the current **transaction ends**                                      |
| `HOLD_CURSORS_OVER_COMMIT` |                             The result set remains **open** even after the current transaction is **committed**                              |

Also check:
- [[Fetching Size]]