# Ders 8 — UDL Template, Strong Types, `std::string_view` Derinlemesine ve `std::optional` Giriş

> **Kurs:** İleri C++ (C++26 uyumlu)
> **Tarih:** 21 Nisan 2025
> **Kapsam:** Literal operator template (variadic `char` non-type parameter pack), strong typing idiom'u (zero-cost abstraction), `std::string_view` (non-owning observer, constructorlar, dangling pointer tehlikesi, null-termination tuzağı, `remove_prefix`/`remove_suffix`, `std::string` ↔ `string_view` dönüşümleri, UDL `sv`/`s` suffix'leri, `constexpr` kullanımı, fonksiyon parametresi olarak tercih), `std::optional` giriş (nullable value semantiği, `std::nullopt`, `value()`, `value_or()`, `has_value()`, monadic interface C++23)

---

## 1. Literal Operator Template — Variadic `char` Parameter Pack

### 1.1 Template Formu

Literal operator fonksiyonu, **non-type parameter pack** olarak `char` parametreleri alan bir template olabilir:

```cpp
template<char... Chars>
constexpr auto operator""_b() {
    constexpr char str[] = { Chars..., '\0' };  // Pack expansion + null terminator

    int result = 0;
    for (int i = 0; str[i] != '\0'; ++i) {
        result = result * 2 + (str[i] - '0');
    }
    return result;
}

// Kullanım:
constexpr auto val = 101_b;     // 5
constexpr auto big = 101110_b;  // 46
```

### 1.2 Derleyicinin Dönüşümü

```
101_b  →  operator""_b<'1', '0', '1'>()
```

Template argümanları olarak her karakter ayrı ayrı non-type argüman olarak geçilir. **Fonksiyonun parametre parantezi boştur** — argümanlar sadece template parametrelerinden gelir.

### 1.3 Template Parametre Paketi Türleri (Hatırlatma)

| Kategori | Syntax | Örnek |
|---|---|---|
| **Type** parameter pack | `template<typename... Ts>` | Tür paketi |
| **Non-type** parameter pack | `template<int... Ns>` | Değer paketi |
| **Template** parameter pack | `template<template<typename> class... Cs>` | Template paketi |
| **UDL** parameter pack | `template<char... Chars>` | Karakter paketi (sadece `char`) |

> **Önemli Not:** Literal operator template'te parametre paketi **sadece `char` türünden** olabilir. Bu kısıtlama standardın bir kuralıdır. Pack expansion (`Chars...`) ile elemanlar comma-separated list'e dönüştürülür — initializer list, fonksiyon argümanları gibi birçok bağlamda kullanılabilir.

---

## 2. Strong Types — Zero-Cost Abstraction

### 2.1 Problem

Primitive türler (int, double) domain kavramlarını yeterince ifade etmez ve tür güvenliği sağlamaz:

```cpp
void set_position(double x, double y, double z);
void apply_force(double newtons);
void set_temperature(double celsius);

// ❌ Derleyici bu karışıklığı yakalayamaz:
set_position(temperature, force, distance);  // Derlenir ama yanlış!
```

### 2.2 Strong Type Wrapper Idiom'u

```cpp
class Meters {
    double value_;

    // Client code'u doğrudan constructor kullanmaktan caydırmak için:
    struct PreventUsage {};

public:
    constexpr explicit Meters(PreventUsage, double v) : value_(v) {}

    // Explicit dönüşüm — örtülü dönüşüm engellenir
    constexpr explicit operator double() const { return value_; }

    // Getter — açık erişim
    [[nodiscard]] constexpr double get() const { return value_; }

    // Stream inserter
    friend std::ostream& operator<<(std::ostream& os, Meters m) {
        return os << m.value_ << "m";
    }
};

// UDL ile strong type oluşturma — tek erişim noktası
constexpr Meters operator""_m(long double v) {
    return Meters{Meters::PreventUsage{}, static_cast<double>(v)};
}
```

### 2.3 Kullanım

```cpp
constexpr auto distance = 3.9_m;        // ✓ Meters türü
// constexpr Meters m{42.0};            // ❌ PreventUsage engeli
auto val = static_cast<double>(distance); // ✓ Explicit dönüşüm
auto val2 = distance.get();              // ✓ Getter
```

### 2.4 Template ile Genelleştirme

Farklı birimler için aynı kodu tekrarlamak yerine, sınıf şablonu kullanılabilir:

