## Statyczny polimorfizm

**Statyczny polimorfizm** to technika programowania, która pozwala na wybór zachowania w czasie kompilacji. W C++ statyczny polimorfizm jest realizowany za pomocą szablonów klas.

Zalety statycznego polimorfizmu:

* lepsza wydajność w porównaniu do dynamicznego polimorfizmu (nie ma kosztów związanych z wywołaniem funkcji wirtualnych)
* brak konieczności dziedziczenia po klasie bazowej - klasa musi jedynie posiadać odpowiednie metody

Wadą statycznego polimorfizmu jest jego statyczność - wybór zachowania odbywa się w czasie kompilacji, co oznacza, że nie można zmienić zachowania w czasie działania programu.

```{code-block} cpp
struct UpperCaseFormatter
{
    std::string format(const std::string& message) const
    {
        std::string result = message;
        std::transform(result.begin(), result.end(), 
                       result.begin(), [](char c) { return std::toupper(c); });   
        return result;
    }
};

struct CapitalizeFormatter
{
    std::string format(const std::string& message) const
    {
        std::string result = message;
        result[0] = std::toupper(result[0]);
        return result;
    }
};

template <typename TFormatter = UpperCaseFormatter>
class Logger 
{
    TFormatter formatter_;
public:   
    Logger() = default;

    Logger(TFormatter formatter)
        : formatter_(std::move(formatter))
    {
    }

    void log(const std::string& message)
    {        
        std::cout << formatter_.format(message) << std::endl;
    }
};
```

```{code-block} cpp
Logger<UpperCaseFormatter> upper_case_logger;
upper_case_logger.log("this is a sample message");

Logger<CapitalizeFormatter> capitalize_logger;
capitalize_logger.log("this is a sample message");

Logger<> default_logger;
default_logger.log("this is a sample message");

Logger logger{CapitalizeFormatter{}};
logger.log("this is a sample message");
```