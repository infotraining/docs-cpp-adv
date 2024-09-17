# Inteligentne wskaźniki

## Zarządzanie zasobami w C++ - RAII

### Mechanizm wyjątków a zasoby

Jawne zarządzanie zasobami w C++ jest trudne, ponieważ wymaga ręcznego zarządzania zasobami. W przypadku wystąpienia wyjątku, zasoby mogą nie zostać zwolnione, co prowadzi do wycieków zasobów.

```{code-block} cpp
void leaky_send(Data* data, const std::string& destination)
{
    auto port = open_port(destination);
    some_mutex.lock();
    
    send(port, data); // it may throw
    may_throw();      // it may throw
    
    some_mutex.unlock(); // leaks if send() throws
    close_port(port);    // leaks if send() throws
    delete data;         // leaks if send() throws
}
```

### RAII - Resource Acquisition Is Initialization 

Technika łączy przejęcie i zwolnienie zasobu z inicjalizacją zmiennych lokalnych i ich automatyczną destrukcją.

* Pozyskanie zasobu jest połączone z konstrukcją, a zwolnienie z automatyczną destrukcją obiektu.
* Ponieważ wywołanie destruktora dla obiektu lokalnego następuje automatycznie, jest zagwarantowane, że zasób zostanie zwolniony od razu, gdy skończy się czas życia zmiennej.
* Mechanizm ten działa również przy wystąpieniu wyjątku. 
* RAII jest kluczową koncepcją przy pisaniu kodu odpornego na wycieki zasobów.

```{code-block} cpp
void safe_send(std::unique_ptr<Data> data, const std::string& destination)
{
    Port port{destination};
    std::lock_guard<std::mutex> lk{some_mutex};
    
    // ...
    send(port, data); // it may throw
    may_throw();      // it may throw
} // automatically unlocks some_mutex, closes the port and deletes the pointer in data
```

#### Klasy RAII

Klasa RAII to klasa, która pozyskuje zasób w konstruktorze a w destruktorze go zwalnia:

```{code-block} cpp
class Port {
    PortHandle port_;
public:
    Port(const std::string& destination) : port_{open_port(destination)} {}
    ~Port() { close_port(port_); }
    operator PortHandle() { return port_; }

    // port handles can't usually be cloned
    // so disable copying and assignment if necessary
    Port(const Port&) = delete;
    Port& operator=(const Port&) = delete;
};
```

```{code-block} cpp
class lock_guard
{
    std::mutex& mtx_;
public:
    lock_guard(std::mutex& mtx) : mtx_{mtx} {
        mtx_.lock();
    }

    ~lock_guard() {
        mtx_.unlock();
    }

    lock_guard(const lock_guard&) = delete;
    lock_guard& operator=(const lock_guard&) = delete;
};
```

#### Kopiowanie obiektów RAII

Klasy implementujące RAII posiadają destruktor, dlatego należy określić sposób zachowania obiektów przy ich kopiowaniu.
Możliwe są następujące strategie:

* Całkowite blokowanie kopiowania oraz transferu prawa własności
* Blokowanie kopiowania przy jednoczesnym umożliwieniu transferu prawa własności do zasobu
* Zezwolenie na kopiowanie obiektów - obiekty współdzielą prawo własności z wykorzystaniem licznika referencji

## Inteligentne wskaźniki

Inteligentne wskaźniki umożliwiają wygodne zarządzanie obiektami, które zostały zaalokowane na stercie za pomocą operatora `new`.
Przejmują odpowiedzialność za wywołanie destruktora dla zarządzanego obiektu oraz zwolnienie pamięci - poprzez wywołanie operatora `delete`.
Inteligentne wskaźniki w C++11 umożliwiają reprezentację prawa własności do zaalokowanego zasobu.
Przeciążając operatory `operator*` oraz `operator->` umożliwiają korzystanie z nich w taki sam sposób, jak wskaźników natywnych.

Dostępne implementacje inteligentnych wskaźników w bibliotece standardowej C++ oraz w bibliotece Boost.


