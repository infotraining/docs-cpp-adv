
# Variadic templates

W C++11 szablony mogą akceptować dowolną ilość (również zero) parametrów. Jest to możliwe dzięki użyciu specjalnego grupowego parametru
szablonu tzw. *parameter pack*, który reprezentuje wiele lub zero parametrów szablonu.

## Parameter pack

*Parameter pack* może być

* grupą parametrów szablonu

```{code-block} cpp
template<typename... Ts> // template parameter pack
class tuple
{
    //...
};

tuple<int, double, string&> t1; // 3 arguments: int, double, string&
tuple<> empty_tuple; // 0 arguments
```

* grupą argumentów funkcji szablonowej

```{code-block} cpp
template <typename T, typename... TArgs>
shared_ptr<T> make_shared(TArgs&&... params)
{
    //...
}

auto sptr = make_shared<Gadget>(42, "ipad"s); // Args as template param: int, std::string
                                              // Args as function param: int&&, std:string&&
```

### Rozpakowanie paczki parametrów

Podstawową operacją wykonywaną na grupie parametrów szablonu jest rozpakowanie jej za pomocą operatora ``...`` (tzw. *pack expansion*).

```{code-block} cpp
template <typaname... Ts>  // template  parameter pack
struct X
{
    tuple<Ts...> data;  // pack expansion
};
```

Rozpakowanie paczki parametrów (*pack expansion*) może zostać zrealizowane przy pomocy wzorca zakończonego elipsą ``...``:

```{code-block} cpp
template <typaname... Ts>  // template  parameter pack
struct XPtrs
{
    tuple<Ts const*...> ptrs;  // pack expansion
};

XPtrs<int, string, double> ptrs; // contains tuple<int const*, string const*, double const*>
```

### Forwardowanie paczki argumentów

Najczęstszym przypadkiem użycia wzorca przy rozpakowaniu paczki parametrów jest implementacja **perfect forwarding'u**:

```{code-block} cpp
template <typename F, typename... TArgs>
void call(F f, TArgs&&... args)
{
    f(std::forward<TArgs>(args)...);
}

void foo(int x, double y, string const& z)
{
    //...
}

call(foo, 42, 3.14, "text"s); // calls foo(42, 3.14, "text")
```

Innym przykładem jest implementacja funkcji ``std::make_unique``:

```{code-block} cpp
template <typename T, typename... TArgs>
std::unique_ptr<T> make_unique(TArgs&&... args)
{
    return std::unique_ptr<T>(new T(forward<TArgs>(args)...);
}
```

## Idiom Head/Tail

Praktyczne użycie **variadic templates** wykorzystuje często idiom **Head/Tail** (znany również **First/Rest**). 

Idiom ten polega na zdefiniowaniu wersji szablonu akceptującego dwa parametry:
- pierwszego ``Head``
- drugiego ``Tail`` w postaci paczki parametrów

W implementacji wykorzystany jest parametr (lub argument) typu ``Head``, po czym rekursywnie wywołana jest implementacja dla rozpakowanej
paczki parametrów typu ``Tail``.

Dla szablonów klas idiom wykorzystuje specjalizację częściową i szczegółową (do przerwania rekurencji):

```{code-block} cpp
template <typename... Types>
struct Count;

template <typename Head, typename... Tail>
struct Count<Head, Tail...>
{
    constexpr static int value = 1 + Count<Tail...>::value; // expansion pack
};

template <>
struct Count<>
{
    constexpr static int value = 0;
};

//...
static_assert(Count<int, double, string&>::value == 3, "must be 3");
```

W przypadku szablonów funkcji rekurencja może być przerwana przez dostarczenie odpowiednio przeciążonej funkcji.
Zostanie ona w odpowiednim momencie rozwijania rekurencji wywołana.

```{code-block} cpp
void print()
{}

template <typename T, typename... Tail>
void print(const T& arg1, const Tail&... args)
{
    std::cout << arg1 << endl;
    print(args...); // args expansion
}
```

## Operator sizeof...


Operator ``sizeof...`` umożliwia odczytanie na etapie kompilacji ilości parametrów w grupie.

```{code-block} cpp
template <typename... Types>
struct VerifyCount
{
    static_assert(Count<Types...>::value == sizeof...(Types),
                  "Error in counting number of parameters");
};
```

## Ograniczenia paczek parametrów

Klasa szablonowa może mieć tylko jedną paczkę parametrów i musi ona zostać umieszczona na końcu listy parametrów szablonu:

```{code-block} cpp
template <size_t... Indexes, typename... Ts>  // error
class Error;
````

Można obejść to ograniczenie w następujący sposób:

```{code-block} cpp
template <size_t... Indexes> struct IndexSequence {};

template <typename Indexes, typename Ts...>
class Ok;

Ok<IndexSequence<1, 2, 3>, int, char, double> ok;
```

Funkcje szablonowe mogą mieć więcej paczek parametrów:

```{code-block} cpp
template <int... Factors, typename... Ts>
void scale_and_print(Ts const&... args)
{
    print(ints * args...);
}

scale_and_print<1, 2, 3>(3.14, 2, 3.0f);  // calls print(1 * 3.14, 2 * 2, 3 * 3.0)
```

```{warning}
Wszystkie paczki w tym samym wyrażeniu rozpakowującym muszą mieć taki sam rozmiar.

````{code-block} cpp
scale_and_print<1, 2>(3.14, 2, 3.0f);  // ERROR
````

## "Nietypowe" paczki parametrów

Podobnie jak w przypadku innych parametrów szablonów, paczka parametrów nie musi być paczką typów, lecz może być paczką stałych znanych na etapie kompilacji

```{code-block} cpp
template <size_t... Values>
struct MaxValue;  // primary template declaration

template <size_t First, size_t... Rest>
struct MaxValue<First, Rest...>
{
    static constexpr size_t rvalue = MaxValue<Rest...>::value;
    static constexpr size_t value = (First < rvalue) ? rvalue : First;    
};

template <size_t Last>
struct MaxValue<Last>
{
    static constexpr size_t value = Last;  // termination of recursive expansion
};

static_assert(MaxValue<1, 5345, 3, 453, 645, 13>::value == 5345, "Error");
```

## Variadic Mixins

**Variadic templates** mogą być skutecznie wykorzystane do implementacji
klas **mixin**

```{code-block} cpp
template <typename... Mixins>
class X : public Mixins...
{
public:
    X(Mixins... mixins) : Mixins(std::move(mixins))... 
    {}
};

struct A
{
    int id_a;

    A(int id) : id_a(id) { std::cout << "A" << std::endl; }
    void foo() { std::cout << "A::foo()" << std::endl; }
};

struct B
{
    std::string id_b;

    B(std::string id) : id_b(std::move(id)) { std::cout << "B" << std::endl; }
    void bar() { std::cout << "B::bar()" << std::endl; }
};

int main()
{
    X<A, B> x{A{1}, B{"one"}};
    x.foo();
    x.bar();
}
```