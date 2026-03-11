# Ders 6 — `std::forward` Detayları, Abbreviated Template Syntax, SFINAE ve Concepts ile Kısıtlama

> **Kurs:** İleri C++ (C++26 uyumlu)
> **Tarih:** 14 Nisan 2025
> **Kapsam:** `std::forward` rvalue overload'undaki `static_assert`, abbreviated template syntax (C++20), `decltype` ile perfect forwarding, lambda'larda perfect forwarding, ref-qualified getter + forwarding, universal reference kısıtlama (concepts `requires`, SFINAE `enable_if`), belirli tür için universal reference (`is_same`, `is_convertible`, `remove_cvref`)

---

## 1. `std::forward` — `static_assert` Güvenlik Mekanizması

### 1.1 İki Overload Hatırlatma

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

### 1.2 `static_assert` Neden Gerekli?

`static_assert`, **rvalue olarak gelen bir argümanın yanlışlıkla lvalue olarak forward edilmesini** compile time'da engeller.

#### Tehlikeli Senaryo:

```cpp
template<typename T>
void call_foo(T&& param) {
    // Kullanıcı yanlışlıkla explicit template argümanı ile:
    foo(forward<MyClass&>(std::move(param)));
    // T = MyClass& → rvalue'yu lvalue olarak forward eder!
}
```

#### Adım Adım Analiz:

```
forward<MyClass&>(rvalue_arg)
  ├── T = MyClass&
  ├── Overload 2 seçilir (argüman rvalue)
  ├── static_cast<MyClass& &&>(t) → static_cast<MyClass&>(t)  (ref collapsing)
  ├── Sonuç: rvalue → lvalue'ya cast edilir ❌
  └── static_assert DEVREYE GİRER → compile time hatası ✓
```

| Senaryo | `T` | `static_assert` | Sonuç |
|---|---|---|---|
| `forward<string&>(lvalue)` | `string&` | Overload 1 seçilir, assert yok | lvalue → lvalue ✓ |
| `forward<string>(rvalue)` | `string` | `!is_lvalue_reference_v<string>` = `true` | rvalue → rvalue ✓ |
| `forward<string&>(rvalue)` | `string&` | `!is_lvalue_reference_v<string&>` = `false` | **HATA** ✓ |

> **Önemli Not:** `static_assert` olmadan, bir rvalue argümanı lvalue referans türüne cast edilerek iletilir; bu, dangling reference veya use-after-move hatalarına yol açar. Bu güvenlik mekanizması compile time'da yakalanması zor bir bug sınıfını ortadan kaldırır.

---

## 2. Abbreviated Template Syntax (C++20)

### 2.1 Temel Kavram

C++20 ile `template` head kısmı yazılmadan, fonksiyon parametresinde `auto` kullanılarak fonksiyon şablonu oluşturulabilir:

```cpp
// Geleneksel template syntax
template<typename T>
void foo(T x);

// Abbreviated template syntax (C++20) — eşdeğer
void foo(auto x);
```

### 2.2 Referans Varyasyonları

| Geleneksel Syntax | Abbreviated Syntax | Parametre Türü |
|---|---|---|
| `template<typename T> void f(T x)` | `void f(auto x)` | Value (kopya) |
| `template<typename T> void f(T& x)` | `void f(auto& x)` | lvalue reference |
| `template<typename T> void f(T&& x)` | `void f(auto&& x)` | **Universal reference** |

```cpp
// Abbreviated syntax ile universal reference
void baz(auto&& x);   // ✓ Universal reference — forwarding reference

// Eşdeğeri:
template<typename T>
void baz(T&& x);
```

> **Önemli Not:** `auto&&` parametresi abbreviated template syntax'te de **universal reference** kurallarına tabidir. `const auto&&` ise universal reference **değildir** — sadece rvalue kabul eder.

---

## 3. Abbreviated Template'de Perfect Forwarding — `decltype` Kullanımı

### 3.1 Problem

Abbreviated syntax'te görünür bir template parametresi (`T`) yoktur. `std::forward<T>(t)` yazılamaz.

### 3.2 Çözüm: `decltype`

```cpp
void call_foo(auto&& t) {
    foo(std::forward<decltype(t)>(t));
}
```

#### Neden Çalışıyor?

