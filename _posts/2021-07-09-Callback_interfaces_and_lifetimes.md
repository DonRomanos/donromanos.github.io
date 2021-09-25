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

The full source code for the sample above can be found [here](https://godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAMzwBtMA7AQwFtMQByARg9KtQYEAysib0QXACx8BBAKoBnTAAUAHpwAMvAFYTStJg1DIApACYAQuYukl9ZATwDKjdAGFUtAK4sGIAGykrgAyeAyYAHI%2BAEaYxBIAzAAcpAAOqAqETgwe3r4BaRlZAqHhUSyx8VzJtpj2jgJCBEzEBLk%2BfoF2mA7ZTS0EpZExcYkpCs2t7fldk4NhwxWj1UkAlLaoXsTI7BzmCWHI3lgA1CYJbgBuPUTE59gmGgCC%2B4fHmGcXbCwkAJ73jxeZgODCOXlO5zcTgmxEwrABz1eoPenzcYmAJEICBYCJezwA9PiTspYQRXCcCAg8AoTtSTphVKlGJlroCYV4HCcAJIsJjATCAkwAdisQoAIucrIjnkcmAoaTy%2BZghF5ogpkMQ8JVBSLAalVbQ8MgQICTmaTpc8K0vGILag8OgTgIAPrhADuzrwvP5EEV/Np3swq0%2BYpOGklpvNluttoAfn7lar1ZrKhBg%2BdQ1gqEwvLQCBHEeKC3inrL5ScVWqNXhUg0GMXAWWFYHlAbqQg4jqpU99dFDcbI2bowQbbQK0nq7XsicmOh0M6FBOU3FfYHK8mtXEAFQnRdV5fEdMJUPhhLd80Wq0j22Xe2O2E/a4Lpeb4irpXr6uVHd7jeVI8nsWF7DqOJzxi2bYKB2b4AScWY5nmDZFmegoygY5afpqU4CF2eptgOzwXphNZ1hAEzoCAIBQS0mDzrWdwXAmrZ9u2nYJA8Zj%2BLumzbJgpDcmuL7fruQlxMGIAnCwC48TsZEyUG/FSb%2BX4rspB6rIOZy6oR5rChK0pPERL7YQwEBoAwEzjvuJHZOYXGoJSYkhnBdSYGSQHmsRJl2U6TLEEwtwZmZAiWV5dY%2BQ50GwVg9DuShOlmmF2QQElOGcRFjmHs58G5vm8WGZ5xnhZxvlxAFJBBal9bpSVkVORmLnZrlDYJWBVVpppwrnhe5p4FQEA5kQJy0KgyAANa0c5SnyQAdCN41phprUXl1mk9ea80TegAC09wPqgT5qa%2BEBKaJh4eetelreadRKNdZqrct627gQFEoHExCopCZxmGYABimwMI6VUnG6WKbAQxKQdBM3mGYX0XC9b2uLQF09VdrUYyWqSapcAXsJp5GUW6cJjc69GQkx0NsdgknSVsOxowmxHCad1navlV2c2hco0m4CCGOEY4Sb2/YCR%2BZ38aLRri/yzGGlBcRS/hSOUYwTB9pgC4C7C85UMQqBSZS1KQvzgt1Lij09vhJqtWbDBC%2B%2BcvU3cJUKPJ4l0%2B7DOYHJPsabqRYGUZ1kmTOc7PuzK7M2dP5ncGB0fQ6AqY9pBXrUpYRjQoM36lBzrREwC1Hf%2BaMXkTVE67R5MEAxbhUyxisMbT1G686xs0g1rfV/rhvt1SCgdflz0nKSWwMFZG7eSK3fzh3/El52yHdQ9QclsBd5OgwrqYB6XpKk7Hz7/yCfXMQmpYJ1acj31J0LlnOeYCwta/It91aSvI%2Bj2549l7pa8jwrvwYgzo4TIAQHfTIDBs4zViMAMIaYF6vUojjTAlxIEPxmq4Ra/ETAAFYLDHwFHgsUh8Wbbl3FnAOFgoFjV2uxF07pPSBjIsgkAj5fZENWBpFC4oeGfwvJnaBOdC4LVWPQ7AjDd7MIPhXDhEAuH8KvvpdeUZN77UOmdMhscRJRyyonc%2Bycr4CPNEImBZUlCsLeho32ZiRGYHgaZdYXtMHYOcYvQ8il77CKwUDN%2Bw9/4qL1LjfGtt06yxUC7Hy00fZowrtcBwFVGKCT0Vue4LjhFIQlNzUs6FmwfnkicEWKsG4K2grhZ40sCLhOZvJNMRS6bgPNrQKxlFeQTW1jRdApsBYOwtuxCAW4O7cNWmvTSINZzzg8do1Jui/xiWMe/MexAJ5SSaX02gEjJmR3mW%2BDxSjU5BNareB038OE7JUm%2BGOsz9mLKemaNZvShYSJsRcg8ZF45oyxppE5jocYYA5FrIhHVU4mIec6dZzz7hSL3iw0ZYoDnhO%2BZUkJZIwnlzYbPGudd7ZC3SY85pWSWq5N5tyAA6oYAgAAVVAihMBPAYAmBpVSInkLuIWbsVT0Xmi5BSwQNK6UMoTGQ%2BSP4PYNLZlPUi3teJbIjtM4Z6ZA5HPCb8reO9YUHyZVwp0Z8L4pyRdfZ6Fc0BeEhpCb6cMABKPRMB4GuI6QwAYlSw1%2BgjNwqsQAoy%2BeKM4Bkcb2tCeMoq05JWTnCpzZeqEniEhOMENyNJwE9DGqDAWkMBapCZBZWkVAnRbDmeG6cENDTXBpDxE40sm6AjCJDXkCCa0zmIMAZA/EmnEC3DuFowBLhLReEas0saACyTAwjfzlAIE4wC4L2iMBSAe/FDRUDcl6D4qBc2OROOZRcbBPoMh2LRGklIApOnXR2CtBt0CAs%2BoTNhXgGB4AAI5eC1hTRifLqW0qUEKwM6St0%2BDYp/K2I9ak%2B24rEgJ60/07ucnIpgHS72PufZTd9Aqv2Mp/QMuyMqdiIpWmM1qsaqVnrqp9N0TBfiNsBo6KCmxaCOtoO7UGJAxoL0OB8DdR0w6oGiNoG4oM6BjmaBNTdNET0nDvSXMIwBNIgd4mB3iPqwXkspah%2Bl6GlSUOgatTi2GBRRvufJnYucL2ApkT6fhJxCMD347QMcAbBA/TMDanY9rJpOqIa6swyjCazX%2BZenYZnfYWas3SMGtmTgMAcuehthhfjG1nVAnYEXUAFoPDSUTYhXMzgYL8H4sJo0XmWRPU83YualeeBwdYtBOB4N4H4DgWhSCoE4G4aw1hDMfH2DwUgBBNCVfWGNEAeCuAzQSFwIUw3JDjYAJzJA0JIMw0hqscEkHVvrTXOC8AUCADQPW%2BvrDgLAJADIehmpIOQSgXaFDKEMHUIQCBUBunq91tAz86ABWyDdoW93Hv1ca691IdBRiyiMO3Ygd6WMA6B8QAA8man7T31sneQE8Jt22OC8GR00fA9XeD8EECIMQ7ApAyEEHStQ63dBcH0IYYwbXLD6C1NtyA6xUAmXR9tciGZTCWGsGYRrum9DkTCF9u7D3EfcF4LXTAuxutun8qkTgPAqs1bWw13gzWODYFUKd24JxVBJH8NtfwkhN0GFnRAWuEPgwQFa7zhnJxcCEBID9Mbqwpf7YGyASQSQZr%2BCSItpIeCA8aH8FwabeDAjLdW6QP7GvNu2B23t9X3D9CcDMLwFgEgNC7bjxtjHyetCp7PpkZwkggA%3D%3D).

# Resources

[Observer vs Publisher-Subscriber](https://betterprogramming.pub/observer-vs-pub-sub-pattern-50d3b27f838c)

[boost signals2 connections](https://www.boost.org/doc/libs/1_46_1/doc/html/boost/signals2/connection.html#id1173908-bb)