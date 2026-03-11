# Ders 5 — Perfect Forwarding, `std::forward` İmplementasyonu, Universal Reference Derinlemesine

> **Kurs:** İleri C++ (C++26 uyumlu)
> **Tarih:** 9 Nisan 2025
> **Kapsam:** Perfect forwarding kavramı, `std::forward` iç yapısı, `std::move` vs `std::forward`, universal reference olmayan durumlar, explicit/partial specialization ve overload resolution, overload vs universal reference, `auto&&` forwarding reference, variadic perfect forwarding

---

## 1. Perfect Forwarding — Kavram

### 1.1 Problem Tanımı

Bir **wrapper fonksiyon** (`call_foo`) argümanları alıp hedef fonksiyona (`foo`) iletirken, argümanların:
- **Value category**'sini (lvalue/rvalue) korumalı
- **const/non-const** niteliğini korumalı

Bu korunma sağlanamazsa overload resolution yanlış fonksiyonu seçer veya sentaks hatası oluşur.

```
Doğrudan çağrı:     foo(arg)       →  doğru overload seçilir
Wrapper üzerinden:  call_foo(arg)  →  call_foo, foo'yu aynı arg ile çağırmalı
                                      value category + constness korunmalı
```

### 1.2 Perfect Forwarding Gerektiren Standart Kütüphane Örnekleri

| Fonksiyon | Ne Yapar? |
|---|---|
| `vector::emplace_back(args...)` | Argümanları constructor'a forward eder |
| `make_unique<T>(args...)` | Argümanları `new T(args...)` constructor'a forward eder |
| `make_shared<T>(args...)` | Aynı — shared_ptr ile |
| `std::invoke(f, args...)` | Argümanları callable'a forward eder |
| `std::apply(f, tuple)` | Tuple elemanlarını fonksiyona forward eder |

### 1.3 Overload Patlaması Problemi

Perfect forwarding olmadan, N parametreli bir fonksiyon için **4^N** overload gerekir:

| Parametre Sayısı | Gerekli Overload Sayısı |
|---|---|
| 1 | 4 (`T&`, `const T&`, `T&&`, `const T&&`) |
| 2 | 16 |
| 3 | 64 |
| 4 | 256 |

```cpp
// 4 overload — sadece 1 parametre için (sürdürülemez)
void call_foo(const MyClass& m)  { foo(m); }
void call_foo(MyClass& m)       { foo(m); }
void call_foo(MyClass&& m)      { foo(std::move(m)); }
void call_foo(const MyClass&& m){ foo(std::move(m)); }

// Perfect forwarding ile: TEK fonksiyon
template<typename T>
void call_foo(T&& t) {
    foo(std::forward<T>(t));
}
```

---

## 2. Universal Reference — Nedir, Ne Değildir?

### 2.1 Universal Reference OLAN Durumlar

```cpp
// 1. Fonksiyon şablonunda T&&
template<typename T>
void func(T&& param);   // ✓ Universal reference

// 2. auto&&
auto&& x = expr;        // ✓ Universal reference

// 3. Lambda parametresinde auto&&
auto lambda = [](auto&& param) { /* ... */ };  // ✓ Universal reference
```

### 2.2 Universal Reference OLMAYAN Durumlar

| Durum | Örnek | Neden |
|---|---|---|
| `const` niteleyici var | `const T&&` | const kaldırır universal özelliği |
| Sınıf şablonunun üye fonksiyonu | `class C<T> { void f(T&&); }` | T zaten belirlenmiş, çıkarım yok |
| Specialization türü | `vector<T>&&` | Bileşik tür, doğrudan `T&&` değil |
| Nested type | `typename T::value_type&&` | Doğrudan `T&&` değil |

