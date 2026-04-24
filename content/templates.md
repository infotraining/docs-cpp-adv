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
    sqrt(2);	 // sqrt<int>(int) 
    sqrt(2.0);   // sqrt(double) 
    sqrt(z);	 // sqrt<double>(complex<double>)
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
std::common_type_t<T1, T2> max_value(T1 a, T2 b)
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

## Szablony klas

Podobnie do funkcji, klasy oraz struktury też mogą być być szablonami sparametryzowanymi typami.

Szablony klas mogą być wykorzystane do implementacji kontenerów, które mogą przechowywać dane typów, które będą definiowane później.

W terminologii obiektowej szablony klas nazywane są *klasami parametryzowanymi*.

W przypadku użycia szablonów klas generowany jest kod tylko dla tych funkcji składowych, które rzeczywiście są wywoływane.

```{code-block} cpp
template <typename T>
class Vector 
{
    size_t size_; 
    T* items_;
    
public: 
    explicit Vector(size_t size);
    
    ~Vector() { delete [] items_; }
    
    //...
    
    const T& operator[](size_t index) const
    {
        return items_[index];
    }

    T& operator[](size_t index)
    {
        return items_[index];
    }

    const T& at(size_t index) const;

    T& at(size_t index);
    
    size_t size() const
    {
        return size_;
    }
};
```

### Implementacja funkcji składowych

Definiując funkcję składową szablonu klasy należy określić jej przynależność do szablonu.

```{code-block} cpp
template <typename T>
Vector<T>::Vector(size_t size) 
    : size_{size}, items_{new T[size]}
{        
    std::fill(items_, items_ + size_, T{});
}

template <typename T>
T& Vector<T>::at(size_t index)
{        
    if (index >= size_)
        throw std::out_of_range("Vector::operator[]");

    return items_[index];
}

template <typename T>
const T& Vector<T>::at(size_t index) const
{        
    if (index >= size_)
        throw std::out_of_range("Vector::operator[]");

    return items_[index];
}
```

```{important}
W przypadku szablonów deklaracje typów i funkcji oraz ich implementacje muszą być w jednym pliku nagłówkowym.
```

### Użycie szablonu klasy

Aby utworzyć zmienne typów szablonowych musimy określić parametry szablonu klasy (do C++17):

```{code-block} cpp
Vector<int> integral_numbers(100);
Vector<double> real_numbers(200);
Vector<std::string> words(665);
Vector<Vector<int>> matrix(10);  
```

Od C++17 kompilator potrafi wydedukować typy argumentów szablonu klasy na podstawie wartości przekazanych do konstruktora. Mechanizm ten nazywany jest *dedukcją typu argumentów szablonu - CTAD*.

## Aliasy szablonów

### Aliasy typów

W C++11 deklaracja `using` może zostać użyta do tworzenia bardziej czytelnych aliasów dla typów - zamiennik dla `typedef`.

```{code-block} cpp
using Id = unsigned int;

using Func = int(*)(double, double);

using DictionaryDesc = std::map<std::string, std::string, std::greater<std::string>>;
```

### Aliasy szablonów

Aliasy typów mogą być parametryzowane typem. Można je wykorzystać do tworzenia częściowo związanych typów szablonowych.

```{code-block} cpp
template <typename T>
using StrKeyMap = std::map<std::string, T>;

StrKeyMap<int> my_map; // std::map<std::string, int>
```

Parametrem aliasu typu szablonowego może być także stała znana w czasie kompilacji:

```{code-block} cpp
template <std::size_t N>
using StringArray = std::array<std::string, N>;

StringArray<1024> arr1; // std::array<std::string, 1024>
```

