---
layout: cover
background: /img/section-title-bg.svg
---

# Semantyka przenoszenia

---
layout: default
---

## Motywacje dla semantyki przenoszenia

- Optymalizacja wydajności:
  - możliwość rozpoznania kiedy mamy do czynienia z obiektem tymczasowym (*temporary object*)
  - możliwość wskazania, że obiekt nie będzie dalej używany - jego czas życia wygasa (*expiring object*)
- Możliwość implementacji obiektów, które nie mogą być kopiowane, ale umożliwiają transfer prawa własności do zasobu:
  - `auto_ptr<T>` w C++98 symulował semantykę przenoszenia za pomocą konstruktora kopiującego i operatora przypisania
  - obiekty kontrolujące zasoby systemowe, które nie mogą być łatwo kopiowane - wątki, pliki, strumienie, itp.

---

## Problem z kopiami obiektów

--- 
class: white-slide
layout: center
---

<img src="/img/copy-from-function-00.jpg" class="img-lg center"/>

--- 
class: white-slide
layout: center
---

<img src="/img/copy-from-function-01.jpg" class="img-lg center"/>

--- 
class: white-slide
layout: center
---

<img src="/img/copy-from-function-02.jpg" class="img-lg center"/>

--- 
class: white-slide
layout: center
---

<img src="/img/copy-from-function-03.jpg" class="img-lg center"/>

--- 
class: white-slide
layout: center
---

<img src="/img/copy-from-function-04.jpg" class="img-lg center"/>

--- 
class: white-slide
layout: center
---

<img src="/img/copy-from-function-05.jpg" class="img-lg center"/>

---

## Czy można to zrobić wydajniej?

--- 
class: white-slide
layout: center
---

<img src="/img/move-from-function-00.png" class="img-lg center"/>

--- 
class: white-slide
layout: center
---

<img src="/img/move-from-function-01.png" class="img-lg center"/>

--- 
class: white-slide
layout: center
---

<img src="/img/move-from-function-02.png" class="img-lg center"/>

--- 
class: white-slide
layout: center
---

<img src="/img/move-from-function-03.png" class="img-lg center"/>

--- 
class: white-slide
layout: center
---

<img src="/img/move-from-function-solution-1.png" class="img-lg center"/>

--- 
class: white-slide
layout: center
---

<img src="/img/move-from-function-solution-2.png" class="img-lg center"/>

---

## Problemy z kopiami obiektów w C++98 - przykład

```cpp
std::vector<std::string> create_and_fill()
{
    std::vector<std::string> vec;

    std::string str = "text";

    vec.push_back(str);

    vec.push_back(str + str); 

    vec.push_back("text"); 

    vec.push_back(str);  

    return vec;
}
```

---

## Ile kopii powstanie?

```cpp{all|7|9|11|13|15}{at:1}
std::vector<std::string> create_and_fill()
{
    std::vector<std::string> vec;

    std::string str = "text";

    vec.push_back(str);

    vec.push_back(str + str); 

    vec.push_back("text"); 

    vec.push_back(str);  

    return vec;
}
```

<v-switch>
  <template #1>
    Wstawienie kopii `str` do kontenera
  </template>
  <template #2>
    Wstawienie kopii tymczasowej warości
  </template>
  <template #3>
    Wstawienie kopii tymczasowej wartości
  </template>
  <template #4>
    Wstawienie kopii `str` do kontenera
  </template>
  <template #5>
    Zwrócenie kontenera - potencjalna kopia dużego obiektu
  </template>
</v-switch>

---

## Jak zoptymalizować ten kod?

```cpp{all|7|9|11|13|15}{at:1}
std::vector<std::string> create_and_fill()
{
    std::vector<std::string> vec;

    std::string str = "text";

    vec.push_back(str);

    vec.push_back(str + str); 

    vec.push_back("text"); 

    vec.push_back(str);  

    return vec;
}
```
<v-switch>
  <template #1>
    ???
  </template>
  <template #2>
    ???
  </template>
  <template #3>
    ???
  </template>
  <template #4>
    ???
  </template>
  <template #5>
    ???
  </template>