```cpp
// UNIVERSAL REFERENCE DEĞİL — Detaylı örnekler:

// 1. const niteleyici
template<typename T>
void f1(const T&& param);  // rvalue reference (sadece rvalue kabul eder)

// 2. Sınıf şablonu üye fonksiyonu
template<typename T>
class Widget {
    void f2(T&& param);    // rvalue reference! T zaten belirlenmiş
    // Widget<int> için: void f2(int&& param) → sadece rvalue

    // Gerçek universal reference için member template:
    template<typename U>
    void f3(U&& param);    // ✓ Universal reference
};

// 3. Specialization türü
template<typename T>
void f4(std::vector<T>&& param);  // rvalue reference

// 4. Nested type
template<typename T>
void f5(typename T::value_type&& param);  // rvalue reference
```

> **Önemli Not:** `std::vector::push_back(T&&)` bir rvalue reference'tır (universal reference **değil**), çünkü `T` sınıf instantiate edilirken zaten belirlenmiştir. Ancak `emplace_back` member template olduğu için universal reference kullanır. Bu fark, API tasarımında kritik öneme sahiptir.

---

## 3. Template Argument Deduction — Universal Reference Bağlamında

### 3.1 Çıkarım Kuralları

| Gönderilen Argüman | `T` Çıkarımı | Fonksiyon Parametresi | Fonksiyon İçinde `T` Kullanımı |
|---|---|---|---|
| **lvalue** (`x`) | `MyClass&` | `MyClass&` (ref collapsing) | `T` = referans türü (**dikkat!**) |
| **const lvalue** (`cx`) | `const MyClass&` | `const MyClass&` | `T` = const referans türü |
| **rvalue** (`MyClass{}`, `move(x)`) | `MyClass` | `MyClass&&` | `T` = non-reference türü |
| **const rvalue** (`move(cx)`) | `const MyClass` | `const MyClass&&` | `T` = const non-reference türü |

### 3.2 `T` Türünün Referans Olmasının Sonuçları

```cpp
template<typename T>
void func(T&& param) {
    T local;  // T = int& ise → SENTAKS HATASI! (referans initialize edilmeli)
              // T = int  ise → OK, default constructor
}

int x = 42;
func(x);        // T = int&  → T local; HATA!
func(42);       // T = int   → T local; OK
```

> **Önemli Not:** Universal reference parametreli fonksiyonlarda `T` doğrudan kullanılırken dikkatli olunmalıdır. `T` bir referans türü olabilir. Referanslıktan arındırılmış tür için `std::remove_reference_t<T>` kullanılmalıdır.

---

## 4. `std::move` vs `std::forward` — Fark

| Özellik | `std::move` | `std::forward` |
|---|---|---|
| **Girdi** | Her şey (lvalue/rvalue) | Her şey (lvalue/rvalue) |
| **Çıktı** | **Her zaman rvalue** (xvalue) | lvalue gelirse **lvalue**, rvalue gelirse **rvalue** |
| **Amaç** | Unconditional rvalue cast | Conditional value category preservation |
| **Template argüman** | Gerekli değil (çıkarım yapılır) | **Zorunlu** (explicit yazılmalı) |
| **Kullanım** | `std::move(x)` | `std::forward<T>(x)` |

```cpp
template<typename T>
void wrapper(T&& arg) {
    // std::move: her zaman rvalue döndürür
    consume(std::move(arg));     // arg lvalue olsa bile rvalue olarak iletilir

    // std::forward: value category'yi korur
    consume(std::forward<T>(arg)); // arg lvalue ise lvalue, rvalue ise rvalue
}
```

---

## 5. `std::forward` İmplementasyonu (C++26)

### 5.1 İki Overload

`std::forward` iki ayrı overload ile implement edilir: biri lvalue'lar, biri rvalue'lar için.

```cpp
#include <type_traits>

// Overload 1: lvalue'lar için
template<typename T>
[[nodiscard]] constexpr T&& forward(std::remove_reference_t<T>& t) noexcept {
    return static_cast<T&&>(t);
}

// Overload 2: rvalue'lar için
template<typename T>
[[nodiscard]] constexpr T&& forward(std::remove_reference_t<T>&& t) noexcept {
    static_assert(!std::is_lvalue_reference_v<T>,
                  "Cannot forward an rvalue as an lvalue!");
    return static_cast<T&&>(t);
}
```

### 5.2 Neden `remove_reference_t` Gerekli?

