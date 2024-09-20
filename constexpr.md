
# Programowanie na etapie kompilacji - constexpr

C++11 wprowadza dwa znaczenia dla "stałej":

-  ``constexpr`` - stała ewaluowana na etapie kompilacji
-  ``const`` - stała, której wartość może zostać ustalona w czasie wykonywania programu i nie może ulec zmianie

```{note}
Modyfikator `constexpr` użyty w deklaracji zmiennej implikuje, że jest ona `const`.

Modyfikator `constexpr` użyty e deklaracji funkcji lub składowej statycznej klasy implikuje również deklarację `inline`.
```

## Stałe wyrażenia - constant expressions

Stałe wyrażenie (**constant expression**) jest wyrażeniem ewaluowanym
przez kompilator na etapie kompilacji. Nie może zawierać wartości, które
nie są znane na etapie kompilacji i nie może mieć także efektów ubocznych.

Stałe wyrażenia mogą być używane jako wartości parametrów NTTP szablonów, rozmiarów tablic, itp.

Jeśli wyrażenie inicjalizujące dla ``constexpr`` nie będzie mogło być
wyliczone na etapie kompilacji kompilator zgłosi błąd:

```{code-block} cpp
int n = 1;
std::array<int, n> a1;  // error: n is not a constant expression

const int cn = 2;
std::array<int, cn> a2; // OK: cn is a constant expression

const size_t tab_size = 1024;              // tab_size is usable in constant expressions
constexpr size_t buff_size = tab_size * 2; // OK - constant expression

std::array<int, buff_size> a3; // OK: buffer_size is a constant expression
```

W wyrażeniu ``contexpr`` można użyć:

-  Wartości typów całkowitych, zmiennoprzecinkowych oraz wyliczeniowych
-  Operatorów nie modyfikujących stanu (np. `+`, `?` i `[]` ale nie `--` lub `++`)
-  Funkcji ``constexpr``
-  Typów literalnych
-  Stałych ``const`` zainicjowanych stałym wyrażeniem


## Stałe constexpr

W C++11 ``constexpr`` przed definicją zmiennej definiuje ją jako stałą,
która musi zostać zainicjowana wyrażeniem stałym.

Stała ``const`` w odróżnieniu od stałej ``constexpr`` nie musi być
zainicjowana wyrażeniem stałym.

```{code-block} cpp
constexpr int x = 7;

constexpr auto prefix = "Data";

constexpr double pi = 3.1415;

constexpr double pi_2 = pi / 2;
```

## Funkcje constexpr

W C++11 funkcje mogą zostać zadeklarowane jako ``constexpr`` jeśli spełniają dwa wymagania: 

* Ciało funkcji zawiera tylko jedną instrukcję ``return`` zwracającą wartość, która nie jest typu `void` 
* Typ wartości zwracanej oraz typy parametrów powinny być typami dozwolonymi dla wyrażeń `constexpr`

C++14 znacznie poluzowuje wymagania stawiane przed funkcjami ``constexpr``. Funkcją ``constexpr`` może zostać dowolna funkcja o ile:

* nie jest wirtualna
* typ wartości zwracanej oraz typy parametrów są typami literalnymi
* zmienne użyte wewnątrz funkcji są zmiennymi typów literalnych
* nie zawiera instrukcji ``asm``, ``goto``, etykiet oraz bloków ``try-catch``
* zmienne użyte wewnątrz funkcji nie są statyczne oraz nie są ``thread-local``
* zmienne użyte wewnątrz funkcji są zainicjowane
* nie alokuje dynamicznie pamięci (używa ``new`` oraz ``delete``)

**Funkcje ``constexpr`` nie mogą mieć żadnych efektów ubocznych.**
Zapisywanie stanu do nielokalnych zmiennych jest błędem kompilacji.

```{note}
W C++20 znacznie poluzowano wymagania stawiane przed funkcjami ``constexpr``. Od C++20 Funkcja ``constexpr`` może:

* być wirtualna
* zawierać instrukcje ``asm`` oraz bloki ``try-catch`` (wciąż nie może zawierać instrukcji `throw`)
* alokować pamięć dynamicznie przy pomocy ``new`` oraz ``delete`` pod warunkiem, że cała zaalokowana pamięć zostanie zwolniona przed opuszczeniem funkcji (można wykorzystywać obiekty `std:vector` i `std::string`, ale nie można ich zwrócić z funkcji)
```

