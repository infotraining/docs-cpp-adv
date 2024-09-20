## Policy Based Design

**Policy Based Design** to technika programowania, która pozwala na wybór zachowania w czasie kompilacji. W C++ jest realizowana za pomocą szablonów klas i techniki statycznego polimorfizmu.

Polega ona na rozdzieleniu klasy algorytmu od tzw. klasy wytycznej, która określa szczegóły implementacyjne (*policy*). Klasa wytycznej jest przekazywana jako parametr szablonu klasy.

Każda wytyczna:

* Ustala interfejs dotyczący jednej konkretnej czynności
* Określa jeden sposób zachowania lub implementacji
* Może być implementowana na wiele sposobów

```{note}
**Policy Based Design** jest podobny do wzorca projektowego **Strategia** (ang. *Strategy*) z tą różnicą, że w przypadku użycia wytycznych wybór zachowania odbywa się w czasie kompilacji.
```

Przykład klasy wektora sparametryzowanej wytycznymi:

 * `RangeCheckPolicy` - wytyczna sprawdzająca poprawność zakresu
 * `LockingPolicy` - wytyczna realizująca mechanizm synchronizacji dostępu do obiektu w środowisku wielowątkowym

```{code-block} cpp
template 
<
    typename T, 
    typename RangeCheckPolicy, 
    typename LockingPolicy = NullMutex
>
class Vector : public RangeCheckPolicy
{
    std::vector<T> items_;
    using mutex_type = LockingPolicy;
    mutable mutex_type mtx_;

public:
    using iterator = typename std::vector<T>::iterator;
    using const_iterator = typename std::vector<T>::const_iterator;

    T& at(size_t index); // implementation is using the range check policy
};
```

Przykładowa klasa wytycznej sprawdzającej poprawność zakresu:

```{code-block} cpp
class ThrowingRangeChecker
{
protected:
    ~ThrowingRangeChecker() = default;

    void check_range(size_t index, size_t size) const
    {
        if (index >= size)
            throw std::out_of_range("Index out of range...");
    }
};
```

Inna implementacja wytycznej kontrolującej zakres indeksów:

```{code-block} cpp
class LoggingErrorRangeChecker
{
public:
    void set_log_file(std::ostream& log_file)
    {
        log_ = &log_file;
    }

protected:
    ~LoggingErrorRangeChecker() = default;

    void check_range(size_t index, size_t size) const
    {
        if ((index >= size) && (log_ != nullptr))
            *log_ << "Error: Index out of range. Index="
                << index << "; Size=" << size << std::endl;
    }

private:
    std::ostream* log_{};
};
```

Implementacja metody ``at()`` wektora z uwzględnieniem wytycznych:

```{code-block} cpp
template 
<
    typename T,
    typename RangeCheckPolicy,
    typename LockingPolicy
>
T& Vector<T, RangeCheckPolicy, LockingPolicy>::at(size_t index)
{
    std::lock_guard<mutex_type> lk{mtx_};  // using the locking policy        
    
    RangeCheckPolicy::check_range(index, size()); // using the range check policy

    return items_[index];
} 
```

Kod klienta korzystający z klasy wektora sparametryzowanej wytycznymi:

```{code-block} cpp
using StdLock = std::mutex;

Vector<int, ThrowingRangeChecker, StdLock> vec = {1, 2, 3};
vec.push_back(4);

try
{
    auto value = vec.at(8);
}
catch(const std::out_of_range& e)
{
    std::cout << e.what() << std::endl;
}
```