| Gelen Argüman | `t`'nin Bildirimdeki Türü | `decltype(t)` | `forward` Sonucu |
|---|---|---|---|
| lvalue (`x`) | `MyClass&` | `MyClass&` | lvalue ✓ |
| const lvalue (`cx`) | `const MyClass&` | `const MyClass&` | const lvalue ✓ |
| rvalue (`MyClass{}`) | `MyClass&&` | `MyClass&&` | rvalue (xvalue) ✓ |

#### Detaylı Analiz — rvalue Durumu:

```
call_foo(MyClass{})  →  t: MyClass&&  →  decltype(t) = MyClass&&

forward<MyClass&&>(t):
  ├── Overload 1 seçilir (t isim → lvalue)
  ├── Parametre: remove_reference_t<MyClass&&>& → MyClass&
  ├── Geri dönüş: MyClass&& && → MyClass&&  (ref collapsing)
  └── static_cast<MyClass&&>(t) → xvalue (rvalue) ✓
```

> **Önemli Not:** `decltype` burada **identifier form**'da kullanılır (`decltype(t)` — parantez yok): değişkenin **declaration type**'ını döndürür. Çift parantez (`decltype((t))`) kullanılırsa ifadenin value category'sine göre sonuç değişir — bu büyük bir hata kaynağıdır.

---

## 4. Lambda İfadelerinde Perfect Forwarding

### 4.1 Generalized Lambda ve Universal Reference

Lambda parametresinde `auto&&` kullanıldığında, derleyicinin oluşturduğu closure type'ın `operator()` fonksiyonu **member template** olur ve parametre **universal reference** olur:

```cpp
// Lambda ile perfect forwarding
auto call_foo = [](auto&& t) {
    foo(std::forward<decltype(t)>(t));
};
```

#### Derleyicinin Ürettiği Kod (Kavramsal):

```cpp
class __lambda_xyz {
public:
    template<typename T>
    auto operator()(T&& t) const {
        return foo(std::forward<T>(t));
    }
};
```

### 4.2 C++20 Explicit Template Lambda Alternatifi

```cpp
// C++20: Explicit template parametresi ile lambda
auto call_foo = []<typename T>(T&& t) {
    foo(std::forward<T>(t));
};
```

| Yöntem | Syntax | `forward` Argümanı |
|---|---|---|
| `auto&&` (C++14+) | `[](auto&& t) { ... }` | `std::forward<decltype(t)>(t)` |
| Explicit template (C++20) | `[]<typename T>(T&& t) { ... }` | `std::forward<T>(t)` |

---

## 5. Ref-Qualified Getter ile Perfect Forwarding

### 5.1 Ref-Qualified Overload Hatırlatma

```cpp
class Person {
    std::string name_;
public:
    // lvalue nesneleri için — kopyalama yok, referans döner
    const std::string& get_name() const & {
        return name_;
    }

    // rvalue nesneleri için — kaynağı taşır (move)
    std::string get_name() && {
        return std::move(name_);
    }
};
```

### 5.2 Generic Kodda Doğru Getter Çağrısı

```cpp
template<typename T>
void process(T&& x) {
    auto s = std::forward<T>(x).get_name();
    // lvalue person → const& overload çağrılır (kopyalama yok)
    // rvalue person → && overload çağrılır (move semantiği)
}
```

`std::forward<T>(x)` ifadesi:
- `x` lvalue ise → lvalue döner → `get_name() const &` çağrılır
- `x` rvalue ise → rvalue döner → `get_name() &&` çağrılır

> **Önemli Not:** `std::forward` olmadan sadece `x.get_name()` yazılırsa, `x` her zaman lvalue olduğu için (isim → lvalue) daima `const &` overload çağrılır. rvalue nesnelerin taşıma optimizasyonu kaybedilir.

---

## 6. `std::forward` Olmadan Perfect Forwarding — `if constexpr`

`std::forward` kullanılmadan, `if constexpr` ile value category'ye göre dallanma yapılabilir:

```cpp
template<typename T>
void call_foo(T&& t) {
    if constexpr (std::is_lvalue_reference_v<T>) {
        foo(t);               // T = lvalue ref → t zaten lvalue, doğrudan ilet
    } else {
        foo(std::move(t));    // T = non-ref → rvalue gelmiş, move ile rvalue yap
    }
}
```

