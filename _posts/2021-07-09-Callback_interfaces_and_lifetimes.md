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
    virtual void on_new_image(Image image) = 0;
};
```

Each one of our sources could register one or more of those `ImageSubscribers` which would be notified once a new image is available.

Our image sources were looking something like this:
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
        m_subscribers.erase(std::remove(m_subscribers.begin(), m_subscribers.end(), subscriber), m_subscribers.end());
    }

    // this would of course come from the actual sensor
    void produce_image()
    {
        for(auto* subscriber : m_subscriber)
        {
            subscriber->on_new_image({});
        }
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

    Subscription(const Subscription& other) = delete;
    Subscription& operator=(const Subscription& other) = delete;
    Subscription(Subscription&& other) = default;
    Subscription& operator=(Subscription&& other) = default;


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

We could've also inherited from the a base class containing a subscription. I choose composition here because it seems more natural to me and leaves more flexibility for the user.

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
    virtual Subscription add_subscriber(ImageSubscriber* subscriber) = 0;
    virtual void remove_subscriber(ImageSubscriber* subscriber) = 0;
    virtual ~ImagePublisher() = default;
};
```

Now we can get to the implementation of our channel:

```cpp
class Channel : public ImageSubscriber, public ImagePublisher, public std::enable_shared_from_this<Channel>
{
public:
    Channel(ImagePublisher& source) : m_source(source){}

    Subscription add_subscriber(ImageSubscriber* subscriber) override
    {
        m_sinks.push_back(subscriber);
        std::shared_ptr<ImagePublisher> shared_this = shared_from_this();
        return Subscription{shared_this, subscriber};
    }

    void on_new_image(Image image) override
    {
        if(m_sinks.empty())
        {
            return;
        }
        std::for_each(m_sinks.begin(), std::prev(m_sinks.end()), [image](ImageSubscriber* sink){sink->on_new_image(std::move(image));});
        m_sinks.back()->on_new_image(std::move(image));
    }

    void remove_subscriber(ImageSubscriber* subscriber) override
    {
        m_sinks.erase(std::remove(m_sinks.begin(), m_sinks.end(), subscriber), m_sinks.end());
    }

private:
    ImagePublisher& m_source;
    std::vector<ImageSubscriber*> m_sinks;
};
```

**Note**: I implemented some move optimizations since image for us is quite a heave object (~1MB) I want to make sure to not make any unnecessary copies.

So far so good. Now we need to adapt our previous implementations. Our `ImageSource` can now internally hold the channel via `shared_ptr`.

```cpp
class ImageSource : public ImagePublisher
{
public:
    ImageSource() : m_channel(std::make_shared<Channel>(*this)){}

    Subscription add_subscriber(ImageSubscriber* subscriber)
    {
        return m_channel->add_subscriber(subscriber);
    }

    void remove_subscriber(ImageSubscriber* subscriber)
    {
        m_channel->remove_subscriber(subscriber);
    }

    void produce_image()
    {
        m_channel->on_new_image({});
    }

private:
    std::shared_ptr<Channel> m_channel;
};
```

Now we can normally instantiate our `ImageSource` like any other object and the user does not need to know anything about lifetimes or `shared_ptr`. I left out the small modifications to the `Subscription` class for brevity, check the full source if you are interested. 

