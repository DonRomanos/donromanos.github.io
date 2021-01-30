---
layout: post
title: A little something about RPC-Calls and Serialization
tags: C++, Serialization, RPC, Protobuf, Cereal, Bitsery
---

Or what is the meaning of those bytes? 

The Problem in a concrete case: We have two separate processes that run on the same platform but in different security environments. The compilers that are used are different as well. Both processes communicate through an sdk that works a little like a driver. We can send a message from one side containing data in a previously allocated memory segment and receive the reponse in the same way.

The message interface looks like this:

```cpp
int send_command(void* request, size_t request_size, void* response, size_t response_size)
```

All calls are synchronous and we do not have to worry about multithreading. On the other side there is a receiver which will receive our command and can respond. The return code will be handed over by the system. 

The receiver entry point looks like this:

```cpp
int command_handler(void* request, size_t request_size, void* response, size_t response_size)
```

We want to do different things on the receiver side based on what the command is. So we need an identifier for our command. Additionally we want to couple the command to the data that is connected to it.

So how should we do it?

# The idea

Suppose we have a function that looks like this:

```cpp
std::string find_name(const std::vector<std::string>& names, std::string_view name);
```

We want to execute the algorithm behind this find in our secure environment while still having the same function signature visible to the user. Somewhere in our program we will define an identifier for this `findName`command:

```cpp
enum class CommandId
{
    find_name,
}
```

Everytime we want to do something and we receive a request in our `command_handler` we will look at the id and determine what to do. The id will always be the first `sizeof(CommandId)` bytes of our request. It will look something along the following lines.

```cpp
std::string find_name_secure(const std::vector<std::string>& names, std::string_view name);

int command_handler(void* request, size_t request_size, void* response, size_t response_size)
{
    int command_id = *static_cast<int*>(request);

    switch(command_id)
    {
        case find_name:
            const std::vector<std::string> names = ???;
            const std::string name = ???;
            std::string result = find_name_secure(names, name);
            std::memcopy(response, result.data(), result.size());
            response_size = result.size(); 
            return 0;
    }
}
```

So whenever we want to perform a different action we will need to extend the `switch` and perform all necessary data conversions ourselves. Since we use variable length types we need to have additional sizes fields and so on... this is quite cumbersome. How can we do better?

# An ideal world

Usually when something like this comes up I imagine the most comfortable way that this could be handled without language or hardware limitations, simply the ideal case.

Ideally I as the client would simply call the function as I would in any other program. So that in my program the code might look like this:

```cpp
auto names = std::vector<std::string>{"Slagathor", "CaptainJoe", "TaserFace"};

auto result = std::string find_name(names, "Slagathor");
```

Thats all, the rest would be taken care of by the program. Arguments and return values would be correctly forwarded. The correct algorithm would be executed.

C++ being a horrible language (we can also call it low level ;)) makes this quite cumbersome. The language itself does not offer any functionality providing any of the two above mentioned features. In other words what we need are:

* Remote Procedure Calls - to determine what logic the user wants to execute
* Serialization and Deserialization of Data Types - to bring data from and to the respective environment

Of course we are not the first ones to encounter those problems. People have written tons of libraries and programs tackling the above problems, so lets take a closer look at them, then I will explain how and why we solved our specific Problem. Some of the more notable ones:

RPC and Serialization