```{important}
Aliasy szablonów nie mogą być specjalizowane.

````{code-block} cpp
template <typename T>
using MyAllocList = std::list<T, MyAllocator>;

template <typename T>
using MyAllocList = std::list<T*, MyAllocator>; // ERRROR
````

### Aliasy i typename

Od C++14 biblioteka standardowa używa aliasów dla wszystkich cech typów, które zwracają typ.

```{code-block} cpp
template <typename T>
using remove_reference_t = typename remove_reference<T>::type;
```

W rezultacie kod odwołujący się do cechy:

```{code-block} cpp
template <typename T>
typename remove_reference<T>::type&& move(T&& arg)
{
    return static_cast<typename remove_reference<T>::type&&>(arg);
}
```

możemy uprościć do:

```{code-block} cpp
template <typename T>
remove_reference_t<T>&& move(T&& arg)
{
    return static_cast<remove_reference_t<T>&&>(arg);
}
```

## Szablony zmiennych

Od C++14 zmienne mogą być parametryzowane przy pomocy typu. 

Takie zmienne nazywamy **zmiennymi szablonowymi** (*variable templates*).

```{code-block} cpp
template<typename T>
constexpr T pi{3.1415926535897932385};
```

Aby użyć zmiennej szablonowej, należy podać jej typ:

```{code-block} cpp
std::cout << pi<double> << '\n';
std::cout << pi<float> << '\n';
```

Parametrami zmiennych szablonowych mogą być stałe znane na etapie kompilacji:

```{code-block} cpp
template<int N>
std::array<int, N> integers{};    

int main()
{
    integers<10>[0] = 42; // sets value for a global array of 10 integers
    
    for (std::size_t i = 0; i < integers<10>.size(); ++i) 
    {   
        std::cout << integers<10>[i] << '\n';    
    }
}
```

### Zmienne szablonowe a jednostki translacji

Deklaracja zmiennych szablonowych może być używana w innych jednostkach translacji.

````{card}
Plik - ``header.hpp``
^^^
```{code-block} cpp
template <typename T>
T value{};

void print();
```
````


````{card}
Plik - ``unit1.cpp``
^^^
```{code-block} cpp
#include "header.hpp"

int main()
{
    value<long> = 42;
    print();
} 
```
````

````{card}
Plik - ``unit2.cpp``
^^^
```{code-block} cpp
#include "header.hpp"

void print()
{
    std::cout << value<long> << '\n'; // OK: prints 42
}
```
````

## Parametry szablonów niebędące typami - NTTP

Parametry szablonów mogą być również wartościami całkowitymi, wskaźnikami, referencjami, wskaźnikami na funkcje, itp. Wartości te muszą być stałymi znanymi w czasie kompilacji.

```{code-block} cpp
template <class T, size_t N>
struct Array 
{
    using value_type = T;        

    T items_[N];
    
    constexpr size_t size() const
    {
        return N;
    }

    constexpr T* data() const
    {
        return items_;
    }
}; 

Array<int, 1024> buffer;
```

## Parametry szablonów z wartościami domyślnymi

Parametrom szablonu klasy można przypisać argumenty domyślne (od C++11 jest to możliwe również dla szablonów funkcji).

* domyślne wartości dla parametrów szablonu, które są typami

```{code-block} cpp
template <typename T, typename Container = std::vector<T>>
class Stack
{
public:
    Stack();
    void push(const T& elem);
    void push(T&& elem);
    void pop();
    T& top() const;
private:
    Container items_;
};

// creating objects
Stack<int> stack_one; // Stack<int, vector<int>>
Stack<int, std::list<int>> stack_two; 
```

* domyślne wartości dla parametrów szablon NTTP

```{code-block} cpp
template <class T, size_t N = 1024>
struct Array 
{
    using value_type = T;        

    constexpr size_t size()
    {
        return N;
    }

    T items_[N];
};

Array<std::byte> buffer{}; // Array<std::byte, 1024>
Array<std::byte, 512> small_buffer{}; 
```

## Szablony jako parametry szablonów

Jeżeli w kodzie szablonu jako parametr ma być użyty inny szablon, to kompilator powinien zostać o tym poinformowany.

```{code-block} cpp
template <
    typename T, 
    template <typename, typename> class Container, // template as template parameter 
    typename Allocator = std::allocator<T>
