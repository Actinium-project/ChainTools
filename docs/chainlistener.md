## Writing a tool for listening to Bitcoin/Altcoin [ZMQ](http://zeromq.org/) notifications in C

In this tutorial we will explore the ZMQ-based notification interface that is available in almost every cryptocoin out there. Bitcoin and its clones offer a set of connection points where clients can 'subscribe' to get notified about certain events happening in the decentralized network, like 'raw transactions', 'raw blocks' etc. This set of interfaces is based on ZMQ (ZeroMQ), a highly scalable networking library written in C that acts like a concurrency framework. ZMQ offers specialized sockets that carry atomic messages which can be transported *in-process*, *inter-process*, *TCP*, *multicast* etc. Those sockets can communicate with each other in various ways, for example *request-reply*, *publish-subscribe*, *fan-out* etc. And unlike many other similar libraries ZMQ has no central *broker* that takes care of forwarding those messages between participants. This of course makes it much easier to use and implement and in fact there exist dozens on different implementations for various languages. In this installment we will use C, the original language of ZMQ.

### Prerequisites

* Cryptocurrency wallet or daemon

A working Bitcoin or Altcoin wallet with activated ZMQ notification interfaces is needed. To activate them open your wallet config an enter the settings below. You can change the ports if needed. If your wallet refuses to publish notifications check if you have linked the ZMQ library.  

```shell
zmqpubrawblock=tcp://127.0.0.1:28332
zmqpubrawtx=tcp://127.0.0.1:28333
zmqpubhashtx=tcp://127.0.0.1:28334
zmqpubhashblock=tcp://127.0.0.1:28335
```

Then restart your wallet and enter this RPC-command in its Debug/Console.

