# Inteligentne wskaźniki - Smart Pointers

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
| ``std::shared_ptr<T>``     |     o       |     o                    |    wewnętrzny       |
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

### Semantyka przenoszenia dla `std::unique_ptr<T>`

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

### Specjalizacja ``std::unique_ptr<T[]>``

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