```cpp
template<typename Tag>
class StrongDouble {
    double value_;
public:
    constexpr explicit StrongDouble(double v) : value_(v) {}
    constexpr explicit operator double() const { return value_; }
    [[nodiscard]] constexpr double get() const { return value_; }
};

// Tag türleri — her birim için farklı specialization
struct MeterTag {};
struct KilogramTag {};
struct CelsiusTag {};

using Meters    = StrongDouble<MeterTag>;
using Kilograms = StrongDouble<KilogramTag>;
using Celsius   = StrongDouble<CelsiusTag>;
```

> **Önemli Not:** Strong type wrapper'lar **zero-cost abstraction**'dır. Optimizasyon seviyesi `-O2` ve üzerinde, `Meters` türünden bir değişken kullanmak ile doğrudan `double` kullanmak arasında **hiçbir assembly farkı yoktur**. Derleyici tüm wrapper katmanını inline eder. Compile-time tür güvenliği sağlanırken runtime maliyeti sıfırdır.

---

## 3. `std::string_view` — Non-Owning String Observer

### 3.1 Temel Kavram

`std::string_view`, bellekte ardışık (contiguous) olarak duran bir karakter dizisinin **sahibi olmayan gözlemcisidir** (non-owning observer):

```cpp
// İç yapısı (kavramsal):
class string_view {
    const char* data_;    // Yazının başlangıç adresi
    std::size_t size_;    // Karakter sayısı
    // sizeof(string_view) = 16 bytes (64-bit) vs sizeof(string) = 32-40 bytes
};
```

| Özellik | `std::string` | `std::string_view` |
|---|---|---|
| **Sahiplik** | Yazının sahibi (owning) | Gözlemci (non-owning) |
| **Bellek yönetimi** | Allocate/deallocate yapar | Allocation **yapmaz** |
| **Kopyalama maliyeti** | O(n) — tüm karakterler kopyalanır | O(1) — sadece pointer + size |
| **Mutator fonksiyonlar** | Var (`push_back`, `append`, `+=` vb.) | **Yok** (salt okunur) |
| **`sizeof`** | 32-40 byte (impl. defined) | 16 byte (64-bit) |
| **`constexpr`** | Kısıtlı (C++20 ile genişletildi) | **Tüm fonksiyonlar** `constexpr` |

### 3.2 Hangi Kaynaklardan Oluşturulabilir?

```cpp
// 1. C-string (null-terminated)
const char arr[] = "hello";
std::string_view sv1{arr};          // null-terminated diziden

// 2. Pointer + size (data constructor)
std::string_view sv2{arr, 3};       // "hel" — ilk 3 karakter

// 3. std::string nesnesinden
std::string str = "world";
std::string_view sv3{str};          // string'in tuttuğu yazıyı gözlemler

// 4. std::array<char> veya std::vector<char>
std::array<char, 5> ar{'h','e','l','l','o'};
std::string_view sv4{ar.data(), ar.size()};

// 5. İteratör çifti (C++20)
std::string_view sv5{str.begin(), str.end()};

// 6. Default constructor — boş gözlemci
std::string_view sv6{};             // data() == nullptr, size() == 0

// 7. nullptr — DELETE EDİLMİŞ (sentaks hatası)
// std::string_view sv7{nullptr};   // ❌ Deleted constructor
```

### 3.3 Constructor Özeti

| Constructor | Parametre | Açıklama |
|---|---|---|
| Default | — | Boş yazı, `data()` = `nullptr`, `size()` = 0 |
| C-string | `const char*` | Null-terminated string gözlemcisi |
| Data + count | `const char*, size_t` | Belirtilen aralığın gözlemcisi |
| `std::string` | `const std::string&` | String'in tuttuğu yazıyı gözlemler |
| İteratör çifti (C++20) | `It first, It last` | `[first, last)` aralığını gözlemler |
| `nullptr` (**deleted**) | `std::nullptr_t` | **Sentaks hatası** — delete edilmiş |

### 3.4 `std::string` ↔ `std::string_view` Dönüşümleri

```cpp
// string → string_view: ÖRTÜLMÜŞbir otomatik dönüşüm
std::string str = "hello";
std::string_view sv = str;          // ✓ Implicit dönüşüm

// string_view → string: EXPLICIT constructor gerekli
std::string_view sv2 = "world";
// std::string s = sv2;             // ❌ Explicit constructor — copy init hata
std::string s{sv2};                 // ✓ Direct initialization
std::string s2(sv2);                // ✓ Direct initialization
auto s3 = std::string(sv2);         // ✓ Explicit prvalue
```