| Klasa wskaźnika            |  Kopiowanie | Transfer prawa własności |  Licznik referencji |
|----------------------------|:-----------:|:------------------------:|:-------------------:|
| ``std::auto_ptr<T>``       |     o       |     o                    |                     |
| ``std::unique_ptr<T>``     |             |     o                    |                     |
| `std::shared_ptr<T>`     |     o       |     o                    |    wewnętrzny       |
| ``boost::scoped_ptr<T>``   |             |                          |                     |
| ``boost::shared_ptr<T>``   |     o       |     o                    |    wewnętrzny       |
| ``boost::intrusive_ptr<T>``|     o       |                          |    zewnętrzny       |

## std::unique_ptr<T>

Klasa szablonowa `std::unique_ptr` służy do zapewnienia właściwego usuwania przydzielanego dynamicznie obiektu. 

Implementuje RAII - destruktor inteligentnego wskaźnika usuwa wskazywany obiekt. Wskaźnik `unique_ptr` nie może być ani kopiowany ani przypisywany, może być jednakże przenoszony.

Przeniesienie prawa własności odbywa się zgodnie z *move semantics* w C++11 - wymaga dla referencji do lvalue jawnego transferu przy pomocy funkcji `std::move()`.

```{code-block} cpp
#include <memory>

void f()
{
    std::unique_ptr<Gadget> ptr_gadget(new Gadget{1, "ipod"});
    ptr_gadget->use();

    may_throw(); 

    std::unique_ptr<Gadget> other_ptr_gadget = std::move(ptr_gadget);
    other_ptr_gadget->use();
} // allocated Gadget is automatically deleted
```

Od C++14 dostępna jest funkcja `std::make_unique()` ułatwiająca tworzenie obiektów z wykorzystaniem `std::unique_ptr`.

```{code-block} cpp
auto ptr_gadget = std::make_unique<Gadget>(1, "ipod");
```

### Semantyka przenoszenia dla std::unique_ptr<T>

Obiekt ``std::unique_ptr`` nie może być kopiowany. Ale dzięki semantyce przenoszenia, może być stosowany tam, gdzie zgodne ze standardem C++03 niekopiowalne obiekty nie mogły działać:

* może być zwracany z funkcji

```{code-block} cpp
std::unique_ptr<Gadget> create_gadget()
{
    static int id_gen = 0;
    const int id = ++id_gen;

    auto ptr_gadget = std::make_unique<Gadget>(id, "Gadget#" + std::to_string(id));
    assert(ptr_gadget->is_valid());

    return ptr_gadget;
}

auto gadget = create_gadget();
if (gadget)
    gadget->use();
```

* może być przekazywany przez wartość (przenoszony) do funkcji jako parametr *sink*

```{code-block} cpp
void sink(unique_ptr<Gadget> gadget)
{
    if (gadget)
        gadget->use();
} // sink takes ownership - deletes the object pointed by gadget

auto gadget = create_gadget();

sink(move(gadget));      // explicitly moving into sink
sink(create_gadget());   // implicit move
```

* może być przechowywany w kontenerach STL (w standardzie C++11)

```{code-block} cpp
std::vector<std::unique_ptr<Gadget>> gadgets;

gadgets.push_back(std::make_unique<Gadget>(42, "smartwatch")); // implicit move to the container
gadgets.push_back(create_gadget()); // implicit move to the container

auto gadget = std::make_unique<Gadget>(665, "smartphone");
gadgets.push_back(std::move(gadget)); // explicit move to the container

for(const auto& g : gadgets)
    g->use();

gadgets.clear(); // elements are automatically destroyed
```

### Wskaźniki klas pochodnych

Wskaźnik do klasy pochodnej może zostać przypisany do wskaźnika do klasy bazowej (*upcasting*). Daje to możliwość stosowania polimorfizmu z wykorzystaniem funkcji wirtualnych.

```{code-block} cpp
Gadget* g = new SuperGadget();
g->do_something();
```

Odpowiednik z użyciem ``unique_ptr``

```{code-block} cpp
std::unique_ptr<Gadget> g = std::make_unique<SuperGadget>();
g->do_something();

//explicit conversion - hard to miss it
auto another_g = std::unique_ptr<Gadget>{ std::make_unique<SuperGadget>() };
```

### Dealokatory

