---
layout: post
title: Uncommon Interfaces
tags: C++, Interfaces, API, Design
---

Imagine we develop a revolutionary new device which can detect at which speeds molecules or atoms move. You just point it to an object and it will give you its atoms speed. Lets name it `TemperatureSensor`.

```cpp
class TemperatureSensor
{
public:
    virtual int get_temperature() const = 0;
};
```

Now we want multiple different implementations of this. For example we want to receive temperatures via network or from a file. So we add the following Implementations:

```cpp
class FileTemperatureSensor : public TemperatureSensor
{
public:
    void set_source_file(const std::filesystem::path& source);

    [[nodiscard]] virtual int get_temperature() const override; 
};

class NetworkTemperatureSensor : public TemperatureSensor
{
public:
    void set_ip(std::string_view ip);

    [[nodiscard]] virtual int get_temperature() const override; 
};
```

So far so good. Now a new Requirement comes in. In our application we want to be able to switch between different implementations at runtime without having to adjust anything. 

So we create a wrapper which enables us todo the switching and at the same time offers the temperature interface.

```cpp
class SwitchableTemperatureSensor : public TemperatureSensor
{
public:
    void add_sensor(std::string_view name, std::unique_ptr<TemperatureSensor> sensor);

    void set_current(std::string_view sensor_name);

    TemperatureSensor& current() const;

    [[nodiscard]] virtual int get_temperature() const  override;
private:
    // ...
};
```

Right but now how can we modify any of the specifics of the implementation at runtime? We only get access to the `TemperatureSensor` reference through current, not to its actual type and implementation.

You can see how I would want to use our class below:

```cpp
int main()
{
    SwitchableTemperatureSensor sensors;
    sensors.add_sensor("Network", std::make_unique<NetworkTemperatureSensor>());
    sensors.add_sensor("Filesystem", std::make_unique<FileTemperatureSensor>());

    sensors.set_current("Network");

    auto& current_sensor = sensors.current();

    // I want todo this now, but how?
    // current_sensor.set_ip("192.168.1.1");
};
```