* [Google Protocol-Buffers](https://developers.google.com/protocol-buffers)
* [Cap'n Proto](https://capnproto.org/rpc.html)

Just Serialization

* [Cereal](https://uscilab.github.io/cereal/)
* [Bitsery](https://github.com/fraillt/bitsery)
* [Boost Serialization](https://www.boost.org/doc/libs/1_75_0/libs/serialization/doc/index.html)

The division above also resembles the different approaches they took. `Protocol-Buffers` and `Cap'n Proto` use an external program and some definition files which will create the code required for serialization and handling rpc-calls for you. 

`Cereal`, `Bitsery` and `Boost Serialization` are simply libraries that you include and then write some code to turn data types into byte streams. They do not offer any RPC mechanisms themselves so that part is completely up to you.

For our specific problem (same platform, different security environment, only synchronous calls) we decided not to use any of them and I will explain below why.

# How most pure C++ Serialization Libraries work

Does not actually matter which one of the ones mentioned you look at, the idea behind them is the same. When you look through the documentation you will see what we have to do in order to serialize custom data types.

```cpp
// Cereal
struct MyRecord
{
  uint8_t x, y;
  float z;
  
  template <class Archive>
  void serialize( Archive & ar )
  {
    ar( x, y, z );
  }
};

// Bitsery
struct MyStruct {
    uint32_t i;
    MyEnum e;
    std::vector<float> fs;
};

template <typename S>
void serialize(S& s, MyStruct& o) {
    s.value4b(o.i);
    s.value2b(o.e);
    s.container4b(o.fs, 10);
}

// Boost
struct MyStruct {
    uint32_t i;
    MyEnum e;
    std::vector<float> fs;

    template<class Archive>
    void serialize(Archive & ar, const unsigned int /* file_version */){
        ar & i & e & fs;
    }
};
```

As you can see for all of them we we provide a templated `serialize` either as member or free function (some support both). In this function we define something I would call the `class layout` simply by listing all its members. It is important that you list the actual members (not some getters that return them by value for example) since we need references to them in order for the deserialization to work.

So whats important the function is templated and it lists all members as lvalues. Now we can define our `Archives` which will determine what happens when we serialize and deserialize. For this we need two classes. We only use placeholders for now but I will show you how we would use them.

```cpp
struct OutputArchive{};
struct InputArchive{};


std::vector<uint8_t> buffer;

// This is how we serialize our things
MyStruct input_data;
InputArchive sink(buffer);
sink.serialize(input_data);

// This is how we deserialize our things
MyStruct result;
OutputArchive source(buffer);
source.deserialize(result);
```

More or less. The details are bit different but the essence is you need two distinct types to which you provide the types you want to serialize to/deserialize from, as well as the memory where the byte stream is.

Both of your types will now use the serialization function internally to do their respective thing. An very stripped down example could be:

```cpp
struct OutputArchive
{
    OutputArchive(std::vector<uint8_t>& buffer) : buffer(buffer){}

    template<typename T>
    void deserialize(T in)
    {
        in.serialize(*this, in); // this calls the member function of our defined type
    }

    // Actual implementation is defined either as operator & (boost), function called value4b (bitsery) or even the constructor (cereal).
    // So when we call serialize on this class, it will call the serialize method on our Data type which will then use the & operator and we end up here.
    template<typename T>
    OutputArchive& operator&(T& in)
    {
        deserialize(in); // For each member we recursively instantiate until we reach an overload below
    }

    // implementations for basic types are provided the rest just recursively instantiates until we reach one of these.
    OutputArchive& operator&(int& in)
    {
        buffer.push_back(reinterpret_cast<uint8_t*>(&in), reinterpret_cast<uint8_t*>(&in) + sizeof(in));
    }

    // Overloads for many more types like std::vector, std::map, all trivially_copyable types, etc.

    std::vector<uint8_t>& buffer;
};
```

Thats the basic idea. It is quite complicated to follow and requires some knowledge about overload resolution and template instantiations to provide a convenient example. Thats why the libraries exist and take this work off of you.

The beauty of this approach lies in a single definition of the serialize function. This means, that operations from serialization and deserialization will always be symmetrical. So there is no source of error if you extend the serialization function, but forget to do the same for the deserialization.

Of course this also comes at a price. Because you have this single definition the function needs to use references in order to know which members to deserialize into. That means your data types need to be default constructible and cannot have const members.

# What we did for our specific Problem

Remember the problem I talked about in the beginning? Actually we can add some more requirements that come from other technical design decisions we have in our code base:

* We use strong types, of which a lot are not default constructable
* We dont expect there to be an unlimited amount of types to be added which need to be serialized appart from 5-6 initial types
* We are only interested in binary serialization

So considering these we decided that we would not follow the approach mentioned before. Instead we provide our own overload based solution taylored exactly to the needs above. This means we do not have this nice symmetrie mentioned above, but we will have to define our serialize and deserialize functions independently from eacht other. Since we dont plan to add many types in the future this was alright for us. And of course we covered the implementation with tests to make sure they are symmetrical.

Here is a very simplified version of our serialization implementation:

```cpp
class BinarySerializer
{
    BinarySerliazer(std::vector<uint8_t>& buffer) : buffer(buffer){};

    template<typename T, typename... Args>
    void serialize(const T& first, Args&&... args)
    {
        serialize(first);
        serialize(args);
    }

    // This is an example for int, we had one overload for all trivially copyable types using sfinae,
    // As well as one for vector and string, we did not need more.
    void serialize(int in)
    {
        buffer.push_back(reinterpret_cast<uint8_t*>(&in), reinterpret_cast<uint8_t*>(&in) + sizeof(in));
    }
}
```

Very similar the deserialization:
```cpp
class BinaryDeserializer
{
    BinarySerliazer(std::vector<uint8_t>& buffer) : buffer(buffer){};

    template<typename T>
    T deserialize()
    {
        static_assert(sizeof(T), "Please provide a specialization for the type you want to deserialize into");
    }

    //... rest of the class
}

// This is an example for int, we had different specializations for different types, in overall there were like 5-6. We could treat all trivially copyable types together.
// and had again implementations for vector and string.
template<>
int BinaryDeserializer::deserialize()
{
    return *reinterpret_cast<int*>(current_buffer_position);
}
```

With this approach this is how most of our Tests looked when we used the above classes:
```cpp
TEST(BinarySerialization, "Vector_can_be_serialized_and_deserialized")
{
    std::vector<uint8_t> buffer;
    std::vector<int> expected = {1,2,4,6,8}; 

    BinarySerializer(buffer).serialize(expected);
    auto result = BinaryDeserializer(buffer).deserialize<std::vector<int> >();

    EXPECT_EQ(expected, result);
}
```

So far I think this was a good decision for our codebase. We had quite compact code that is easy to understand. The downside is that our BinarySerializer and Deserializers are currently dependency magnets since we provide overloads for different types. So far this is not a problem for us, but time will tell.

For future types we will probably use friend specialized template functions next to those types instead of in the serializer.

So far, so good. But we only tackled the serialization but what about the RPC part?

# The suprising end

Well we never actually solved that part. We settled on some convenience, but with the serialization in place our code was already pretty maintainable (at least for now). We thought about using some macro or template magic to create function bodies for us that would take care of all the forwarding. However with the current C++ facilities this is not easily possible (or at least we did not find a satisfying way).

Maybe with C++23 we get compile time reflection which offers us whole new possibilities that might solve this problem. For now if you really need to do it, I think a code generator based on `llvm` looks most promising to me. Maybe I will find the time and post about it in another article.

My most important learning here was to adapt your approach to your requirements, the more you know about them the better you can choose. We implemented both, a simple version of what those libraries do, as well as our final approach I described above. That was very valuable to have as comparison and make a good decision. 

And remember: **This is the way!** (At least our way, for you it might look different based on your requirements ;))