Używając wskaźnika `std::unique_ptr` można zdefiniować własny dealokator, który będzie odpowiedzialny za prawidłowe zwolnienie zasobu. Umożliwia to
kontrolę nad zasobami innymi niż obiekty dynamicznie alokowane na stercie lub wymagającymi specjalnej obsługi w fazie destrukcji.

Aby użyć własnego dealokatora należy podać jego typ jako drugi parametr szablonu `std::unique_ptr<T, Dealloc>` oraz przekazać instancję dealokatora jako drugi parametr konstruktora:

```{code-block} cpp
void f() 
{
    std::unique_ptr<FILE, int(*)(FILE*)> file{fopen("test.txt"), &fclose};

    std::vector<char> buffer(1024);
    read_file_to_buffer(file.get(), vec.data(), vec.size());

    // rest of the code
} // fclose() is called for an opened file
```

```{important}
Dealokator dla `unique_ptr` wywoływany jest tylko, jeśli wewnętrzny wskaźnik jest rózny od `nullptr`!
```

### Idiom PIMPL

Wskaźnik `std::unique_ptr` świetnie nadaje się do stosowania tam, gdzie wcześniej stosowane były wskaźniki zwykłe albo obiekty typu `std::auto_ptr` (obecnie mający status *deprecated*), np. do implementacji idiomu PIMPL

#### PIMPL - Private Implementation:

* minimalizuje zależności na etapie kompilacji
* separuje interfejs od implementacji
* ukrywa implementację przed klientem


Plik ``bitmap.hpp``:
********************

```{code-block} cpp
class Bitmap
{
public:
    // rest of the interface...
    
    ~Bitmap(); // must be only declared
private:
    class Impl; // forward declaration

    std::unique_ptr<Impl> pimpl_; // pointer to implementation
};
```

Plik ``bitmap.cpp``:
******************

```{code-block} cpp
#include "bitmap.hpp"

class Bitmap::Impl
{
    std::vector<Pixel> pixels_;
    int width_;
    int height_;
};

Bitmap::Bitmap() : pimpl_ {std::make_unique<Impl>()}
{
    // implementation details...
}

Bitmap::~Bitmap() = default; // important! after defintion of Bitmap::Impl()
```

### Specjalizacja std::unique_ptr<T[]> dla tablic

Klasa ``std::unique_ptr<T[]>`` jest specjalizacją szablonu `std::unique_ptr` dla tablic obiektów typu `T`.

* Klasa ta stanowi lepszą alternatywę dla klasycznych tablic przydzielanych dynamicznie.
* Zgodnie z zasadą RAII, klasa `std::unique_ptr<T[]>` automatycznie zwalnia pamięć po tablicy przy wyjściu z zakresu.
* Oferuje przeciążony operator indeksowania (`operator[]`) umożliwiający stosowanie naturalnej składni odwołań do elementów tablicy.
* Destruktor wykorzystuje operator `delete []` aby automatycznie usunąć wskazywaną tablicę.

```{important}
Klasa `std::unique_ptr<T[]>` jest użyteczna w przypadku gdy wywołujemy kod *legacy* korzystający z tablic alokowanych dynamicznie.

Gdy chcemy użyć tablicy w nowym kodzie, zalecane jest użycie kontenerów STL, takich jak `std::vector<T>`.
```

```{code-block} cpp
namespace LegacyCode 
{
    int* create_buffer(size_t size)
    {
        return new int[size];
    }
}

void use_legacy_buffers()
{
    const size_t size = 1024;

    std::unique_ptr<int[]> buffer{create_buffer(size)};

    for(size_t i = 0; i < size; ++i)
        buffer[i] = i;

    may_throw();
    
    // rest of the code
} // buffer is automatically deleted
```

## std::shared_ptr

`std::shared_ptr<T>` jest szablonem nieingerencyjnego wskaźnika zliczającego odniesienia do wskazywanych obiektów.

Działanie:

* konstruktor tworzy licznik odniesień i inicjuje go wartością 1
* konstruktor kopiujący lub kopiujący operator przypisania inkrementują licznik odniesień
* destruktor zmniejsza licznik odniesień, jeżeli wartość spada do 0, to obiekt jest zwalniany poprzez wywołanie dealokatora (domyślnie jest nim operator ``delete``)