</v-switch>

---
layout: default
---

## lvalue vs. rvalue

Aby umożliwić implementację semantyki przenoszenia C++11 wykorzystuje podział obiektów na:

<v-clicks>

- **lvalue**
  - obiekt posiada nazwę
  - można pobrać adres obiektu
- **rvalue**
  - nie można pobrać adresu
  - zwykle nienazwany obiekt tymczasowy (np. obiekt zwrócony z funkcji)
  - z obiektów rvalue możemy transferować stan pozostawiając je w poprawnym, ale nieokreślonym stanie

</v-clicks>

---
layout: default
---

## lvalue vs. rvalue

```cpp {all|1|3|7|9|10|11}{at:1}
double pi = 3.14;

double& ref_pi = pi;

std::string foo(std::string str);

auto bar = foo("Hello"); 

std::vector<int> vec;
vec.push_back(42);
vec[0] = int{665}; 
```

<v-switch>
  <template #1>
    <code>pi</code> jest lvalue, <code>3.14</code> jest rvalue
  </template>
  <template #2>
    <code>ref_pi</code> jest lvalue, <code>pi</code> jest lvalue
  </template>
  <template #3>
    <code>foo("Hello")</code> zwraca rvalue, <code>bar</code> jest lvalue
  </template>
  <template #4>
    <code>vec</code> jest lvalue
  </template>
  <template #5>
    <code>42</code> jest rvalue - przekazane jako argument funkcji
  </template>
  <template #6>
    <code>vec[0]</code> jest lvalue (<code>int& operator[](size_t)</code>), <code>int{665}</code> jest rvalue
  </template>
</v-switch>

---

## lvalue vs. rvalue

<v-clicks>

* Operacja przenoszenia, która wiąże się ze zmianą stanu jest niebezpieczna dla obiektów lvalue, ponieważ obiekt może zostać użyty po wykonaniu takiej operacji.
* **Operacje przenoszenia są bezpieczne dla obiektów rvalue!!!**

</v-clicks>

---

## Referencje rvalue

* C++11 wprowadza referencje do rvalue - **rvalue references**
  * składnia: ``T&&``
  * muszą zostać zainicjowane i nie mogą zmienić odniesienia
  * służą do identyfikacji operacji, które implementują przenoszenie
  * klasyczne referencje ``T&`` od C++11 są *lvalue referencjami*

---

## Reguły wiązania referencji

* Wprowadzenie *referencji rvalue* rozszerza reguły wiązania referencji.

---

## Reguły wiązania referencji - C++98


* **lvalue** mogą być wiązane do **lvalue referencji**

```cpp
int x = 10;
int& ref_x = x;
```

* **rvalue** mogą być wiązane do **const lvalue referencji**

```cpp
std::string get_name();
const std::string& name = get_name();
```

---

## Reguły wiązania referencji - C++11

* **rvalue** mogą być wiązane do **rvalue referencji**

```cpp
std::string&& name = get_name();
```

* **lvalue** nie mogą być wiązane do **rvalue referencji**

```cpp
int&& rvref_x = x; // ERROR
```

---

## Wykorzystanie referencji rvalue

* Używając **rvalue references** możemy przeciążyć funkcje w celu optymalnej obsługi zarówno lvalue jak i rvalue: 

```cpp
template <typename T>
class vector
{
public:
    void push_back(const T& item); // wstawia kopię obiektu
    void push_back(T&& item);      // wstawia z przenoszeniem
};

vector<std::string> vec;
std::string str = "text";

vec.push_back(str);       // wywołuje push_back(const T&)
vec.push_back("text");    // wywołuje push_back(T&&)
vec.push_back(str + str); // wywołuje push_back(T&&)
```

---