>
class Stack
{
    Container<T, Allocator> items_;
public:
    Stack() = default;
    void push(const T& elem);
    void push(T&& elem);
    void pop();
    T& top() const;
};

template <typename T, template <typename, typename> class Container, typename Allocator>
void Stack<T, Container, Allocator>::push(const T& elem)
{
    items_.push_back(elem);
}

template <typename T, template <typename, typename> class Container, typename Allocator>
void Stack<T, Container, Allocator>::push(T&& elem)
{
    items_.push_back(std::move(elem));
}

//... rest of the implementation

int main()
{
    Stack<int, std::vector> stack_v;
    stack_v.push(1);
    stack_v.push(2);
    stack_v.push(3);

    Stack<int, std::deque> stack_d;
    stack_d.push(1);
}

```

## Specjalizacja szablonów klas

Specjalizacja szablonów klas pozwala na dostosowanie implementacji szablonu do konkretnego typu argumentu.
W przypadku szablonów klas specjalizacja może być częściowa lub pełna.

* Szablon ogólny

```{code-block} cpp
template <typename T>
class Holder
{
    T value_;

public:
    explicit Holder(T value)
        : value_{std::move(value)}
    { }

    const T& value() const { return value_; }
};
```

* Specjalizacja częściowa

```{code-block} cpp
template <typename T>
class Holder<T*>
{
    std::unique_ptr<T> value_;

public:
    explicit Holder(T* value) noexcept
        : value_{value}
    {
        assert(value != nullptr);
    }

    const T& value() const { return *value_; }
};
```

* Specjalizacja pełna

```{code-block} cpp
template <>
class Holder<const char*>
{
    std::string_view value_;

public:
    explicit Holder(const char* value) noexcept
        : value_{value}
    { }

    std::string_view value() const { return value_; }
};
```

```{note}
W przypadku specjalizacji szablonów klasy możliwe jest rozszerzenie interfejsu klasy specjalizowanej względem klasy ogólnej poprzez dodanie nowych funkcji składowych (np. specjalizacja `std::vector<bool>` dodaje metodę `flip()`, której nie ma w szablonie ogólnym).
```

Tworząc instancję szablonu `Holder` dla konkretnej wartości typu `T` kompilator wybierze odpowiednią specjalizację:

```{code-block} cpp
Holder<int> h1{42};
assert(h1.value() == 42);

Holder<int*> h2{new int{42}};
assert(h2.value() == 42);

Holder<const char*> h3{"Hello, World!"};
assert(h3.value() == "Hello, World!");
assert(h3.value().size() == 13);
```

### Specjalizacje częściowe szablonu klasy

Dla szablonu klasy:

```{code-block} cpp
template <class T1, class T2>
class MyClass 
{
    //...
}; 
```

możemy utworzyć następujące specjalizacje częściowe:

```{code-block} cpp
template <typename T>
class MyClass<T, T> 
{ 
    // specjalizacja częściowa: drugim typem jest T
}; 
```

```{code-block} cpp
template <typename T>
class MyClass<T, int> 
{
    // specjalizacja częściowa: drugim typem jest int 
}; 
```

```{code-block} cpp
template <typename T1, typename T2>
class MyClass<T1*, T2*> 
{
    // oba parametry są wskaźnikami 
}; 
```

Poniższe przykłady pokazują, które wersje szablonu klasy zostaną utworzone:

```{code-block} cpp
MyClass<int, float> mif;      // uses MyClass<T1,T2>
MyClass<float, float> mff;    // uses MyClass<T,T>
MyClass<float, int> mfi;      // uses MyClass<T,int>
MyClass<int*, float*> mp;     // uses MyClass<T1*,T2*>
```

W przypadku, gdy więcej niż jedna specjalizacja pasuje wystarczająco dobrze zgłaszany jest błąd dwuznaczności:

```{code-block} cpp
MyClass<int, int> me1;   // ERROR: matches MyClass<T, T> & MyClass<T, int>