```{code-block} cpp
#include <memory>
#include <cassert>
#include <map>
#include <string>

using namespace std;

class Gadget { /* implementation */ };

std::map<std::string, std::shared_ptr<Gadget>> gadgets;

void using_gadgets()
{
    auto p1 = std::make_shared<Gadget>(1, "ipad");  
    assert(p1.use_count() == 1); // reference counter = 1

    {
        std::shared_ptr<Gadget> p2 = p1; // copy of shared_ptr
        assert(p1.use_count() == 2);     // reference counter = 2

        gadgets.insert(std::make_pair("ipad", p2)); // copy of shared_ptr to a std container
        assert(p1.use_count() == 3);                      // reference counter == 3

        p2->use();
    }  // destruction of p2 decrements reference counter
    
    assert(p1.use_count() == 2); // reference counter = 2
}  // destruction of p1 decrements reference counter = 1

int main()
{
    using_gadgets();
    assert(gadgets["ipad"].use_count() == 1); // reference counter = 1

    gadgets.clear(); // reference counter = 0 - gadget is removed
}
```

### Przydatne metody z interfejsu shared_ptr<T>

* `T* get() const`
  
  * zwraca przechowywany wskaźnik.

* `void reset()`
  
  * zwalnia prawo własności do zarządzanego obiektu.

* `template <typename Y*> void reset(Y* ptr)`
  
  * zamienia obiekt zarządzany na obiekt wskazywany przez `ptr`.

* `bool unique() const`
  
  * zwraca `true`, jeśli obiekt `std::shared_ptr` jest jedynym właścicielem przechowywanego wskaźnika.

* `long use_count() const`

  * zwraca wartość licznika odwołań do wskaźnika przechowywanego w obiekcie `std::shared_ptr`. Przydatna w diagnostyce.

* `void swap(shared_ptr<T>& other)`

  * wymienia wskaźniki między dwoma obiektami `std::shared_ptr`. Wymienia wskaźniki oraz liczniki odwołań.

* `explicit operator bool() const`

  * umożliwia konwersję do wartości logicznej, np. `if (p && p->is_valid())`

### Funkcja std::make_shared<T>()

Używanie `std::shared_ptr`` eliminuje konieczność jawnego stosowanie operatora ``delete`` 
Aby uniknąć także używania operatora ``new`` należy stosować funkcję pomocniczą ``make_shared()``, która pełni rolę fabryki wskaźników `std::shared_ptr``. Funkcja przekazuje (przez *perfect forwarding*) swoj parametry do konstruktora obiektu typu `T`, który jest alokowany na stercie.

```{code-block} cpp
struct Gadget
{
    Gadget(int id, string name) : id_{id}, name_{std::move(name)} {}
    // ...
    
private:
    int id_;
    std::string name_;
};

std::shared_ptr<Gadget> x = std::make_shared<Gadget>(1, "ipad");
```

* Stosowanie funkcji `std::make_shared<Gadget>(1, "ipad")` jest wydajniejsze niż konstrukcja `std::shared_ptr<Gadget>(new Gadget(1, "ipad"))`, ponieważ alokowany jest tylko jeden segment pamięci, w którym umieszczany jest wskazywany obiekt oraz blok kontrolny z licznikami odniesień.

* Stosowanie `std::make_shared()` jest również bezpieczniejsze, ponieważ eliminuje możliwość wycieku pamięci w przypadku wystąpienia wyjątku podczas alokacji pamięci dla obiektu.

```{code-block} cpp
void f(shared_ptr<Gadget>, int);
int may_throw();

void bad()
{
    f(shared_ptr<Gadget>{new Gadget(2, "wearable")}, may_throw()); // potential memory leak
}

void ok()
{
    f(std::make_shared<Gadget>(2, "wearable"), may_throw());
}
```

### Dealokatory

Niekiedy pojawia się potrzeba zastosowania wskaźnika `std::shared_ptr` do kontroli zasobu takiego typu, że jego zwolnienie nie sprowadza się do prostego wywołania operatora ``delete``.
Takie przypadki `std::shared_ptr` obsługuje przy pomocy dealokatorów użytkownika.

Dealokator jest:

* obiektem funkcyjnym odpowiedzialnym za zwolnienie zasobu, kiedy liczba referencji spadnie do zera.
* przekazywany jako drugi argument konstruktora `std::shared_ptr`.