| Dönüşüm Yönü | Tür | Sentaks |
|---|---|---|
| `string` → `string_view` | **Implicit** | `string_view sv = str;` ✓ |
| `string_view` → `string` | **Explicit** | `string s{sv};` ✓, `string s = sv;` ❌ |

### 3.5 Non-Mutating Üye Fonksiyonlar

`std::string`'in tüm salt-okunur fonksiyonları `string_view`'da da mevcuttur:

```cpp
std::string_view sv = "Bjarne Stroustrup";

sv.size();         // 17
sv.length();       // 17 (size ile eşdeğer)
sv.empty();        // false
sv[0];             // 'B'
sv.at(5);          // 'e' (bounds-checked)
sv.front();        // 'B'
sv.back();         // 'p'
sv.data();         // const char* — yazının adresi

// Arama fonksiyonları
sv.find("Str");           // 7
sv.rfind("r");            // 15
sv.find_first_of("aeiou");
sv.find_last_not_of(" ");

// Karşılaştırma
sv.compare("hello");
sv.starts_with("Bj");    // true (C++20)
sv.ends_with("up");       // true (C++20)
sv.contains("rou");       // true (C++23)

// Substring — string_view döndürür (kopyalama YOK)
sv.substr(7, 10);         // "Stroustrup" — string_view döner
```

> **Önemli Not:** `string_view::substr()` **bir `string_view` döndürür**, `string::substr()` ise **bir `string` döndürür**. Bu kritik farktır: `string::substr()` O(n) kopyalama yaparken, `string_view::substr()` O(1)'dir — sadece pointer aritmetiği. Yoğun substring işlemleri yapan kodlarda `string_view` kullanımı **büyük performans kazancı** sağlar.

### 3.6 `remove_prefix` ve `remove_suffix`

`string_view`'a özgü iki mutator-benzeri fonksiyon vardır — bunlar yazıyı değiştirmez, **gözlem penceresini daraltır**:

```cpp
std::string_view sv = "[[important]]";

sv.remove_prefix(2);   // sv = "important]]"  — baştan 2 karakter kırp
sv.remove_suffix(2);   // sv = "important"    — sondan 2 karakter kırp

// Orijinal yazı DEĞİŞMEZ — sadece pointer/size güncellenir
```

| Fonksiyon | İşlem | Maliyet |
|---|---|---|
| `remove_prefix(n)` | `data_ += n; size_ -= n;` | O(1) |
| `remove_suffix(n)` | `size_ -= n;` | O(1) |

### 3.7 UDL Suffix'leri: `s` vs `sv`

```cpp
using namespace std::string_literals;
using namespace std::string_view_literals;

auto str = "hello"s;     // std::string
auto sv  = "hello"sv;    // std::string_view

// Fark:
// "hello"s  → heap allocation (kopyalama)
// "hello"sv → sadece pointer + size (allocation yok)
```

### 3.8 `constexpr` String İşlemleri

Tüm `string_view` fonksiyonları `constexpr`'dir — compile-time string manipülasyonu mümkündür:

```cpp
constexpr std::string_view greeting = "Hello, World!";
constexpr auto len = greeting.size();           // 13 — compile-time
constexpr auto pos = greeting.find("World");    // 7  — compile-time
constexpr auto sub = greeting.substr(0, 5);     // "Hello" — compile-time
constexpr bool hw  = greeting.starts_with("He"); // true — compile-time

static_assert(len == 13);
static_assert(pos == 7);
static_assert(hw);
```

> **Önemli Not:** `string_view`'ın tüm fonksiyonlarının `constexpr` olması, compile-time string parsing, validation ve transformation işlemlerini mümkün kılar. `consteval` fonksiyonlar içinde `string_view` kullanarak, konfigürasyon string'lerinin derleme zamanında doğrulanması sağlanabilir — runtime overhead sıfırdır.

---

## 4. `std::string_view` — Dangling Pointer Tehlikesi

### 4.1 Temel Kural

`string_view` gözlemlediği yazının **sahibi değildir**. Yazının ömrü sona ererse `string_view` **dangling** hale gelir:

```cpp
// ❌ Tanımsız davranış — geçici nesne
std::string_view sv = std::string{"temporary"};
// string geçici nesne burada yok edilir
std::println("{}", sv);  // UB! Dangling pointer

// ❌ Tanımsız davranış — scope dışı
std::string_view get_name() {
    std::string local = "hello";
    return local;  // local yok edilecek → dangling
}

// ✓ Güvenli — string literal statik ömürlü
std::string_view sv2 = "safe";  // String literal programın sonuna kadar yaşar
```