![rpc_getzmqsubscriptions](https://raw.githubusercontent.com/Actinium-project/ACM-Designs/master/random/getzmqsubscriptions.png)


* GCC, CLANG, or MS C++ compiler

You will also need a C compiler toolchain for generating the binary we are about to write. This code has been tested with GCC and CLANG but it should also work with Microsoft's C++ compilers.

* ZMQ & CZMQ libraries and headers

And finally, we will need ZMQ and [CZMQ](http://czmq.zeromq.org/czmq4-0:_start), the *high-level* C-binding to ZMQ. CZMQ makes working with ZMQ's sockets easier as it wraps the undelying code in shorter semantics. It also hides the differences between ZMQ versions which of course makes working with legacy code more pleasant. To install these libraries use the install tool of your choice like **apt** or **brew**, or download them directly from ZMQ's [home page](http://zeromq.org/intro:get-the-software).

**macOS**

```shell
brew install zmq
brew install czmq
```

**Linux Debian 9**

```shell
echo "deb http://download.opensuse.org/repositories/network:/messaging:/zeromq:/release-stable/Debian_9.0/ ./" >> /etc/apt/sources.list
wget https://download.opensuse.org/repositories/network:/messaging:/zeromq:/release-stable/Debian_9.0/Release.key -O- | sudo apt-key add
apt-get install libzmq3-dev
apt-get install libczmq-dev
```

### Introduction to ZMQ notifications

Presumably, most cryptocurrency users have little or no experience with the code that constitutes their wallets let alone parts that take care of sending notifications via ZMQ. But in our case knowing what actually happens on the other side of the fence is of crucial importance. ZMQ itself knows nothing about the messages it's sending around. All ZMQ sees are arrays of bytes with lenghts prepended. It is programmer's responsibility to convert them into a proper format. As ZMQ strives to be usable by any programming language out there it simply can't dictate their 'correct' formatting. For example in C there is no concept of strings at all. What we call a 'string' in C is an array of bytes with an appeneded **0** that marks its end. Not so with other languages which often have a dedicated string type, use no end-indicators, or maybe even indicate their string lengths by prepending a number before the actual content. Using different environments also means *looking differently at data*. Therefore, the only thing ZMQ sees is a bunch of bytes with a length indicator prepended. Like this:

![zmq_string](https://raw.githubusercontent.com/Actinium-project/ACM-Designs/master/random/zmq_string.png)

And to understand how cryptocurrency software formats their data we must look into their code. Here is an example from Bitcoin's [current master branch](https://github.com/bitcoin/bitcoin/blob/master/src/zmq/zmqpublishnotifier.cpp).

![btc_send_zmq_message](https://raw.githubusercontent.com/Actinium-project/ACM-Designs/master/random/btc_send_zmq_message.png)

By looking at the parameters of the *SendMessage* function as well as from the comments we recognize that Bitcoin is sending a three-part or, in ZMQ-lingo, *multipart* messages. Every message begins with a *command* that is essentialy the *topic* a client subscribes to. The second part is the *payload* that can be a **transaction hash*, a **blockhash**, a **raw block**, or a **raw transaction**. The third part of the message is the length indicator written in **little-endian** [format](https://en.wikipedia.org/wiki/Endianness). 

The place where all those bytes get packed and pushed out via ZMQ is the internal **zmq_send_multipart** [function](https://github.com/bitcoin/bitcoin/blob/master/src/zmq/zmqpublishnotifier.cpp#L30).

![zmq_send_multipart](https://raw.githubusercontent.com/Actinium-project/ACM-Designs/master/random/zmq_send_multipart.png)

It accepts a **variable range** of arguments (note the three **dots** at the end of the argument list) and each of those gets converted into a proper ZMQ message that takes [data and size](https://github.com/bitcoin/bitcoin/blob/master/src/zmq/zmqpublishnotifier.cpp#L30) to initialize the next chunk. 

Ultimately the message gets pushed out via **zmq_msg_send** [function](https://github.com/bitcoin/bitcoin/blob/master/src/zmq/zmqpublishnotifier.cpp#L43). 

One important part in this mechanism is the flag **ZMQ_SNDMORE** that indicates the end of a **multipart message**. As long as there are *more* messages to be sent this flag will be set. After the last message has left it switches to **0** to inform the receiving party that there won't be any more parts belonging to this message. One can think of **multipart messages** as boxes containing separate entities, called *frames*, inside. A *frame* is a chunk of data that can be processed individually. Multipart messages can contain any number of frames. In Bitcoin's case we have three frames that represent different types of data.

Now the question is: How do we get these messages? Or better, how do we get them *programmatically*?

We'll take C of course to write a client that subscribes to those *publishers* and reads incoming messages the same way they've been written. This is the reason why one should always *look into the* code that originally produced data to be able to consume it properly.

### Using ZMQ libraries to receive messages 

Before we get them we have to connect to available publishers from our wallet or daemon. With the help from CZMQ library we can do it easily by invoking [this function](https://github.com/Actinium-project/ChainTools/blob/master/bin/chainlistener.c#L41).

![open_new_socket](https://raw.githubusercontent.com/Actinium-project/ACM-Designs/master/random/zmq_open_new_socket.png)

As we are using the *publish-subscribe* [pattern](http://zguide.zeromq.org/page:all#Chapter-Advanced-Pub-Sub-Patterns) to receive messages, our two parameters are **zmqserver** and **topic**. The *zmqserver* is just the URL of one the publishers we're interested in. 

```shell
zmqpubrawblock=tcp://127.0.0.1:28332
zmqpubrawtx=tcp://127.0.0.1:28333
zmqpubhashtx=tcp://127.0.0.1:28334
zmqpubhashblock=tcp://127.0.0.1:28335
```

The *topic* is the term after the *zmqpub* prefix from the same line. If for example we're interested in receiving **block hashes** we'd use **tcp://127.0.0.1:28335** as *zmqserver* and **hashblock** as *topic*. Additionally, our tool contains a [small piece of logic](https://github.com/Actinium-project/ChainTools/blob/master/bin/chainlistener.c#L33) that parses the parameters entered via command line so we can start it by giving it the **zmqserver** and **topic** directly.

![zmq_tool_params](https://raw.githubusercontent.com/Actinium-project/ACM-Designs/master/random/zmq_tool_params.png)

#### Receiving multipart-messages

Having subscribed to one of the publishers the next task will be to process those messages in a meaningful way. And as we have already seen the topics we can subscribe to will have different types and message sizes, but they'll always comprise of three parts, that is **frames**:

* Topic name, a string like 'rawtx', 'hashblock' etc.
* Payload, a hexadecimal value
* Length indicator in *little-endian* format, a numeric value

And here we should distinguish between two steps:

* Receving data
* Processing data

The receiving part is the same for all clients based on ZMQ. We just connect to the publisher and wait for a message to arrive. As ZMQ treats all messages as pure bytes with a length indicator prepended *it seems* that there is [not much to be done](https://github.com/Actinium-project/ChainTools/blob/master/bin/chainlistener.c#L46) from our side:

![zmq_receive_messages](https://raw.githubusercontent.com/Actinium-project/ACM-Designs/master/random/zmq_receive_messages.png)

However, the very moment we receive such a message we're immediately obliged to define the way we're gonna look at it. Failing to do so would subsequently lead to a data loss or defect formatting. Therefore, we're not only receiving messages but also preparing a 'picture' of them. If we hover over the *zsock_recv* function we get this help window:

![zsock_recv_hover](https://raw.githubusercontent.com/Actinium-project/ACM-Designs/master/random/zmq_hover_receive_picture.png)

As we see these messages aren't *mere bunch of bytes* but a bit more sophisticated. 

Let's look into the [available options](https://github.com/zeromq/czmq/blob/master/README.md#zsock---high-level-socket-api-that-hides-libzmq-contexts-and-sockets) to learn more about 'pictures':

![zsock_recv](https://raw.githubusercontent.com/Actinium-project/ACM-Designs/master/random/zsock_recv.png)

Among other picture-types we also see our own parameter **m** that indicates messages which contain **frames**. This of course fits nicely to the fact that our wallet is sending messages containing multiple frames. Now, one thing must be mentioned here: as we are using the CZMQ library *to abstract away all the tedious stuff* from ZMQ we enjoy the 'luxury' of not having to type everything separately. We simply let our client receive a message in a format we expect it to have, that's it. A simple 'm' as parameter is enough to save us from the repetitious and error prone work with raw bytes.

#### Unpacking and processing messages

However, there is still some work to be done as not everything can be automated away. We're still coding in C, aren't we?

The multipart message we just received contains data in certain formats and just blindly converting them to strings in hope *it'll work somehow* would only lead to crippled hash values, weird hexadecimals and other garbage. Therefore, we must know that not all data can be 'pop'-ed with the same function. Let's recall the types of data we're receiving:

* Topic name, a string
* Payload, a hexadecimal value
* Length indicator in little-endian format, a numeric value

For the **first frame** we can use the *zmsg_popstr* function, this is the easy part.
The **second frame** is a bit harder as is contains data of various length and quality. Therefore we first get it out of the multipart message as a *frame* via *zmsq_pop* function that returns an object of type *zframe_t*. This way we can later convert it into a human-readable format. Had we used *zmsg_popstr* we'd only see a part of the whole data structure. The **third frame** is a number that we convert into an *unsigned int* as we have no negative lengths in our case.

But our **second frame** needs some additional massage as we need to know its length and also have to convert its data into hexadecimals. For this we use two new functions: 

*zframe_strhex* 

![zframe_strhex](https://raw.githubusercontent.com/Actinium-project/ACM-Designs/master/random/zframe_strhex.png)

*zframe_size* 

![zframe_size](https://raw.githubusercontent.com/Actinium-project/ACM-Designs/master/random/zframe_size.png)

Here we can see that there is no need for us to touch this data directly as CZMQ offers [functions](https://github.com/Actinium-project/ChainTools/blob/master/bin/chainlistener.c#L53) that take care of such tasks.

![zmq_unpacking_messages](https://raw.githubusercontent.com/Actinium-project/ACM-Designs/master/random/zmq_unpacking_messages.png)

At the end we print these messages in our console:

![zmq_print_data](https://raw.githubusercontent.com/Actinium-project/ACM-Designs/master/random/zmq_print_data.png)

Last but not least, [we free the memory](https://github.com/Actinium-project/ChainTools/blob/master/bin/chainlistener.c#L59) we've been using so far. As already stated in the docs ZMQ expects the receiving side to take care of memory management.

An this is how our tool would present us those real-time data:

![chainlistener_demo](https://raw.githubusercontent.com/Actinium-project/ACM-Designs/master/random/chainlistener_demo.png)