```{code-block} cpp
class FileCloser
{
public:
    void operator()(FILE* file)
    {
        if (file)
            fclose(file);
    }
};

void use_device()
{
    std::shared_ptr<FILE> safe_file(fopen("data.txt", "+rw"), FileCloser{});

    may_throw(); // może wylecieć wyjątek

    //... rest of the code
} // FileCloser::operator() is called for the file - file is closed
```

### Rzutowania między wskaźnikami std::shared_ptr<T>

Problemy związane z rzutowaniami między wskaźnikami `std::shared_ptr` rozwiązują trzy funkcje szablonowe:

* `template<class T, class U> shared_ptr<T> static_pointer_cast(shared_ptr<U> const & r)`
  
  * Jeśli `r` jest pusty, zwraca pusty `shared_ptr<T>`
  * W innym przypadku `shared_ptr<T>` przechowuje kopię `static_cast<T*>(r.get())` i współdzieli prawo własności z `r`

* `template<class T, class U> shared_ptr<T> const_pointer_cast(shared_ptr<U> const &r)`
  
  * Jeśli `r` jest pusty, zwraca pusty `shared_ptr<T>`
  * W innym przypadku `shared_ptr<T>` przechowuje kopię `const_cast<T*>(r.get())` i współdzieli prawo własności z `r`

* `template<class T, class U> shared_ptr<T> dynamic_pointer_cast(shared_ptr<U> const & r)`

  * Jeśli `dynamic_cast<T*>(r.get())` zwraca wartość różną od `nullptr`, zwracany jest obiekt `shared_ptr<T>`, który współdzieli prawo własności do `r`
  * W przeciwnym wypadku zwracany jest pusty `shared_ptr<T>`

```{code-block} cpp
class Base 
{  
public:
    virtual ~Base() = default;
    virtual void foo() {}
};

class Derived : public Base {}
{
public:
    void foo() override {}
    void bar() {}
};

int main()
{
    auto p = std::make_shared<Derived>();
    std::shared_ptr<Base> pb = p; // downcasting

    auto pd = std::dynamic_pointer_cast<Derived>(pb);
    if (pd)
        pd->bar();
}
```

### Zależności cykliczne między wskaźnikami std::shared_ptr

Problemem dla wskaźników `shared_ptr` są zależności cykliczne, które powodują wycieki zasobów (pamięci).

```{code-block} cpp
struct Cyclic
{
    std::shared_ptr<Cyclic> self;
    Cyclic() = default;
};

void memory_leak()
{
    std::shared_ptr<Cyclic> ptr = make_shared<Cyclic>();
    ptr->self = ptr;
}  
```

```{important}
Cykle muszą być przerywane za pomocą wskaźników `std::weak_ptr`
```

### Tworzenie wskaźnika std::shared_ptr ze wskaźnika this

Niekiedy istnieje konieczność utworzenia inteligentnego wskaźnika `shared_ptr` ze wskaźnika `this`.
Oznacza to, że obiekt danej klasy będzie zarządzany za pośrednictwem inteligentnego wskaźnika.
W ogólności rozwiązaniem problemu konwersji `this` do `shared_ptr` jest użycie wskaźnika typu `std::weak_ptr`, który jest obserwatorem wskaźników `shared_ptr` (pozwala na obserwowanie wskazywanego obiektu bez ingerowania w wartość licznika odwołań).
Zdefiniowanie w klasie składowej typu ``weak_ptr`` i zainicjowanie jej wartością `this` umożliwia późniejsze pozyskiwanie wskaźników `shared_ptr`.
Aby nie trzeba było za każdym razem pisać tego samego kodu, można wykorzystać dziedziczenie po pomocniczej klasie `std::enable_shared_from_this`.

Klasa `enable_shared_from_this<T>` umożliwia obiektowi typu `T`, który jest zarządzany przez instancję `std::shared_ptr<T>` bezpieczne tworzenie dodatkowych wskaźników typu `std::shared_ptr<T>`, które współdzielą prawo własności do zarządzanego obiektu.

Przykład nieprawidłowego generowania instancji `shared_ptr` ze wskaźnika `this`:
********************************************************************************