Forward'ın template argümanı **explicit** olarak verilir (çıkarım yapılmaz). `T` bir referans türü olabilir. `remove_reference_t` ile parametrenin gerçek türü elde edilir.

### 5.3 Adım Adım Analiz

#### lvalue Gönderildiğinde:

```
Senaryo: forward<string&>(param)   // T = string&

Overload 1 seçilir (param lvalue):
  Parametre: remove_reference_t<string&>& → string&
  Geri dönüş: T&& → string& && → string&  (ref collapsing)
  static_cast<string&>(param) → lvalue ✓
```

#### rvalue Gönderildiğinde:

```
Senaryo: forward<string>(param)    // T = string

Overload 1 seçilir (param isim → lvalue):
  Parametre: remove_reference_t<string>& → string&
  Geri dönüş: T&& → string&&
  static_cast<string&&>(param) → rvalue (xvalue) ✓
```

### 5.4 Neden `std::forward`'ın Template Argümanı Explicit?

```cpp
template<typename T>
void wrapper(T&& arg) {
    foo(std::forward<T>(arg));  // T explicit olarak yazılmalı
    // foo(std::forward(arg));  // HATA! Template argüman çıkarımı yapılmaz
}
```

Çünkü `forward`'ın parametresi `remove_reference_t<T>&` formunda — `T`'yi operand türünden çıkarım yoluyla elde etmek imkansız. `T`'nin ne olduğu, **çağıran fonksiyonun** template parametresinden gelir.

---

## 6. Perfect Forwarding Olmadan da Çalışan Alternatif

```cpp
template<typename T>
void call_foo(T&& param) {
    if constexpr (std::is_lvalue_reference_v<T>) {
        foo(param);              // T = lvalue ref → doğrudan ilet
    } else {
        foo(std::move(param));   // T = non-ref → rvalue olarak ilet
    }
}
```

Bu yaklaşım doğru çalışır ancak `std::forward` kadar idiomatik değildir ve variadic parametrelerde kullanılamaz.

---

## 7. Explicit (Full) Specialization ve Overload Resolution

### 7.1 Kritik Kural: Specialization Overload Resolution'a Katılmaz

```cpp
// Primary template 1
template<typename T>
void func(T param) { std::println("primary T"); }

// Primary template 2 (overload)
template<typename T>
void func(T* param) { std::println("T*"); }

// Template 1'in explicit specialization'ı
template<>
void func<int*>(int* param) { std::println("explicit spec int*"); }

int x = 42;
func(&x);
// Overload resolution: Template 1 vs Template 2
// Template 2 daha spesifik (partial ordering rules) → Template 2 seçilir
// Template 1'in specialization'ı KULLANILMAZ!
```

> **Önemli Not:** Overload resolution **sadece primary template'ler** arasında yapılır. Seçilen primary template'in explicit specialization'ı varsa **o zaman** kullanılır. Bu kural, template specialization'ın beklenmedik şekilde görmezden gelinmesine yol açabilir.

### 7.2 Specialization'ın Hangi Template'e Ait Olduğu

```cpp
template<typename T> void f(T) {}     // Primary 1
template<typename T> void f(T*) {}    // Primary 2

// Bu specialization hangi template'in?
template<> void f<int*>(int*) {}      // Primary 1'in (T = int*)
template<> void f<int>(int*) {}       // Primary 2'nin (T = int)
```

---

## 8. Universal Reference Parametrenin 3 Kullanım Amacı

| # | Amaç | Açıklama |
|---|---|---|
| 1 | **Perfect Forwarding** | Argümanları başka fonksiyona value category'yi koruyarak iletme |
| 2 | **Value Category Dependent Code** | lvalue/rvalue'ya göre farklı compile-time kod dalları |
| 3 | **Constness Dependent Code** | const/non-const'a göre farklı compile-time kod dalları |

