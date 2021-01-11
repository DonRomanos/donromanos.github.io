# S-E-R-I-A-L-I-Z-A-T-I-O-N

Or what is the meaning of those bytes? 

The Problem in a concrete case: We have two separate processes that run on the same platform but in different security environments. The compilers that are used are different as well. Both processes communicate through an sdk that works a little like a driver. We can send a message from one side containing data in a previously allocated memory segment and receive the reponse in the same way.

The message interface looks like this:

```cpp
int send_command(void* request, size_t request_size, void* response, size_t reponse_size)
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

Everytime we want to do something and we receive a request in our `command_handler` we will look at the id and determine what to do. The id will always be the first `sizeof(int)` bytes of our request. I will look something along the following lines.

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
What I would like to do is somehow tie together the method that I am calling to the command that will executed in the secure environment.

Ideally this would involve no more than:
```cpp
register_command(&find_name, &find_name_secure)
```

And after this whenever I call `find_name` it would automatically construct the message containing a unique identifier as well as all the data I pass in. At the same time it would extend the switch for my case handle the reconstruction of the data, trigger the `find_name_secure` function, assemble the result into the response and voila we're done.

Without some fancy things which might be valid depending on your use cases and resources this is not easily possible in C++. We would need some macro magic or a code generator in order to create the body of our `find_name` function. 




