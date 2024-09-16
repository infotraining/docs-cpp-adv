# Semantyka przenoszenia (*Move semantics*)

## Motywacja dla semantyki przenoszenia

* Optymalizacja wydajności:
  * unikanie zbędnego kopiowania obiektów tymczasowych (*temporary objects*)
  * możliwość wskazania, że obiekt nie będzie dalej używany - jego czas życia
    wygasa (*expiring object*)

* Implementacja obiektów, które kontrolują zasoby systemowe (np. pamięć, uchwyty do plików, muteksy, itp.). Takie obiektu, nie powinny być kopiowane, ale powinny umożliwiać transfer prawa własności do zasobu:

  * `auto_ptr<T>` w C++98 symulował semantykę przenoszenia za pomocą konstruktora kopiującego i operatora przypisania

```cpp
void create_and_insert(vector<string>& coll)
{
    string str = "text";

    coll.push_back(str); // insert a copy of str
                         // str is used later
    
    coll.push_back(str + str); // insert a copy of temporary value
                               // unnecessary copy in C++98
    
    coll.push_back("text"); // insert a copy of temporary value
                            // unnecessary copy in C++98
    
    coll.push_back(str);  // insert a copy of str
                          // unnecessary copy in C++98
    
    // str is no longer used
}
```

## lvalues i rvalues

Aby umożliwić implementację semantyki przenoszenia C++11 wykorzystuje podział obiektów na:

* **lvalue**

  - obiekt posiada nazwę lub jest referencją `&`
  - można pobrać adres obiektu 

* **rvalue**

  - nie można pobrać adresu
  - zwykle nienazwany obiekt tymczasowy (np. obiekt zwrócony z funkcji przez wartość)
  - z obiektów rvalue możemy transferować stan pozostawiając je w poprawnym, ale nieokreślonym stanie

Przykłady:

```{code-block} cpp
double dx;
double* ptr; // dx and ptr are lvalues

std::string foo(std::string str); // foo and str are lvalues

// foo's return is rvalue
foo("Hello"); // temp string created for call is rvalue

std::vector<int> vec; // vec is lvalue

vi[5] = 0; // vi[5] is lvalue - returns int&
```

* Operacja przenoszenia, która wiąże się ze zmianą stanu, jest niebezpieczna dla obiektów lvalue, ponieważ obiekt może zostać użyty po wykonaniu
takiej operacji.

* Operacje przenoszenia są bezpieczne dla obiektów rvalue. Takie obiekty są tymczasowe i nie będą używane po wykonaniu operacji przenoszenia.

## Referencje rvalue - *rvalue references*

C++11 wprowadza referencje do rvalue - **rvalue references**, które zachowują się podobnie jak klasyczne referencje z C++98 (zwane od C++11 **lvalue references**).

* składnia: `T&&`
* muszą zostać zainicjowane i nie mogą zmienić odniesienia
* służą do identyfikacji operacji, które implementują przenoszenie

Wprowadzenie referencji do rvalue rozszerza reguły wiązania referencji:

* Tak jak w C++98:

  - **lvalues** mogą być wiązane do **lvalue references**
  - **rvalues** mogą być wiązanie do **const lvalue references**

* W C++11:

  - **rvalues** mogą być wiązane do **rvalue references**
  - **lvalues** nie mogą być wiązane do **rvalue references**

```{code-block} cpp
std::string full_name(const std::string& first, const std::string& last)
{
    return first + " " + last;
}

std::string fname = "John"; // fname is lvalue

std::string& ref_fname = fname; // lvalue reference is bound to an lvalue
const std::string& cref_full_name = full_name(fname, "Doe"); // const lvalue reference is bound to an rvalue

std::string&& rref_full_name = full_name("John", "Doe"); // rvalue reference is bound to an rvalue
std::string&& rref_fname = fname; // ERROR - rvalue reference cannot be bound to an lvalue
```

```{important}
Teoretycznie możliwe jest tworzenie referencji typu ``const T&&``. Są one poprawne składniowo, ale nie mają sensu.
```

## Implementacja semantyki przenoszenia

### Przeciążanie funkcji za pomocą referencji rvalue

Przy pomocy lvalue referencji i  rvalue referencji możemy przeciążać funkcje. W ten sposób możemy zaimplementować funkcje, które przyjmują obiekty tymczasowe (rvalue) i obiekty, które będą dalej używane (lvalue).