### 4.2 Güvenli ve Tehlikeli Senaryolar

| Senaryo | Güvenli? | Neden |
|---|---|---|
| `string_view sv = "literal";` | ✓ | String literal statik ömürlü |
| `string_view sv = str;` (str yaşıyor) | ✓ | str ömrü boyunca geçerli |
| `string_view sv = std::string{...};` | ❌ | Geçici nesne hemen yok edilir |
| `return local_string;` (→ `string_view`) | ❌ | Otomatik ömürlü nesne scope'ta biter |
| `sv = str; str.clear();` | ❌ | Reallocation → eski bellek geçersiz |

### 4.3 Null-Termination Tuzağı

```cpp
char arr[] = {'C', '+', '+', '!'};  // null-terminated DEĞİL

std::string_view sv{arr, 4};        // ✓ Güvenli — size bilgisi var
std::println("{}", sv);             // ✓ "C++!" — inserter size kullanır

// ❌ Tehlikeli — data() null-terminated değil
printf("%s", sv.data());            // UB! %s null karakter arar
some_c_api(sv.data());              // UB! C API null-terminated bekler
```

> **Önemli Not:** `string_view::data()` **null-terminated string döndürmeyi garanti etmez**. C API'lerine veya `printf`'e `data()` geçerken yazının null-terminated olduğundan emin olunmalıdır. Güvenli yol: `std::string temp{sv}; c_api(temp.c_str());` — geçici string oluşturup `c_str()` kullanmak.

---

## 5. `std::string_view` Fonksiyon Parametresi Olarak

### 5.1 Tercih Kuralı

```cpp
// ❌ Gereksiz kopyalama riski
void process(std::string s);

// ❌ Sadece null-terminated kabul eder
void process(const char* s);

// ✓ En esnek ve verimli — tüm string kaynaklarını kabul eder
void process(std::string_view sv);
```

| Parametre Türü | `string` kabul? | `const char*` kabul? | `string_view` kabul? | Kopyalama? |
|---|---|---|---|---|
| `std::string` | ✓ | ✓ (implicit) | ✓ (explicit) | **Evet** |
| `const std::string&` | ✓ | ✓ (implicit) | ✓ (explicit) | Hayır |
| `const char*` | ✓ (`.c_str()`) | ✓ | ✗ | Hayır |
| `std::string_view` | ✓ (implicit) | ✓ (implicit) | ✓ | **Hayır** (16 byte kopya) |

> **Önemli Not:** Salt-okunur string parametreleri için `std::string_view` tercih edilmelidir. `const std::string&` yerine `string_view` kullanmak, hem `std::string` hem `const char*` hem de `string_view` argümanlarını **allocation olmadan** kabul eder. `string_view` kopyalaması sadece 16 byte'tır (pointer + size) — bu nedenle **değer semantiği ile** (by value) geçmek referans ile geçmekten genellikle daha verimlidir.

---

## 6. `std::optional` — Nullable Value Semantiği

### 6.1 Kavram

`std::optional<T>`, bir değeri **tutabilir veya tutmayabilir** — "değer yok" durumunu type-safe şekilde ifade eder:

```cpp
#include <optional>

std::optional<int> find_index(std::string_view sv, char c) {
    if (auto pos = sv.find(c); pos != std::string_view::npos) {
        return static_cast<int>(pos);
    }
    return std::nullopt;  // "değer yok"
}
```

### 6.2 Oluşturma Yolları

```cpp
// 1. Değerli oluşturma
std::optional<int> opt1 = 42;
std::optional<int> opt2{42};
auto opt3 = std::make_optional(42);

// 2. Boş oluşturma
std::optional<int> opt4;               // Boş — default constructed
std::optional<int> opt5 = std::nullopt; // Boş — explicit

// 3. In-place construction (kopyalama/taşıma yok)
std::optional<std::string> opt6{std::in_place, "hello"};
std::optional<std::string> opt7{std::in_place, 5, 'x'};  // "xxxxx"
```

### 6.3 Erişim ve Sorgulama

```cpp
std::optional<int> opt = 42;

// Varlık kontrolü
if (opt.has_value()) { /* ... */ }
if (opt) { /* ... */ }            // operator bool

// Değer erişimi
int val = opt.value();            // Değer yoksa std::bad_optional_access fırlatır
int val2 = *opt;                  // UB eğer boşsa! (kontrol yok)
int val3 = opt.value_or(0);       // Değer yoksa 0 döndürür

// Pointer-like erişim (sınıf türleri için)
std::optional<std::string> os = "hello";
auto len = os->size();            // 5
```

