# Ders 1 — Value Categories, Referans Türleri ve `decltype` Kuralları

> **Kurs:** İleri C++ (C++26 uyumlu)
> **Tarih:** 24 Mart 2025
> **Kapsam:** Expression value categories, referans bağlama kuralları, `decltype` tür çıkarımı

---

## 1. İfade (Expression) Temelleri

Her ifadenin iki temel niteliği vardır:

| Nitelik | Açıklama |
|---|---|
| **Data Type** (Tür) | İfadenin ürettiği değerin türü (`int`, `double`, `std::string` vb.) |
| **Value Category** (Değer Kategorisi) | İfadenin bellekteki konumu ve taşınabilirliğiyle ilgili sınıflandırması |

### 1.1 İfade Türü Hakkında Kritik Kurallar

- İfadelerin türü **referans türü olamaz**. `int& r = x;` bildiriminde `r` ifadesinin türü `int`'tir, `int&` değil.
- İfadelerin türü **pointer türü olabilir**: `int* p = &x;` → `p` ifadesinin türü `int*`.
- `void` bir türdür; `void` fonksiyona yapılan çağrı ifadesinin türü `void`'dir.
- **Integral promotion** kuralı: `char c; +c` ifadesinin türü `int`'tir (karakter aritmetiği `int`'e yükseltilir).

```cpp
char c1, c2;
// c1 + c2 ifadesinin türü int'tir (integral promotion)
static_assert(std::is_same_v<decltype(c1 + c2), int>);
```

---

## 2. Value Categories (Değer Kategorileri)

### 2.1 Birincil (Primary) Değer Kategorileri

Her ifade **tam olarak bir** birincil kategoriye aittir:

| Kategori | Tam Adı | Kimliği Var mı? | Taşınabilir mi? |
|---|---|---|---|
| **lvalue** | Locator Value | Evet | Hayır |
| **prvalue** | Pure R-Value | Hayır | Evet |
| **xvalue** | Expiring Value | Evet | Evet |

### 2.2 Bileşik (Combined) Değer Kategorileri

| Kategori | Eşdeğer | Tanım |
|---|---|---|
| **glvalue** | lvalue ∪ xvalue | Kimliği olan (has identity) tüm ifadeler |
| **rvalue** | prvalue ∪ xvalue | Taşınabilir (can be moved from) tüm ifadeler |

### 2.3 Venn Şeması

```
         ┌──────────────────────────────────────────┐
         │              glvalue                      │
         │   ┌──────────┐    ┌──────────┐           │
         │   │  lvalue   │    │  xvalue  │           │
         │   │           │    │          │           │
         │   │ identity  │    │ identity │           │
         │   │ !movable  │    │ movable  │           │
         │   └──────────┘    └────┬─────┘           │
         └────────────────────────┼──────────────────┘
                                  │
         ┌────────────────────────┼──────────────────┐
         │              rvalue    │                   │
         │                  ┌────┴─────┐             │
         │                  │  xvalue  │             │
         │                  │          │             │
         │   ┌──────────┐  └──────────┘             │
         │   │ prvalue   │                           │
         │   │!identity  │                           │
         │   │ movable   │                           │
         │   └──────────┘                            │
         └───────────────────────────────────────────┘
```

> **Önemli Not:** xvalue hem glvalue hem rvalue kümesinin elemanıdır. Bu, xvalue ifadelerin hem kimliğe sahip hem de taşınabilir olduğu anlamına gelir — move semantiğinin temel taşıdır.

> **Önemli Not:** Value category bir **değişkenin** değil, bir **ifadenin** niteliğidir. `int&& r = 10;` bildiriminde `r` ifadesinin value category'si **lvalue**'dur (bir değişken ismi olduğu için), data type'ı ise `int`'tir.

---

## 3. Hangi İfade Hangi Kategoride?

### 3.1 lvalue İfadeler

