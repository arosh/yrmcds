Server-side locking support.

Motivation
----------

Locking is one of the things people often want to have with memcached.

Though `add` has been used to implement locks at client-side [1][] [2][],
this approach is fragile if clients are not very stable;  if a client
acquires a lock then dies, the lock will never be released.  Moreover,
objects used to implement locks may be evicted.

yrmcds introduces a server-side locking mechanism to resolve this problem.

Overview
--------

A lock associates an existing object and a client connection.  Locked
objects will neither be expired nor be evicted.  Modification commands
such as `set`, `replace`, or `delete` will fail if the object is locked,
unless the client is holding the lock.  `get` succeeds even though the
object is locked by another client.  If a client is disconnected, all locks
held by the client will be released.

Although lock commands are available in both ASCII and binary protocols,
use of binary protocol is strongly recommended.

ASCII protocol
--------------

* `lock KEY\r\n`

    acquires a server-side lock of the object associated with KEY.
    The response will be one of:

    - "OK\r\n", to indicate success.
    - "LOCKED\r\n", if another client has already locked the object.
    - "NOT_FOUND\r\n", if the object does not exist.

* `unlock KEY\r\n`

    releases a server-side lock.  The response will be one of:

    - "OK\r\n", to indicate success.
    - "CLIENT_ERROR MESSAGE\r\n", if the object is not locked by the client,
      or if the object does not exist.

* `unlock_all\r\n`

    releases all locks held by the client.  The response will be "OK\r\n".

Additionally, modification commands such as `set` and `delete` will fail
when the object is locked by another client.  The response will be

"LOCKED\r\n"

in this case.

Binary protocol
---------------

The binary protocol is enhanced with new `Lock`, `Unlock`, `UnlockAll`,
`LaG` (lock and get), `LaGK` (lock and get with key), and
`RaU` (replace and unlock) commands.

New opcodes are:

    - 0x40 (Lock)
    - 0x41 (LockQ)
    - 0x42 (Unlock)
    - 0x43 (UnlockQ)
    - 0x44 (UnlockAll)
    - 0x45 (UnlockAllQ)
    - 0x46 (LaG)
    - 0x47 (LaGQ)
    - 0x48 (LaGK)
    - 0x49 (LaGKQ)
    - 0x4a (RaU)
    - 0x4b (RaUQ)

New response statuses are:

    - 0x0010 Locked
    - 0x0011 Not locked

### Lock, Lock Quietly

Request:

    - MUST NOT have extras.
    - MUST have key.
    - MUST NOT have value.

Lock an object.  The response-packet contains no extra data, and the result
of the operation is signaled through the status code.  Specifically,
0x0001 (not found) is returned when the object does not exist, and 0x0010
(locked) is returned when the object has been locked by another client.

### Unlock, Unlock Quietly

Request:

    - MUST NOT have extras.
    - MUST have key.
    - MUST NOT have value.

Unlock a locked object.  The response-packet contains no extra data, and
the result of the operation is signaled through the status code. Specifically,
0x0001 (not found) is returned when the object does not exist, and 0x0011
(not locked) is returned when the object is not locked by the client.

### Unlock All, Unlock All Quietly

Request:

    - MUST NOT have extras.
    - MUST NOT have key.
    - MUST NOT have value.

Unlock all locked objects held by the client.  The response-packet
contains no extra data, and the result of the operation is signaled
through the status code.  In fact, this command will succeed unconditionally.

### Lock and Get, Lock and Get Quietly, Lock and Get Key, Lock and Get Key Quietly

Request:

    - MAY have extras.
    - MUST have key.
    - MUST NOT have value.

Response (if found):

    - MUST have extras.
    - MAY have key.
    - MAY have value.

Lock an object and get the data atomically.

Optional extra data for Lock and Get request is 4 byte expiration time.
If extra data exists, the object's expiration time will be renewed.

The response is the same as that of `Get` or `GetK` command.  Specifically,
if the object is locked, the response with status code = 0x0010 (locked)
will be returned.  `LaGQ` and `LaGKQ` will return the response with status
code = 0x0001 (not found) if the object is not found.

### Replace and Unlock, Replace and Unlock Quietly

Request:

    - MUST have extras.
    - MUST have key.
    - MUST have value.

Replace a locked object then unlock it atomically.  The client need to
lock the object in advance.

The extra data are the same as that of `Replace` binary command, i.e.,
4 byte flags and 4 byte expiration time.

The response is almost the same as that of `Replace` binary command.
Specifically, 0x0011 (not locked) is returned when the object is not
locked by the client.

[1]: http://www.regexprn.com/2010/05/using-memcached-as-distributed-locking.html
[2]: http://russellneufeld.wordpress.com/2012/05/24/using-memcached-as-a-distributed-lock-from-within-django/