A full working code example can be found [here](https://godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAMzwBtMA7AQwFtMQByARg9KtQYEAysib0QXACx8BBAKoBnTAAUAHpwAMvAFYTStJg1DIApACYAQuYukl9ZATwDKjdAGFUtAK4sGIAMykrgAyeAyYAHI%2BAEaYxCAA7ABspAAOqAqETgwe3r4BaRlZAqHhUSyx8cm2mPaOAkIETMQEuT5%2BgXaYDtmNzQSlkTFxiSkKTS1t%2BZ0TA2FDFSPJAJS2qF7EyOwc5v5hyN5YANQm/m4Abt1ExKfYJhoAgrv7h5gnZ2wsJACet/dPZj2DAOXmOpzcTnGxEwrD%2Bj2ewNe7zcYmAJEICBYcIBQJBYLONHoCm%2B40wWP8d3hgJeoLe4K8DBIWGh6AA%2BiwmKlsf8DkwFAojgAVMmpOJMAgbTBCRgKEj/EwJKyPVJeaK0PDIED/I46o7nPAtLxiI4AP2FLFFxHFkulDFlxAgy3eABEjlgqEwvLQCKclQ9dXqDRLjWECEdgJgCKyCCKxRLoY6jmg7WHTq6NL75QlnZn4Y9efyjgAxOiYc2W63Q232o4gI4qtUaoWxq3xqUyuXwxX/BvqzXa3XnVB4dBHJRR2UbLaswmYCDJ8ZjgjoEAgWfE0ksVepcUIcxJMfrTaYZYKqzZ3NPR4BkwAVis98Z%2BAUomI6DvOdvrv1hpDgnDkbRi2lZzk6C5hqglzEMQI6YAOOpnvBAZHNC8YMEcGb%2BH6N4Xl2OZYfK%2BYGIWESRgA7iQADW5ZxjaHbELW9aqn2zYWrRVb0Vmfq9hqWrXoOw6juOrKTseEDjCuIBQmEwCsvqmBkUceCpKe3YXgReb%2Brqd4PhYT54C%2BzTvl%2BH6Br%2BtBKf%2BEZRjGbGtpKibgUckFxDBWBIYh/HIShkYbOhXCXjhrpZvhfo8sRApCGRhDIAgTBqmWwFttWJCMTxyCsRWyWcV23HMbxSFDiORxMOgbJKHaJDicuq7SUYRzMGwpBLpJDJ4AAjl4mCsqkBA3GcNH2RxlX9dgY70aeXknN2U0BiwIn0QoOmNXBX4ui1q5fJc4kTYF2m4VeWk6kVQmAcgGzQoI1WSXVwDjSNrIrZNR3Tdh3lHPN53QYwqb%2BK680VfaCgAHQ0Aw6A7Q9T17QhB0efey0YAZr7GZ%2BrqDSBKU3GYB5fZdBCOQI4weTNL0BqhflHAAVBAn0XT9AC0txKMm6CnhpL0Kjmmk3gjj5I4Zb4fqZP7BhZoYATZSUOWBRNhs5UFuXBU2eWTuoU8Q6F099ghMxSLMCOgevYNZQF2SBjow9N3OHakMHnOK7BIV4mT1VjACynLrRJq4MkycSYGyHJcmcPtSX1MnNWHbWdd1vX9W4GPZSNfwUpeSEe17AOLVbmepKuhBxql2v42e2cjSDriW%2BpYU14RDwSxyYSW7lGfRQQsXxfQSd0SN91A1bgMkCDpXlfREDmGYpEEBRxCUZPUc1SAHKUd1MddeC0%2Bz9R0vDfatyOuzb06kPxAj2VC0jRPZhmCWRIkrZC8bcvTCr6y69wWcd%2BJebyf7xSh905TVPiDYSeMfrXynuRKik8j71wDJ6Ig%2B4kz00EJfGsaZ%2B7D2BuAq6cC8K%2Bg4KsWgnBby8D8BwLQpBUCcDcNYawh4px0kBDwUgBBNBENWJREAt4uDA38FwBIvDJCCIAJz%2BAABwaEkGYaQJCOCSHIRw6hnBeAKBABoNhHDVhwFgEgTAqhuheGuOQSgzRgAKGUIYWoQgECoDIhQ1haALR0HFNkKx4RaC2PsRQqhzjUilniLyIw0ZiAMkoqQfxgSADyxjvEOOUQY7oDxiAWNUUEQxyBGj4AobwfgggRBiHYFIGQghFAqHUJQnQegDBGBQPQyw%2Bg8DRHUZAVYqBerZHURwBmEk0ymEsNYMwVDRJbD0BJMIHibF2ISdwXgfVMDbFYWRK0%2Bc5nENIUoqpKiODYEycY1KqgJFJAZkkSQSZal3QgH1cJToIB0MGY0o4uBCCpV2FwZY8ztFcJAJICRwMkgSNkRI28QKNBJC4KI28KR5GKNIL43gNCOBqI0VoqpywNkcDMLwLcXANCaIRTsr56LVhQUyM4SQQA%3D%3D)

# Uncommon interfaces

In the following I want to compare the different options we have for this scenario. It is not possible to introduce a (usual) common interface for our `ip` and `file_source` since they reprensent completely independent properties.

## Dynamic Casting

This is the most straightforward way and might be acceptable in a case like this.

```cpp
SwitchableTemperatureSensor sensors;
sensors.add_sensor("Network", std::make_unique<NetworkTemperatureSensor>());
sensors.add_sensor("Filesystem", std::make_unique<FileTemperatureSensor>());

sensors.set_current("Network");

auto& network_sensor = dynamic_cast<NetworkTemperatureSensor&>(sensors.current());

network_sensor.set_ip("192.168.0.1");
```

Notice i purposely left out the error handling since we should know what we are doing when we request the "Network" sensor.

Working code can be found [here](https://godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAMzwBtMA7AQwFtMQByARg9KtQYEAysib0QXACx8BBAKoBnTAAUAHpwAMvAFYTStJg1DIApACYAQuYukl9ZATwDKjdAGFUtAK4sGIAMykrgAyeAyYAHI%2BAEaYxCAAHFykAA6oCoRODB7evnppGY4CoeFRLLHxAOwatpj2RQxCBEzEBDk%2BfoF2mA5ZTS0EJZExcYnJCs2t7Xnjk4Nhw%2BWj1QCUtqhexMjsHOb%2BYcjeWADUJv5uAG49RMRn2CYaAIJ7B0eYp%2BdsLCQAnncPzzM%2BwYhy8JzObicE2ImFY/yeLxBbw%2BbjEwBIhAQLHhgOBoPB5xo9AUPwmmGx/nuCKBrzB7whXgYJCwMPQAH0WEwUjiAYcmAoFMcACrklJxJgETaYISMBQkAEmSpWJ4pLzRWh4ZAgAHHXXHC54VpeMTHAB%2BIpYYuIEqlMoYcuIEBWHwAIscsFQmF5aAQzsrHnr9YbJSawgRjsBMAQ2QRReLJTCncc0Pbw2c3Ro/QrKi6swinnyBccAGJ0TAWq02mF2h3HEDHVXqzXCuPWhPS2XyhFKgGNjVanV6i6oPDoY5KaNyzbbNlEzAQFMTccEdAgEBzklklhrlIShDmABs442W0wK0VVhzeeeT0DJgArFZH0z8ApRMR0A/c/e3QajaHBAjKMY1bKt52dRdw1QK5iGIUdMEHXUL0QwNjhhBMGGOTN/H9O8r27XMcIVAsDCLCIowAdxIABrCt41tTtiDrBs1X7FtLXo6tGOzf0%2B01bVbyHEcxwnNk8BSCAJlXEBoTCYA2QNTAKOOcTz0EpCe3U1CpLXNAvDTc4IVOMwzBlAhHCMFSUmOIhjLMFEjPEhzzmXaTXFoa88MI/0UIfJ8LBfPA3xaT8fy/IN/1oFTAMjaNYw4tspSTSDjmguI4KwXzNIDVDdXQzZMK4Ty9UVN1s284jHkLQUhAowhkAQJh1XLUD2xrEhmL45B2MrNruO7XjWP4lDh1HY4mHQdklHtEhJJXNdZMs5g2FIVy10ZPAAEcvEwNkUgIW5zjoxKuJmw7sHHRi1JyjTcNy44WDZaaHQUPzloQn9XTWkBviuSSruKjTc3zG79WEy7o2QTYYUEObpMW4BLrOtl3uuvC7tyx6odgxgDLdR7npIBQADoaAYdB/uR1HAdOfCb1Bvy3owIL31C783WOsD2tuMwj2xmGCGSgQJiyjHUPy4hMIAKggLHodxgBaO4lBTdBzyIrTSsqu9HyZ19Wa/cK/xDKKwyAuLWqSiDhfDVKYIyhDNey%2B60KjAqHrZfnFeVnoBHQJXKVikCErAp0aa1kGUjgi4JXYFCvAySzuYAWS5L6dJARlmTiTB2U5blzgzhHVozjbtt2/bDrcTm%2BrO/5KWvFCU7TgnGNejXQeblI10IeMOrlnHBAvVuzpJ1ww6vDutanp4zc5MIw4Gpu6oIBqmvoGuGLOpGXppwniBJiapsYiBzDMciCCo4hqLPkv5p%2BphqN2sudohC%2Br9oy3TodO4nXVsX96H0mk9E%2BZ9SzElJPFW%2B31ORPzZC/BC5xwEtRDrXH%2BlI/6Ny0oA4mokvawzPu/GiZ9/7az1F6Igh5jjhEvjREB290zuh%2BMtTUnt%2BS%2BnOEQ6%2Bm9v7yl5r/HB%2BDBYrFISDQMNCP70IdLg4C4lT4mS4AATjMMTLgB4EjEw0KokhjdJ4WA4GsWgnB7y8D8BwLQpBUCcDcNYawx5pz0iBDwUgBBNAGLWNREA94uDE38FwSo3jJD%2BMUf4BIGhJBmGkEYjgkhTFuMsZwXgCgQA1FceYgxpA4CwCQJgVQPR9IkHIJQFowAFDKEMHUIQCBUAUTMc4tAlo6ASiyOU8ItAqk1LMRYhpKQyzxD5EYGMxBGTUVID0vpAB5fSHTanxNyT0R4xBSmJKCHk5ATR8BmN4PwQQIgxDsCkDIQQigVDqHSaQXQyQDBGBQLYyw%2Bg8DRGSZANYqB9pZGSRwBWUl0ymEsNYMwFipynj0FJMIrTKnVNmdwXgB1MA7GcRRa03cYWGOMXE85ViODYDWQUpiqgEgHgVgeSQyZrmIwgAdEZzoIA2P%2Bfc44uBCAdT2FwFYsK3EiNIJ4yQGj1GRISPeBIgKDxKPvAefQnBYmkC6bwLFSSUkuM5WijgZheDbi4BoGosqEkcA5ekrlMEMjOEkEAA%3D%3D%3D).

## Using Type Erasure plus the Visitor Pattern

I first attempted to use a variant, however that comes into conflict with the `SwitchableTemperatureSensor` taking ownership of its contained `TemperatureSensors`.

With `std::variant` you cannot easily use references and we would need variants of different types for storing the dedicated items and for accessing them, plus some glue code to bring them together. It seemed like a lot of fuss so I implemented the standard type easure technique manually and offered to access the current sensor via a dedicated visitor:

```cpp
class SwitchableTemperatureSensor : public TemperatureSensor
{
public:
    class Visitor
    {
    public:
        virtual ~Visitor() = default;
        virtual void visit(FileTemperatureSensor& sensor) = 0;
        virtual void visit(NetworkTemperatureSensor& sensor) = 0;
    };

    template<typename T>
    void add_sensor(std::string name, std::unique_ptr<T> sensor)
    {
        m_sensors[name] = std::make_unique<TypeErased<T>>(std::move(sensor));
    }

    void set_current(std::string sensor_name)
    {
        m_current = m_sensors.find(sensor_name);
    }

    void accept(Visitor& visitor)
    {
        return m_current->second->accept(visitor);
    }

    [[nodiscard]] virtual int get_temperature() const  override
    {
        return m_current->second->get_temperature();
    }
private:
    class TypeErasedI : public TemperatureSensor
    {
    public:
        virtual ~TypeErasedI() = default;
        virtual void accept(Visitor& visitor) = 0;
    };

    template<typename T>
    class TypeErased : public TypeErasedI
    {
    public:
        TypeErased(std::unique_ptr<T> element) : m_element(std::move(element)){}
        
        void accept(Visitor& visitor) override
        {
            visitor.visit(*m_element);
        }

        [[nodiscard]] virtual int get_temperature() const  override
        {
            return m_element->get_temperature();
        }
    private:
        std::unique_ptr<T> m_element;
    };

    using SensorMap = std::unordered_map<std::string, std::unique_ptr<TypeErasedI>>;

    SensorMap m_sensors;
    SensorMap::iterator m_current{m_sensors.end()};
};
```

Compared to the `dynamic_cast` it added quite a lot of boilerplate so might not be the best solution. Here is how the usage looks like:

```cpp
class MyVisitor : public SwitchableTemperatureSensor::Visitor
{
public:
    virtual void visit(FileTemperatureSensor& sensor) override
    {
        std::cout << "FileTemperatureSensor" << std::endl;
    }
    virtual void visit(NetworkTemperatureSensor& sensor) override
    {
        std::cout << "NetworkTemperatureSensor" << std::endl;
    }
};


int main()
{
    SwitchableTemperatureSensor sensors;
    sensors.add_sensor("Network", std::make_unique<NetworkTemperatureSensor>());
    sensors.add_sensor("Filesystem", std::make_unique<FileTemperatureSensor>());
    MyVisitor visitor;

    sensors.set_current("Network");
    sensors.accept(visitor);

    sensors.set_current("Filesystem");
    sensors.accept(visitor);
}
```

Full working code for this can be found [here](https://godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAMzwBtMA7AQwFtMQByARg9KtQYEAysib0QXACx8BBAKoBnTAAUAHpwAMvAFYTStJg1DIApACYAQuYukl9ZATwDKjdAGFUtAK4sGIMwCcpK4AMngMmAByPgBGmMQSAMzSAA6oCoRODB7evv5BaRmOAmER0SxxCVzJtpj2xQxCBEzEBDk%2BfoG19VlNLQSlUbHxSdIKza3teV3j/YPllaMAlLaoXsTI7BzmieHI3lgA1CaJbgBumA4kJ9gmGgCCO3sHmMenbCwkAJ43d49muwY%2By8RxObic42ImFYvweTyBLzebjEwBIhAQLFh/0BwNBpxo9AUX3GmExiVucIBzxBrzBXgYJCwUPQAH0WEwUlj4bjaaczi08IYCFyHvsmAoFIcACqklLxJgEdaYISMBTXOEAdisDxSXhitDwyBAf0OpsOZzwrS8YkOAD8ZSw5cQFUqVQw1cQIEs3gARQ5YKhMLy0YWJbX3M3my2Km3hAiHYCYAgsgiy%2BWKqFew5od3xk5%2BjQncMmDU%2Bot/P5iiWHABidEwDqdLqhbo9hxAh11%2BsN0rTzozytV6seWr%2BXYNRpNZrOqDw6EOSmTavWmxZBMwEBz4wXBHQIBA66JJJY%2B5SCoQ5gAbAu1htMEsS1ZS%2BW4Q9IyYAKxWL8M/AKUTEOgn5lh%2BfoWlasaCAmSYpn2zYbt6W7xqgFzEMQc6YFOpqPlhkaHFCGYMIchZhrhJZlpqZaka%2B9xVpKkRJgA7iQADWjbpq6Q7EO2nZ6hOvaOhxLZcRWo46nxhrGm%2B06zvOi4sngKQQOMe4gJC4TACyFqYIxhyKQ%2B0nYWJEZ4aaKn7mgXh5qcYLHGYZgqgQjhGHpKSHEQdlmEitmKd5pw7qpri0C%2BJlGVRxaGccX6fhYv54P%2BLRAaBwFRhBtB6VBibJqmgn9kqWZIYcKHxOhWBkcZpmmgR6xEVwIXvqWxyUSFFY0XRhxCIxhDIAgTD6g2cEDq2JA8eOPbsXlwnusOOHid2k6Re1ABq8WEMODXhpGY0LaFeHgTG6W2itRQkFm%2Bb%2BpggbBqGm2Vft1rpTOc5RkUEB1vQE3wcNxBXguXHeudJG3aZ902k984Wq9DEEMxxBsYNnHTT9ZjXkoSMA4kBb1Wa5EtZFOUpAYqZggQXxyswbDSlikbg4cTDoKyaMespu77upLkU5gpABfu9J4AAjl4mAsikBA/acUo3H96PlcDeEsCyTMkAoMWcyl53mSA7IscLfOC5hEtkzgzpKEBEu/OSLOqZ8FzKf9D7UbtuM0TTsl/cmyDrFCghW2zYsadLHospzBlOxVlUK57aGMNZfoK0rxAKAAdDQDDoHbSPB6w97Y2FrW7bTTDIJsosQMda3I9ekMV6HG24ZG1XEERkdezHAC0NxKDm6Ad%2BSRclwQEDV0QxAO3LzuPJFMWqxg8UAUlIFgdGD0ZfGWWwbl8EFQI25FahpWYVP4emY3zcslH3sEL32BdwIPc3OvBNCQhufHM%2BOrofyqZSbt7VSkb2ATaYHQAASVGhJZAAkmxDREkfOW20f6VVSgdO0/85SAPFMAkBZ1MYXSuiGV%2BNNl5gzdv3TApdy4j1%2BsPEgGMsaOwauFfOkYCZEwNm4Um5Ns5U3JLhP%2BACgHzg7NtaUAjMGgNlrhBB9czRoONuI32IA9ZCxFmLMEktySHDqKSGO3oOwK20WwH2msbYbkMboh8o5367XfJFV2z0yEUNWlQlGL0a57xKhhGRRk5ZIJocQJOw8IAACoDH0CMQQMe3i34UUnjYnG0UfyzwSoBYCKVQbpTjNBbKCNMyIR3vGDxaEvF2JxsfJB%2BEkw1UOGEnRghr6P1yS/BhlUJ5bU/gqdg0TNbKOFqLcWbgNHYBqSycxghX640drhLwGQXLfQALIcl9DzJRDJALxGAWyDkYJNbs2ANzHpDABYqP6eosRpsQEW1uFMyKCylnxy4irFppo7kpH3IQdMI0W7R3GVqB5SNk6uC9JM4sz4bm0QMNWeZXxKEjWERAjqXUCA9T6h9Jp319ywp%2BpqcM0jIoZPNG7IJ70BqbxgUjX6CdvTFWKWVOB3TWYoDWNZNwtlzBmBJZ9clHp2V%2BTcCsoKEzrFELSoS56QToaw3hmSxGPLXFUqKQfSRpSzKMssiytl9lJWsS5bK649k%2BUCrTsFZ5MTRJMJolk9k4RgU4twp1bqvV%2Bq6qmm2BOTy5buqTvTRmXEIDsu1XDdlBzGXa11kc/WYJA3SugXq8W2AvRRMil6n1is/XspJUeHKwaVlhpZL0sEnL0UiUtksJNcTIxevkhfGO/qtVMVYuy8tuFoVYrcVQ8FlbHneuLuQwe/jm3Ju7dW1uPsM31izaSJteNdopt7aXAdLUwUWA4CsWgnAPy8D8BwLQpBUCcFZZYawN4Vy0gBDwUgBBNCrpWCxEAH4uBJ0SFwDUD7JAvoCIkAAHBoSQZhpDro4JILd1692cF4AoEAGhL3XpWHAWASBMCqEuFZEg5BKAtGAAoZQhg6hCAQKgRi26L1oEdHQBUWQcMRFoPhwj27d2kZSPWBIYojApmIPSFipBGPMYAPJWVo0R0DSHLj3GIFh8DwRkPICaPgbdvB%2BCCBEGIdgUgZCCEUCodQO6dB6AMEYFA1hrD6DwDESDkAVioFFlkSDHA24qXzKYI9lgzC7uXHePQKlwhUbwwRoT3BeBi0wFsC9jFnRvIC2ujdIGdNgY4NgaTqHuKqC/ZeNul5JDZn08AQ4EAxace9BAQ9VhLDc1wBXOyz6liBdg7ekAkgv1J0vF%2B/9X6PwtY0JeLgAQPyXn0JwYDpB6O8H3RwCDUGYM6bLf1jgZheAni4BoaDw24s1amysVCGRnCSCAA%3D).

## Having the Switchable class not own its Sensors

We could also refrain from taking ownsership and only act out the switching while control still lies with the abstraction above us.

Something like this:

```cpp
class SwitchableTemperatureSensor : public TemperatureSensor
{
public:
    void add_sensor(std::string name, TemperatureSensor* sensor)
    {
        m_sensors[name] = sensor;
    }

    void set_current(std::string sensor_name)
    {
        m_current = m_sensors.find(sensor_name);
    }

    [[nodiscard]] virtual int get_temperature() const  override
    {
        return m_current->second->get_temperature();
    }
private:
    using SensorMap = std::unordered_map<std::string,TemperatureSensor*>;

    SensorMap m_sensors;
    SensorMap::iterator m_current{m_sensors.end()};
};

// Usage
int main()
{
    auto network_sensor = std::make_unique<NetworkTemperatureSensor>();
    auto file_sensor =  std::make_unique<FileTemperatureSensor>();
    SwitchableTemperatureSensor sensors;
    sensors.add_sensor("Network", network_sensor.get());
    sensors.add_sensor("Filesystem", file_sensor.get());

    sensors.set_current("Network");

    network_sensor->set_ip("192.168.0.1");
};
```

This is quite straightforward and is in many cases probably the right choice. The downside is that anyone in the abstraction layer above the `SwitchableTemperatureSensor` has access to your sensors since the ownership lies with them. This can become a problem in large codebases.

## Introducing a Domain Specific Language

Another way is to not rely on the compiler and C++ but make our own "language" for treating the properties here. A simple example could be a string representation.


Imagine something like this:
```cpp
void set_property(std::string_view property, std::string_view value)
{
    // Error handling, interpretation of values, forwarding to the right methods and more...
}
```

We can then use it like this:

```cpp
int main()
{
    SwitchableTemperatureSensor sensors;
    sensors.add_sensor("Network", std::make_unique<NetworkTemperatureSensor>());
    sensors.add_sensor("Filesystem", std::make_unique<FileTemperatureSensor>());

    sensors.set_current("Network");

    sensors.set_property("ip", "192.168.0.1");
};
```

This might be useful for certain cases, however you are (likely) trading for more runtime error handling and some logic to glue everything together.

## End of the story

These are currently all the ways that I can see approaching the problem when you cannot use a common interface. Hence the term "Uncommon Interfaces". I might extend the list in the future. 

If you have a good idea which I am not aware of please let me know.
