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

Operacja przenoszenia, która wiąże się ze zmianą stanu, jest niebezpieczna dla obiektów l-value, ponieważ obiekt może zostać użyty po wykonaniu
takiej operacji.

Operacje przenoszenia są bezpieczne dla obiektów rvalue.