| Fonksiyon | Boşken Davranış | Kullanım |
|---|---|---|
| `value()` | `std::bad_optional_access` fırlatır | Güvenli erişim |
| `operator*` | **Tanımsız davranış** | Performans-kritik (kontrol yapıldıktan sonra) |
| `value_or(default)` | `default` değerini döndürür | Fallback değer gerektiğinde |
| `has_value()` / `operator bool` | `false` | Varlık kontrolü |

### 6.4 Monadic Interface (C++23)

```cpp
std::optional<std::string> name = get_name();

// transform: değer varsa dönüştür, yoksa nullopt
auto upper = name.transform([](const std::string& s) {
    auto result = s;
    std::ranges::transform(result, result.begin(), ::toupper);
    return result;
});  // optional<string>

// and_then: optional döndüren zincir (flatmap)
auto first_char = name.and_then([](const std::string& s) -> std::optional<char> {
    if (s.empty()) return std::nullopt;
    return s.front();
});  // optional<char>

// or_else: değer yoksa alternatif optional döndür
auto result = name.or_else([] {
    return std::make_optional<std::string>("default");
});  // optional<string>
```

### 6.5 `std::optional` vs Alternatifler

| Yaklaşım | Avantaj | Dezavantaj |
|---|---|---|
| `std::optional<T>` | Type-safe, değer semantiği, stack allocation | `sizeof(T) + 1` byte overhead |
| Sentinel değer (`-1`, `""`) | Ek maliyet yok | Tür güvenliği yok, sentinel geçerli değer olabilir |
| `T*` (pointer) | Null kontrolü mümkün | Sahiplik belirsiz, lifetime yönetimi zor |
| `std::expected<T,E>` (C++23) | Hata bilgisi taşır | Daha karmaşık API |

> **Önemli Not:** `std::optional` değer semantiğine sahiptir — heap allocation **yapmaz**, değeri kendi içinde (in-place) tutar. Bu, `std::unique_ptr<T>` ile "nullable" ifade etmekten çok daha verimlidir. Low-latency sistemlerde `optional` tercih edilmelidir çünkü allocation/deallocation maliyeti yoktur. Ancak `sizeof(optional<T>)` = `sizeof(T) + alignment padding + bool flag` olduğu unutulmamalıdır.

---

## 7. `std::optional` — Dangling Reference Tehlikesi

```cpp
// ❌ Tanımsız davranış — referans semantiği ile karıştırma
std::optional<const std::string&> ref_opt;  // C++26'da bile sorunlu

// ✓ Güvenli — değer semantiği
std::optional<std::string> val_opt = "hello";

// ❌ reference_wrapper ile workaround — dikkatli kullanılmalı
std::optional<std::reference_wrapper<const std::string>> ref_opt2;
```

---

## 8. Vocabulary Types Genel Bakış

C++17 ile eklenen üç "vocabulary type":

| Sınıf | Amaç | Kullanım Sıklığı |
|---|---|---|
| `std::optional<T>` | "Değer var mı yok mu?" | Yüksek |
| `std::variant<Ts...>` | "Bu değerlerden biri" (type-safe union) | Yüksek (özellikle generic kodda) |
| `std::any` | "Herhangi bir tür" (type-erased) | Düşük (spesifik senaryolar) |

---

## 9. Özet Tablosu

| Konu | Anahtar Kavram | Kritik Detay |
|---|---|---|
| UDL template | `template<char... Chars>` | Sadece `char` non-type pack, parametre parantezi boş |
| Strong types | Zero-cost abstraction | Assembly'de primitive ile aynı, compile-time güvenlik |
| `string_view` | Non-owning observer | 16 byte, allocation yok, tüm fonksiyonlar `constexpr` |
| `string_view` dangling | Gözlemlenen nesne ölürse UB | Geçici string → `string_view` en yaygın hata |
| Null-termination | `data()` null-terminated garanti etmez | C API'ye geçerken `std::string` üzerinden |
| `substr` farkı | `string::substr` → string (O(n)) | `string_view::substr` → string_view (O(1)) |
| `remove_prefix/suffix` | Gözlem penceresini daraltır | O(1) — pointer/size güncelleme |
| `optional` | Nullable value semantics | Stack allocation, `value_or` ile güvenli erişim |
| `optional` monadic (C++23) | `transform`, `and_then`, `or_else` | Fonksiyonel zincir, `nullopt` propagation |