```cpp
template<typename T>
void multi_purpose(T&& param) {
    // Amaç 1: Perfect forwarding
    target_func(std::forward<T>(param));

    // Amaç 2: Value category dependent
    if constexpr (std::is_lvalue_reference_v<T>) {
        std::println("lvalue received");
    } else {
        std::println("rvalue received");
    }

    // Amaç 3: Constness dependent
    if constexpr (std::is_const_v<std::remove_reference_t<T>>) {
        std::println("const object");
    } else {
        std::println("non-const object");
    }
}
```

---

## 9. `auto&&` — Forwarding Reference

`auto&&` de bir forwarding reference'tır ve aynı deduction kuralları geçerlidir:

```cpp
int x = 42;
const int cx = 99;

auto&& r1 = x;           // auto = int&    → int&   (lvalue ref)
auto&& r2 = cx;          // auto = const int& → const int& (lvalue ref)
auto&& r3 = 42;          // auto = int     → int&&  (rvalue ref)
auto&& r4 = std::move(x);// auto = int     → int&&  (rvalue ref)
```

### 9.1 Lambda'da `auto&&` ile Perfect Forwarding

```cpp
// C++20 lambda ile perfect forwarding
auto wrapper = []<typename T>(T&& arg) {
    return target(std::forward<T>(arg));
};

// Veya auto&& ile (C++14+)
auto wrapper2 = [](auto&& arg) {
    return target(std::forward<decltype(arg)>(arg));
};
```

---

## 10. Variadic Perfect Forwarding

Birden fazla parametre için perfect forwarding — variadic template ile:

```cpp
template<typename... Args>
auto make_widget(Args&&... args) {
    return Widget(std::forward<Args>(args)...);
}

// Emplace fonksiyonları böyle çalışır:
template<typename T, typename... Args>
std::unique_ptr<T> make_unique(Args&&... args) {
    return std::unique_ptr<T>(new T(std::forward<Args>(args)...));
}
```

---

## 11. Universal Reference ve Overload — Dikkat Edilmesi Gerekenler

### 11.1 Universal Reference Overload Her Şeyi Çeker

```cpp
class Widget {
    std::string name;
public:
    // Bu ikisi arasında overload resolution beklendiği gibi çalışmayabilir:
    void set_name(const std::string& n) { name = n; }  // (1)

    template<typename T>
    void set_name(T&& n) { name = std::forward<T>(n); } // (2)

    // set_name("hello") → (2) seçilir (const char[6] → exact match)
    // set_name(str)      → (2) seçilir (string& → exact match, (1) const& dönüşüm gerektirir)
};
```

> **Önemli Not:** Universal reference parametreli fonksiyonlar, overload resolution'da neredeyse **her zaman kazanır** çünkü exact match üretirler. Bu, `const T&` overload'ların beklenmedik şekilde devre dışı kalmasına yol açar. Çözüm: SFINAE, `if constexpr`, veya C++20 concepts ile kısıtlama.

### 11.2 Concepts ile Kısıtlama (C++20/26)

```cpp
#include <concepts>
#include <string>

class Widget {
    std::string name;
public:
    // Sadece string-like türler için universal reference
    template<std::convertible_to<std::string> T>
    void set_name(T&& n) {
        name = std::forward<T>(n);
    }
};
```

---

## 12. Low-Latency Sistemler İçin Kritik Noktalar

> **Önemli Not:** Perfect forwarding **sıfır overhead**'dir — `std::forward` bir `static_cast`'tır ve hiçbir runtime instruction üretmez. Ancak template instantiation compile time'ı artırır. Header-heavy projelerde bu, build time'ı ciddi şekilde etkileyebilir.

> **Önemli Not:** Variadic perfect forwarding (`Args&&... args` + `std::forward<Args>(args)...`), constructor delegation pattern'larında **en verimli** yoldur. `emplace_back` yerine `push_back` kullanmak, gereksiz temporary nesne oluşturarak allocation + copy/move maliyeti ekler.

> **Önemli Not:** Universal reference parametreli fonksiyonlar, **her farklı argüman türü/value category kombinasyonu** için ayrı bir template instantiation üretir. Bu, binary boyutunu artırabilir (code bloat). Performans-kritik ama binary-boyut-hassas embedded sistemlerde bu trade-off değerlendirilmelidir.
