---
layout: post
title:  "Integrating zeromq with boost::asio"
keywords: c++11 network programming boost asio zeromq
published: true
categories: cpp11 en
---

<img src="/img/snippet.jpg" alt="show me the code" width="627px"/>

Boost::asio and zeromq are two powerful network programming beasts. Boost::asio provides 
[Proactor pattern](http://www.artima.com/articles/io_design_patterns2.html) based asynchronous network I/O, asynchronous name resolution, timers and other utilities in a clean and concise API. Whereas zeromq allows to easily implement very powerful and flexible networking patterns.
Both toolkits are quite complementary and combining them allows for very interesting possibilities in terms of application design.
Fortunatly, both boost::asio and zeromq allow to easily interoperate with third party code and this is what I am going to explore in this post.

*TL;DR: a functionning example code of integrating a zeromq REQ socket sending a message to zeromq REP socket and receiving back a response, integrated with boost::asio Proactor is available [here](https://github.com/iyedb/boost_asio_zeromq)* 

Boost::asio allows us to assign an existing file descriptor to a boost::asio socket object. In our case the file descriptor is the socket file descriptor of a zeromq REQ socket. So first create a zeromq socket, obtain the associated file descriptor and assign it to a boost::asio socket. The code below does just that:

{% highlight c++ %}

void *zmq_sock_ = zmq_socket (zmq_ctx, ZMQ_REQ);
int zfd;
size_t optlen = sizeof (zfd);
zmq_getsockopt (zmq_sock_, ZMQ_FD, &zfd, &optlen);
sock_.assign (boost::asio::ip::tcp::v4(), zfd);

{% endhighlight %}

The `assing` method takes a protocol argument (which is a tcp v4 socket in our case) and the actual file descriptor. This is basically all what is needed to hook zeromq to boost:asio. The socket object will then be used to register with boost::asio `io_service` (the Proactor) and then perform the actual I/O operations - using zeromq - and without blocking when the file descriptor is ready for reading or writing. The following class `asio_zmq_req_socket` does just that and actually mimics the original boost::asio tcp/ip [socket](http://www.boost.org/doc/libs/1_55_0/doc/html/boost_asio/reference/ip__tcp/socket.html) with `async_send` and `async_recv` methods. 

{% highlight c++ %}

class asio_zmq_req_socket {
public:
	asio_zmq_req_socket(boost::asio::io_service& __io_service, std::string __srv_addr):
		sock_(__io_service) 
	{
		void *zmq_ctx = get_zmq_context ();
    	zmq_sock_ = zmq_socket (zmq_ctx, ZMQ_REQ);

    	if (zmq_sock_ == NULL) {
    		throw zmq_exception("could not create a zmq REQ socket");
    	}

    	int zfd;
    	size_t optlen = sizeof (zfd);
    	zmq_getsockopt (zmq_sock_, ZMQ_FD, &zfd, &optlen);
    	sock_.assign (boost::asio::ip::tcp::v4(), zfd);

    	int rc = zmq_connect (zmq_sock_, __srv_addr.c_str ());
    	if (rc) {
    		throw zmq_exception("zmq socket: could not connect to " + __srv_addr);
    	}
	}

	template <typename ConstBufferSequence, typename Handler>
	void async_send (const ConstBufferSequence & buffer, Handler handler) {
		zsock_send_op<ConstBufferSequence, Handler> send_op (zmq_sock_, buffer, 
			handler);
		sock_.async_write_some (boost::asio::null_buffers(), send_op);
	}

	template <typename MutableBufferSequence, typename Handler>
	void async_recv (const MutableBufferSequence & buffer, Handler handler) {
		zsock_recv_op<MutableBufferSequence, Handler> recv_op (zmq_sock_, buffer, 
			handler);
		sock_.async_read_some (boost::asio::null_buffers(), recv_op);
	}

	~asio_zmq_req_socket() {
		// the order matters
		sock_.close(); 
		zmq_close (zmq_sock_);	
	}
private:
	void *zmq_sock_ = nullptr;
	boost::asio::ip::tcp::socket sock_;
};

{% endhighlight %}

The template member functions `async_send (const ConstBufferSequence & buffer, Handler handler)` takes a ConstBufferSequence buffer to send and a handler that will be called once the buffer has been sent. To do this, it creates a `zsock_send_op<ConstBufferSequence, Handler> send_op` object and passes it as a [WriteHandler](http://www.boost.org/doc/libs/1_55_0/doc/html/boost_asio/reference/WriteHandler.html) to `async_write_some`. Of particular importance here, we pass `boost::asio::null_buffers()` as the buffer to send. This tells boost::asio `io_service` not to actually send anything but to call the handler when the socket is ready for writing (thus effectively acting as an Reactor instead of a Proactor if you think about it). When this happens, our `send_op` object will be called and only then, our real buffer will be sent using zeromq:

{% highlight c++ %}

template <typename ConstBufferSequence, typename Handler>
class zsock_send_op {
public:

	zsock_send_op(const void *__zmq_sock, const ConstBufferSequence & __buffers,
	 Handler __handler): 
		buffers_(__buffers),
		handler_(__handler),
		zmq_sock_(__zmq_sock) 
	{
	}

	void operator()(const boost::system::error_code &ec,
	 std::size_t bytes_transferred) {

		boost::system::error_code zec;
		std::size_t send_rc = 0;

		if (!ec) {
			int flags = 0;
			size_t fsize = sizeof (flags);
			zmq_getsockopt(const_cast<void *>(zmq_sock_), ZMQ_EVENTS, &flags, &fsize);

			if (flags & ZMQ_POLLOUT) {
				const void* buf = boost::asio::buffer_cast<const void*>(buffers_);
				int buf_size = boost::asio::buffer_size(buffers_);
			    send_rc = zmq_send (const_cast<void *>(zmq_sock_), 
			    	const_cast<void *>(buf), buf_size, 0);
			    
			    if (send_rc == -1)
			    {
			    	zec.assign(errno, boost::system::system_category());
			    }
			}
			handler_ (zec, send_rc);
		} // ec
		else {
			handler_ (ec, send_rc);	
		}
	}

private:
	ConstBufferSequence buffers_;
	Handler handler_;
	const void *zmq_sock_;
};

{% endhighlight %}

Notice the method `void operator()(const boost::system::error_code &ec, std::size_t bytes_transferred)` where all the magic takes place. First, it makes our `zmq_send_op` object comply with the `WriteHandler` concept boost::asio expects for write operation handlers. Then, when called by boost::asio `io_service`, it checks if a write operation can be made on the zeromq socket (`ZMQ_POLLOUT` is set) and if so, sends the buffer with a call to `zmq_send`. And finally, calls the handler we originally passed to `async_send`.

The `zmq_send_op / zmq_recv_op`  is the crux of this design. You can think of it as a closure containing the data to send and the handler to call, in which `zmq_send` routine will eventually execute. Actually it can even be implemented as C++11 lambda but for the sake of keeping thinks simple, I used a basic class. The `async_recv` uses a `zsock_recv_op` and is exactly in the same vein.

#### Putting it all together

The following is a simple example of how our `asio_zmq_req_socket` gets used:

{% highlight c++ %}

class my_zmq_req_client: 
	public std::enable_shared_from_this<my_zmq_req_client> 
{
public:
	my_zmq_req_client(boost::asio::io_service& __io_service,
		const std::string& __addr):
	zsock_(__io_service, __addr)
	{	
		
	}
	void send(const std::string& __message) {
		print("my_zmq_req_client: sending message:", __message);
		zsock_.async_send (
				boost::asio::buffer(__message),
				boost::bind(&my_zmq_req_client::handle_send, shared_from_this(),
					boost::asio::placeholders::error,
					boost::asio::placeholders::bytes_transferred 
				)
		);
	}
private:
	void handle_send(const boost::system::error_code &ec,
			std::size_t bytes_transferred)
	{
		if (ec) {
			std::cout << "failed to send buffer\n";
		}
		else {
			print("my_zmq_req_client: message sent");
			zsock_.async_recv (
				boost::asio::buffer(recv_buffer_), 
				boost::bind (&my_zmq_req_client::handle_recv, shared_from_this(), 
					boost::asio::placeholders::error,
					boost::asio::placeholders::bytes_transferred)
			);	
		}
	}
	void handle_recv(const boost::system::error_code &ec,
							std::size_t bytes_transferred) 
	{
		if (!ec) {
			// if no error happened but no bytes were received because the zmq socket 
			// is not POLL_IN ready (check the zero_mq zmq_getsockopt: it says that 
			// a socket can be reported as writable by the OS but not yet for zeromq) 
			// schedule another async recv operation. 
			// This is less likely to happen for send operations
			// and in practice never had to do it.
			if (bytes_transferred == 0)
				zsock_.async_recv (boost::asio::buffer(recv_buffer_), 
					boost::bind (&my_zmq_req_client::handle_recv, shared_from_this(), 
						boost::asio::placeholders::error,
						boost::asio::placeholders::bytes_transferred));
			else {
				printf("my_zmq_req_client: zmq REP: %s\n", recv_buffer_.data());
			}
		}
	}
	asio_zmq::asio_zmq_req_socket zsock_;
	std::array<char, 256> recv_buffer_;
};

int main(int, char**) {
	asio_zmq::get_zmq_context();
	boost::asio::io_service io_service;
	std::string srv_addr("tcp://127.0.0.1:11155");
	std::string msg("zmq REQ: hello from client");
	{
		auto zclient = std::make_shared<my_zmq_req_client>(io_service, srv_addr);
		zclient->send(msg);	
	}
	io_service.run();
	zmq_ctx_term(asio_zmq::get_zmq_context()); 
}

{% endhighlight %}

`my_zmq_req_client` uses an `asio_zmq_req_socket` and defines a receive handler along with a send handler. It sends a message to a zmq REP server at address `srv_addr` and receives back a response in `recv_buffer_` buffer then just prints it. The Proactor `io_service` runs out of work, `run` returns and the programs exits.

That's about it. The full code is available at this [github repo](https://github.com/iyedb/boost_asio_zeromq) along with a python zmq REP test server. Check it out. 







