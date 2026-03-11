# Ders 2 — Move Semantics, `std::move` İmplementasyonu ve Special Member Functions

> **Kurs:** İleri C++ (C++26 uyumlu)
> **Tarih:** 26 Mart 2025
> **Kapsam:** Move semantiği motivasyonu, `std::move` iç yapısı, `remove_reference` meta function, alias template, reference collapsing, universal reference deduction, `noexcept`, special member functions

---

## 1. Move Semantics — Motivasyon ve Temel İlke

### 1.1 Amaç

Move semantiğinin tek amacı: **gereksiz kopyalamaları (deep copy) taşıma (move) işlemine dönüştürmek.**

- Bir nesne **kaynak** (resource) tutuyorsa (heap bellek, dosya handle'ı, soket vb.), kopyalama o kaynağın klonlanmasını gerektirir.
- Eğer kaynak nesne artık kullanılmayacaksa (ölmek üzere, geçici nesne vb.), kaynağı **kopyalamak** yerine **çalmak** (ownership transfer) çok daha ucuzdur.

```
Kopyalama (deep copy):   kaynak → yeni kaynak tahsis et → veriyi kopyala
Taşıma (move):           kaynak → sahipliği devret → eski nesneyi boşalt
```

### 1.2 Taşıma Her Zaman Avantaj Sağlamaz

> **Önemli Not:** Move semantiği **yalnızca** dinamik kaynak tutan sınıflar için performans avantajı sağlar. Sabit boyutlu veri tutan sınıflarda (örneğin `std::array`, C-style dizi, sadece POD üyeleri olan sınıflar) taşıma ile kopyalama arasında **hiçbir fark yoktur** — her iki durumda da byte-by-byte kopyalama yapılır.

```cpp
// Move semantiğinin FAYDASIZ olduğu sınıf örneği:
class Sensor {
    std::array<double, 1024> readings;  // sabit boyutlu, heap yok
    int id;
    // Move = Copy → taşımanın avantajı yok
};

// Move semantiğinin FAYDALI olduğu sınıf örneği:
class Buffer {
    std::unique_ptr<std::byte[]> data;  // heap allocation
    std::size_t size;
    // Move: pointer swap → O(1), Copy: memcpy → O(n)
};
```

### 1.3 Moved-From State ve Destructor

> **Önemli Not:** Taşınmış (moved-from) bir nesne hâlâ **geçerli bir nesne**dir ve destructor'ı **mutlaka çağrılacaktır**. Bu nedenle move constructor/assignment, kaynak nesneyi destructor'ın güvenle çalışabileceği bir duruma bırakmalıdır. Tipik olarak pointer üyeler `nullptr`'a, size üyeleri `0`'a set edilir.

---

## 2. `std::move` vs `std::ranges::move` (Algoritma)

İki farklı `move` fonksiyonu birbiriyle karıştırılmamalıdır:

| Fonksiyon | Başlık Dosyası | Görevi |
|---|---|---|
| `std::move(obj)` | `<utility>` | Value category'yi xvalue'ya cast eder. **Hiçbir şeyi taşımaz.** |
| `std::ranges::move(range, dest)` | `<algorithm>` | Gerçek bir algoritma — range'deki öğeleri hedef range'e **taşır**. |

### 2.1 `std::ranges::move` — Algoritma

```cpp
#include <algorithm>
#include <vector>
#include <string>
#include <print>

int main() {
    std::vector<std::string> src{"alpha", "beta", "gamma"};
    std::vector<std::string> dst(src.size());

    // ranges::move dahili olarak std::move (utility) kullanarak
    // her elemanı taşır:
    std::ranges::move(src, dst.begin());

    for (const auto& s : dst)
        std::println("{}", s);  // alpha, beta, gamma

    // src'deki string'ler artık moved-from state'te (boş)
}
```

### 2.2 Klasik `copy` Algoritmasının İmplementasyonu (Referans)

```cpp
// Basitleştirilmiş copy implementasyonu
template<typename InIter, typename OutIter>
constexpr OutIter copy(InIter first, InIter last, OutIter dest) {
    while (first != last) {
        *dest++ = *first++;   // kopyalama ataması
    }
    return dest;
}

// move algoritması: aynı yapı, std::move eklenmiş
template<typename InIter, typename OutIter>
constexpr OutIter move(InIter first, InIter last, OutIter dest) {
    while (first != last) {
        *dest++ = std::move(*first++);  // taşıma ataması
    }
    return dest;
}
```

> **Önemli Not:** STL algoritmalarının **hiçbiri** exception throw etmez. Hedef range'in yeterli kapasiteye sahip olduğundan emin olmak tamamen programcının sorumluluğundadır. Yetersiz range = **undefined behavior**.

---

## 3. `std::move` — İç Yapısı (Under the Hood)

### 3.1 `std::move` Ne Yapar, Ne Yapmaz?

```
std::move(expr) ≡ static_cast<T&&>(expr)
```

- **Yapar:** Operandının value category'sini **xvalue**'ya dönüştürür (lvalue → xvalue, prvalue → xvalue).
- **Yapmaz:** Hiçbir veri taşımaz, hiçbir kaynak transfer etmez, hiçbir side-effect üretmez.

> **Önemli Not:** `std::move` adı yanlış bir isimdir; doğru adı `move_cast` olabilirdi. Fonksiyon yalnızca bir **cast** işlemidir. Gerçek taşıma, elde edilen xvalue ifadesinin bir move constructor veya move assignment operatörü tarafından **kullanılması** ile gerçekleşir.

### 3.2 Tam İmplementasyon (C++26)

```cpp
#include <type_traits>

template<typename T>
[[nodiscard]] constexpr std::remove_reference_t<T>&& move(T&& param) noexcept {
    return static_cast<std::remove_reference_t<T>&&>(param);
}
```

Bu kısa fonksiyonun anlaşılması için birkaç alt kavram gereklidir:

---

## 4. `remove_reference` Meta Function

### 4.1 Meta Function Kavramı

| Normal Fonksiyon | Meta Function |
|---|---|
| Runtime'da **değer** hesaplar | Compile time'da **tür** veya **değer** hesaplar |
| `int square(int x)` | `template<typename T> struct remove_reference` |
| Input: değer → Output: değer | Input: tür → Output: tür |

### 4.2 Partial Specialization ile İmplementasyon

```cpp
// Primary template — referans olmayan türler için
template<typename T>
struct remove_reference {
    using type = T;
};

// Partial specialization — lvalue referans türleri için
template<typename T>
struct remove_reference<T&> {
    using type = T;
};

// Partial specialization — rvalue referans türleri için
template<typename T>
struct remove_reference<T&&> {
    using type = T;
};
```

| Template Argümanı | Eşleşen Specialization | `type` Sonucu |
|---|---|---|
| `int` | Primary | `int` |
| `int&` | `T&` partial spec. | `int` |
| `int&&` | `T&&` partial spec. | `int` |
| `const int&` | `T&` partial spec. | `const int` |

> **Önemli Not:** `remove_reference` yalnızca referanslığı kaldırır; `const` / `volatile` niteleyicileri korunur. `const int&` → `const int` (const kalır).

### 4.3 Alias Template (`_t` Suffix Konvansiyonu)

`typename` yazma zahmetinden kurtulmak için standart kütüphane her transformation trait için bir alias template sunar:

```cpp
// Standart kütüphanedeki alias template
template<typename T>
using remove_reference_t = typename remove_reference<T>::type;

// Kullanım karşılaştırması:
// Eski (zahmetli):
typename std::remove_reference<T>::type&&

// Modern (temiz):
std::remove_reference_t<T>&&
```

### 4.4 `typename` Anahtar Sözcüğü ve Relaxed `typename` (C++20)

```cpp
// C++20 öncesi: typename zorunlu (dependent type bağlamında)
template<typename T>
using my_alias = typename std::remove_reference<T>::type;

// C++20 sonrası: birçok bağlamda typename artık opsiyonel (P0634R3)
// Derleyici, bağlamdan türün tür olduğunu çıkarabilir
template<typename T>
using my_alias = std::remove_reference<T>::type;  // OK since C++20
```

---

## 5. Alias Template — Genel Kullanım

### 5.1 Template Kategorileri (C++26)

| # | Kategori | Örnek |
|---|---|---|
| 1 | Class Template | `template<typename T> class vector` |
| 2 | Function Template | `template<typename T> T max(T, T)` |
| 3 | Variable Template | `template<typename T> constexpr bool is_integral_v` |
| 4 | Alias Template | `template<typename T> using remove_ref_t = ...` |
| 5 | Concept | `template<typename T> concept integral = ...` |

### 5.2 Pratik Örnekler

```cpp
#include <set>
#include <functional>
#include <utility>

// Örnek 1: Aynı türden pair için kısaltma
template<typename T>
using epair = std::pair<T, T>;

epair<int> coords{3, 4};       // std::pair<int, int>
epair<double> range{0.0, 1.0}; // std::pair<double, double>

// Örnek 2: Büyükten küçüğe sıralanan set
template<typename T>
using gset = std::set<T, std::greater<T>>;

gset<int> scores{90, 85, 95};  // 95, 90, 85 sırasında
```

---

## 6. Universal (Forwarding) Reference ve Template Argument Deduction

### 6.1 Universal Reference Nedir?

```cpp
template<typename T>
void func(T&& param);   // ← Universal reference (forwarding reference)
```

Bir parametrenin universal reference olması için **iki koşul** zorunludur:

| Koşul | Açıklama |
|---|---|
| **Tür çıkarımı olmalı** | `T` çıkarım ile belirlenmelidir |
| **Tam olarak `T&&` formunda olmalı** | `const T&&`, `vector<T>&&` vb. **değildir** |

### 6.2 Universal Reference OLMAYAN Örnekler

```cpp
// (1) const niteleyicisi var → rvalue reference
template<typename T>
void f1(const T&& param);  // rvalue reference, universal DEĞİL

// (2) Tür çıkarımı yok, T önceden belirlenmiş → rvalue reference
template<typename T>
class Widget {
    void f2(T&& param);    // rvalue reference, universal DEĞİL
    // Çünkü T, sınıf instantiate edilirken zaten belirlendi

    // Gerçek universal reference için member template gerekir:
    template<typename U>
    void f3(U&& param);    // universal reference ✓
};

// (3) Bileşik tür → rvalue reference
template<typename T>
void f4(std::vector<T>&& param);  // rvalue reference, universal DEĞİL
```

> **Önemli Not:** `vector<T>::push_back(T&&)` bir rvalue reference parametredir, universal reference **değildir** — çünkü `T`, sınıf oluşturulurken zaten belirlenmiştir. Bu nedenle `push_back` yalnızca rvalue alır. Ancak `emplace_back` member template olduğu için universal reference kullanır ve her şeyi kabul eder.

### 6.3 Deduction Kuralları

| Argüman | `T` Çıkarımı | Parametrenin Gerçek Türü |
|---|---|---|
| **rvalue** (örn: `42`, `std::move(x)`) | `int` | `int&&` (rvalue ref) |
| **lvalue** (örn: `x`) | `int&` | `int& &&` → `int&` (reference collapsing) |
| **const lvalue** (örn: `cx`) | `const int&` | `const int&` |
| **const rvalue** (örn: `std::move(cx)`) | `const int` | `const int&&` |

```cpp
template<typename T>
void func(T&& param);

int x = 42;
const int cx = 99;

func(42);              // T = int,        param: int&&
func(x);               // T = int&,       param: int&   (ref collapsing)
func(cx);              // T = const int&, param: const int&
func(std::move(x));    // T = int,        param: int&&
func(std::move(cx));   // T = const int,  param: const int&&
```

### 6.4 Universal Reference Parametresinin Önemli Özelliği

Universal reference parametresi:
- Gelen argümanın **value category** bilgisini (lvalue/rvalue) **korur** (retain eder)
- Gelen argümanın **const/non-const** bilgisini **korur**
- **prvalue vs xvalue ayrımını KORUMAZ** — her iki rvalue türü de aynı şekilde işlenir

`const T&` parametresi ise her şeyi kabul eder ama value category bilgisini **kaybeder**.

---

## 7. Reference Collapsing (Referans Çökmesi)

### 7.1 Kurallar Tablosu

| Oluşan Referans | Sonuç |
|---|---|
| `T& &` | `T&` |
| `T& &&` | `T&` |
| `T&& &` | `T&` |
| `T&& &&` | `T&&` |

> **Önemli Not:** Yalnızca `&& + &&` birleşimi rvalue reference üretir. Diğer tüm kombinasyonlar lvalue reference'a çöker. Kısaca: **lvalue reference her zaman kazanır** (bulaşıcıdır).

### 7.2 Reference Collapsing Oluşan 4 Bağlam

| # | Bağlam | Örnek |
|---|---|---|
| 1 | Template argument deduction | `func(lvalue)` → `T = int&` → `int& &&` → `int&` |
| 2 | `auto&&` type deduction | `auto&& r = lvalue;` → `int& &&` → `int&` |
| 3 | `typedef` / `using` bildirimi | `using ref = int&; using rref = ref&&;` → `int&` |
| 4 | `decltype` | `decltype((lvalue))&&` → `int& &&` → `int&` |

```cpp
int x = 42;

// Bağlam 2: auto&& ile reference collapsing
auto&& r1 = x;          // auto = int&  → int& && → int&  (lvalue ref)
auto&& r2 = 42;         // auto = int   → int&&           (rvalue ref)

// Bağlam 3: typedef/using ile reference collapsing
using LRef = int&;
using Result = LRef&&;  // int& && → int&  (lvalue ref)
static_assert(std::is_lvalue_reference_v<Result>);  // true

// Bağlam 4: decltype ile reference collapsing
decltype(x)&& r3 = std::move(x);     // int&& (x isim, decltype(x) = int)
// decltype((x))&& r4 = ???;          // int& && → int& (ifade, lvalue → int&)
```

---

## 8. `std::move` İmplementasyonu — Adım Adım Analiz

```cpp
template<typename T>
[[nodiscard]] constexpr std::remove_reference_t<T>&& move(T&& param) noexcept {
    return static_cast<std::remove_reference_t<T>&&>(param);
}
```

### 8.1 lvalue Argüman Gönderildiğinde

```
std::move(x)   // x: int lvalue

T çıkarımı:  T = int&
Parametre:   int& && → int&  (reference collapsing)

Geri dönüş türü:
  remove_reference_t<int&>&&  →  int&&

static_cast hedef türü:
  remove_reference_t<int&>&&  →  int&&

Sonuç: int&& → xvalue ✓
```

### 8.2 rvalue Argüman Gönderildiğinde

```
std::move(42)  // 42: int prvalue

T çıkarımı:  T = int
Parametre:   int&&

Geri dönüş türü:
  remove_reference_t<int>&&  →  int&&

static_cast hedef türü:
  remove_reference_t<int>&&  →  int&&

Sonuç: int&& → xvalue ✓
```

> **Önemli Not:** `remove_reference_t` olmadan, lvalue argümanda `T = int&` olduğu için `T&&` ifadesi `int& &&` → `int&` olarak çökerdi. Bu da **lvalue reference** döndürürdü — ki move'un amacına tamamen aykırıdır. `remove_reference_t` bu tuzağı önler.

---

## 9. `noexcept` Specifier

### 9.1 Temel Kural

```cpp
void safe_func() noexcept;       // Bu fonksiyon exception ATMAZ
void risky_func();               // Bu fonksiyon exception ATABİLİR
void maybe_func() noexcept(expr);// expr true ise noexcept, false ise değil
```

### 9.2 Move Operasyonlarında `noexcept`'in Kritik Önemi

> **Önemli Not:** STL konteynerleri (özellikle `std::vector`), eleman taşırken move constructor'ın `noexcept` olup olmadığını **compile time'da kontrol eder**. Eğer move constructor `noexcept` **değilse**, konteyner güvenlik nedeniyle **kopyalama** yapar. Bu, beklenmedik performans kayıplarına yol açar.

```cpp
class Widget {
    std::string data;
public:
    // noexcept olmadan: vector reallocation'da KOPYALAMA yapılır!
    Widget(Widget&& other) : data(std::move(other.data)) {}

    // noexcept ile: vector reallocation'da TAŞIMA yapılır ✓
    Widget(Widget&& other) noexcept : data(std::move(other.data)) {}
};
```

### 9.3 `noexcept` Operatörü

```cpp
// noexcept operatörü: bir ifadenin exception atıp atmadığını compile time'da sorgular
constexpr bool is_safe = noexcept(std::declval<Widget>().~Widget());

// Koşullu noexcept ile delegasyon:
template<typename T>
void wrapper(T&& val) noexcept(noexcept(process(std::forward<T>(val)))) {
    process(std::forward<T>(val));
}
```

`noexcept` specifier vs `noexcept` operatörü:

| Kullanım | Rolü | Örnek |
|---|---|---|
| `void f() noexcept;` | **Specifier** — fonksiyonun garantisi | Fonksiyon exception atmaz |
| `noexcept(expr)` | **Operatör** — compile-time sorgu | `noexcept(x + y)` → `true`/`false` |
| `void f() noexcept(noexcept(g()));` | İkisi birlikte | f, g kadar güvenli |

---

## 10. Special Member Functions ve Move Semantiği

### 10.1 Altı Özel Üye Fonksiyon

| # | Fonksiyon | C++11 Öncesi | C++11 Sonrası |
|---|---|---|---|
| 1 | Default Constructor | Var | Var |
| 2 | Destructor | Var | Var |
| 3 | Copy Constructor | Var | Var |
| 4 | Copy Assignment | Var | Var |
| 5 | Move Constructor | — | **Yeni** |
| 6 | Move Assignment | — | **Yeni** |

### 10.2 Move Constructor ve Move Assignment İmzaları

```cpp
class MyClass {
public:
    MyClass(MyClass&& other) noexcept;              // Move constructor
    MyClass& operator=(MyClass&& other) noexcept;   // Move assignment
};
```

### 10.3 Derleyicinin Örtülü (Implicit) Bildirim Kuralları

> **Önemli Not:** Aşağıdaki kurallar **Rule of Five** uygulamasında kritik öneme sahiptir. Bir special member function'ı kendiniz bildirdiğinizde, derleyicinin diğerlerini otomatik üretip üretmeyeceğini bilmelisiniz.

**Move constructor/assignment'ın implicitly declared olma koşulları:**

Derleyici move member'ları **yalnızca** şu koşulların **tümü** sağlanırsa implicit olarak bildirir:

1. Kullanıcı tarafından bildirilmiş **copy constructor** yok
2. Kullanıcı tarafından bildirilmiş **copy assignment** yok
3. Kullanıcı tarafından bildirilmiş **move constructor** yok (move assignment için)
4. Kullanıcı tarafından bildirilmiş **move assignment** yok (move constructor için)
5. Kullanıcı tarafından bildirilmiş **destructor** yok

```cpp
// Senaryo 1: Derleyici TÜM special member'ları üretir
class A {};  // 6/6 implicit ✓

// Senaryo 2: Destructor yazıldı → move member'lar YOK
class B {
public:
    ~B() {}  // destructor var → move ctor/assign implicitly declared DEĞİL
    // copy ctor/assign: implicit (ama deprecated!)
};

// Senaryo 3: Copy constructor yazıldı → move member'lar YOK
class C {
public:
    C(const C&) = default;  // copy ctor var → move ctor/assign YOK
};
```

### 10.4 `= default` ve `= delete`

```cpp
class Resource {
public:
    Resource() = default;
    ~Resource() = default;

    // Kopyalama yasakla:
    Resource(const Resource&) = delete;
    Resource& operator=(const Resource&) = delete;

    // Taşımaya izin ver (derleyici üretsin):
    Resource(Resource&&) noexcept = default;
    Resource& operator=(Resource&&) noexcept = default;
};
```

### 10.5 Move Constructor Gövdesi — Tipik İmplementasyon

```cpp
#include <utility>
#include <cstddef>

class DynamicBuffer {
    std::byte* ptr_ = nullptr;
    std::size_t size_ = 0;

public:
    // Move constructor: kaynağı çal, eski nesneyi güvenli duruma bırak
    DynamicBuffer(DynamicBuffer&& other) noexcept
        : ptr_{std::exchange(other.ptr_, nullptr)}
        , size_{std::exchange(other.size_, 0)}
    {}

    // Move assignment: self-assignment kontrolü + kaynak çalma
    DynamicBuffer& operator=(DynamicBuffer&& other) noexcept {
        if (this != &other) {
            delete[] ptr_;
            ptr_  = std::exchange(other.ptr_, nullptr);
            size_ = std::exchange(other.size_, 0);
        }
        return *this;
    }

    ~DynamicBuffer() { delete[] ptr_; }
};
```

> **Önemli Not:** `std::exchange(obj, new_val)` fonksiyonu, `obj`'nin eski değerini döndürüp `obj`'ye `new_val` atadığı için move idiomlarında idealdir. Tek satırda hem "çal" hem "sıfırla" yapılır.

---

## 11. `std::move` ile Bir Nesneyi Ne Zaman Taşımalı?

### 11.1 Karar Ağacı

```
Nesneyi taşımalı mıyım?
│
├── Nesne bundan sonra kullanılacak mı?
│   └── EVET → TAŞIMA! (lvalue olarak kullan)
│
├── Nesne geçici (temporary) mi?
│   └── EVET → Otomatik taşınır (std::move gerekmez)
│
├── Nesne bir fonksiyonun lokal değişkeni ve return ediliyor mu?
│   └── EVET → std::move KULLANMA! (NRVO'yu engeller)
│
└── Nesne artık kullanılmayacak mı?
    └── EVET → std::move kullan ✓
```

### 11.2 Önemli Anti-Pattern'ler

```cpp
// YANLIŞ: return'de std::move kullanmak (NRVO'yu engeller)
std::string make_name() {
    std::string result = "hello";
    return std::move(result);  // KÖTÜ! Copy elision'ı engeller
    // return result;          // DOĞRU! NRVO uygulanır
}

// YANLIŞ: const nesneye std::move uygulamak
const std::string name = "Necati";
std::string copy = std::move(name);  // Move DEĞİL, kopyalama yapar!
// Çünkü: std::move(name) → const string&&
// const string&& → move ctor'a değil, copy ctor'a bağlanır
```

---

## 12. Özet: `std::move` Anahtar Noktalar

| Konu | Detay |
|---|---|
| `std::move` ne yapar? | `static_cast<T&&>` — value category'yi xvalue yapar |
| `std::move` ne yapmaz? | Hiçbir veriyi taşımaz |
| Neden `remove_reference_t` gerekli? | lvalue argümanda `T = T&` olur; `T&&&` → `T&` çöker; doğru cast için referans kaldırılmalı |
| Neden `noexcept`? | Move member'lar `noexcept` olmalı; aksi halde STL konteynerleri kopyalama yapar |
| Neden `constexpr`? | Compile-time bağlamda da kullanılabilmesi için |
| Universal reference parametre neden? | Hem lvalue hem rvalue argüman kabul edebilmek için |

---

## 13. Low-Latency Sistemler İçin Kritik Noktalar

> **Önemli Not:** High-frequency trading ve oyun motorları gibi düşük gecikmeli sistemlerde, `noexcept` move constructor'ın yokluğu `std::vector::push_back` gibi operasyonlarda **beklenmedik deep copy**'lere yol açar. Bu, mikrosaniye hassasiyetinde çalışan sistemlerde kabul edilemez gecikme spike'ları oluşturur. **Her performans-kritik sınıfın move constructor'ı `noexcept` olmalıdır.**

> **Önemli Not:** `std::move` bir cast olduğu için runtime maliyeti **sıfırdır** — herhangi bir instruction üretmez. Ancak yanlış kullanımı (const nesnelere uygulanması, return value'da kullanılması) sessizce kopyalama tetikler ve profiler'da şüpheli sıcak noktalar oluşturur.