```cpp {all|7|9|11|13}{at:1}
std::vector<std::string> create_and_fill()
{
    std::vector<std::string> vec;

    std::string str = "text";

    vec.push_back(str);

    vec.push_back(str + str); 

    vec.push_back("text"); 

    vec.push_back(str);  

    return vec;
}
```

<v-switch>
  <template #1>
    <code>str</code> to lvalue <code>-> vec.push_back(const string&)</code>
  </template>
  <template #2>
    <code>str + str</code> to rvalue <code>-> vec.push_back(T&&)</code>
  </template>
  <template #3>
    <code>"text"</code> jest konwertowane do rvalue <code>-> vec.push_back(T&&)</code>
  </template>
  <template #4>
    <code>str</code> to lvalue <code>-> vec.push_back(const string&)</code><br/>
    Tutuaj chcielbyśmy dokonać transferu stanu, ponieważ <code>str</code> nie jest już więcej używany , ale <code>str</code> jest lvalue.
  </template>
</v-switch>

---

## std::move

* Jeśli chcemy dokonać transferu stanu z obiektu lvalue, musimy go **jawnie skonwertować do rvalue** za pomocą funkcji `std::move`
* `std::move()` produkuje wyrażenie **xvalue** (*expiring value* - xvalue jest rvalue) umożliwiając efektywny transfer stanu obiektu do obiektu docelowego

```cpp
std::vector<int> vec = {1, 2, 3, 4, 5};

std::vector<int> backup_vec = vec;

std::vector<int> target_vec = std::move(vec);
```

---

```cpp {all|13}{at:1}
std::vector<std::string> create_and_fill()
{
    std::vector<std::string> vec;

    std::string str = "text";

    vec.push_back(str);

    vec.push_back(str + str); 

    vec.push_back("text"); 

    vec.push_back(std::move(str));  

    return vec;
}
```
<v-switch>
  <template #1>
    Teraz <code>str</code> jest przekazywany jako rvalue, więc wywołany zostanie <code>vec.push_back(T&&)</code>, który może dokonać transferu stanu z <code>str</code> do elementu wewnątrz kontenera.
  </template>
</v-switch>

---

## std::swap w C++98

```cpp
template <typename T>
void swap(T& a, T& b)
{
    T temp = a; // copy a to temp
    a = b;      // copy b to a
    b = temp;   // copy temp to b
}
```

* Ta implementacja jest poprawna, ale może być nieefektywna dla typów, które są kosztowne do kopiowania (np. duże kontenery, obiekty zarządzające zasobami systemowymi, itp.)

---

## std::swap w C++11

```cpp
template <typename T>
void swap(T& a, T& b)
{
    T temp = std::move(a); // tries to move a to temp
    a = std::move(b);      // tries to move b to a
    b = std::move(temp);   // tries to move temp to b
}
```

---
layout: center
---

<div class="text-xl text-center">

Transfer stanu z obiektu rvalue do obiektu docelowego jest możliwy tylko wtedy, gdy
klasa obiektu implementuje **semantykę przenoszenia**

</div>

---

## Semantyka przenoszenia w klasach

* Aby klasa obsługiwała semantykę przenoszenia, musi implementować:
  * konstruktor przenoszący
  * przenoszący operator przypisania
  
---

## Funkcje specjalne w C++11

*  C++11 istnieje sześć specjalnych funkcji składowych klasy:

```cpp {all|4|5|6|7|8|9}{at:1}
class X
{
public:
    X();
    ~X();
    X(const X&);
    X& operator=(const X&);
    X(X&&);
    X& operator=(X&&);
};
```

<v-switch>
  <template #1>
    Konstruktor domyślny
  </template>
  <template #2>
    Destruktor
  </template>
  <template #3>
    Konstruktor kopiujący
  </template>
  <template #4>
    Operator przypisania kopiującego
  </template>
  <template #5>
    Konstruktor przenoszący
  </template>
  <template #6>
    Operator przypisania przenoszącego
  </template>
