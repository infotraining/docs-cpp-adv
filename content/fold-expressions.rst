Fold expressions - C++17
========================

Wyrażenia fold w językach funkcjonalnych
----------------------------------------

Koncept *redukcji* jest jednym z podstawowych pojęć w językach
funkcjonalnych.

**Fold** w językach funkcjonalnych to rodzina funkcji wyższego rzędu
zwana również **reduce**, **accumulate**, **compress** lub **inject**.
Funkcje **fold** przetwarzają rekursywnie uporządkowane kolekcje danych
(listy) w celu zbudowania końcowego wyniku przy pomocy funkcji
(operatora) łączącej elementy.

Dwie najbardziej popularne funkcje z tej rodziny to `fold` (*fold left*) i
`foldr` (*fold right*).

Przykład:

Redukcja listy `[1, 2, 3, 4, 5]` z użyciem operatora `(+)`:

-  użycie funkcji ``fold`` - redukcja od lewej do prawej

--------------

.. code:: haskell

    fold (+) 0 [1..5]

.. code:: console

    (((((0 + 1) + 2) + 3) + 4) + 5)

-  użycie funkcji ``foldr`` - redukcja od prawej do lewej

--------------

.. code:: haskell

    foldr (+) 0 [1..5]

.. code:: console

    (1 + (2 + (3 + (4 + (5 + 0)))))

Redukcja w C++98 - std::accumulate
----------------------------------

W C++ redukcja jest obecna poprzez implementację algorytmu
``std::accumulate``.

.. code:: c++

    #include <vector>
    #include <numeric>
    #include <string>
    
    using namespace std::string_literals;
    
    std::vector<int> vec = {1, 2, 3, 4, 5};
    
    std::accumulate(std::begin(vec), std::end(vec), 
                    "0"s, // initial value
                    [](const std::string& reduction, int item) { 
                        return "("s + reduction +  " + "s + std::to_string(item) + ")"s;
                    }
    );


.. parsed-literal::

    (((((0 + 1) + 2) + 3) + 4) + 5)


Fold expressions w C++17
------------------------

Wyrażenia typu *fold* umożliwiają uproszczenie rekurencyjnych
implementacji dla zmiennej liczby argumentów szablonu.

Przykład z wariadyczną funkcją ``sum(1, 2, 3, 4, 5)`` z wykorzystaniem
*fold expressions* może być w C++17 zaimplementowany następująco:

.. code:: c++

    template <typename... Args>
    auto sum(Args&&... args)
    {
        return (... + args);
    }


.. code:: c++

    sum(1, 2, 3, 4, 5);


.. parsed-literal::

    (int) 15


Składnia wyrażeń fold
---------------------

Niech :math:`e = e_1, e_2, \dotso, e_n` będzie wyrażeniem, które
zawiera nierozpakowany *parameter pack* i :math:`\otimes` jest
*operatorem fold*, wówczas **wyrażenie fold** ma postać:

-  Unary **left fold**

   :math:`(\dotso\; \otimes\; e)`

który jest rozwijany do postaci
:math:`(((e_1 \otimes e_2) \dotso ) \otimes e_n)`

-  Unary **right fold**

   :math:`(e\; \otimes\; \dotso)`

który jest rozwijany do postaci
:math:`(e_1 \otimes ( \dotso (e_{n-1} \otimes e_n)))`

Jeśli dodamy argument nie będący paczką parametrów do operatora ``...``,
dostaniemy dwuargumentową wersję **wyrażenia fold**. W zależności od
tego po której stronie operatora ``...`` dodamy dodatkowy argument
otrzymamy:

-  Binary **left fold**

   :math:`(a \otimes\; \dotso\; \otimes\; e)`

który jest rozwijany do postaci
:math:`(((a \otimes e_1) \dotso ) \otimes e_n)`

-  Binary **right fold**

   :math:`(e\; \otimes\; \dotso\; \otimes\; a)`

który jest rozwijany do postaci
:math:`(e_1 \otimes ( \dotso (e_n \otimes a)))`

Operatorem :math:`\otimes` może być jeden z poniższych operatorów C++:

.. code:: cpp

    +  -  *  /  %  ^  &  |  ~  =  <  >  <<  >>
    +=  -=  *=  /=  %=  ^=  &=  |=  <<=  >>=
    ==  !=  <=  >=  &&  ||  ,  .*  ->*

Elementy identycznościowe
-------------------------

Operacja fold dla pustej paczki parametrów (*parameter pack*) jest
ewaluowana do określonej wartości zależnej od rodzaju zastosowanego
operatora. Zbiór operatorów i ich rozwinięć dla pustej listy parametrów
prezentuje tabela:

+--------------------+--------------------------------------------------+
| Operator           | Wartość zwracana jako element identycznościowy   |
+====================+==================================================+
| ``&&``             | ``true``                                         |
+--------------------+--------------------------------------------------+
| \|\|               | ``false``                                        |
+--------------------+--------------------------------------------------+
| ``,``              | ``void()``                                       |
+--------------------+--------------------------------------------------+

Jeśli operacja fold jest ewaluowana dla pustej paczki parametrów dla
innego operatora, program jest nieprawidłowo skonstruowany
(*ill-formed*).

Przykłady wyrażeń fold
----------------------

Wariadyczna funkcja przyjmująca dowolną liczbę argumentów
konwertowalnych do wartości logicznych i zwracająca ich iloczyn logiczny
(``operator &&``):

.. code:: c++

    template <typename... TArgs>
    bool all_true(TArgs... args)
    {
        return (... && args);
    }



.. code:: c++

    bool result = all_true(true, true, false, true);

    assert(result == false);


--------------

Funkcja ``print()`` wypisująca przekazane argumenty. Implementacja
wykorzystuje wyrażenie *binary left fold* dla operatora ``<<``:

.. code:: c++

    #include <iostream>
    
    template <typename... TArgs>
    void print(const TArgs&... args)
    {
        (std::cout << ... << args) << "\n";
    }

.. code:: c++

    print(1, 2, 3, 4);


.. parsed-literal::

    1234


--------------

Funkcja ``sum`` zwracająca sumę argumentów przekazanych do funkcji:

.. code:: c++

    template <typename... TArgs>
    constexpr auto sum(const TArgs&... args)
    {
        return (... + std::forward<TArgs>(args));
    }

    static_assert(sum(1, 2, 3, 4, 5) == 15);


--------------

Implementacja wariadycznej wersji algorytmu ``foreach()`` z
wykorzystaniem funkcji `invoke()``:

.. code:: c++

    #include <iostream>
    
    template <typename F, typename... TArgs>
    auto invoke_for_all(F&& f, TArgs&&... args)
    {
        return (..., f(std::forward<TArgs>(args)));
    }

    struct Printer
    {
        int counter = 0;

        template <typename T>
        void operator()(T&& arg) { std::cout << arg; ++counter; }
    };

.. code:: c++

    #include <string>    
    using namespace std::literals;

    invoke_for_all(Printer{}, 42, " is the answer\n");

    Printer printer;
    invoke_for_all(printer, "Hello", " "s, "world", '!', '\n');
    assert(printer.counter == 5);

--------------

Implementacja wariadycznych wersji algorytmów ``count()`` oraz
``count_if()`` działających na listach typów:

.. code:: c++

    #include <type_traits>
    #include <iostream>
    
    // count the times a predicate Predicate is satisfied in a typelist Lst
    template <template<typename> class Predicate, typename... Lst>
    constexpr size_t count_if = ((Predicate<Lst>::value ? 1 : 0) + ...); 
        
    // count the occurences of a type V in a typelist L
    template <class V, class... Lst>
    constexpr size_t count = ((std::is_same<V, Lst>::value ? 1 : 0) + ...); 



.. code:: c++

    static_assert(count_if<std::is_integral, float, unsigned, int, double, long> == 3);
    static_assert(count_if<std::is_const, float, unsigned, int, double, long> == 0);
    static_assert(count_if<std::is_pointer, float, unsigned, int, double*, long> == 1);
    
    static_assert(count<float, unsigned, int, double, long, float> == 1);


