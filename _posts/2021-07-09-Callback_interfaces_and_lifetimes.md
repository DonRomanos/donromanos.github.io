---
layout: post
title: Callback interfaces and lifetimes
tags: C++, Callback, shared_ptr, publisher subscriber
---

# That damned shared_ptr
Not long ago I was implementing an abstraction for different sources of images. One of the sources was a camera, one was a network socket. Both of those communicated via callbacks. 

Those callbacks were plain c-functions, however we were using certain objects for our computations which we require to track some state.

So we defined an interface for receiving our images:
```cpp
class ImageSubscriber
{
public:
    virtual void on_new_image(Image&& image) = 0;
};
```

Each one of our sources could register one or more of those `ImageSubscribers` which would be notified once a new image is available.

Our image sources implemented an interface looking like this:

```cpp
class ImageSource
{
public:
    void add_subscriber(ImageSubscriber* subscriber)
    {
        m_subscribers.push_back(subscriber);
    }

    void remove_subscriber(ImageSubscriber* subscriber)
    {
            m_subscribers.erase(std::remove(m_subscribers.begin(), m_subscribers.end(), subscriber));
    }

private:
    std::vector<ImageSubscriber*> m_subscribers;
};
```

This leads us to the problem that now arises due to the lifetime of the object involved. Before we can call any of our `Subscribers` we have to make sure that they still exist, otherwise terrible things will happen.

One (too common) way todo that is to use a `shared_ptr`. Just wrap your `Subscribers` into a shared_ptr and you will never have a problem with their lifetimes:

```cpp

class ImageSource
{
// ...
private:
    std::vector<std::shared_ptr<ImageSubscriber>> m_subscribers;
};
```

Unfortunately now your Subscriber lifetimes are also determined by the actual device and furthermore we force the user to use a shared_ptr for all of their objects for which they want to receive images. Whether it is the right choice for the job or not.

# An alternative

What we want is that when our object goes out of scope and its lifetime ends that it will be automatically removed from the list of subscribers.

Lets do that using a `Subscription` object which we add as member to our class.

```cpp

class Subscription
{
public:
    Subscription(ImageSource* source, ImageSubscriber* subscriber) : m_source(source), m_subscriber(subscriber)
    {
        m_source->add_subscriber(m_subscriber);
    }

    ~Subscription()
    {
        m_source->remove_subscriber(m_subscriber);
    }

private:
    ImageSource* m_source;
    ImageSubscriber* m_subscriber;
};

class IWantToUseAnImage : public ImageSubscriber
{
public:
    IWantToUseAnImage(ImageSource* source) : m_subscription(source, this) {}

    void on_new_image(Image&& image) override
    {
        // Do something with image
    } 

private:
    Subscription m_subscription;
};

```

We could've also inherited from the subscription, however that does not feel right as well. Composition seems more natural to me in this case.

Now we solved the lifetime problem for our `ImageSource` since now every object will properly unregister itself once its lifetime has ended due the destructor of the subscription member being called.

However by introducing this object we have actually introduced the very same problem just on the other side. When the lifetime of our `ImageSubscriber` exceeds that of the `ImageSource` we will try to unregister our `Subscriber` from a nonexisting object leading to a segmentation fault or worse.

You can choose to ignore this problem (since the camera is usually instantiated early for many cases this will be good enough), however it can lead to hard to detect bugs. So can we do better?

# That damned shared_ptr yet again

Righ this is exatly the problem that `shared_ptr` and `weak_ptr` try to solve. We could just have our subscription hold a `weak_ptr` to the source and check whether it is still alive before we unsubscribe. 

In order to do that we need a `shared_ptr` yet again, which our weak pointer can reference. This means we are back to shared ownership of our device and the problem I already mentioned above. So can we achieve this without shared ownership for useres of our classes? Turns out we can.

Currently we have an observer pattern in place (meaning the source and sink directly know each other). Lets switch that to a real Publisher-Subscriber pattern and introduce a channel object as broker in between them.

First we introduce a new interface for our publishing functionality

```cpp
class ImagePublisher
{
public:
    virtual void publish_image(Image&& image) = 0;
    virtual void add_subscriber(ImageSubscriber* subscriber) = 0;
    virtual void remove_subscriber(ImageSubscriber* subscriber) = 0;
};
```

Now we can get to the implementation of our channel:

```cpp
class Channel : public ImageSubscriber, public ImagePublisher
{
public:
    Channel(ImagePublisher& source) : m_source(source){}

    void publish_image(Image&& image) override
    {
        if(m_sinks.empty())
        {
            return;
        }
        if(m_sinks.size() > 1)
        {
            std::for_each(m_sinks.begin(), m_sinks.end() - 1, [image](ImageSubscriber* sink){sink->on_new_image(std::move(image));});
        }
        m_sinks.back().on_new_image(std::move(image));
    }

    void add_subscriber(ImageSubscriber* subscriber) override
    {
        m_sinks.push_back(subscriber);
    }

    void remove_subscriber(ImageSubscriber* subscriber) override
    {
        if(auto source = m_source.lock())
        {
            source.remove_subscriber(subscriber);
        }
        else
        {
            std::cerr << "Found subscriber without publisher." std::endl;
        }
    }

private:
    ImagePublisher& m_source;
    std::vector<ImageSubscriber*> m_sinks;
};
```

**Note**: I implemented some move optimizations since image for us is quite a heave object (~1MB) I want to make sure to not make any unnecessary copies.

So far so good. Now we need to adapt our previous implementations. Our `ImageSource` can now internally hold the channel via `shared_ptr`.

```cpp
class ImageSource : ImagePublisher
{
public:
    ImageSource() : m_channel(std::make_shared<Channel>(*this)){}

    void add_subscriber(ImageSubscriber* subscriber)
    {
        m_channel->add_subscriber(subscriber);
    }

    void remove_subscriber(ImageSubscriber* subscriber)
    {
        m_channel->remove_subscriber(subscriber);
    }

    // Call on_new_image whenever an image is available etc.

private:
    std::shared_ptr<Channel> m_channel;
};
```

Now we can normally instantiate our `ImageSource` like any other object and the user does not need to know anything about lifetimes or `shared_ptr`s. 