Przykład funkcji ``constexpr`` w C++11:

```{code-block} cpp
constexpr int factorial(int n)
{
    return (n == 0) ? 1 : n * factorial(n-1);
}
```

Przykład funkcji ``constexpr`` w C++14:

```{code-block} cpp
// C++14 constexpr functions may use local variables and loops
#if __cplusplus >= 201402L
constexpr int factorial_cxx14(int n)
{
    int res = 1;
    while (n > 1)
        res *= n--;
    return res;
}
#endif // C++14
```

Funkcja ``constexpr`` może zostać użyta w kontekście, w którym wymagana
jest stała ewaluowana na etapie kompilacji (np. rozmiar tablicy natywnej
lub stała będąca parametrem szablonu):

```{code-block} cpp
#include <array>

constexpr size_t square(size_t n)
{
    return n * n;
}

const int n = 3; 

int tab_1[factorial(2)]; 
int tab_2[factorial(size)]; 
std::array<int, factorial(square(2))> tab_3;
```

### Instrukcje warunkowe w funkcjach constexpr

Pominięty blok kodu w instrukcji warunkowej nie jest ewaluowany na
etapie kompilacji.

```{code-block} cpp
#include <stdexcept>

constexpr int low = 0;
constexpr int high = 99;

constexpr int check(int i)
{
    if (low <= i && i < high)
        return i;
    else
        throw std::out_of_range("range error");
}

int main()
{
    constexpr int value_1 = check(50);  // OK

    constexpr int value_2 = check(200);  // ERROR
}
```

## Typy literalne

C++11 wprowadza pojęcie **typu literalnego** (*literal type*), który
może być użyty w stałym wyrażeniu ``constexpr``:

Typem literalnym jest:

* Typ arytmetyczny (całkowity, zmiennoprzecinkowy, znakowy lub logiczny)
* Typ referencyjny do typu literalnego (np: ``int&``, ``double&``)
* Tablica typów literalnych
* Klasa, która:
  
  - ma trywialny destruktor (może być ``default``) 
  - wszystkie niestatyczne składowe i typy bazowe są typami literalnymi 
  - jest agregatem lub ma przynajmniej jeden konstruktor ``contexpr``, który nie jest konstruktorem kopiującym lub przenoszącym (konstruktor musi mieć pustą implementację, ale umożliwia inicjalizację składowych na liście inicjalizującej)

```{code-block} cpp
class Point
{
    double x_, y_;
public:
    constexpr Point(double x, double y) : x_(x), y_(y) {}
    constexpr int x() const { return x; }
    constexpr int y() const { return y; }
};

constexpr Point p1(1.0, 2.0);
```

Typy literalne mogą być używane w funkcjach `constexpr` jako:
* typy parametrów funkcji
* typy zwracane przez funkcje

```{note}
Od C++17 `std::array<T, N>` i `std::string_view` są typami literalnymi.
```

```{code-block} cpp
template <typename T, size_t N>
constexpr auto sum(const std::array<T, N>& data)
{
    T result{};
    
    for (const auto& value : data)
    {
        result += value;
    }
    
    return result;
}

constexpr std::array<int, 5> data{1, 2, 3, 4, 5};
constexpr auto result = sum(data); // result = 15: evaluated at compile time
```

## Obliczenia na etapie kompilacji - lookup tables

Funkcje `constexpr` mogą być wykorzystane do obliczeń na etapie kompilacji, co pozwala na optymalizację kodu i zwiększenie wydajności programu. 


```{code-block} cpp
constexpr size_t fibonacci(size_t n)
{
    if (n == 0 || n == 1)
        return n;
    
    return fibonacci(n - 1) + fibonacci(n - 2);
}

template <size_t N, typename F>
constexpr auto create_lookup_table(F f)
{
    std::array<size_t, N> result{};
    
    for (size_t i = 0; i < N; ++i)
    {
        result[i] = f(i);
    }
    return result;
}

constexpr auto fibonacci_table = create_lookup_table<20>(fibonacci); // evaluated at compile time
```