The full source code for the sample above can be found [here](https://godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAMzwBtMA7AQwFtMQByARg9KtQYEAysib0QXACx8BBAKoBnTAAUAHpwAMvAFYTStJg1DIApACYAQuYukl9ZATwDKjdAGFUtAK4sGe1wAyeAyYAHI%2BAEaYxBKSAKykAA6oCoRODB7evnrJqY4CQSHhLFExXGYAbLaY9vkMQgRMxASZPn5c1bXpDU0EhWGR0bEJCo3Nrdkdo739xaUSlQCUtqhexMjsHOYAzMHI3lgA1CbbbgBumA4kJ9gmGgCCO3sHmMenbCwkAJ43d49muwY%2By8RxObico2ImFYvweTyBLzebjEwBIhAQLFhjwekK8DkOAEkWExgJg/iYAOxWCkAEROVjhD32TAUCkJxNJQi8EQUyGIeFK5Kpf0S3NoeGQID%2BhxlhzOeGaXjEctQeHQhwEAH0QgB3TV4DmYCBEkmvA2mxZvGmHDT06Wy%2BWK5UAPxNnO5vP5pQglpO1qwVCYXloBDtcNpYex92ZrMOXJ5fLwiTqkb%2BMbZbpUYrwCgQ0SFDPuooi4sl9pljoIStocY9ieT6UOTHQ6E1CjrXuixsN8c9AuiACpDu2E53iL7ttbbdtC7K5Qqq8qzqr1VDPhc2x3%2B8Ru6be4nSkOR33ShOp5G55Xq4dXYblNnc12z4cA0GQ6mIzPyUyDLH9/yGwEAsRWzMsHjnf8kzqCBRnQEAQFzJpMFbZNiDBTN7xLHM8zQ7ZbkqYdVnWTBSHZPct0PYcKOiS0QEOFg2yIjYYKYzBlnozdR23GDqPHctjmFcDZUpOlGXuCCt0AhgIDQBhRlrLipPMCoNQIHDnywegCDJL8hJlSClII1BEmiJgiFwmkZIEeSDJTIy1Joq0XxqTBtIvWVbPSCBPKAyplNU9SnNfYNQ108SPMkuyVOM0zzL9bzIvSZT/NQBzxyCzBAxC1M9JvHzpMWfjKVnOdZTwKgICDIhDloVBkAAa2QpyGIUViADpaoan1Ctyudiv40rZU6xr0AAWhuNdUA3Y8Dy7FreMKsLBplESBtlGolDWlbBPC5bZVg%2BCNmIYgkTBY4zDMAAxVYGHVfLDh1dFVgIQ5MPFR9iDa8wzFO05hwIOCQFcWh3MG1bcvBqNEn5M4zPYfiDpAHVoXqzVUPQu8Hxwm4ONatYNlBsj3S4yj5pJ/MwtWymfxZNk3AQQwQhrOji1LInMEg0pSNZiV2be7Dom50D/sBxgmBLTA2wZqFWyoYhUAYtSczBenGZqLF%2BoeHmwN2w5VYYJnd1JfmPv8vHiNo3HWJY/G2P6iMxIkxS6ibFtOJPLtM05wcqPJ9KpuifksCKna9pa4J6oUNrRVzTUIiYLqZrHRaSsGxHEJltGCFwtwMKxinsGHaXkM1JW2T9IukNl%2BXFYQHMfUJucoSrYgGAUvslKpDOS7L0ik%2B3KnU4E0So0vFcNQYbVMD1c1SSNs1DUtAPjrVMkIdD5byogcOGEjtrMBYZMvm6raBKHvbm7WBhG%2BEh3dbnRH%2BGITVoWQBBt7bCOo6iYBgh9PuAbwWhpgM4H9Ui7yjq4bqpETBxAsLPMkcRLJe14keCOhUu4R3GnhLUup9SGhgoAkA64jQIMWCnESFDeqyh3nveOXVFjYOwLg6e%2BDTSEMBiQiAZCqG60hvxZcapDiTWmrxee3tiBHgWhqC4K9g7r3PjQz%2BED97EBZEaRGIijS0O/pgX%2BBVSI6P3rdf%2BvsPbjkMcoveUDyGE34VrGGcMpS5TzlhU2BEWqsUJojC4Vwc4oL9gOHGOiPx0mptGX8GYeysUOCzYWrj3rY3DIWbWzjdZe2tpbBib81a0A4fBYkjUpZVxVgzA26s8IQAHGXWxwo778Xus2Vs/dvQBPMVIv2PU%2BEb0GpfVuHEcnlNoEwpp7tZo7haTROx9TcqCNXAfAOYyxziNQWY8ZXS%2Bo9NKtkspTMmFaKWdxSZfElq3xHgI8e0MMB4klgghuCjT47NyUwlhM8CH2xpLwzZ5yHF4FhtpNJD8iHdxQtnUpuTgmakGUzUJOUIm00JAAdUMAQAAKqgRQmB7gMEzLEw42t2YSOAlrUCgLZQEmRYIdFmLsWZnEaxI8rEsmHKkjbYiIy3bHIgDU30dSfm6zmRPKebz2G4rITIwOq8Q6KJlIjNAXgXpgjOt9AASpcTAfymqGEOAgr6F1fpuBFvBYG0zrTfiLI4gFDTEoCFxja6%2BlNPyFj%2BAAehdYcAIrk2Rv0uPVB6DMXoM0SCZOSOqqAajWGsgCLtnriguGyIi%2BL85oQeMEF6xI/5pqbMQYAyBSI5MkUOJowAzhdM1rrN1hwACyTBgjCOhK1NuT8XyqiMIcXuNVyquQNK8VA4aHKHFku2NgJ1MCqA2MhNkakzIan7XmfF8t0A3JOgjIhXgGB4AAI5eElujU4FKUXUqULSw0OMh0%2BApkPcte0Mm20IrbG%2BMpz0jqcojQpkt11bp3ehSlaKMXHpxaeypylzYbC%2BWc81c5K2ovnalHCD0mBfGzTddUuZVi0HVGIVqD0SD1T7nsV4A7%2B5SQ1BEbQlwXqPVoDWRojVB1IVnYcddLTgjAH4re4i97iLTJlUiw9/6sWAdNMOCO/VKigbJE60%2BEno6LpuWwue4Htojwfu1K5S6NgKaNLw/ifS27TmdVJsSHBli0E4HEXgfgOBaFIKgTgbhrDWC4xsc62weCkAIJoEzyx6ogDiFwNqbmKT%2BckFwCkABObYAAODQkgzDSDMxwSQlmvO2c4LwBQIANAea88sOAsAkBjsuAqkg5BKDFoUMoQwNQhAIFQDqKz7m0CHzoGZdIVWma1fq1ZmzzXEh0CGMyIwpdiDrrw31gbxAADyCqusNdS0V5A9wc2ZY4LwRbDR8BWd4PwQQIgxDsCkDIQQmK1Cpd0B0AwRgUCOcsPoAUmXIDLGMnUVbo1YJ%2BlMJYawZgbMSb0LBYIHWat1fm9wXg2dMCbHczqNRiROA8FM%2BZlL1neB2Y4NgVQxXzKHFUFFioo0KiSEHVd4AhxuWjd3paCADnvt3cOLgQgJBXNcEWBD3LPmQCSCi21CoUX4tRbiPzjQFQuDhbiFURLyXSA9bR%2Bl2wWWcuo/IfoTgZheAsAkBobLsu0traV1oFXsjUjOEkEAA%3D).

# Resources

[Observer vs Publisher-Subscriber](https://betterprogramming.pub/observer-vs-pub-sub-pattern-50d3b27f838c)