```{code-block} cpp
#include <memory>

void do_stuff(std::shared_ptr<A> p)
{
    // using p
}

class A
{
public:
    void call_do_stuff()
    {
        do_stuff(std::shared_ptr<A>(this));
    }
};

int main()
{
    auto p = std::make_shared<A>();
    p->call_do_stuff();
}
```

Prawidłowe tworzenie instancji `std::shared_ptr`` ze wskaźnika `this:

```{code-block} cpp
#include <memory>

void do_stuff(std::shared_ptr<A> p)
{
    // using p
}

// ...

class A : public std::enable_shared_from_this<A>
{
public:
    void call_do_stuff()
    {
        do_stuff(shared_from_this());
    }
};

int main()
{
    auto p = std::make_shared<A>();
    p->call_do_stuff();
}
```

## std::weak_ptr

Najbardziej znanym problemem, związanym ze wskaźnikami opartymi na zliczaniu odniesień, są odniesienia cykliczne.
Występują one w sytuacji, gdy kilka obiektów trzyma wskaźniki do siebie nawzajem, przez co licznik odniesień nie spada do zera i wskazywane obiekty nie są nigdy kasowane.
Rozwiązaniem tego problemu jest zastosowanie słabych wskaźników – obiektów `std::weak_ptr`.

Wskaźnik typu `std::weak_ptr` obserwuje wskazywany obiekt, ale nie ma kontroli nad czasem jego życia i nie może zmieniać jego licznika odniesień.

Nie udostępnia operatorów dereferencji. Aby mieć dostęp do wskazywanego obiektu konieczne jest dokonanie konwersji do `std::shared_ptr`.

Gdy wskazywany obiekt został już skasowany konwersja na `std::shared_ptr` daje w wynik:

* wskaźnik pusty - w przypadku metody `lock()`
  
  ```{code-block} cpp
  shared_ptr<Gadget> ptr_g = make_shared<Gadget>(1, "ipad");
  
  std::weak_ptr<Gadget> weak_g = ptr_g; 
  assert(ptr_g.use_count() == 1); // reference counter = 1
  
  ptr_g.reset();  // reference counter = 0, gadget is deleted
  
  std::shared_ptr<Gadget> temp_g = weak_g.lock();
  
  if (temp_g)
  {
      temp_g->use();
  }
  ```

* zgłasza wyjątek `std::bad_weak_ptr`  - w przypadku konstruktora `std::shared_ptr<T>(const std::weak_ptr<T>&)`

  ```{code-block} cpp
  std::shared_ptr<Gadget> ptr_g = make_shared<Gadget>(1, "ipad");
  std::weak_ptr<Gadget> weak_g;
  
  weak_g_ = ptr_g; // reference counter = 1
  ptr_g.reset();   // gadget is destroyed
  
  try
  {
      std::shared_ptr<Gadget> temp_g(weak_g); // throws std::bad_weak_ptr
  }
  catch(const std::bad_weak_ptr& e)
  {
      std::cout << e.what() << std::endl;
  }
  ```

### Przechowywanie std::weak_ptr w kontenerach asocjacyjnych

Aby przechować wskaźniki `std::weak_ptr` w kontenerach asocjacyjnych należy klasy `std::owner_less` jako parameteru definiującego sposób porównania
wskaźników.

Klasa `std::owner_less` dostarcza implementację umożliwiającą porównanie wskaźników `std::shared_ptr` oraz `std::weak_ptr` na podstawie prawa własności, a nie wartości
przechowywanych wskaźników. Dwa wskaźniki są uznane za równoważne tylko wtedy, gdy oba są puste lub oba zarządzają tym samym obiektem (współdzielą blok kontrolny).

```{code-block} cpp
struct Key { /* implementation */ };
struct Value { /* implementation */ };

std::map<std::shared_ptr<Key>, Value, std::owner_less<std::shared_ptr<Key>>> my_map;

std::set<std::weak_ptr<Key>, std::owner_less<std::weak_ptr<Key>>> my_set;
```

### Zastosowanie std::weak_ptr

Wskaźniki `std::weak_ptr` stosuje się do:

* zrywania cyklicznych zależności dla `shared_ptr`
* współużytkowania zasobu bez przejmowania odpowiedzialności za zarządzanie nim
* eliminowania ryzykownych operacji na wiszących wskaźnikach