```{code-block} cpp
template <typename T>
class vector
{
public:
    void push_back(const T& item);  // accepts lvalue - inserts a copy of item into a vector

    void push_back(T&& item);       // accepts rvalue - moves item into container
};

// ...

void create_and_insert(vector<string>& coll)
{
    string str = "text";

    coll.push_back(str); // coll.push_back(const string&) - inserts a copy of str (lvalue)
                         // str is used later

    coll.push_back(str + str); // coll.push_back(string&&) - moves temporary object into container

    coll.push_back("text");    //coll.push_back(string&&) - moves temporary object into container

    coll.push_back(std::move(str));  // coll.push_back(string&&) - moves expiring str into container
                                     // str is no longer used
}
```

### Implementacja semantyki przenoszenia w klasach

Aby zaimplementować semantykę przenoszenia dla klasy należy zapewnić jej:

* konstruktor przenoszący - przyjmujący jako argument **rvalue reference**
* przenoszący operator przypisania - przyjmujący jako argument **rvalue reference**

```{important}
Konstruktor przenoszący i przenoszący operator przypisania są nowymi specjalnymi funkcjami składowymi klas w C++11.
```

#### Funkcje specjalne klas w C++11

Od C++11 istnieje sześć specjalnych funkcji składowych klasy:

* konstruktor domyślny - ``X();``
* destruktor - ``~X();``
* konstruktor kopiujący - ``X(const X&);``
* kopiujący operator przypisania - ``X& operator=(const X&);``
* konstruktor przenoszący - ``X(X&&);``
* przenoszący operator przypisania - ``X& operator=(X&&);``

```{important}
Klasa wspiera semantykę przenoszenia, jeśli posiada konstruktor przenoszący i przenoszący operator przypisania.
```

### Implementacja konstruktora przenoszącego

Implementacja konstruktora przenoszącego powinna przenieść zasoby obiektu tymczasowego do obiektu docelowego i pozostawić obiekt tymczasowy w poprawnym, ale nieokreślonym stanie.

Implementując przenoszący operator przypisania należy najpierw zwolnić zasoby obiektu docelowego, a następnie przenieść zasoby obiektu źródłowego do obiektu docelowego.

```{code-block} cpp
class DataSet
{
private:
    std::string name_;
    int* data_;
    size_t size_;
public: 
    DataSet(std::string name, std::initializer_list<int> data)
        : name_(name), data_(new int[data.size()]), size_(data.size())
    {
        std::copy(data.begin(), data.end(), data_);
    }

    ~DataSet()
    {
        delete[] data_;
    }

    // move constructor
    DataSet(DataSet&& source)
        : name_(std::move(source.name_)) // move string using its move constructor
        , data_(source.data_) // copy pointer from source.data_ to data_
        , size_(source.size_) // copy size from source.size_ to size_
    {
        source.data_ = nullptr; // set source.data_ to nullptr - no longer owns the resource
        source.size_ = 0; 
    }

    // move assignment operator
    DataSet& operator=(DataSet&& source)
    {
        if (this != &source)
        {
            delete[] data_; // release the resource

            name_ = std::move(source.name_); // move string using its move constructor
            data_ = source.data_; // copy pointer from source.data_ to data_
            size_ = source.size_; // copy size from source.size_ to size_

            source.data_ = nullptr; // set source.data_ to nullptr - no longer owns the resource
            source.size_ = 0;
        }
        return *this;
    }

    //... rest of the class
};
```

#### Implementacja z std::exchange()

Przy samodzielnej implementacji konstruktora przenoszącego i przenoszącego operatora przypisania można skorzystać z funkcji ``std::exchange()`` z biblioteki standardowej C++.

```{code-block} cpp
#include <utility>

class DataSet
{
    DataSet(DataSet&& source)
        : name_(std::std::move(source.name_)); 
        , data_(std::exchange(source.data_, nullptr)) // move pointer from source.data_ to data_  and set source.data_ to nullptr
        , size_(std::exchange(source.size_, 0)) // move size from source.size_ to size_ and set source.size_ to 0
    {
    }

    DataSet& operator=(DataSet&& source)
    {
        if (this != &source)
        {
            delete[] data_; // release the resource

            name_ = std::move(source.name_); // move string using its move constructor
            data_ = std::exchange(source.data_, nullptr); // move pointer from source.data_ to data_  and set source.data_ to nullptr
            size_ = std::exchange(source.size_, 0); // move size from source.size_ to size_ and set source.size_ to 0
        }
        return *this;
    }
};
```