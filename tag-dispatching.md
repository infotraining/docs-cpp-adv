## Tag dispatching

Czasami pożądane jest dostarczenie wyspecjalizowanych implementacji
dla wybranej funkcji lub klasy w celu poprawy wydajności lub uniknięcia problemów.

Przykładem może być implementacja funkcji `advance()`, która przesuwa
iterator `it` o zadaną `n` ilość kroków. 

Generyczna implementacja może operować na dowolnym typie iteratora:

```{code-block} c++
template<typename InputIterator, typename Distance>
void advance(InputIterator& x, Distance n)
{
    while (n > 0) 
    {  
        ++x;
        --n;
    }
}
```

Nie jest to implementacja optymalna dla iteratorów o swobodnym dostępie (np. `vector<int>::iterator`).

Optymalizacja funkcji polega na utworzeniu grupy funkcji pomocniczych, które mogą być dopasowane do odpowiedniego rodzaju 
iteratora za pomocą "taga", który umożliwia przeciążenie tych funkcji.

```{code-block} c++
template<typename Iterator, typename Distance>
void advance_impl(Iterator& x, Distance n, std::input_iterator_tag)
{
    // complexity - O(N)
    while (n > 0) 
    {  
        ++x; 
        --n;
    }
}

template<typename Iterator, typename Distance>
void advance_impl(Iterator& x, Distance n, std::random_access_iterator_tag) 
{
    // complexity - O(1)
    x += n;
}
```

Funkcja `advance()` po prostu przyjmuje argumenty i przekazuje je do funkcji pomocniczej. Na podstawie "taga"
odbywa się dopasowanie odpowiedniej implementacji.

```{code-block} c++
template<typename Iterator, typename Distance>
void advance(Iterator& x, Distance n)
{
    advance_impl(x, n,
        typename std::iterator_traits<Iterator>::iterator_category{});
}
```

### Cechy iteratorów

Klasa cech `std::iterator_traits` umożliwia dostęp do informacji o iteratorze, takich jak:

* typ elementu, na który wskazuje iterator - `std::iterator_traits<Iterator>::value_type`
* kategoria iteratora - `std::iterator_traits<Iterator>::iterator_category`

```{code-block} c++
std::vector<int> vec;
using Iterator = std::vector<int>::iterator;

static_assert(std::is_same_v<std::iterator_traits<Iterator>::value_type, int>);
```

Biblioteka standardowa definiuje zbiór typów opisujących kategorię iteratora:

```{code-block} c++
namespace std 
{
    struct input_iterator_tag { };
    struct output_iterator_tag { };
    struct forward_iterator_tag : public input_iterator_tag { };
    struct bidirectional_iterator_tag : public forward_iterator_tag { };
    struct random_access_iterator_tag : public bidirectional_iterator_tag { };
}

static_assert(
    std::is_same_v
    <
        std::iterator_traits<Iterator>::iterator_category, 
        std::random_access_iterator_tag
    >
);
```