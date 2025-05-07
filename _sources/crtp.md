# CRTP

**Curiously Recurring Template Pattern (CRTP)** to idiom, który polega na utworzeniu klasy bazowej, która jest szablonem, a następnie dziedziczenie po niej w klasie pochodnej, podając jako argument szablonu klasę pochodną.

```{code-block} c++
template <typename Derived>
struct Base
{
    void interface()
    {
        // ...
        static_cast<Derived*>(this)->implementation();
        // ...
    }

    static void static_interface()
    {
        // ...
        Derived::static_sub_func();
        // ...
    }
};

struct Derived : Base<Derived> // CRTP pattern
{
    void implementation()
    {
        std::cout << "Derived::implementation()\n";
    }

    static void static_sub_func()
    {
        std::cout << "Derived::static_sub_func()\n";
    }
};

int main()
{
    Derived d;
    d.interface(); // output: Derived::implementation()
    Derived::static_interface(); // output: Derived::static_sub_func()
}
```

W powyższym przykładzie klasa `Base` używa CRTP do wywołania metody `implementation()` z klasy pochodnej `Derived`. Metoda `Base::static_interface()` wywołuje statyczną metodę `static_sub_func()` z klasy pochodnej.