The full source code for the sample above can be found [here](https://godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAMzwBtMA7AQwFtMQByARg9KtQYEAysib0QXACx8BBAKoBnTAAUAHpwAMvAFYTStJg1DIApACYAQuYukl9ZATwDKjdAGFUtAK4sGEjQAcpK4AMngMmAByPgBGmMQSAMwA7KQADqgKhE4MHt6%2B/kEZWY4CYRHRLHEJXCm2mPalDEIETMQEeT5%2BXIH1jTktbQTlUbHxSakKre2dBT1BU0MjldUTAJS2qF7EyOwc5onhyN5YANQmiW4AbpgOJBfYJhoAggdHJ5jnl2wsJACeDyerzMhwYxy8ZwubicU2ImFYgJebzBHy%2BbjEwBIhAQLERrxesK8DlOAEkWExgJggSZklZkgARC5WJEvY5MBQKUnkylCLwxBTIYh4arU2lAtJ82h4ZAgIGneWnK54dpeMSK1B4dCnAQAfQiAHcdXhuZgIGSKZ9jRa1l96acNEy5QqlSq1QA/c08vkCoXVCA2i52rBUJheWgER1IhmR/HPNkc068/mCvBpJoxoHxzmelSSvAKBDxUXM54SmJSmVO%2BUugiq2iJ70ptM5U5MdDoHUKRu%2B%2BJmk1Jn3C%2BIAKlOXeTPeIAcSdodiRLCsVytraquGq1cN%2BN073aHxD7FoHKeqo/Hg%2Bq09nMcXNbrpw9JuUeYLvcvp2DofDGej8%2BprIMCZHkKzYCMW4p5pWLyLkBqZNBAUzoCAIAFm0mAdmmxBQjmT7lvmhaYYkjxmAAbGOWw7JgpBcoeu4nmOtHxDaICnCwnbkbs8HsZgGwsTuE57vBDFTlW5xilBCo0oyLLPNBu4gQwEBoAwUwNvx8nmKRqAEPhb5YPQBBUr%2B4nyjB6kkdqaTxEwRAEfSikCCppnpuZWk6ba74NJgBnXgqTk5BAfmgSRGnatpjHuR%2BYYRkZMm%2BXJzmaZZxDWfcM4BfFOQaSFrnhYGHkhlFGbGfegUKWsIk0gui4KngVAQKGRCnLQqDIAA1mh7msQoXEAHTNW1/rlcVi6VSJ1UKv17XoAAtA8m6oNuZ7Hr2XVCeVMXjfKkljQqDRKDtW1ibFm0KghSG7MQxBolC5xmGYABiWwMFqpWnPq2JbAQpw4VKL7ED15hmNdlxjgQiEgK4tA%2BeN23FbDsZpEKVzWewIlnSA%2Brwq1OoYVhj7PvhDy8d12y7ND1FevxdGrVTRYxdt9P/uynJuAghgRPWzFlhWFOYDB1RUdz0q8z9eHxILEGg%2BDjBMOWmCdmzcIdlQxCoKx2n5lCrPsw0eKjS8QuQcdpzawwHMHpSot/SFJMUUxxNcZxpPcaN0bSbJalNK27Z8eevY5vzI70bTU7ajcl2alScNHSdXXhK1Cg9RKBY6jETADUtk7rVV43oyhSs4wQBFuNhBN09gY6K2hOoa5yeX59XKtqzXCD5v65OLnCtbEAwqmDuptINx2tdUZne4MznolSbGN7rtqDB6pghpWpSFuWiaNoLfEQpYBVMebbVEBxwwCc9ZgLBpn8g0HaJk8nV32wMB3Elu8bi7o/wxA6vCyAIEfnbx0TnEYA4R/SjzBkhRGmArj/yyCfROrhBpURMAAVgsCvKkKC7IByEqeeO5VB7x1moRXUBojQmnghAkAW5TQYLWNnSSDDhoKmPqfNOA01jEOwKQpe5CLSUPBjQiAdCmHG3hiJNcmpTjzUWkJNegdiCnjWmHbekc953xYQA%2BBZ9kpKAEUhGRppWFAMwCAsqVFjFn2emA4OfspwWK0afRB9DybiINkjFGspiql1wtbcyXUuLk3RjcO4xccEh2HETYx35GSMzjABbM/YuKnC5pLHxv1CZRhLIbLxxsA6O3tqxX%2BOtaD6OoUwdqCtULoC1mzM2utCIQGHLXFxYpX4iVem2DsY8/ThLsUokOQ0xH73Gg/HuvFin1NoFwrpvtlr7h6YxVx7TiqSI3OfLeczJzyNwbY%2BZQyRojOqkUupHMuGGK2QJRZwkNov2nhIueiMMBEnlhg9u0cNHyhOSUrhPDl4UNdvSURhz7nFQAPRgpNmIesfy%2BGUjeoWCI4dWy9wwacfMrZkZ0FlvQU4XlkAA2kojPAyMDK5PflQoehdi6mw5lEnUky6X0x/CWTMCTSQAHVDAEAACqoEUJgZ4DAcwpNOIbXmCiwIGwguShUJIuWCD5QKoVOZ5FcVPFxQplz5JOwojMn21yIAtIDG00Fxs1nz0Xv8/hIq6EqIjrvD5N90ZoC8F9KEN1AYACVbiYBJR1Qw6KTQAzusDNwUskKQ2WXaP8pYPFko6RlAQxMk1P2ZbE1lLwIWnBCF5Tkv9bitQRdZU4bM0iWWUuiqg2pth7OAl7T6UobicnImKsumEXjhC%2BuSUBXbWzEGAMgKixTFGjjaMAK4Qz9bG2zQAWSYOEaR8Juq90/u%2BDURhTgjyarVLyxpPioGrWFU4SkuxsCupgVQuw0Kcm0iWw9W7CxitVugF5V00ZUK8AwPAABHLw8tcaXHldypVSgVUmiJqenwdNJ7TpOvk52ZFnbP3lFB897l0bkkqV%2B39/6sIKt5fysDwqIONI0rbXYwK7mxsXNmnlT6cpXX1EwP4/anpagLFsWgWoxDdTeiQVqo8jifGPWPeS2oYjaFuF9d6tB6ytHaie1C2pq1fp6eEYAIkEMUSQxRZZnzOUgaI4KkjFoxzx1GiRCjVIWXOt6k819uw4Wmio4dae797Mvpec59ucTO5eUfvaGJRVngcA2LQTgKDeB%2BA4FoUgqBOBuGsNYXTuxbqJB4KQAgmgwsbFaiAFBXAeoZeSIVyQXBkgAE5EgBA0JIMw0gIscEkNFnL8XOC8AUCADQWWcsbDgLAJAl7bhupIOQSg46FDKEMA0IQCBUD6hi5ltAF9sVNGmxzObC2YtxZW2kOg4w2RGBrsQL9gm9sHeIAAeTdVtxbbXhvIGeAOrrHBeCPZaPgGLvB%2BCCBEGIdgUgZCCAFWoNruguD6EMMYZLlh9DCi65ADYqB5KvemghQMphLDWDMHF6zegELhA27N%2Bb93uC8CLpgPYmX9TJTSJwHg4XIutdi7wBLHBsCqBGzZU4qgAjEWmsRSQJ6DCbqNadk%2BNoIBJex3D04uBCAkHS1wNYFO%2Bt5ZAJIAIPViIBAawEFBeuNDES4JVlBxF9CcBa6QHbbOOu2G6711n9DLccDMLwFg/geu2/a29p3WgXfhyyM4SQQA%3D).

# Resources

[Observer vs Publisher-Subscriber](https://betterprogramming.pub/observer-vs-pub-sub-pattern-50d3b27f838c)