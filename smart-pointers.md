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
    close_port(port);  // leaks if send() throws
    delete data;          // leaks if send() throws
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
    may_throw();   // it may throw
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