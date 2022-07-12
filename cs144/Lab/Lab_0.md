​	Welcome to CS144: Introduction to Computer Networking. In this warmup, you will set up an installation of Linux on your computer, learn how to perform some tasks over the Internet by hand, write a small program in C++ that fetches a Web page over the Internet, and implement (in memory) one of the key abstractions of networking: a reliable stream of
bytes between a writer and a reader. We expect this warmup to take you between 2 and 6 hours to complete (future labs will take more of your time). Three quick points about the
lab assignment:

+  It’s a good idea to read the whole document before diving in!

+  Over the course of this 8-part lab assignment, you’ll be building up your own implementation of a significant portion of the Internet—a router, a network interface, and the TCP protocol (which transforms unreliable datagrams into a reliable byte stream).Most weeks will build on work you have done previously, i.e., you are building up your own implementation gradually over the course of the quarter, and you’ll continue to use your work in future weeks. This makes it hard to “skip” a checkpoint.

  

+ The lab documents aren’t “specifications”—meaning they’re not intended to be con-sumed in a one-way fashion. They’re written closer to the level of detail that a software
  engineer will get from a boss or client. We expect that you’ll benefit from attending the lab sessions and asking clarifying questions if you find something to be ambiguous and you think the answer matters. (If you think something might not be fully specified, sometimes the truth is that it doesn’t matter—you could try one approach or the other
  and see what happens.)





### Start

​	前面还有一些热身lab(获取网页,给自己发送电子邮件),按照文档指示一个一个输入就可以了





###  Writing webget

让我们利用系统的 tcp 实现一个 webget



利用 TCPSocket 与 Host 建立连接，然后发送请求头即可。就像之前的2.1,只不过这里我们是用代码实现,2.1节是键盘直接输入进终端页面`GET /hello HTTP/1.1`,`Host: cs144.keithw.org`,3.4节Host path都已经给你了.

想了解HTTP请求头格式的各个部分的意思,可以看P67(自顶向下)



注意:文档中请求头里的每一行的换行必须是 `\r\n` 不能是 `\n`，发完请求后需要 `shutdown(SHUT_WR)` 不然服务端等待你后续的请求。然后就是必须读到 `eof` 后才结束，不然可能会读入的不完整。



```c++
// apps\webget.cc
TCPSocket sock;
sock.connect(Address(host, "http"));//与host机器采用http服务建立连接
string data_send = "GET " + path + " HTTP/1.1\r\nHost: " + host + "\r\n" + "Connection: close\r\n\r\n";		//\r\n就是换行的意思
sock.write(data_send);	//写入要发出的请求
while (!sock.eof())
{
     auto data_recv = sock.read();
     std::cout << data_recv;	//打印回复
}
sock.close();
```





### An in-memory reliable byte stream

<!--它让我们实现一个 ByteStream，这个代码注释也挺详细的(在每个函数旁边写了这个函数的作用)-->



在本实验中，你需要实现在内存中的字节流(利用内存做数据缓冲区):（有点类似于管道）。字节被“input”端写入，并且能够被“output”端读取。当读取方读到了流的末尾，其将会遇到EOF标志。

​	要求:

- 字节流可以从**写入端**写入，并以**相同的顺序**，从**读取端**读取
- 字节流是有限的，写者可以终止写入。而读者可以在读取到字节流末尾时，产生EOF标志，不再读取。
- 所实现的字节流必须支持**流量控制**，以控制内存的使用。该对象初始化时使用“capacity”,设置其能够占用内存的最大空间. 当所使用的缓冲区爆满时，将禁止写入操作。直到读者读取了一部分数据后，空出了一部分缓冲区内存，才让写者写入。
- 写入的字节流可能会很长，必须考虑到字节流大于缓冲区大小的情况。即便缓冲区只有1字节大小，所实现的程序也必须支持正常的写入读取操作。

​	

方案:

+ 利用 `std::deque<char>` 来维护缓冲区。
+ 利用 `std::string` 和双指针来维护缓冲区。(大部分采用这种)





cs144所提供的的接口,我们要补充完整:

writer:

```c++
// Write a string of bytes into the stream. Write as many
// as will fit, and return the number of bytes written.
size_t write(const std::string &data);

// Returns the number of additional bytes that the stream has space for
size_t remaining_capacity() const;

// Signal that the byte stream has reached its ending
void end_input();

// Indicate that the stream suffered an error
void set_error();
```



reader:

```c++
// Peek at next "len" bytes of the stream
std::string peek_output(const size_t len) const;

// Remove "len" bytes from the buffer
void pop_output(const size_t len);

// Read (i.e., copy and then pop) the next "len" bytes of the stream
std::string read(const size_t len);

bool input_ended() const; // `true` if the stream input has ended
bool eof() const; // `true` if the output has reached the ending
bool error() const; // `true` if the stream has suffered an error
size_t buffer_size() const; // the maximum amount that can currently be peeked/read
bool buffer_empty() const; // `true` if the buffer is empty
size_t bytes_written() const; // Total number of bytes written
size_t bytes_read() const; // Total number of bytes popped
```





Answer:

```c++
// lib\byte_stream.hh
   bool _error{};  //!< Flag indicating that the stream suffered an error.
   std::string buffer_ = "";
   size_t capacity_ = 0;
   size_t used_ = 0;
   size_t total_read_ = 0;
   size_t total_write_ = 0;
   bool input_end_ = false;
```

```c++
// lib\byte_stream.cc

template <typename... Targs>
void DUMMY_CODE(Targs &&.../* unused */) {}

using namespace std;
ByteStream::ByteStream(const size_t capacity) : capacity_(capacity) {}

size_t ByteStream::write(const string &data) {
   DUMMY_CODE(data);
   // 获取数据大小
   size_t writed_size = (remaining_capacity() >= data.size() ? data.size() : remaining_capacity());
   // 保存数据并更新信息
   buffer_ += data.substr(0, writed_size); //写入缓存
   used_ += writed_size;
   total_write_ += writed_size;
   return writed_size;
}

//! \param[in] len bytes will be copied from the output side of the buffer
string ByteStream::peek_output(const size_t len) const { return buffer_.substr(0, len); }

//! \param[in] len bytes will be removed from the output side of the buffer
void ByteStream::pop_output(const size_t len) {
   buffer_.erase(0, len);
   total_read_ += len;
   used_ -= len;
}

//! Read (i.e., copy and then pop) the next "len" bytes of the stream
//! \param[in] len bytes will be popped and returned
//! \returns a string
std::string ByteStream::read(const size_t len) {
   // 获取数据大小
   size_t read_size = (len <= used_ ? len : used_);
   // 返回数据
   std::string temp = peek_output(read_size);
   pop_output(read_size);
   return temp;
}

void ByteStream::end_input() { input_end_ = true; }

bool ByteStream::input_ended() const { return input_end_; }

size_t ByteStream::buffer_size() const { return used_; }

bool ByteStream::buffer_empty() const { return (used_ == 0); }

bool ByteStream::eof() const { return input_end_ && used_ == 0; }

size_t ByteStream::bytes_written() const { return total_write_; }

size_t ByteStream::bytes_read() const { return total_read_; }

size_t ByteStream::remaining_capacity() const { return capacity_ - used_; }
```





使用deque容器:

```c++
class ByteStream {
  private:
    // Your code here -- add private members as necessary.
    std::deque<char> _buffer = {};
    size_t _capacity = 0;
    size_t _read_count = 0;
    size_t _write_count = 0;
    bool _input_ended_flag = false;
    bool _error = false;  //!< Flag indicating that the stream suffered an error.
    //......

```

```c++
ByteStream::ByteStream(const size_t capacity) : _capacity(capacity) {}

size_t ByteStream::write(const string &data) {
    size_t len = data.length();
    if (len > _capacity - _buffer.size()) {
        len = _capacity - _buffer.size();
    }
    _write_count += len;
    for (size_t i = 0; i < len; i++) {
        _buffer.push_back(data[i]);
    }
    return len;
}

//! \param[in] len bytes will be copied from the output side of the buffer
string ByteStream::peek_output(const size_t len) const {
    size_t length = len;
    if (length > _buffer.size()) {
        length = _buffer.size();
    }
    return string().assign(_buffer.begin(), _buffer.begin() + length);
}

//! \param[in] len bytes will be removed from the output side of the buffer
void ByteStream::pop_output(const size_t len) {
    size_t length = len;
    if (length > _buffer.size()) {
        length = _buffer.size();
    }
    _read_count += length;
    while (length--) {
        _buffer.pop_front();
    }
    return;
}

void ByteStream::end_input() { _input_ended_flag = true; }

bool ByteStream::input_ended() const { return _input_ended_flag; }

size_t ByteStream::buffer_size() const { return _buffer.size(); }

bool ByteStream::buffer_empty() const { return _buffer.size() == 0; }

bool ByteStream::eof() const { return buffer_empty() && input_ended(); }

size_t ByteStream::bytes_written() const { return _write_count; }

size_t ByteStream::bytes_read() const { return _read_count; }

size_t ByteStream::remaining_capacity() const { return _capacity - _buffer.size(); }

```