| İfade Türü | Örnek | Açıklama |
|---|---|---|
| Değişken isimleri | `x`, `name` | Rvalue referans bile olsa: `int&& r = 5;` → `r` ifadesi lvalue |
| Fonksiyon isimleri | `foo` | `std::move(foo)` bile lvalue üretir |
| String literalleri | `"hello"` | Dizi gibi davranır → lvalue |
| Prefix `++`/`--` | `++a`, `--a` | Nesnenin kendisini döndürür |
| Atama operatörleri | `a = b`, `a += b` | Compound assignment dahil |
| Üye erişimi (lvalue nesne) | `obj.member`, `ptr->member` | Nesne lvalue ise erişim de lvalue |
| `.*` / `->*` operatörleri | `x.*mp`, `ptr->*mp` | Member pointer erişimi |
| Dizi erişimi | `arr[n]` | Dizi lvalue ise |
| Dereference | `*p` | Her zaman lvalue |
| lvalue ref döndüren fonksiyon çağrısı | `foo()` where `int& foo();` | Geri dönüş türü `T&` ise |
| lvalue ref'e cast | `static_cast<T&>(expr)` | Hedef tür lvalue referans ise |
| Referans olan non-type template parametresi | `template<int& R>` içinde `R` | lvalue |

```cpp
#include <string>
#include <utility>

int& get_ref();
void example() {
    int x = 42;
    int&& rr = std::move(x);

    // Hepsi lvalue:
    x;                          // değişken ismi
    rr;                         // rvalue ref değişkeni AMA ifade olarak lvalue!
    "hello world";              // string literal
    ++x;                        // prefix increment
    get_ref();                  // lvalue ref döndüren fonksiyon
    *(&x);                      // dereference
}
```

> **Önemli Not:** `int&& rr = std::move(x);` bildiriminde `rr`'nin **data type**'ı `int`'tir (ifade türü referans olamaz) ve **value category**'si **lvalue**'dur (bir isimdir). Tür ile kategoriyi karıştırmak en sık yapılan hatadır.

### 3.2 prvalue İfadeler

| İfade Türü | Örnek |
|---|---|
| Tüm literaller (string hariç) | `42`, `3.14`, `'A'`, `true`, `nullptr` |
| Aritmetik operatörler | `a + b`, `a * b` |
| Karşılaştırma operatörleri | `a == b`, `a <=> b` |
| Mantıksal operatörler | `a && b`, `!a` |
| Postfix `++`/`--` | `a++`, `a--` |
| Adres operatörü | `&x` |
| `this` pointeri | Üye fonksiyon içinde `this` |
| Enumerator'ler | `Color::Red` |
| Lambda ifadeleri | `[](int x){ return x; }` |
| Geçici nesne oluşturma | `std::string{"temp"}` |
| Referans döndürmeyen fonksiyon çağrısı | `foo()` where `int foo();` |
| Sign operatörü (`+`, `-`) | `+x`, `-x` |
| Non-type template parametresi (referans olmayan) | `template<int N>` içinde `N` |
| `requires` ifadeleri | `requires { ... }` (C++20) |

```cpp
#include <string>

int foo();
void example() {
    int a = 10;

    // Hepsi prvalue:
    42;                         // integer literal
    a + 1;                      // aritmetik
    a > 0;                      // karşılaştırma
    &a;                         // adres operatörü
    foo();                      // non-ref döndüren fonksiyon
    std::string{"geçici"};      // temporary object
    [](int x){ return x * 2; };// lambda ifadesi
    +a;                         // sign operatörü (C/C++'da prvalue!)
}
```

> **Önemli Not:** Lambda ifadeleri derleyici tarafından closure type türünden **geçici nesne** (temporary object) olarak ele alınır, bu nedenle her zaman prvalue'dur. `auto f = []{}; ` bildiriminde `f` ifadesi lvalue'dur (isimlendirilmiş değişken), ama sağ taraftaki lambda ifadesinin kendisi prvalue'dur.

### 3.3 xvalue İfadeler

| İfade Türü | Örnek |
|---|---|
| Rvalue ref döndüren fonksiyon çağrısı | `std::move(x)`, `std::forward<T>(x)` |
| rvalue nesnenin üye erişimi | `std::move(obj).member`, `MyClass{}.x` |
| rvalue nesne + `.*` | `std::move(obj).*mp` |
| rvalue dizi erişimi | `std::move(arr)[0]` |
| rvalue ref'e cast | `static_cast<T&&>(expr)` |