| Gelen | `T` Çıkarımı | `is_lvalue_reference_v<T>` | Yapılan İşlem |
|---|---|---|---|
| lvalue | `MyClass&` | `true` | `foo(t)` — lvalue olarak ilet |
| rvalue | `MyClass` | `false` | `foo(std::move(t))` — rvalue olarak ilet |

> **Önemli Not:** Bu yaklaşım tek parametreli durumlarda çalışır ancak variadic parametrelerde (`Args&&...`) uygulanamaz. İdiomatik yol her zaman `std::forward<T>(t)` kullanmaktır.

---

## 7. Universal Reference Kısıtlama — Sadece rvalue Kabul Etme

### 7.1 Problem Tanımı

Universal reference parametreli bir fonksiyonun **sadece rvalue** ile çağrılmasını, lvalue ile çağrılmasının **yasaklanmasını** istiyoruz.

### 7.2 Çözüm 1: C++20 Concepts — `requires` Clause

```cpp
template<typename T>
    requires (!std::is_lvalue_reference_v<T>)
void foo(T&& t) {
    bar(std::forward<T>(t));
    // veya: bar(std::move(t));  — sadece rvalue geldiği garanti
}
```

#### Davranış:

```cpp
std::string name = "Ayse";

foo(name);              // ❌ Constraint karşılanmadı — overload set'ten çıkarılır
foo(std::string{"Ali"}); // ✓ T = string, constraint sağlandı
foo(std::move(name));    // ✓ T = string, constraint sağlandı
```

| `T` Çıkarımı | `is_lvalue_reference_v<T>` | `requires` Sonucu |
|---|---|---|
| `string&` (lvalue) | `true` | `false` → overload set'ten çıkar |
| `string` (rvalue) | `false` | `true` → fonksiyon çağrılabilir |

> **Önemli Not:** Concepts constraint'i `static_assert`'ten farklıdır. `static_assert` fonksiyon **seçildikten sonra** hata verir. `requires` clause ise fonksiyonu **overload set'ten çıkarır** — başka bir viable overload varsa o seçilir, yoksa "no matching overloaded function found" hatası oluşur.

### 7.3 Çözüm 2: SFINAE — `std::enable_if_t` (Pre-C++20)

```cpp
template<typename T,
         typename = std::enable_if_t<!std::is_lvalue_reference_v<T>>>
void foo(T&& t) {
    bar(std::forward<T>(t));
}
```

#### SFINAE Mekanizması:

```
T = string& (lvalue gönderildi)
  ├── !is_lvalue_reference_v<string&> → false
  ├── enable_if_t<false> → type üyesi YOK
  ├── Substitution failure → overload set'ten çıkarılır
  └── Başka overload yoksa: "no matching overloaded function found"

T = string (rvalue gönderildi)
  ├── !is_lvalue_reference_v<string> → true
  ├── enable_if_t<true> → void (default)
  ├── Substitution başarılı → overload set'te kalır
  └── Fonksiyon çağrılır ✓
```

---

## 8. SFINAE — Substitution Failure Is Not An Error

### 8.1 Temel Prensip

Template argüman substitution'ı sırasında oluşan tür hataları **compile error değildir** — ilgili fonksiyon overload set'ten çıkarılır:

```cpp
template<typename T>
typename T::value_type process(T x);  // T::value_type yoksa → SFINAE

process(42);  // T = int, int::value_type yok → substitution failure
              // → overload set'ten çıkar (hata DEĞİL)
              // → başka viable overload yoksa → HATA
```

### 8.2 `std::enable_if` / `std::enable_if_t` Yapısı

```cpp
// enable_if implementasyonu (kavramsal)
template<bool Condition, typename T = void>
struct enable_if {};                    // false açılımı: type YOK

template<typename T>
struct enable_if<true, T> {
    using type = T;                     // true açılımı: type VAR
};

template<bool Condition, typename T = void>
using enable_if_t = typename enable_if<Condition, T>::type;
```

| `Condition` | `enable_if_t<Condition>` | Sonuç |
|---|---|---|
| `true` | `void` (default) | Substitution başarılı — overload set'te kalır |
| `false` | **tanımsız** (`type` yok) | Substitution failure — overload set'ten çıkar |

### 8.3 `enable_if` Kullanım Konumları