</v-switch>

---
class: white-slide
---

<div class="text-center">

Funkcje specjalne w C++11

</div>

<img src="/img/move-semantics-rules-1.svg " class="img-lg center"/>


---
class: white-slide
---

<div class="text-center">

Funkcje specjalne w C++11

</div>

<img src="/img/move-semantics-rules-2.svg " class="img-lg center"/>

---
class: white-slide
---

<div class="text-center">

Funkcje specjalne w C++11

</div>

<img src="/img/move-semantics-rules-3.svg " class="img-lg center"/>

---
class: white-slide
---

<div class="text-center">

Funkcje specjalne w C++11

</div>

<img src="/img/move-semantics-rules-4.svg " class="img-lg center"/>

<div class="bottom-center">

Deklaracja destruktora powoduje wyłączenie domyślnej implementacji semantyki przenoszenia

</div>

---
class: white-slide
---

<div class="text-center">

Funkcje specjalne w C++11

</div>

<img src="/img/move-semantics-rules-5.svg " class="img-lg center"/>

<div class="bottom-center">

Deklaracja operacji kopiowania powoduje wyłączenie domyślnej implementacji semantyki przenoszenia

</div>

---
class: white-slide
---

<div class="text-center">

Funkcje specjalne w C++11

</div>

<img src="/img/move-semantics-rules-6.svg " class="img-lg center"/>

<div class="bottom-center">

Deklaracja operacji kopiowania powoduje wyłączenie domyślnej implementacji semantyki przenoszenia

</div>

---
class: white-slide
---

<div class="text-center">

Funkcje specjalne w C++11

</div>

<img src="/img/move-semantics-rules-7.svg " class="img-lg center"/>

<div class="bottom-center">

Deklaracja operacji przenoszenia powoduje usunięcie semantyki kopiowania

</div>

---
class: white-slide
---

<div class="text-center">

Funkcje specjalne w C++11

</div>

<img src="/img/move-semantics-rules-8.svg " class="img-lg center"/>

<div class="bottom-center">

Deklaracja operacji przenoszenia powoduje usunięcie semantyki kopiowania

</div>

---
class: white-slide
---

<div class="text-center">

Funkcje specjalne w C++98

</div>

<img src="/img/special-functions-c++98.svg " class="img-lg center"/>

---
class: white-slide
---

<div class="text-center">

Funkcje specjalne w C++11

</div>

<img src="/img/move-semantics-rules-8.svg " class="img-lg center"/>

---
class: white-slide
---

<div class="text-center">

Funkcje specjalne w C++11 (kod)

</div>

<img src="/img/special-functions-code-0.svg " class="img-lg center"/>

---
class: white-slide
---

<div class="text-center">

Funkcje specjalne w C++11 (kod)

</div>

<img src="/img/special-functions-code-1.svg " class="img-lg center"/>

---
class: white-slide
---

<div class="text-center">

Funkcje specjalne w C++11 (kod)

</div>

<img src="/img/special-functions-code-2.svg " class="img-lg center"/>

---
class: white-slide
---

<div class="text-center">

Funkcje specjalne w C++11 (kod)

</div>

<img src="/img/special-functions-code-2.svg " class="img-lg center"/>

---
class: white-slide
---

<div class="text-center">

Funkcje specjalne w C++11 (kod)

</div>

<img src="/img/special-functions-code-3.svg " class="img-lg center"/>

---
class: white-slide
---

<div class="text-center">

Funkcje specjalne w C++11 (kod)

</div>

<img src="/img/special-functions-code-4.svg " class="img-lg center"/>

---
class: white-slide
---

<div class="text-center">

Funkcje specjalne w C++11 (kod)

</div>

<img src="/img/special-functions-code-5.svg " class="img-lg center"/>

---
class: white-slide
---

<div class="text-center">

Funkcje specjalne w C++11 (kod)

</div>

<img src="/img/special-functions-code-6.svg " class="img-lg center"/>