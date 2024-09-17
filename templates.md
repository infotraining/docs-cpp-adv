# Szablony

Szablony implementują koncepcję **programowania generycznego** (*generic programming*) - programowania używającego typów danych jako parametrów.

Szablony zapewniają mechanizm, za pomocą którego funkcje lub klasy mogą być implementowane dla ogólnych typów danych (nawet takich, które jeszcze nie istnieją).
  * zastosowanie szablonów klas umożliwia parametryzację kontenerów ze względu na typ elementów, które są w nich przechowywane
  * szablony funkcji umożliwiają implementację algorytmów działających w ogólny sposób dla różnych struktur danych

W szczególnych przypadkach implementacja szablonów klas lub funkcji może być wyspecjalizowana dla pewnych typów.

Parametry szablonów mogą być:

* typami
* stałymi typów całkowitych lub wyliczeniowych znanymi w chwili kompilacji
* szablonami

## Szablony funkcji

**Szablon funkcji** - funkcja, której typy argumentów lub typ zwracanej wartości zostały sparametryzowane.

Składnia definicji szablonu funkcji:

```{code-block}
template <comma_separated_list_of_parameters>
return_type function_name(list_of_args)
{
    //...
}
```

Przykład definicji szablonu funkcji `max_value()`:

```{code-block} cpp
template<typename T>
T max_value(T a, T b)
{
    return b < a ? a : b;
}
```

Użycie szablonu funkcji `max_value()`:

```{code-block} cpp
int result1 = max_value<int>(4, 9);

double x = 4.50;
double y = 3.14;
double result2 = max_value(x, y); // max_value<double>(x, y)

std::string s1 = "mathematics";
std::string s2 = "math";
assert(max_value(s1, s2) == "mathematics"s);
```

### Dedukcja typów argumentów szablonu

W momencie wywołania funkcji szablonowej `max_value()` następuje proces dedukcji tych typów argumentów szablonu, które nie zostały podane w ostrych nawiasach `<>`. Typy argumentów szablonu dedukowane są na podstawie typu argumentów funkcji.

Reguły dedukcji typów dla szablonów funkcji:

* Każdy parametr funkcji może brać udział (lub nie) w procesie dedukcji parametru szablonu
* Wszystkie dedukcje są przeprowadzane niezależnie od siebie
* Na zakończenie procesu dedukcji kompilator sprawdza, czy każdy parametr szablonu został wydedukowany i czy nie ma konfliktów między wydedukowanymi parametrami
* Każdy parametr funkcji, który bierze udział w procesie dedukcji musi ściśle pasować do typu argumentu funkcji - niejawne konwersje są zabronione

```{code-block} cpp
template<typename T, typename U>
void f(T x, U y) {}

template<typename T>
void g(T x, T y) {}

int main()
{
    f(1, 2); // void f(T, U) [T = int, U = int]
    g(1, 2); // void g(T, T) [T = int]
    g(1, 2u); // error: no matching function for call to g(int, unsigned int)
}
```

W przypadku, gdy w procesie dedukcji wykryte zostaną konflikty:

```{code-block} cpp
short f();

auto val = max_value(f(), 42); // ERROR - no matching function
```

istnieją dwa rozwiązania:

```{code-block} cpp
// #1
auto val1 = max_value(static_cast<int>(f()), 42); // OK

// #2
auto val2 = max_value<int>(f(), 42); // OK
```

### Tworzenie instancji szablonu

Koncepcja szablonów wykracza poza zwykły model kompilacji (konsolidacji). Cały kod szablonu powinien być umieszczony w jednym pliku nagłówkowym. Dołączając następnie zawartość pliku nagłówkowego do kodu aplikacji umożliwiamy generację i kompilację kodu dla konkretnych typów.

Tworzenie **instancji szablonu** - proces, w którym na podstawie szablonu generowany jest kod, który zostanie skompilowany.

Utworzenie instancji szablonu jest możliwe tylko wtedy, gdy dla typu podanego jako argument szablonu zdefiniowane są wszystkie operacje używane przez szablon, np. operatory ``<``, ``==``, ``!=``, wywołania konkretnych metod, itp.

#### Fazy kompilacji szablonu

Proces kompilacji szablonu przebiega w dwóch fazach:

1. Na etapie definicji szablonu, ale bez tworzenia jego instancji kod jest sprawdzany pod względem poprawności bez uwzględniania parametrów szablonu:

   - wykrywane są błędy składniowe
   - wykrywane jest użycie nieznanych nazw (typów, funkcji, itp. ), które nie zależą od parametru szablonu
   - sprawdzane są statyczne asercje, które nie zależą od parametru szablonu

2. Podczas tworzenia instancji szablonu, kod szablonu jest jeszcze raz sprawdzany. W szczególności sprawdzane są części, które zależą od parametrów szablonu.

```{code-block} cpp
template<typename T>
void foo(T t)
{
    undeclared();   // first-phase compile-time error if undeclared() unknown
    undeclared(t);  // second-phase compile-time error if undeclared(T) unknown
    static_assert(sizeof(int) > 10, "int too small");  // always fails if sizeof(int)<=10
    static_assert(sizeof(T) > 10, "T too small");  // fails if instantiated for T with size <=10                              
}
```