```cpp
// 1. Default template argümanında (en yaygın)
template<typename T,
         typename = std::enable_if_t</* constraint */>>
void foo(T&& t);

// 2. Geri dönüş değeri türünde
template<typename T>
std::enable_if_t</* constraint */, void>
foo(T&& t);

// 3. Fonksiyon parametresinde (nadir)
template<typename T>
void foo(T&& t, std::enable_if_t</* constraint */, int> = 0);
```

---

## 9. Belirli Tür İçin Universal Reference — Tür Kısıtlama

### 9.1 Problem

Universal reference parametrenin **yalnızca belirli bir tür** (örneğin `std::string`) için çalışmasını istiyoruz. Doğrudan `std::string&&` yazmak çözüm değildir — bu rvalue reference olur, universal reference olmaz:

```cpp
// ❌ YANLIŞ — Bu universal reference DEĞİL
void foo(std::string&& s);  // Sadece string rvalue kabul eder

// ❌ YANLIŞ — const lvalue ref her şeyi kabul eder ama value category bilgisi kaybolur
void foo(const std::string& s);

// ✓ DOĞRU — Universal reference + tür kısıtlaması
template<typename T>
    requires std::is_same_v<std::remove_cvref_t<T>, std::string>
void foo(T&& s);
```

### 9.2 Kısıtlama Stratejileri

#### Strateji A: Tam Tür Eşleşmesi — `std::is_same_v` + `std::remove_cvref_t`

```cpp
template<typename T>
    requires std::is_same_v<std::remove_cvref_t<T>, std::string>
void foo(T&& t) {
    bar(std::forward<T>(t));
}
```

`remove_cvref_t` gereklidir çünkü `T`, universal reference deduction sonucu `string&`, `const string&` veya `string` olabilir:

| Gelen Argüman | `T` Çıkarımı | `remove_cvref_t<T>` | `is_same_v` |
|---|---|---|---|
| `string` lvalue | `string&` | `string` | `true` ✓ |
| `const string` lvalue | `const string&` | `string` | `true` ✓ |
| `string` rvalue | `string` | `string` | `true` ✓ |
| `int` lvalue | `int&` | `int` | `false` ❌ |
| `const char*` rvalue | `const char*` | `const char*` | `false` ❌ |

#### Strateji B: Dönüşebilirlik — `std::is_convertible_v` / `std::convertible_to`

```cpp
// string'e dönüşebilen her tür kabul edilir (const char*, string_view, vb.)
template<typename T>
    requires std::convertible_to<T, std::string>
void foo(T&& t) {
    bar(std::forward<T>(t));
}
```

```cpp
foo("hello");                    // ✓ const char* → string dönüşümü var
foo(std::string{"world"});       // ✓ string → string (kendisi)
foo(std::string_view{"test"});   // ✓ string_view → string dönüşümü var
foo(42);                         // ❌ int → string dönüşümü yok
```

### 9.3 SFINAE ile Aynı Kısıtlama (Pre-C++20)

```cpp
template<typename T,
         typename = std::enable_if_t<
             std::is_same_v<std::remove_cvref_t<T>, std::string>>>
void foo(T&& t) {
    bar(std::forward<T>(t));
}
```

### 9.4 Concepts vs SFINAE Karşılaştırması

| Özellik | Concepts (`requires`) | SFINAE (`enable_if`) |
|---|---|---|
| **Standart** | C++20+ | C++11+ |
| **Okunabilirlik** | Yüksek | Düşük |
| **Hata mesajı** | "constraints not satisfied" | "no matching overloaded function" |
| **Composability** | `requires (A && B \|\| C)` | İç içe `enable_if` — karmaşık |
| **Tercih** | **Modern kodda varsayılan** | Legacy / C++20 öncesi projeler |

> **Önemli Not:** `std::is_same_v<T, std::string>` yerine `std::is_same_v<std::remove_cvref_t<T>, std::string>` kullanılmalıdır. Universal reference deduction'da `T`, lvalue geldiğinde referans türü (`string&`) olur — `remove_cvref_t` olmadan constraint her zaman başarısız olur.

---

## 10. Concepts ve SFINAE — Overload Set Davranışı

### 10.1 Ortak Davranış: Overload Set'ten Çıkarma

