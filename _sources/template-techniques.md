# Szablony - techniki programowania

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

## SFINAE + enable_if

Jednym z podstawowych narzędzi w meta-programowaniu w C++ jest szablon
**enable if** wykorzystujący mechanizm **SFINAE**.

Motywacją dla stosowania `enable_if` jest chęć dostarczenia dla
klientów uniwersalnego interfejsu, bez zmuszania ich do podejmowania
decyzji o wyborze określonych implementacji. Decyzje takie są
podejmowane przez twórców bibliotek. Dobór optymalnych
implementacji w zależności od typów danych dokonywany jest na etapie
kompilacji.

Chcemy uniknąć kodu, który często wygląda tak:

```{code-block} c++
// Don't even dare to pass an array of complex objects to this function!!!
template <typename T>
void store_blob(const T* src, size_t n)
{
    // impl
}
```

### SFINAE

**Substitition Failure Is Not An Error (SFINAE)** jest mechanizmem
wykorzystywanym przez kompilator w trakcie dedukcji typów dla argumentów
szablonu. Szablony, dla których nie udaje się podstawić określonego typu
jako typu argumentu szablonu są ignorowane i nie powodują błędów
kompilacji.

W praktyce jeśli dedukcja typów argumentów dla szablonu funkcji znajdzie
przynajmniej jedno dopasowanie, błędne podstawienia są usuwane z listy
funkcji kandydujących do wywołania (*overload resolution*) i nie
zgłaszają błędów kompilacji.

Załóżmy następującą implementację szablonów funkcji:

```{code-block} c++
template <typename T> void foo(T arg)
{}

template<typename T> void foo(T* arg)
{}
```

Wywołanie `foo(42)` powoduje utworzenie instancji szablonu funkcji
``foo<int>(int)``. Ponieważ kompilatorowi nie udaje się podstawić
prawidłowo typu `int` dla drugiej implementacji działa SFINAE -
nieudane podstawienie nie generuje błędów kompilacji a implementacja
znika z listy funkcji kandydujących do wywołania. Jeśli brakowałoby
pierwszej implementacji funkcji (tej prawidłowo dopasowanej) kompilator
zgłosiłby błąd.

### Szablon enable_if

Rozważmy dwie implementacje funkcji z identycznymi interfejsami, ale
różnymi wymaganiami odnośnie typu danych:

```{code-block} c++
template <typename T>
void do_stuff(T t)       // for small objects
{
    //~~~
}

template <typename T>
void do_stuff(T const& t) // for large objects
{
    //~~~
}
```

Sygnatury tych funkcji są zbyt podobne, aby można było zastosować
klasyczne przeciążenie funkcji. Rozwiązaniem tego problemu jest
zastosowanie szablonu `enable_if` i mechanizmu SFINAE.

Typowa implementacja szablonu `enable_if` wygląda następująco:

```{code-block} c++
template <bool Condition, typename T = void>
struct enable_if
{
    using type = T;
};

template <typename T>
struct enable_if<false, T>
{};
```

Szablon ogólny jest sparametryzowany wartością logiczną (zwykle
wyrażeniem logicznym, które jest ewaluowane na etapie kompilacji) oraz
typem (domyślnie `void`). Wewnątrz struktury definiowany jest typ
`type`. W wersji specjalizowanej dla wartości logicznej `false`
takiej definicji typu brakuje.

- Jeśli wyrażenie `Condition` przekazane jako pierwszy argument
  szablonu ewaluowane jest do wartości `true`:
   
  -  ``enable_if<Condition>`` ma składową `type`
  -  i ``enable_if<Condition>::type`` odnosi się do tego typu

- Jeśli wyrażenie `Condition` ewaluowane jest do `false`

  -  ``enable_if<Condition>`` nie ma składowej `type`
  -  i ``enable_if<Condition>::type`` jest nieprawidłowym odwołaniem
  -  w rezultacie podstawienie nie udaje się (działa SFINAE)

W celu ułatwienia korzystania z `enable_if` C++14 definiuje poniższy
alias szablonu:

```{code-block} c++
template <bool Condition, typename T = void>
using enable_if_t = typename enable_if<Condition, T>::type;
```

Wykorzystanie szablonu `enable_if` do problemu wyboru implementacji
wygląda następująco:

```{code-block} c++
#include <iostream>

template <typename T>
auto do_stuff(T t) -> std::enable_if_t<(sizeof(T) < 8)>  // for small objects
{
    std::cout << "do_stuff(Small Object)\n";
}

template <typename T>
auto do_stuff(T const& t) -> enable_if_t<(sizeof(T) > 8)> // for large objects
{
    std::cout << "do_stuff(Large Object)\n";
}
```

```{code-block} c++
do_stuff('A'); // OK, small object
do_stuff(std::string("abc")); // OK, large object
```

W zależności od rozmiaru typu wydedukowanego w procesie tworzenia
instancji szablonu jest tworzona i później wywoływana odpowiednia
instancja szablonu funkcji `do_stuff()`.

Próba utworzenia instancji dla typu ``double`` zgłasza błąd
kompilacji.

```{code-block} c++
do_stuff(3.14);
```

```{code-block} bash
error: no matching function for call to do_stuff
 note: candidate template ignored: disabled by enable_if [with T = double]
```

Kompilator nie jest w stanie wygenerować żadnej instancji szablonu dla
typu o rozmiarze 8 bajtów.

Mechanizm SFINAE zapewnia dwufazowe dopasowanie funkcji do wywołania
(*two-phase overload resolution*): 

1. W fazie pierwszej `enable_if` i SFINAE eliminują funkcje kandydujące, dla których nie można zrealizować podstawienia
2. W fazie drugiej dopasowana może zostać tylko jedna funkcja z grupy funkcji kandydujących do wywołania

### Cechy typów i enable_if

Często jako w wyrażeniu logicznym, będącym pierwszym argumentem szablonu
`enable_if`, wykorzystywane są cechy typów:

```{code-block} c++
#include <type_traits>

template <typename T>
auto store_item(const T& item) -> std::enable_if_t<std::is_trivially_copyable<T>::value>
{
    // memcpy or other low-level operation are allowed
}

template <typename T>
auto store_item(T const& item) -> std::enable_if_t<!std::is_pod<T>::value>
{
    // memcpy or other low-level operation are not allowed
}
```

SFINAE może zostać użyte również do wprowadzenia ograniczeń jeśli chodzi
i zakres typów, dla których realizowane jest tworzenie instancji
szablonu:

```{code-block} c++
struct Shape
{
    //...
};

struct Rectangle : Shape
{
    //...
};

template <typename T>
auto do_stuff_with_shape(T const& shape) -> std::enable_if_t<std::is_base_of_v<Shape, T>>
{
    //...
}
```

### Domyślne argumenty szablonów funkcji

W C++11 argumenty szablonów funkcji mogą przyjmować wartości domyślne.

Pozwala to na czytelniejszą dla programisty implementację SFINAE z
`enable_if`:

```{code-block} c++
template <
    typename T,
    typename = std::enable_if_t<std::is_base_of_v<Shape, T>>
>
void do_other_stuff_with_shape(T const& shape) 
{
    //...    
}
```

Aby podnieść jeszcze bardziej czytelność kodu możemy zdefiniować
pomocniczy alias:

```{code-block} c++
template <typename T>
using IsShape = std::enable_if_t<std::is_base_of<Shape, T>::value>;
```

i wykorzystać go do implementacji szablonu funkcji z ograniczeniem
SFINAE:

```{code-block} c++
template <
    typename T,
    typename = IsShape<T>
>
void do_other_stuff_with_shape_alt(T const& shape) 
{
    //...    
}
```

### Ograniczenia w szablonach klas

SFINAE oraz `enable_if` mogą również zostać użyte dla szablonów klas.

Możemy na przykład ograniczyć możliwość tworzenia instancji szablonów
dla typów zmiennoprzecinkowych:

