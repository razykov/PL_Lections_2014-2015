# Лекция 11 


## Перегрузка унарных операций

Для класса `Date` очень полезными будут операции `d++` и `++d`. Рассмотрим реализацию таких операторов.


### Префиксная унарная операция (@a `++a` `--a`) (общий случай)

Префиксная унарная операция может быть определена двумя способами:

1. В виде функции-члена: `@a ~ a.operator@()`.
2. В виде внешней функции: `@a ~ operator@(a)`.

Определим эту операцию для класса `Date`

```cpp
class Date
{
    …
    public:
    …
    // определение префиксного ++
    Date & operator++()
    {
        add_days(1);
        return *this;
    }
};
```


### Постфиксная унарная операция (a@ `a++` `a--`)

При перегрузке префиксной и постфиксной унарных операций встает вопрос, об интерпретации записи типа `operator@`.
В первых версиях языка префиксная и постфиксная операции определялись одинаково, но 1998 году для их разделения был введен фиктивный параметр типа `int`.

Постфиксная унарная операция также может быть определена двумя способами:

1. В виде функции-члена `a@ ~ a.operator@(int)`.
2. В виде внешней функции `a@ ~ operator@(a, int)`.

Рассмотрим ее реализацию для класса `Date`:

```cpp
class Date
{
    …
    public:
    …
    // определение постфиксного ++
    // d++ возвращает значение до увеличения, 
    // значит возвращать нужно копию, а не ссылку.
    Date operator++(int)
    {
        Date d = *this;
        add_days(1); // ~ ++(*this);
        return d;
    }
};
```

Как видим, из-за копирования всего объекта операция `d++` будет выполняется дольше, поэтому надо выбирать в пользу `++d`. 
Это верно для своих типов, но для встроенных типов компилятор производит оптимизацию. В результате записи `i++` и `++i`, например, для переменных типа `int`, равноценны.


## Перегрузка операции != 

Операция `!=` симметрична, поэтому предпочтительным выбором будет перегрузка оператора `operator!=` как внешней функции. Однако, для увеличения производительности, объявим функцию другом класса — таким образом она станет `inline`.

```cpp
class Date
{
    …
    public:
    …
    friend bool operator!=(const Date &d1, const Date &d2)
    {
        return !(d1==d2);
    }  
};
```

Как видим, использование ранее определенной операции `==` позволяет не обращаться к внутренним полям класса напрямую. Так как описанная функция является `inline`, то в результате будет произведена следующая замена.

`d1!=d2` ~ `!(d1==d2)` ~ `!(d1.d == d2.d && d1.m == d2.m && d1.y == d2.y)`


## Класс динамического массива

Попробуем реализовать аналог класса `vector`.

```cpp
template<typename T>
class myvector
{
    T* data;
    int sz;
    
  public:
    myvector(int size): sz(size)
    {
        data = new T[sz];
    }
    
    int size()
    {
        return sz;
    }
};
```


### Константные функции-члены

Теперь напишем процедуру, которая выведет на экран содержимое нашего вектора.

```cpp
void print(const myvector<int> &v)
{
    for(int i = 0 ; i < v.size(); i++)
    cout << v[i] << ' ';
}
```

Такой код не скомпилируется: ошибка произойдет из-за функции-члена `v.size()` так как эта функция, потенциально может изменить состояние объекта, в то время как формальный параметр `v` объявлен с модификатором `const`.

Для решения этой проблемы функцию `size()` надо определить как константный.

```cpp
int size() const
{
    return sz;
}
```
 

## Перегрузка операции []

Теперь рассмотрим перегрузку операции `operator[]` для того же класса `myvector`.

```cpp
template <typename T>
class myvector
{
    T* data;
    int sz;
    
  public:
    …
    T& operator[](int n)
    {
        if(n < 0 || n >= sz)
            throw n;
        return data[n];
    }
};
```

Так как `operator[]` ― `inline`, то будет произведена замена:

`v[i]` ~ `v.operator[](i)` ~ `v.data[i]`

При компиляции процедуры `print` вновь произойдет ошибка. На этот раз причиной будет функция `v[i]`. Для корректной компиляции в данном случае необходимо определить вторую функцию.

```cpp
T operator[](int n) const
{
	if(n < 0 || n >= sz)
		throw n;
	return data[n];
}
```


## Деструктор и его необходимость

Деструктор — коренная особенность C++

Рассмотрим такой случай:

```cpp
{
    myvector<int> v(100000);
    v[0] = v[99999];
    …
}
```

Внутри `v` создается массив в динамической памяти на 100000 элементов, а после работы с ним освобождения этого блока не происходит. Возникает утечка памяти. 
Для борьбы с утечками памяти в таких случаях в C++ был придуман деструктор.

Реализуем деструктор для класса `myvector`.

```cpp
template <typename T>
class myvector
{
    T* data;
    int sz;
    
  public:
    ~myvector()
    {
        delete[] data;
    }
};
```

Деструкторы всегда вызываются **неявно** в **определенный момент времени**. Для локальных объектов при выходе из блока, а для глобальных при завершении программы.

Принципиальное отличие этого подхода от сборщика мусора: сборщик мусора вызывается в недетерминированный момент времени, а деструкторы всегда вызываются в детерминированный момент времени, когда заканчивается время жизни объекта.