Hem concepts hem SFINAE, constraint sağlanmadığında fonksiyonu **overload set'ten çıkarır** (sentaks hatası vermez):

```cpp
template<typename T>
    requires (!std::is_lvalue_reference_v<T>)
void foo(T&& t) { /* ... */ }

void foo(const std::string& s) { /* fallback overload */ }

std::string name = "test";
foo(name);  // Universal ref overload çıkarılır → fallback çağrılır ✓
```

### 10.2 `static_assert` vs Constraint Farkı

| Mekanizma | Etki Zamanı | Overload Resolution | Kullanım Yeri |
|---|---|---|---|
| `static_assert` | Fonksiyon **seçildikten sonra** | Katılmaz — hata verir | Fonksiyon gövdesi |
| `requires` / SFINAE | Fonksiyon **seçilmeden önce** | Overload set'ten çıkarır | Fonksiyon bildirimi |

> **Önemli Not:** Overload set'in boş kalması durumunda her iki mekanizma da hata üretir, ancak hata mesajları farklıdır. Concepts "associated constraints not satisfied" şeklinde **neyin yanlış olduğunu** açıkça belirtir; SFINAE ise sadece "no matching overloaded function found" der — debugging açısından concepts çok daha üstündür.

---

## 11. `std::remove_cvref_t` (C++20)

Universal reference ile tür karşılaştırması yaparken `T`'den hem referans hem cv-qualifier'ları temizlemek gerekir:

```cpp
// remove_cvref_t = remove_cv_t<remove_reference_t<T>>
static_assert(std::is_same_v<std::remove_cvref_t<const std::string&>, std::string>);
static_assert(std::is_same_v<std::remove_cvref_t<std::string&&>, std::string>);
static_assert(std::is_same_v<std::remove_cvref_t<const volatile int&>, int>);
```

| `T` | `remove_reference_t<T>` | `remove_cvref_t<T>` |
|---|---|---|
| `string&` | `string` | `string` |
| `const string&` | `const string` | `string` |
| `string&&` | `string` | `string` |
| `const string&&` | `const string` | `string` |

---

## 12. Özet — Perfect Forwarding Kalıpları

| Durum | Kalıp |
|---|---|
| Klasik template | `template<typename T> void f(T&& t) { g(std::forward<T>(t)); }` |
| Abbreviated template | `void f(auto&& t) { g(std::forward<decltype(t)>(t)); }` |
| Lambda (auto) | `[](auto&& t) { g(std::forward<decltype(t)>(t)); }` |
| Lambda (explicit) | `[]<typename T>(T&& t) { g(std::forward<T>(t)); }` |
| Sadece rvalue (concepts) | `requires (!is_lvalue_reference_v<T>)` |
| Sadece rvalue (SFINAE) | `enable_if_t<!is_lvalue_reference_v<T>>` |
| Belirli tür (concepts) | `requires is_same_v<remove_cvref_t<T>, Target>` |
| Dönüşebilir tür | `requires convertible_to<T, Target>` |
| `forward` olmadan | `if constexpr (is_lvalue_reference_v<T>) f(t); else f(std::move(t));` |

---

## 13. Low-Latency Sistemler İçin Kritik Noktalar

> **Önemli Not:** Abbreviated template syntax (`auto&&`) ile yazılan fonksiyonlar, klasik `template<typename T>` ile **aynı makine kodunu** üretir — ek runtime overhead yoktur. Build time etkisi de minimaldir çünkü aynı template instantiation mekanizması kullanılır.

> **Önemli Not:** SFINAE tabanlı kısıtlamalar, overload set oluşturma aşamasında derleyicinin daha fazla substitution denemesi yapmasına neden olur. Çok sayıda SFINAE-constrained overload bulunan header-heavy projelerde bu **compile time'ı ölçülebilir şekilde artırır**. Concepts, derleyiciye daha doğrudan semantik bilgi verdiği için genellikle daha hızlı derleme süresi sağlar.

> **Önemli Not:** `if constexpr` ile manuel forwarding, `std::forward` ile eşdeğer makine kodu üretir. Ancak `std::forward` compiler'ın **inlining ve devirtualization** optimizasyonlarına daha tanıdık bir kalıp sunduğu için, optimizasyon seviyesi düşük debug build'lerde bile doğru çalışması garanti edilir.
