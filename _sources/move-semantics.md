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