### Specjalizacja funkcji szablonowych

W specjalnych przypadkach istnieje możliwość zdefiniowania funkcji specjalizowanej.

Zdefiniujmy najpierw pierwszy szablon funkcji:

```{code-block} cpp
template <typename T>
bool is_greater(T a, T b)
{
    return a > b;
}
```

Jeśli wywołamy ten szablon dla literałów znakowych `"abc"` i `"def"` utworzona zostanie instancja szablonu, która porówna 
adresy przechowywane we wskaźnikach a nie tekst. Aby zapewnić prawidłowe porównanie c-łańcuchów musimy dostarczyć specjalizowaną wersję szablonu dla parametru ``const char*``:

```{code-block} cpp  
template <>
bool is_greater<const char*>(const char* a, const char* b)
{
    return strcmp(a, b) > 0;
}

is_grater(4, 5); // wywołanie funkcji szablonowej

is_greater("a", "g"); // wywołanie funkcji specjalizowanej dla const char*
```

Ponieważ podawanie typu specjalizowanego w ostrych nawiasach jest redundantne można specjalizację szablonu funkcji
zapisać w sposób następujący:

```{code-block} cpp
template <>
bool is_greater(const char* a, const char* b)
{
    return strcmp(a, b) > 0;
}
```

W praktyce zamiast stosować jawną specjalizację szablonu funkcji można wykorzystać przeciążenie funkcji `is_greater()`:

```{code-block} cpp
bool is_greater(const char* a, const char* b)
{
    return strcmp(a, b) > 0;
}
```

#### Połączenie przeciążania szablonów, specjalizacji i przeciążania funkcji

Przykład wykorzystania specjalizacji lub przeciążenia szablonu funkcji:

```{code-block} cpp
template <typename T> T sqrt(T); // basic template sqrt<T>

template <> float sqrt(float);   // specialization of template sqrt<float>

template <typename T> complex<T> sqrt(complex<T>); // overloaded template sqrt for complex<T> arguments

double sqrt(double); // overloaded function sqrt(double)

//...

void f(complex<double> z)
{
    sqrt(2);	// sqrt<int>(int) 
    sqrt(2.0); // sqrt(double) 
    sqrt(z);	// sqrt<double>(complex<double>)
    sqrt(3.14f); // sqrt<float>(float)
}
```

### Przeciążanie szablonów

W programie może obok siebie istnieć mając tę samą nazwę:

* kilka szablonów funkcji – byle produkowały funkcje o odmiennych argumentach,
* funkcje, o argumentach takich, że mogłyby zostać wyprodukowane przez któryś z szablonów (funkcje specjalizowane),
* funkcje o argumentach takich, że nie mógłby ich wyprodukować żaden z istniejących szablonów (zwykłe przeładowanie).

### Adres wygenerowanej funkcji

Możliwe jest pobranie adresu funkcji wygenerowanej na podstawie szablonu.

```{code-block} cpp
template <typename T> void f(T* ptr)
{
    cout << "funkcja szablonowa f(T*)" << endl;
}

void h(void (*pf)(int*))
{
    cout << "h( void (*pf)(int*)" << endl;
}

int main() 
{
    h(&f<int>);    // przekazanie adresu funkcji wygenerowanej
                // na podstawie szablonu
}
```

### Parametry szablonu dla wartości zwracanych przez funkcję

W przypadku, gdy funkcja szablonowa ma zwrócić typ inny niż typy argumentów (lub przynajmniej rozważana jest taka możliwość) możemy 
zastosować następujące rozwiązania:

#### Parametr szablonu określający zwracany typ

```{code-block} cpp
template<typename TReturn, typename T1, typename T2>
TReturn max_value(T1 a, T2 b);
```

Ponieważ nie ma związku pomiędzy typami argumentów a typem zwracanym, wywołując szablon należy określić jawnie typ zwracany.

```{code-block} cpp
max_value<double>(4, 7.2);
```

#### Dedukcja typu zwracanego z funkcji (C++14)

```{code-block} cpp
template<typename T1, typename T2>
auto multiply(T1 a, T2 b)
{
    return a * b;
}
```

#### Użycie klasy cech (*type traits*)

```{code-block} cpp
#include <type_traits>

template<typename T1, typename T2>
std::common_type_t<T1, T2> max (T1 a, T2 b)
{
    return b < a ? a : b;
}
```

### Domyślne parametry szablonu

Definiują parametry szablonu, możemy określić dla nich wartości domyślne. Mogą one odwoływać się do wcześniej zdefiniowanych parametrów szablonu.

```{code-block} cpp
#include <type_traits>

template<typename T1, typename T2, 
         typename TResult = std::common_type_t<T1,T2>>
TResult max_value(T1 a, T2 b)
{
    return b < a ? a : b;
}
```

Wywołując szablon funkcji możemy pominąć argumenty z domyślnymi wartościami:

```{code-block} cpp
auto val_1 = max_value(1, 3.14); // ::max<int, double, double>
```

lub jawnie podać odpowiedni argument:

```{code-block} cpp
auto val_2 = max_value<int, short, double>(1, 4); 
```