```cpp
#include <utility>

struct Widget {
    int data;
};

int&& rval_func();

void example() {
    Widget w{42};

    // Hepsi xvalue:
    std::move(w);                  // std::move → T&& döndürür
    rval_func();                   // rvalue ref döndüren fonksiyon
    Widget{}.data;                 // prvalue nesnenin member erişimi
    static_cast<Widget&&>(w);      // rvalue ref'e cast

    // Dizi örneği:
    using Arr5 = int[5];
    Arr5{1,2,3,4,5}[0];           // rvalue dizinin elemanına erişim → xvalue
}
```

---

## 4. Referans Türleri ve Bağlama Kuralları

### 4.1 Üç Referans Kategorisi

```cpp
std::string& r1  = s;    // (1) lvalue reference
std::string&& r2 = expr; // (2) rvalue reference
// (3) forwarding (universal) reference — yalnızca tür çıkarımı bağlamında:
template<typename T>
void func(T&& param);    // T&& burada forwarding reference
```

> **Önemli Not:** `T&&` ifadesi **yalnızca tür çıkarımı varsa** forwarding reference'tır. Aksi halde sıradan rvalue reference'tır. Bu ayrım perfect forwarding için kritiktir.

### 4.2 Bağlama Kuralları Tablosu

| Referans Türü | lvalue | prvalue | xvalue |
|---|---|---|---|
| `T&` (lvalue ref) | Bağlanır | **HATA** | **HATA** |
| `const T&` (const lvalue ref) | Bağlanır | Bağlanır | Bağlanır |
| `T&&` (rvalue ref) | **HATA** | Bağlanır | Bağlanır |
| `const T&&` (const rvalue ref) | **HATA** | Bağlanır | Bağlanır |

```cpp
#include <string>
#include <utility>

void example() {
    std::string name{"Necati"};

    // lvalue reference — sadece lvalue'ya bağlanır:
    std::string& r1 = name;              // OK: lvalue
    // std::string& r2 = std::string{}; // HATA: prvalue → lvalue ref'e bağlanamaz
    // std::string& r3 = std::move(name);// HATA: xvalue → lvalue ref'e bağlanamaz

    // const lvalue reference — her şeye bağlanır:
    const std::string& cr1 = name;              // OK: lvalue
    const std::string& cr2 = std::string{};     // OK: prvalue
    const std::string& cr3 = std::move(name);   // OK: xvalue

    // rvalue reference — sadece rvalue'ya bağlanır:
    std::string&& rr1 = std::string{};          // OK: prvalue
    std::string&& rr2 = std::move(name);        // OK: xvalue
    // std::string&& rr3 = name;                // HATA: lvalue → rvalue ref'e bağlanamaz
}
```

> **Önemli Not:** `const T&&` pratikte neredeyse hiç kullanılmaz; ancak function overload resolution kurallarını anlamak için varlığını bilmek gerekir.

---

## 5. Fonksiyon Geri Dönüş Türü ve Value Category İlişkisi

| Geri Dönüş Türü | Çağrı İfadesinin Value Category'si |
|---|---|
| `T` (non-reference) | **prvalue** |
| `T&` (lvalue reference) | **lvalue** |
| `T&&` (rvalue reference) | **xvalue** |

```cpp
#include <string>

std::string  by_value();  // çağrı → prvalue
std::string& by_lref();   // çağrı → lvalue
std::string&& by_rref();  // çağrı → xvalue

// std::move'un xvalue üretmesinin sebebi:
// std::move(x) → geri dönüş türü T&& → xvalue
```

---

## 6. `decltype` ile Value Category Tespiti

`decltype` specifier'ının **iki farklı çıkarım kuralı** vardır:

### 6.1 Kural 1 — Operand Bir İsim (Identifier) İse

Çıkarım, ismin **bildirim türünü** (declared type) birebir verir:

```cpp
int x = 5;
int& r = x;
int&& rr = 10;
const int& cr = x;

struct S { int mx; };
S obj;

static_assert(std::is_same_v<decltype(x),       int>);
static_assert(std::is_same_v<decltype(r),        int&>);
static_assert(std::is_same_v<decltype(rr),       int&&>);
static_assert(std::is_same_v<decltype(cr),       const int&>);
static_assert(std::is_same_v<decltype(obj.mx),   int>);       // üye erişimi de isim
```

### 6.2 Kural 2 — Operand Bir İfade (Expression) İse

Çıkarım, ifadenin **value category**'sine göre belirlenir:

| Value Category | `decltype` Ürettiği Tür |
|---|---|
| **lvalue** | `T&` |
| **prvalue** | `T` |
| **xvalue** | `T&&` |

```cpp
#include <utility>

int x = 5;

// Parantez eklemek ismi ifadeye dönüştürür:
static_assert(std::is_same_v<decltype(x),    int>);    // isim → declared type
static_assert(std::is_same_v<decltype((x)),   int&>);   // ifade + lvalue → T&

// Value category tespiti:
static_assert(std::is_same_v<decltype(42),              int>);    // prvalue → T
static_assert(std::is_same_v<decltype(std::move(x)),    int&&>);  // xvalue  → T&&
static_assert(std::is_same_v<decltype((x)),             int&>);   // lvalue  → T&
```

> **Önemli Not:** `decltype(x)` ile `decltype((x))` **farklı sonuç üretir**. Parantez, ismi bir ifadeye dönüştürür. Bu fark, `decltype(auto)` kullanımında beklenmeyen referans döndürme hatalarına yol açabilir — dikkat edilmesi gereken kritik bir noktadır.

### 6.3 `decltype` ile Value Category Test Fonksiyonu (C++26)

```cpp
#include <type_traits>
#include <print>

template<typename T>
consteval void print_value_category() {
    if constexpr (std::is_lvalue_reference_v<T>)
        std::println("lvalue");
    else if constexpr (std::is_rvalue_reference_v<T>)
        std::println("xvalue");
    else
        std::println("prvalue");
}

// Kullanım:
// print_value_category<decltype((expr))>();

int main() {
    int x = 42;
    print_value_category<decltype((x))>();              // lvalue
    print_value_category<decltype((std::move(x)))>();   // xvalue
    print_value_category<decltype((42))>();              // prvalue
}
```

---

## 7. Özet Karar Ağacı

```
İfadenin value category'si nedir?
│
├── İsim mi (değişken, fonksiyon, string literal)?
│   └── EVET → lvalue
│       (Dikkat: rvalue ref değişkeni bile lvalue!)
│
├── std::move() / static_cast<T&&>() / rvalue ref döndüren fonksiyon çağrısı mı?
│   └── EVET → xvalue
│
├── Geçici nesne / literal / aritmetik / lambda mı?
│   └── EVET → prvalue
│
└── Üye erişimi mi? (obj.member)
    ├── obj lvalue ise → lvalue
    └── obj rvalue ise → xvalue
```

---

## 8. Low-Latency ve Yüksek Performans İçin Kritik Noktalar

> **Önemli Not:** Move semantiğinin temel motivasyonu gereksiz kopyalamaları ortadan kaldırmaktır. Bir nesne **xvalue** kategorisindeyse, kaynaklarının "çalınabileceği" (move edilebileceği) anlamına gelir. High-frequency trading, oyun motorları ve gerçek zamanlı sistemlerde bu ayrım doğrudan gecikme (latency) farkı yaratır — özellikle `std::string`, `std::vector` gibi heap allocation yapan türlerde.

> **Önemli Not:** `const T&` her value category'ye bağlanabilmesi sayesinde generic kod yazarken güvenli bir "catch-all" olarak kullanılır, ancak bu durumda move optimization devre dışı kalır. Performans-kritik kodda overload setinde `T&&` parametresinin bulunması, rvalue argümanlarda kopyalama yerine taşıma yapılmasını sağlar.