MyClass<int*, int*> me2; // ERROR: matches MyClass<T, T> & MyClass<T1*, T1*>
```

## Składowe jako szablony

Składowe klas mogą być szablonami. Dotyczy to:

* wewnętrznych klas pomocniczych,
* funkcji składowych.

```{code-block} cpp
template <typename T>
class Stack 
{
    std::deque<T> items_;
public:
    template <typename IItem>
    void push(TItem&& item)
    {
        items_.push_back(std::forward<TItem>(item));
    }

    //... rest of the implementation
}; 
```

## Nazwy zależne od typów

Gramatyka języka C++ nie jest niezależna od kontekstu. Aby sparsować np. definicję funkcji, potrzebna jest znajomość kontekstu, w którym funkcja jest definiowana.

Przykład problemu:

```{code-block} cpp
template <typename T>
auto dependent_name_context1(int x)
{
    auto value = T::A(x);
    
    return value;
}
```

Standard C++ rozwiązuje problem przyjmując założenie, że dowolna nazwa, która jest zależna od
parametru szablonu odnosi się do zmiennej, funkcji lub obiektu.

```{code-block} cpp
struct S1
{
    static int A(int v) { return v * v; }
};

auto value2 = dependent_name_context1<S1>(10); // OK - T::A(x) was parsed as a function call
```

Słowo kluczowe ``typename`` umożliwia określenie, że dany symbol (identyfikator) występujący w kodzie szablonu i zależny od parametru szablonu jest typem, np. typem zagnieżdżonym – zdefiniowanym wewnątrz klasy.

```{code-block} cpp
struct S2
{
    struct A
    {
        int x;
        
        A(int x) : x{x}
        {}
    };
};

template <typename T>
auto dependent_name_context2(int x)
{
    auto value = typename T::A(x); // hint for a compiler that A is a type
    
    return value;
}

auto value = dependent_name_context2<S2>(10); // OK - T::A was parsed as a nested type
```

Przykład użycia słowa kluczowego ``typename`` dla zagnieżdżonych typów definiowanych w kontenerach:

```{code-block} cpp
template <class T> 
class Vector 
{      
public:
    using value_type = T;
    using iterator = T*;
    using const_iterator = const T*;    

    // ...
    // rest of implementation
};

template <typename Container>
typename Container::value_type sum(const Container& container)
{
    using result_type = typename Container::value_type;

    result_type result{};

    for(typename Container::const_iterator it = container.begin(); it != container.end(); ++it)
        result += *it;

    return result;
} 
```

Podobne problemy dotyczą również nazw zależnych od parametrów szablonu i odwołujących się do zagnieżdżonych definicji innych szablonów:

```{code-block} cpp
struct S1 { static constexpr int A = 0; }; // S1::A is an object
struct S2 { template<int N> static void A(int) {} }; // S2::A is a function template
struct S3 { template<int N> struct A {}; }; // S3::A is a class template
int x;

template<class T>
void foo() 
{
    T::A < 0 > (x); // if T::A is an object, this is a pair of comparisons;                    
                    // if T::A is a typename, this is a syntax error;                        
                    // if T::A is a function template, this is a function call;        
                    // if T::A is a class or alias template, this is a declaration.
}

foo<S1>(); // OK
```

Aby określić, że dany symbol zależny od parametru szablonu to szablon funkcji piszemy:

```{code-block} cpp
template <typename T>
voi foo()
{
    T::template A<0>();
}
```

Aby określić, że dany symbol zależny od parametru szablonu to szablon klasy piszemy:

```{code-block} cpp
template <typename T>
voi foo()
{
    typename T::template A<0>();
}
```