```{code-block} c++
template <typename T>
using FloatingPoint = std::enable_if_t<std::is_floating_point_v<T>>;

template <
    typename T,
    typename = FloatingPoint<T>
> 
class Data
{
    //... rest of the class
};


Data<double> d1; // OK

Data<int> d2;    // ERROR: no type named type in std::enable_if<false, void>
```

### SFINAE i przeciążone konstruktory

SFINAE może rozwiązać problemy związane z przeciążonymi konstruktorami
klas i ich czasami zaskakującym zachowaniem.

Załóżmy, że implementujemy klasę ``Heap``, która potrzebuje dwóch wersji
konstruktorów:

```{code-block} c++
#include <iostream>

template <typename T>
class Heap
{
public:
    Heap(size_t n, T const& v)
    {
        std::cout << "Heap(size_t, T)\n"; 
    }
    
    template <typename InputIt>
    Heap(InputIt start, InputIt end)  // range init using pair of input iterators
    {
        std::cout << "Heap(InputIt, InputIt)" << std::endl;
    }
};
```

Obecność konstruktora definiowanego jako szablon może dać zaskakujący
rezultat:

```{code-block} c++
Heap<int> h(5, 0); // range init constructor called - output: Heap(InIt, InIt)
```

Powyższy kod spowoduje wywołanie konstruktora przyjmującego jako
argumenty parę iteratorów, ponieważ ta wersja konstruktora gwarantuje
lepsze dopasowanie argumentów.

Możemy uniknąć takiego zachowania wprowadzając ograniczenie dla
konstruktora wykorzystującego iteratory:

```{code-block} c++
#include <iterator>

template <typename It>
using IteratorCategory_t = typename std::iterator_traits<It>::iterator_category;

template <typename It>
using InputIterator = std::is_base_of<std::input_iterator_tag, IteratorCategory_t<It>>;

template <typename It>
using IsInputIterator_t = std::enable_if_t<InputIterator<It>::value>;

template <typename T>
class Heap
{
public:
    Heap(size_t n, T const& v)
    {
        std::cout << "Heap(size_t, T)\n"; 
    }

    template <
        typename InputIt,
        typename = IsInputIterator_t<InputIt>
    >
    Heap(InputIt start, InputIt end)  // range init using pair of input iterators
    {
        std::cout << "Heap(InputIt, InputIt)" << std::endl;
    }
};
```

Teraz przy wywołaniu

```{code-block} c++
Heap<int>(5, 0); // output: Heap(size_t, T)
```

konstruktor z iteratorami znika z funkcji kandydujących do wywołania.

Natomiast, gdy przekażemy konstruktorowi zakres zdefiniowany przez
iteratory, to odpowiednia wersja konstruktora jest instancjonowana i
później wywołana.

```{code-block} c++
auto range = { 1, 2, 3 };
    
Heap<int>(range.begin(), range.end()); // output: Heap(InputIt, InputIt)
```

## CRTP

**Curiously Recurring Template Pattern (CRTP)** to idiom, który polega na utworzeniu klasy bazowej, która jest szablonem, a następnie dziedziczenie po niej w klasie pochodnej, podając jako argument szablonu klasę pochodną.

```{code-block} c++
template <typename Derived>
struct Base
{
    void interface()
    {
        // ...
        static_cast<Derived*>(this)->implementation();
        // ...
    }

    static void static_interface()
    {
        // ...
        Derived::static_sub_func();
        // ...
    }
};

struct Derived : Base<Derived> // CRTP pattern
{
    void implementation()
    {
        std::cout << "Derived::implementation()\n";
    }

    static void static_sub_func()
    {
        std::cout << "Derived::static_sub_func()\n";
    }
};

int main()
{
    Derived d;
    d.interface(); // output: Derived::implementation()
    Derived::static_interface(); // output: Derived::static_sub_func()
}
```

W powyższym przykładzie klasa `Base` używa CRTP do wywołania metody `implementation()` z klasy pochodnej `Derived`. Metoda `Base::static_interface()` wywołuje statyczną metodę `static_sub_func()` z klasy pochodnej.